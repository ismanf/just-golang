# `structs` — `HostLayout` Marker (since 1.25)

## TL;DR

`structs.HostLayout` (since 1.25) is a zero-sized embedded marker telling the Go compiler "this struct must match the platform C ABI layout exactly" — same field offsets, alignment, padding as a C struct of the same shape. Use when interoperating with cgo, syscalls, ioctls, kernel UAPI structs, or hardware register layouts. Without it, Go is free to reorder or repack fields in future versions; with it, the compiler commits to the C ABI.

## Mental Model

```
Normal Go struct:
   Field layout follows Go's rules: source order, alignment per arch.
   Tomorrow the compiler may reorder for packing.

struct with structs.HostLayout (1.25+):
   Field layout matches the host platform's C layout for the same shape.
   Future versions of the compiler commit to NOT reordering.
   
Use cases: cgo, syscall, kernel UAPI, hardware MMIO regions.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"structs"
	"unsafe"
)

type CFoo struct {
	_  structs.HostLayout // marker; zero size
	A  uint32
	B  uint8
	C  uint32
}

func main() {
	fmt.Println(unsafe.Sizeof(CFoo{})) // matches sizeof in C
	fmt.Println(unsafe.Offsetof(CFoo{}.C))
}
```

## Deep Dive

### What `HostLayout` guarantees

When the first field (or embedded marker) is `structs.HostLayout`:

- The struct's field offsets follow the platform's C ABI rules.
- Padding insertions follow C alignment rules for the platform.
- The compiler will not silently re-pack fields in future Go versions.

Without `HostLayout`, Go's struct layout is unspecified — historically it has matched C, but the Go team reserves the right to optimize. `HostLayout` makes the commitment explicit.

### Where it matters

- **cgo struct sharing**: `C.struct_foo` and a Go mirror with `HostLayout` are guaranteed bit-equivalent across versions.
- **syscalls**: `syscall.Stat_t`, ioctl request/response structs.
- **Kernel UAPI**: networking structs (`sockaddr_in`, `ip_mreq`), uring submission queues.
- **Hardware MMIO**: device register layouts.
- **File formats**: binary structures read from disk as a single blob.

### Without `HostLayout`

Most existing code passes structs across cgo or syscall boundaries without `HostLayout` and works. The reason: the Go compiler currently matches C for these layouts. `HostLayout` future-proofs you.

### What it does NOT do

- Doesn't change layout for normal Go-only structs.
- Doesn't fix endianness — bytes still need conversion if running cross-arch.
- Doesn't insert padding you forgot in your C definition.
- Doesn't enable C++ name-mangled members or vtables.

### Comparison with manual alignment

Pre-1.25 patterns:

```go
type CFoo struct {
	A uint32
	B uint8
	_ [3]byte // manual pad to match C
	C uint32
}
```

Works, but error-prone — different C compilers and platforms pack differently. `HostLayout` delegates the work to the Go compiler.

### Combining with `golang.org/x/sys`

Many syscall-adjacent packages already use the equivalent (internal markers); `structs.HostLayout` standardizes the pattern for user code.

## Standard Library Hooks

- `unsafe.Sizeof`, `unsafe.Offsetof`, `unsafe.Alignof` — measure actual layout.
- `syscall` and `golang.org/x/sys/unix` structs — many will adopt `HostLayout` over time.

## Real-World Patterns

### 1. Mirror a C struct exactly

```go
// In C:
// struct timespec { time_t tv_sec; long tv_nsec; };

import "structs"

type Timespec struct {
	_      structs.HostLayout
	Sec    int64 // time_t on most 64-bit unixes
	Nsec   int64
}
```

### 2. ioctl payload

```go
type IfReq struct {
	_     structs.HostLayout
	Name  [16]byte  // ifr_name
	Flags uint16
	_     [22]byte  // padding to sizeof(struct ifreq)
}
```

### 3. Memory-mapped device register block

```go
type UART struct {
	_   structs.HostLayout
	DR  uint32 // data register
	RSR uint32 // receive status
	_   [16]byte
	FR  uint32 // flag register
}

// Cast a mapped region:
regs := (*UART)(unsafe.Pointer(mappedAddr))
regs.DR = 'A'
```

Use case: bare-metal Go (TinyGo), driver development.

### 4. File header struct

```go
type ELFHeader struct {
	_       structs.HostLayout
	Ident   [16]byte
	Type    uint16
	Machine uint16
	Version uint32
	Entry   uint64
	// ...
}
```

Read with `binary.Read` against a `bytes.Reader`.

### 5. Sharing with cgo

```go
/*
#include <stdint.h>
struct Pkt { uint32_t id; uint8_t kind; uint32_t len; };
*/
import "C"
import "structs"

type Pkt struct {
	_    structs.HostLayout
	ID   uint32
	Kind uint8
	Len  uint32
}

// Now (*Pkt)(unsafe.Pointer(&cPkt)) is sound across compilers.
```

## Anti-Patterns & Gotchas

**Adding `HostLayout` to every Go struct.** Unnecessary; only for ABI-bound structs.

**Forgetting endianness.** `HostLayout` doesn't byte-swap.

**Mismatching the C definition.** Use `cgo -godefs` or hand-verify field types/sizes.

**Putting `HostLayout` *not* as the first field.** It must be first for the marker to apply correctly (it's zero-sized but its presence drives the layout decision).

**Assuming `unsafe.Sizeof` matches automatically without `HostLayout`.** It usually does today, but no guarantee.

**Using `HostLayout` on a struct with a `string` or slice field.** Those types are Go-specific; can't be ABI-shared with C.

## Performance Notes

- Zero cost at runtime — it's a marker for the compiler.
- Layout may insert more padding than Go's defaults on some platforms; tiny memory overhead.
- For data-heavy structs (millions of instances), measure with `unsafe.Sizeof` and consider whether the C alignment is worth it.

## How Big Companies Use It

- **TinyGo** users (microcontroller code) will adopt for MMIO register blocks.
- **Tailscale's networking** has hand-rolled equivalents that may migrate.
- **Cilium / eBPF Go loaders** use C struct mirrors heavily; `HostLayout` makes their code future-proof.
- **Stdlib `syscall` and `x/sys/unix`** are expected to adopt over time.

## Source Code References

Pinned to `go1.26`.

- `structs`: [`src/structs/structs.go`](https://github.com/golang/go/blob/master/src/structs/structs.go).
- Compiler handling: [`src/cmd/compile/internal/types/`](https://github.com/golang/go/tree/master/src/cmd/compile/internal/types).
- 1.25 release notes: https://go.dev/doc/go1.25.

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/structs.
- Proposal #66408: https://github.com/golang/go/issues/66408.
- "cgo: structs and the ABI" — go.dev/wiki/cgo.

## Exercises / Self-Check

1. Add `structs.HostLayout` to a syscall struct (e.g., a custom ioctl). Compare `unsafe.Sizeof` before and after.
2. Mirror a small C struct via `cgo -godefs`; then convert it to a hand-written Go type with `HostLayout`.
3. Write an MMIO-like register struct and read fields via `unsafe.Pointer` cast. (Don't actually run on hardware; treat as a thought experiment.)
4. Why doesn't `HostLayout` work on a struct containing `string`? Trace what the Go runtime expects.
5. Find an existing project that hand-pads C structs in Go; propose migrating to `HostLayout`.
