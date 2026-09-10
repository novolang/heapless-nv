# heapless-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Fixed-capacity collections for the tier that has no allocator: a
**bounded vector**, a **bounded UTF-8 buffer**, a **bounded FIFO** and a
**bounded linear-probing map**.  Each is a `@value` struct of two or
three integers, each has its capacity fixed when it is made, and none of
them panics when it is full.

They are the frame buffers and the small tables under a deferred-logging
device: a message assembled into a bounded buffer, sensor readings
queued behind an interrupt, an interned index looked up in a table sized
at build time.

## Adding it, and checking it

```bash
novo pkg add heapless-nv     # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/heapless_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: heapless.<fn>`.  They turn green
one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use heapless

// The two-step: the collection answers where, the caller writes there,
// and a second call records that the write happened.
fn main() [io]
    var buf = bytes.zeros(4)
    var v = heapless.vec(4)

    match heapless.vec_push_at(v)
        None     => println("full")
        Some(at) =>
            buf = bytes.set_u8(buf, at, 0xAB)
            v = heapless.vec_pushed(v)

    println("${heapless.vec_len(v)}")   // 1
```

## How the capacity is fixed, and what that costs

**Not at the type, because the language has no way to put it there.**
A `@value` struct takes no type parameters and there are no const
generics (SPEC § 14.2), so `Vec<T, N>` — heapless's own spelling — has
no counterpart: every element type and every capacity would have to be a
distinct type written out by hand.  So the capacity is a **field**, set
when the collection is made.

That has one cost and it is worth saying plainly: **the capacity is not
checked at compile time.**  `heapless.vec(8)` over a four-byte buffer is
a program that will answer slot 4 and let the caller write past the end
of its own storage.  A compile-time `N` would have caught that; a field
cannot.  What the field buys instead is that one `BoundedVec` works over
a `Bytes` on a host, a static region on a device and a window into a DMA
buffer, which a container parameterised on `[T; N]` could not do.

**And the storage is the caller's**, which is the other half of the same
finding.  A `@value` struct is built whole and replaced whole (SPEC
§ 14.3) — there is no field assignment, and none into a `[T; N]` element
either — so a vector that owned `[u8; 256]` would copy 256 bytes on
every push.  Quadratic behaviour in the one operation the package exists
to make cheap is not a trade; it is a wrong answer.  So each collection
holds the bookkeeping, the caller holds the elements, and every write is
two calls:

```novo
match heapless.vec_push_at(v)
    None     => dropped = dropped + 1
    Some(at) =>
        buf = bytes.set_u8(buf, at, b)
        v = heapless.vec_pushed(v)
```

What the language would need for the container to own its elements is an
unboxed aggregate that can be updated in place — a mutable `@value`
binding, or a borrow of one field of one — together with a const generic
so the capacity can be a type.  The lane's report names both.

## Nothing panics on a full container

heapless's rule, kept: every operation that can fail on a full or empty
collection answers `?Int`, and `None` is the whole of the failure.

heapless spells it `Result<(), T>` — the item comes back in the `Err`,
so a caller that could not push has not lost what it was pushing.  Here
the item **never left the caller**: the collection was told where to put
one, not given one.  So the guarantee is stronger and the signature is
smaller, and there is no error type in this package at all.

There are two panics, and both are for a program that is wrong rather
than a program that is full: `heapless.map` on a capacity that is not a
power of two, because a map that probed with the wrong mask would answer
wrong values forever, and the ordinary out-of-range panic a caller gets
from its own storage.

## `ringbuf-nv` lives here

novo-lang's package grid carries a `ringbuf-nv` row described as
"heapless's single-producer queue" (`embedded`, `core`, P1).  That is
`BoundedFifo`, and it is in this package rather than in one of its own.

The reason is that it is the same upstream type.  A separate `ringbuf-nv`
would port `heapless::spsc::Queue`, and this package ports
`heapless::Deque` — which are one queue in the reference implementation
and would be two ports to keep in step here, with the same wrap
arithmetic written twice and a caller having to know which of them their
other bounded collection came from.  One package, one FIFO.

**It is not `bbqueue-nv`, and the distinction is the load-bearing one
for anybody choosing between them.**  `BoundedFifo` moves **elements**,
one at a time, and wraps by index arithmetic.  `bbqueue-nv` moves
**bytes**, in contiguous runs, and wraps by a watermark so that a frame
is never split across the end of the buffer.  A caller queueing sensor
readings wants this one; a caller writing a whole log frame out of an
interrupt wants that one, because a frame handed back in two pieces is
what the grant model exists to prevent.

## The layer, and why

`core`.  Every function is total, from values to values: the
collections are two or three integers each, and none of them reads or
writes a byte.  There is no effect to declare because there is nothing
here to perform.

`tests/embedded_probe.nv` is the device claim in a form that either
builds or does not, and it exercises **all four** collections over one
inline array.  It takes the whole public surface rather than a core
subset of it, because nothing here touches `Bytes`, `Str` or `Result`.

## The load-bearing interface

Four types, and the fact that every write is a question and then a
record.

```novo
pub @value
struct BoundedVec
    cap: Int
    len: Int

pub fn vec_push_at(v: BoundedVec) -> ?Int
pub fn vec_pushed(v: BoundedVec) -> BoundedVec
```

`?Int` and not a `Result`, because there is one way to fail and it does
not need a name.  Two calls and not one, because the element is never
this package's to hold.  And `vec_push_at` **consumes nothing** — asking
twice answers twice — which is what makes the pair safe to write in
either order and what a caller who decided not to write after all
depends on.

The map is the type where the split earns the most:

```novo
pub fn map_slot(m: BoundedMap, hash: Int, probe: Int) -> Int
pub fn map_max_probes(m: BoundedMap) -> Int
```

A lookup is a loop the caller writes, comparing its own keys, with this
package supplying only where to look next.  That is not a limitation
here — it is the only way one map serves keys of a type a `@value`
struct could not be generic over, and it means the probe rule has one
home instead of one per key type.

`BoundedStr` is the type whose whole content is one rule: a code point
goes in **whole or not at all**.  `str_push_at` takes the lead byte,
answers `None` when the remaining room is smaller than the character it
begins, and so a buffer that ran out mid-message still holds valid
UTF-8.  That is the case a logger truncating at a fixed buffer's end
meets on its first non-ASCII line.

## The reference implementation

`heapless` (MIT/Apache-2.0, the Rust Embedded working group): `Vec<T, N>`,
`String<N>`, `Deque<T, N>` / `spsc::Queue<T, N>`, and `FnvIndexMap<K, V, N>`
with its power-of-two capacity and linear probing.  Its test suite is
the oracle for the wrap arithmetic, the probe walk and the load factor.

Three things change, and each is named where it happens: the capacity is
a field rather than a const generic; the elements are the caller's
rather than the collection's; and `Result<(), T>` becomes `?Int`,
because an item that never left the caller needs no returning.

## Status

| function | implemented |
| --- | --- |
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
