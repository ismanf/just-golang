# Google & gVisor — A User-Space Kernel in Go

## TL;DR

**gVisor** is Google's open-source **application kernel** written in Go. It implements a substantial subset of the Linux syscall ABI in user space, intercepts container syscalls, and emulates them — providing an isolation boundary between untrusted code and the host kernel. gVisor runs in **Google Cloud Run, App Engine, and Cloud Functions**, isolating customer workloads from each other and from the host. The single biggest gotcha: **gVisor trades syscall performance for isolation**. Syscalls that would take ~100 ns on a real kernel take ~µs through gVisor's intercept + emulation path. CPU-bound workloads are fine; syscall-heavy workloads (databases, network proxies) pay a real cost.

## Mental Model

```
    Container (untrusted Go/Python/etc.)
         │   makes syscall (read, write, mmap, ...)
         ▼
    ┌────────────────────────────────────────────────┐
    │  Sentry  (gVisor's user-space "kernel", in Go) │
    │   - implements Linux ABI                        │
    │   - manages process state, FDs, mm, scheduler   │
    │   - has its own page table, signal handling     │
    └─────────┬──────────────────────────────────────┘
              │ (subset of host syscalls — narrow surface)
              ▼
    ┌────────────────────────────────────────────────┐
    │  Host Linux kernel                              │
    │   - sees only sentry's narrow syscall surface   │
    │   - host kernel bugs are mostly out of reach    │
    └────────────────────────────────────────────────┘

    Two platforms for syscall intercept:
      - ptrace: works anywhere; slow.
      - KVM:    needs nested virt; faster.
      - systrap (since 2022): SECCOMP-based; fastest.
```

The key insight: rather than relying on the host Linux kernel's namespace/cgroup-based isolation (which has had many CVEs), gVisor **re-implements the kernel ABI in a separately-isolated process**, presenting a much narrower attack surface to the host.

## Syntax & Basic Usage

```bash
# Install runsc (the OCI runtime gVisor ships):
$ curl https://storage.googleapis.com/gvisor/releases/release/latest/runsc \
    -o /usr/local/bin/runsc
$ chmod +x /usr/local/bin/runsc

# Configure Docker to use it:
$ cat /etc/docker/daemon.json
{
  "runtimes": {
    "runsc": {
      "path": "/usr/local/bin/runsc"
    }
  }
}
$ systemctl restart docker

# Run a container under gVisor:
$ docker run --runtime=runsc -it alpine sh
/ # uname -a
Linux 4.4.0 ... (the version gVisor presents — not the host)
```

`runsc` is the OCI-compatible runtime that wraps gVisor; Docker, containerd, and Kubernetes can all use it.

## Deep Dive

### Architecture overview

gVisor has three main components, all in Go:

1. **Sentry** (`runsc/sentry/`): the user-space kernel. Implements process/thread management, virtual memory, file system, network stack, signal handling, and Linux syscalls.
2. **Gofer** (`runsc/fsgofer/`): a sidecar process that proxies filesystem access between the container and the host (over a 9P/LISAFS protocol). Sentry doesn't have direct host filesystem access; the gofer does.
3. **runsc**: the CLI/OCI runtime that creates sandboxes, manages lifecycle, hooks into containerd.

### Why Go

gVisor was implemented in Go (rather than C, C++, or Rust) for these reasons documented by the team:

- **Memory safety** prevents an entire class of kernel vulnerabilities (use-after-free, buffer overflow). Go's bounds-checked slices and GC eliminate ~70% of historical Linux kernel CVE classes.
- **Garbage collection** is acceptable because Sentry isn't latency-critical the way Linux is. A few-ms GC pause is fine when amortized over millions of syscalls.
- **Strong stdlib** for concurrency, network protocols (TCP/IP stack reuses Go's `net` design abstractions), and serialization.
- **Internal expertise**: by 2017 Google had Go expertise at scale; the gVisor team could ship a complex system without inventing language tooling.

The team is candid about Go's downsides for this work in talks: GC pauses, atomic-heavy hot paths, and the inability to use `unsafe.Pointer` arithmetic forced some workarounds.

### Syscall intercept platforms

gVisor supports three "platforms" — mechanisms for catching guest syscalls:

#### ptrace platform

The original. The host Linux kernel ptrace-attaches to the guest process; every syscall traps to Sentry, which emulates and resumes. Works on any Linux; high overhead.

```
Guest → SYSCALL → host kernel traps → ptrace_event → Sentry handles → resume guest
```

Cost: ~5–10 µs per syscall, vs. ~100 ns native.

#### KVM platform

Sentry runs as a *guest* VM (using `/dev/kvm`). The guest process's syscalls trap into Sentry directly, without going to the host kernel first.

Cost: ~1–2 µs per syscall. Requires KVM (nested virt if you're in a cloud VM).

#### systrap platform (2022+)

Uses SECCOMP filters + signal delivery to redirect syscalls into Sentry. Faster than ptrace, doesn't need KVM. The current production platform on Google Cloud.

Cost: ~0.5–1 µs per syscall.

The platform abstraction lives in [`gvisor.dev/gvisor/pkg/sentry/platform/`](https://github.com/google/gvisor/tree/master/pkg/sentry/platform).

### Sentry's syscall table

```go
// gvisor/pkg/sentry/syscalls/linux/sys_read.go (excerpt; Apache-2.0 © The gVisor Authors)
func Read(t *kernel.Task, sysno uintptr, args arch.SyscallArguments) (uintptr, *kernel.SyscallControl, error) {
    fd := args[0].Int()
    addr := args[1].Pointer()
    size := args[2].SizeT()

    file := t.GetFile(fd)
    if file == nil { return 0, nil, linuxerr.EBADF }
    defer file.DecRef(t)

    n, err := readv(t, file, addr, int(size))
    if err != nil { return 0, nil, err }
    return uintptr(n), nil, nil
}
```

Each Linux syscall has a Go implementation. Sentry has implementations for ~250+ syscalls — the subset most container workloads need. Unimplemented syscalls return `ENOSYS`.

The full table: [`pkg/sentry/syscalls/linux/`](https://github.com/google/gvisor/tree/master/pkg/sentry/syscalls/linux).

### The network stack — `netstack`

gVisor implements its own **TCP/IP stack in Go** ([`pkg/tcpip/`](https://github.com/google/gvisor/tree/master/pkg/tcpip)). The stack handles:

- IPv4, IPv6.
- TCP, UDP, ICMP.
- Socket-style API exposed to Sentry.
- Network namespaces.

The stack is used in two ways:

1. **netstack mode**: gVisor processes all packets itself; only forwards finished packets to the host.
2. **host network mode**: gVisor uses the host kernel's network stack via `socket()` calls.

netstack is what made gVisor practical for Google Cloud — it's also reused by **Tailscale** (see `20-big-tech/09-tailscale.md`).

### Memory management

Sentry has its own page tables. Guest virtual addresses don't map directly to host physical addresses; Sentry maintains the mapping. mmap'd files go through Sentry → Gofer → host.

Costs:
- Page faults are intercepted and serviced by Sentry.
- mmap is slower than native (because it crosses the Gofer).
- Direct memory access via `mmap` of `/dev/shm` etc. is restricted.

### Performance characteristics

Per gVisor benchmarks (https://gvisor.dev/docs/architecture_guide/performance/):

- **CPU-bound workloads**: near-native (no syscalls).
- **Syscall-heavy workloads**: 10–30% slower depending on syscall.
- **I/O-heavy workloads**: 30–50% slower due to Gofer round-trips.
- **Network throughput** (netstack): ~50% of native; significantly improved with `--platform=systrap`.

Trade-off: you accept performance loss in exchange for a much narrower attack surface to the host kernel.

### Use cases

1. **Google Cloud Run / Functions / App Engine**: every customer container runs under gVisor. Customer code can't escape to the host or to other customers via kernel bugs.
2. **GKE Sandbox**: opt-in for Kubernetes pods. `runtimeClassName: gvisor`.
3. **Compromised containers**: gVisor is recommended for running any container you don't trust (downloaded binaries, untrusted images).
4. **Multi-tenant SaaS**: each tenant in its own sandbox.

### Limitations

- Not every Linux syscall is implemented. Older or obscure ones (`fanotify`, `bpf`, `kexec_*`) return `ENOSYS`.
- Performance gaps compared to native.
- Specialized workloads that need direct hardware access (FPGAs, GPUs) don't work.
- The Gofer overhead makes filesystem-heavy workloads (database log writers) painful.
- Some applications check `uname` and refuse to run on the version gVisor reports.

### Comparison to alternatives

- **Linux namespaces + cgroups** (Docker default): smallest overhead, largest attack surface. Kernel CVEs reach all containers.
- **Firecracker / Kata Containers** (microVM): real KVM-backed VMs. More overhead than gVisor (boot ~125 ms), but mature isolation. Used by AWS Lambda.
- **gVisor**: in between. Process-level isolation; user-space kernel; lower boot than microVM.
- **WebAssembly runtimes** (Wasmtime, WasmEdge): even narrower interface; great for custom workloads, limited general-purpose use.

### Real-world deployment

Google Cloud Run runs tens of millions of containers daily under gVisor. The gVisor team publishes performance and security data: https://gvisor.dev.

### Go-specific engineering challenges

#### 1. Long-running Sentry processes can't tolerate stop-the-world GC

A few-ms GC pause means a few-ms latency spike on whatever syscall hit during STW. The team measures this carefully; mitigations include:

- Aggressive allocation reduction in hot syscall paths.
- `sync.Pool` for per-syscall scratch state.
- Pre-allocated free lists for common structures (`Task`, `FileDescription`, `VMA`).

#### 2. Unsafe pointer arithmetic

Implementing memory management is hard in Go without pointer arithmetic. gVisor uses `unsafe.Pointer` heavily in `pkg/safemem/`, with careful audit. Test coverage is high.

#### 3. Avoiding allocations during syscall handling

Allocations during syscalls trigger GC pressure that affects every other workload sharing the Sentry process. The team profiles `runtime/pprof/allocs:objects` continuously.

#### 4. Goroutines per task

Each guest thread is backed by a Sentry goroutine. A container with 100 threads has 100+ goroutines in Sentry. This is fine (goroutines are cheap), but it stresses the scheduler when guest processes spawn rapidly.

### Operating gVisor

`runsc debug --stacks <container-id>` dumps Sentry's goroutine stacks — essentially `runtime.Stack(buf, true)` output. Invaluable for diagnosing stuck containers.

`runsc events --stats` exposes per-container resource usage.

`gvisor-profiling` tools wrap Go's pprof for production diagnostics.

## Standard Library Hooks

gVisor uses, but doesn't ship as, the stdlib. As a Go project, it imports widely:

- `syscall` and `golang.org/x/sys/unix`: low-level access to host syscalls (a tiny, allowed set).
- `runtime`, `runtime/pprof`, `runtime/trace`: heavy profiling.
- `unsafe`: pointer manipulation for memory management.
- `sync/atomic`: lock-free data structures.
- `encoding/binary`: ABI marshalling.
- `context`: cancellation through syscalls (gVisor extends this internally).

## Real-World Patterns

### 1. Sandbox an OCI container

```bash
$ docker run --runtime=runsc -d --name=untrusted \
    -v "$PWD":/work nginx
```

Same docker CLI; just specify `--runtime=runsc`.

### 2. GKE Sandbox pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: untrusted
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: untrusted:latest
```

`runtimeClassName: gvisor` selects gVisor on a node that has it installed.

### 3. Inspect a sandbox at runtime

```bash
$ runsc list
ID          PID    STATUS  ...
$ runsc debug --stacks <id> > stacks.txt
$ runsc debug --profile-heap=heap.pprof <id>
$ go tool pprof heap.pprof
```

Profiles attribute time/memory to Sentry's Go code — a fascinating view of "what is the kernel doing".

### 4. Custom runsc config

```bash
$ runsc install --runtime=runsc-perf -- \
    --platform=systrap --network=netstack
```

Then in `daemon.json` reference `runsc-perf`. Useful for tuning per-workload.

### 5. Use netstack outside gVisor

```go
import "gvisor.dev/gvisor/pkg/tcpip/stack"

s := stack.New(stack.Options{
    NetworkProtocols:   []stack.NetworkProtocolFactory{ipv4.NewProtocol},
    TransportProtocols: []stack.TransportProtocolFactory{tcp.NewProtocol},
})
```

Standalone use of `netstack` as a user-space TCP/IP stack — Tailscale and several VPN projects do this. Lets your Go program implement networking without `net.Dial` and the host kernel's network stack.

## Anti-Patterns & Gotchas

**Running CPU/IO-heavy databases under gVisor.** The Gofer overhead and syscall cost will visibly degrade throughput. Use a microVM or bare metal.

**Assuming all Linux syscalls work.** Some are unimplemented. Check the list: https://gvisor.dev/docs/user_guide/compatibility/.

**Mixing gVisor and host-network containers in one Kubernetes pod.** Network expectations diverge. Choose one networking mode per workload.

**Treating gVisor as "Docker but secure".** It's a re-implementation; performance differs.

**Pointer arithmetic in user code expecting kernel semantics.** Some `/proc` and `/sys` paths return synthetic data from Sentry.

**Calling unsupported syscalls (e.g., `bpf`).** Returns `ENOSYS`. Apps that probe-and-fail-back handle it; apps that error out crash.

**Relying on kernel side-channel timings.** gVisor's timing differs; security-by-timing assumptions fail.

**Long-running mmap'd databases.** Page-fault overhead through Gofer is substantial; consider direct read/write APIs instead.

**Debugging without `runsc debug`.** A stuck Sentry isn't visible via `htop`; you need gVisor-aware tooling.

**Expecting "uname" to lie cleanly.** gVisor reports its emulated kernel version; some software refuses to run.

## Performance Notes

(From gVisor's published benchmarks; varies by workload.)

- Container startup: ~50–100 ms (faster than Firecracker's ~125 ms cold-boot).
- Memory overhead per sandbox: ~50 MiB (Sentry + Gofer).
- Syscall overhead: 1–5 µs (systrap), 5–10 µs (ptrace).
- TCP throughput (netstack): ~5–15 Gbps single connection; ~50% native.
- HTTP request/response (small): ~50% native QPS.
- Filesystem `read()` 4 KB: ~3× native latency (Gofer round-trip).
- Filesystem `read()` 1 MB: ~30% slower than native (bulk transfer amortized).

## How Big Companies Use It

- **Google Cloud Run / Functions / App Engine**: production multi-tenant isolation.
- **Google Kubernetes Engine (GKE Sandbox)**: opt-in pod sandboxing.
- **AWS** uses **Firecracker** (Rust microVM) for Lambda — different approach, but cited by gVisor team as the alternate point in the design space.
- **Tailscale** uses gVisor's netstack as the user-space TCP/IP stack for its `tsnet` library: https://tailscale.com/blog/userspace-networking.
- **Docker Desktop** has experimented with gVisor for Mac/Windows isolation.
- **Sysdig**, **Falco**, security researchers use gVisor as a hardened test harness for analyzing malicious binaries.
- **GKE customers** running multi-tenant SaaS use gVisor for tenant isolation in dedicated pods.

## Source Code References

Pinned to `google/gvisor` master.

- Sentry entry point: [`pkg/sentry/sentry.go`](https://github.com/google/gvisor/blob/master/pkg/sentry/sentry.go).
- Syscall table: [`pkg/sentry/syscalls/linux/`](https://github.com/google/gvisor/tree/master/pkg/sentry/syscalls/linux).
- TCP/IP stack: [`pkg/tcpip/`](https://github.com/google/gvisor/tree/master/pkg/tcpip).
- Filesystem (VFS): [`pkg/sentry/vfs/`](https://github.com/google/gvisor/tree/master/pkg/sentry/vfs).
- Memory management: [`pkg/sentry/mm/`](https://github.com/google/gvisor/tree/master/pkg/sentry/mm).
- Kernel scheduler (task management): [`pkg/sentry/kernel/`](https://github.com/google/gvisor/tree/master/pkg/sentry/kernel).
- Platforms: [`pkg/sentry/platform/`](https://github.com/google/gvisor/tree/master/pkg/sentry/platform).
- Gofer: [`runsc/fsgofer/`](https://github.com/google/gvisor/tree/master/runsc/fsgofer).
- runsc: [`runsc/`](https://github.com/google/gvisor/tree/master/runsc).

(Apache-2.0 © The gVisor Authors.)

## Further Reading

- gVisor documentation: https://gvisor.dev.
- "The True Cost of Containing: A gVisor Case Study" — Young et al., HotCloud 2019.
- "gVisor: Defense in depth for container workloads" — Google Security blog: https://security.googleblog.com.
- "What's the difference between Firecracker and gVisor?" — comparative analysis: https://news.ycombinator.com/.
- Adin Scannell, "gVisor — sandboxing Go" (GopherCon talks).
- "Tailscale's userspace networking" (uses netstack): https://tailscale.com/blog/userspace-networking.
- Brendan Burns, "Sandboxing containers" (general): https://www.brendangregg.com/.
- LWN, "gVisor: A sandbox for containers": https://lwn.net/Articles/771833/.
- gVisor source-tree tour (KubeCon talks): https://kccncna.com.

## Exercises / Self-Check

1. Why does gVisor implement its own TCP/IP stack instead of using the host kernel's? What does it gain and lose?
2. Sentry catches a `mmap` syscall from the guest. Walk through the steps: where does the page table change happen, where does the host get involved, and where does the Gofer fit in?
3. Run a syscall-heavy benchmark (`sysbench fileio` or similar) on Docker vs Docker + gVisor. Quantify the overhead. Hypothesize what the bottleneck is, then verify with `runsc debug --profile-cpu`.
4. Why is Go acceptable for kernel-level work despite its GC? What workloads would still benefit from C or Rust?
5. Use `gvisor.dev/gvisor/pkg/tcpip/stack` directly in a small Go program to create a user-space TCP server. Compare the API surface to the stdlib `net` package.
