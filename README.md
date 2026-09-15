# heapless-nv

A **fixed-capacity collection** is one whose maximum size is decided when it
is made and never changes, so that it needs no allocator. This package brings
four of them to novo-lang: a bounded vector, a bounded UTF-8 text buffer, a
bounded first-in-first-out queue, and a bounded hash map. It is a port of the
Rust crate [heapless](https://docs.rs/heapless) by the Rust Embedded working
group. [can-nv](https://novo-lang.org/packages/can-nv) is built on it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What it is

An ordinary vector grows by asking an allocator for more memory. A
microcontroller often has no allocator, and a program that must never fail
for want of memory cannot have one. A fixed-capacity collection answers that
by refusing to grow: it is given its capacity once, and an operation that
would exceed it fails in a way the caller can see.

Each of the four collections here is a **`@value` struct** of two or three
integers, which means it lives in the caller's stack frame or in static
memory with no header, no reference count and nothing to free.

**The collection holds the bookkeeping and the caller holds the elements.**
A vector here is a capacity and a length. The bytes, or the values, live in a
buffer the caller owns: a `Bytes` on a machine with an operating system, a
fixed array in static memory on a device, or a window into a region a
peripheral writes into.

That makes every write two calls. The first asks the collection where the
element goes and answers nothing when there is no room. The caller writes it
there. The second records that the write happened.

```novo
match heapless.vec_push_at(v)
    None     => dropped = dropped + 1
    Some(at) =>
        buf = bytes.set_u8(buf, at, b)
        v = heapless.vec_pushed(v)
```

The **map** is an open-addressed table with **linear probing**: a key hashes
to a slot, and if that slot is taken the next one is tried, and so on. The
capacity is a power of two so that the wrap is a mask. Removing an entry
leaves a **tombstone**, a slot marked as used-and-now-empty, because an empty
slot would end a probe chain that runs through it. A table can therefore be
full of tombstones while holding nothing, and rehashing is what clears them.

| Type | Fields | What it holds |
| --- | --- | --- |
| `BoundedVec` | capacity, length | Elements in order, appended and removed at the end. |
| `BoundedStr` | capacity, length | UTF-8 text, measured in bytes, never cut inside a character. |
| `BoundedFifo` | capacity, head, length | Elements in arrival order, added at one end and taken from the other. |
| `BoundedMap` | capacity, length, dead | Slots for keys and values, probed linearly. |

| Quantity | Value |
| --- | --- |
| Integers in a collection | 2 or 3 |
| Bytes the package allocates | 0 |
| Value a failed operation answers | nothing |
| Map capacity | a power of two |
| Map capacity for n entries | the next power of two at or above 2n |

## Install

```
novo pkg add heapless-nv
```

## Example

```novo
use std.bytes
use heapless

fn main() [io]
    // The collection keeps the bookkeeping. This is the caller's storage.
    var buf = bytes.zeros(4)

    // A vector of at most four elements. Two integers, no allocation.
    var v = heapless.vec(4)

    // Ask where the next element goes. `None` means the vector is full,
    // and asking does not change it.
    match heapless.vec_push_at(v)
        None     => println("full")
        Some(at) =>
            // Write the element into your own storage, at that index.
            buf = bytes.set_u8(buf, at, 0xAB)
            // Then record that the write happened.
            v = heapless.vec_pushed(v)

    println("${heapless.vec_len(v)}")
```

Build and test with `novo pkg build` and
`novo test --isolate tests/heapless_tests.nv`. Today `novo test` fails on
purpose: every test reaches a `not implemented: heapless.<fn>` panic. The
tests are the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `heapless` | All four collections: the vector, the text buffer, the queue and the map, with the question-and-record pair for each write and the accessors that report capacity, length and fullness. |

## How to choose an entry point

**`BoundedVec` when elements are appended and read by index.**

**`BoundedStr` when the elements are text.** It is a vector of bytes with one
extra rule, and the rule is the reason it is a separate type.

**`BoundedFifo` when elements are added at one end and taken from the
other.** A sensor reading queued behind an interrupt and read by the main
loop is the case.

**`BoundedMap` when a value is looked up by a key.** Size it with
`heapless.map_capacity_for`, which applies the load factor rather than
leaving it to the caller.

**Use [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv) instead of
`BoundedFifo` when the elements are bytes of one frame.** `BoundedFifo` moves
elements one at a time and wraps by index arithmetic, so a frame that crosses
the wrap comes back in two pieces. bbqueue-nv moves bytes in contiguous runs
and wraps by a watermark, so a frame never does.

## The rules a user needs

1. **Nothing here panics because a collection is full or empty.** Every
   operation that can fail answers an optional index, and the absence is the
   whole of the failure. The reference implementation returns the item in an
   error, so a caller that could not push has not lost it. Here the item
   never left the caller.
2. **Asking where consumes nothing.** `vec_push_at` and its three siblings
   leave the collection unchanged, so asking twice answers twice, and a
   caller that decides not to write after all has changed nothing.
3. **The capacity is a field, and it is not checked against your storage.**
   novo-lang has no integer type parameters and a `@value` struct takes no
   type parameter at all (SPEC section 14.2), so the capacity cannot be part
   of the type. `heapless.vec(8)` over a four-byte buffer will answer slot 4
   and let the caller write past the end of its own storage. Nothing in this
   package can catch that.
4. **A code point goes into a `BoundedStr` whole or not at all.**
   `str_push_at` takes the character's lead byte and answers nothing when the
   remaining room is smaller than the character that byte begins. A buffer
   that runs out mid-message therefore still holds valid text, which is the
   case a logger truncating at a fixed length meets on its first non-ASCII
   line. `heapless.str_seq_len` says how many bytes a lead byte announces.
5. **A map's capacity must be a power of two, and a capacity that is not
   panics.** A map that probed with the wrong mask would answer wrong values
   for ever, so this is a mistake in the program rather than a value the
   program should carry.
6. **Size a map with `map_capacity_for`, not with the number of entries.**
   Open addressing degrades sharply as a table fills, because the probe
   chains join up. `map_capacity_for(3)` is 8 and `map_capacity_for(10)` is
   32.
7. **The lookup loop is yours.** `map_slot` answers which slot to try on
   probe number `n`, and `map_max_probes` says when to stop. The comparison
   of keys is the caller's, because the keys are. That is what lets one map
   serve key types a `@value` struct could not be generic over.
8. **A removal leaves a tombstone, so a map can be full while holding
   nothing.** `map_dead` counts them and `map_needs_rehash` says when to
   rebuild.
9. **`vec_swap_remove_at` does not preserve order.** It moves the last
   element into the hole, which is one write rather than a shift of
   everything after it.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Here it covers the whole
public surface. Nothing in the package uses `Bytes`, `Str`, `Result` or a
list, so there is no host-only half to leave out.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not. It exercises all four collections over one inline array.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

## What is not included

- **The storage.** A `@value` struct is built whole and replaced whole
  (SPEC section 14.3), so a vector that owned a 256-byte array would copy all
  256 bytes on every push. That is quadratic behaviour in the one operation
  the package exists to make cheap. The elements stay with the caller.
- **A capacity in the type.** See rule 3.
- **An error type.** See rule 1.
- **A hash function.** `map_slot` takes a hash the caller computed. Which
  hash a key deserves is the caller's decision, and a map that chose one
  would be a map for one kind of key.
- **Iteration.** Each collection answers indices, and the loop over them is
  the caller's, for the same reason the map's lookup loop is.
- **A byte queue that never splits a frame.** See "How to choose an entry
  point".

## Related packages

- [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv) is the byte queue
  whose unit of work is a contiguous run rather than an element. A caller
  writing a whole log frame out of an interrupt wants that one.
- [can-nv](https://novo-lang.org/packages/can-nv) builds its bounded transmit
  queue on this package.
- [bitfield-nv](https://novo-lang.org/packages/bitfield-nv) is the other
  small-value package for a device: a hardware register's bits, described
  once and read by the description.
- [fixedpoint-nv](https://novo-lang.org/packages/fixedpoint-nv) is the
  arithmetic for the same machines, where there is no floating-point unit.
- `std.list` and `std.map` in the standard library are the growable
  collections. They allocate, they own their elements, and they are not
  available on a device with no allocator.

## Tests

```bash
novo test --isolate tests/heapless_tests.nv   # 22 tests
```

The oracle is `heapless`'s own test suite: the wrap arithmetic of the queue,
the probe walk of the map, and the load factor. The upstream types are
`Vec`, `String`, `Deque`, `spsc::Queue` and `FnvIndexMap`.

The tests compile today and fail at run, each on the
`not implemented: heapless.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `heapless.BoundedVec`, `.BoundedStr`, `.BoundedFifo`, `.BoundedMap` | declared |
| `heapless.vec`, `.vec_push_at`, `.vec_pushed` | no |
| `heapless.vec_pop_at`, `.vec_popped`, `.vec_at`, `.vec_swap_remove_at` | no |
| `heapless.vec_truncated`, `.vec_cleared` | no |
| `heapless.vec_len`, `.vec_capacity`, `.vec_is_full`, `.vec_is_empty` | no |
| `heapless.bounded_str`, `.str_seq_len`, `.str_push_at`, `.str_pushed` | no |
| `heapless.str_cleared`, `.str_len`, `.str_capacity`, `.str_remaining` | no |
| `heapless.str_is_full`, `.str_is_empty` | no |
| `heapless.fifo`, `.fifo_enqueue_at`, `.fifo_enqueued` | no |
| `heapless.fifo_dequeue_at`, `.fifo_dequeued`, `.fifo_peek_at`, `.fifo_at` | no |
| `heapless.fifo_cleared`, `.fifo_len`, `.fifo_capacity` | no |
| `heapless.fifo_is_full`, `.fifo_is_empty` | no |
| `heapless.map_capacity_for`, `.map`, `.map_slot`, `.map_max_probes` | no |
| `heapless.map_inserted`, `.map_removed`, `.map_needs_rehash`, `.map_cleared` | no |
| `heapless.map_len`, `.map_capacity`, `.map_dead` | no |
| `heapless.map_is_full`, `.map_is_empty` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
