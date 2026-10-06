# heapless-nv

**Status: WITHDRAWN.** Do not add this package. The four fixed-capacity
collections it described are now part of the language: `Vec[T; N]`,
`String[N]`, `Deque[T; N]` and `Map[K, V; N]`, specified in SPEC section 14.8
("The heapless family"). Version 0.0.4 is the last release, and it exists to
say so.

heapless-nv was published as an interface only. Every function was declared
with its full signature and every body was a `todo()` that panics when called,
so no program has depended on a working body.

## What replaces each type

| heapless-nv | The language |
|---|---|
| `BoundedVec` | `Vec[T; N]` |
| `BoundedStr` | `String[N]` |
| `BoundedFifo` | `Deque[T; N]` |
| `BoundedMap` | `Map[K, V; N]` |

Each member of the family is a `@value` value whose capacity is part of its
type and whose elements are stored inline, in the stack frame for a local and
in static storage for a module-level `var`. A write at the capacity answers
`Err(Full)` and leaves the value unchanged. Building, pushing, indexing and
iterating allocate nothing.

## What the family does not do

`Map[K, V; N]` takes a key of an integer type or `Bool` only. heapless-nv's
map left the keys and their comparison to the caller, so it could index by
any key. A program that needs a bounded map keyed by text keeps its own table
over a `Vec[T; N]` until the family covers it.

`String[N]` appends bytes. `push_str` adds every byte of a `Str` or none of
them, so text appended that way stays valid UTF-8. heapless-nv's rule that a
code point goes in whole applied to text arriving one byte at a time, and with
`String[N]` that check is the caller's.

## Related packages

- [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv) is not replaced by
  the family and stays. It moves bytes a contiguous run at a time and never
  splits a frame at the wrap, which a `Deque[u8; N]` does.

## Tests

The interface still builds, and its suite still runs, from a clone of this
repository:

```sh
novo pkg build
novo test tests/heapless_tests.nv
```

Every test fails on the `not implemented: heapless.<fn>` panic
of the `todo()` it reaches, since no body will be written. The suite is kept
because it records what each function was specified to do.
`tests/embedded_probe.nv` is a program, not a test: it builds the interface
for a Cortex-M4 target to show that no signature needs an allocator.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
