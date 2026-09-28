# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.3 — 2026-09-29

**Withdrawn.**  The language now has `Vec[T; N]`, `String[N]`,
`Deque[T; N]` and `Map[K, V; N]` (SPEC section 14.8): fixed-capacity
collections with the capacity in the type, the storage inline and
`Err(Full)` at the capacity.  They replace `BoundedVec`, `BoundedStr`,
`BoundedFifo` and `BoundedMap`, and a package cannot wrap them, so no
0.1.0 will be published.  This release changes the README, the
manifest's description and this file; the interface is unchanged and
every body is still `todo()`.

- The README says the package is withdrawn, names the member of the
  family that replaces each type, and says what the family does not
  do: `Map[K, V; N]` takes integer and `Bool` keys only.
- The manifest keeps `stability = "draft"`, the only value an
  interface release may carry.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

- The README's question-and-record snippet is a whole program now.  It
  used `v`, `buf`, `b` and `dropped` without binding any of them, so
  `novo doc` could not compile it (`E2003`) and `novo pkg publish`
  refused the release over it.  It builds its storage with
  `bytes.zeros` and its vector with `heapless.vec` before the pair of
  calls it is there to show.

## 0.0.1 — 2026-09-10

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- `BoundedVec`, `BoundedStr`, `BoundedFifo` and `BoundedMap` — the four
  collections as `@value` structs of two or three integers each, so a
  program with no allocator can hold one on its stack or in a static
  cell.
- Every write is a question and then a record: `vec_push_at` answers the
  slot or `None`, the caller writes there, and `vec_pushed` records that
  it happened.  Asking consumes nothing, so a caller that decided not to
  write after all has changed nothing.
- `BoundedStr`'s one rule: a code point goes in whole or not at all.
  `str_push_at` takes the lead byte, so a buffer that ran out
  mid-message still holds valid UTF-8 — the case a logger truncating at
  a fixed buffer's end meets on its first non-ASCII line.
- `BoundedMap`'s probe rule as `map_slot` and `map_max_probes`, with the
  power-of-two capacity and the load factor in `map_capacity_for`.  The
  lookup loop is the caller's, because the keys and the comparison are.
- `map_removed` and `map_needs_rehash`: a removal in an open-addressed
  table leaves a tombstone rather than an empty slot, so a table can be
  full while holding nothing.

**`ringbuf-nv` is `BoundedFifo`.**  The must-have plan's `ringbuf-nv`
row is "heapless's single-producer queue", which is the same upstream
type this package ports; two packages would mean the same wrap
arithmetic twice, so the row is absorbed here.  It is not `bbqueue-nv`:
this moves elements one at a time, that moves bytes in contiguous runs
and never splits a frame.

**Two constraints shaped every signature**, and the README says what the
language would need to lift them.  A `@value` struct takes no type
parameters and there are no const generics, so the capacity is a field
rather than part of the type — which means a capacity that does not
match the caller's buffer is not caught at compile time.  And a `@value`
struct is built whole and replaced whole (SPEC § 14.3), so a collection
that owned its elements would copy the whole store per write.
