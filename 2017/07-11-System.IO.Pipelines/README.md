# API Review for `System.IO.Pipelines`

Status: **Needs more work** | [API Reference](System.IO.Pipelines.md) |
[Video](https://www.youtube.com/watch?v=IXg58zMsPug)

We've taken a high-level look at `System.IO.Pipelines`. You can find more
details in this [speclet](https://github.com/dotnet/corefxlab/blob/master/docs/specs/pipelines-io.md).

The general value prop of `System.IO.Pipelines` is to provide the illusion of an
infinite buffer that a writer can write to and a reader can read from. It
automatically maintains buffers and provides pooling.

In practical terms, it's a programming model for `Span<T>` and `Buffer<T>` that
focuses on reducing allocations and efficient buffer pooling.

## Feedback 1

* `IPipe` should probably be an abstract class
* `IPipe.Reset()` shouldn't exist on the abstraction
* `IPipeWriter` exposes `Alloc()` which returns a buffer
    - It seems the API is lying; the `WritableBuffer` should be a mode on the
      writer, rather than a produced entity
    - It results in fairly long calls where the developer is seemingly chasing
      the thing that is the actual buffer they can write to.
* `WritableBuffer`
    - should be merged with `IPipeWriter`
    - `AsReableBuffer()` seems wrong (or, as David Fowler would say "trust me, it's wrong")
* Pooling types should be in a nested namespace, e.g.
  `System.IO.Pipelines.Pooling`
* `IScheduler` seems like a BCL-ish concept and shouldn't live in pipelines
    - We should make sure we don't have an abstraction and add one if necessary
    - Should probably be an abstract class

## Feedback 2

* We cannot converge `ReadableBuffer` with `IPipeReader`
    - Which means we should probably not doing on the writer otherwise we'd
      break the symmetry
* We should change the semantics of advancement
    - If you didn't consume, you clearly examined the data and decided that you
      don't want to consume it because you don't have enough data
    - So if you don't call `Advance()` then you examined everything but didn't
      consume
    - This would mean that `Advance()` only has to take the consumed range
* We're using cursors in order to abstract the fact that the buffers might not
  be consecutive; it's similar to what Netty does.
    -  Krzysztof is proposing to use ints for advances/moves and only use
       something like a cursor for the start location. If the start is implicit
       it avoid having to frequently pass large-ish structs around.
* Expose low-level APIs to party on multiple segments