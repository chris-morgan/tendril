# tendril

**Warning**: This library is at a very early stage of development, and it
contains a substantial amount of `unsafe` code. Use at your own risk!

[![Build Status](https://travis-ci.org/kmcallister/tendril.svg?branch=master)](https://travis-ci.org/kmcallister/tendril)

[API Documentation](https://kmcallister.github.io/docs/tendril/tendril/index.html)

## Introduction

`Tendril` is a compact string/buffer type, optimized for zero-copy parsing.
Tendrils have the semantics of owned strings, but are sometimes views into
shared buffers. When you mutate a tendril, an owned copy is made if necessary.
Further mutations occur in-place until the string becomes shared, e.g. with
`clone()` or `subtendril()`.

Buffer sharing is accomplished through thread-local (non-atomic) reference
counting, which has very low overhead. The Rust type system will prevent
you at compile time from sending a tendril between threads. (See below
for thoughts on relaxing this restriction.)

Whereas `String` allocates in the heap for any non-empty string, `Tendril` can
store small strings (up to 8 bytes) in-line, without a heap allocation.
`Tendril` is also smaller than `String` on 64-bit platforms — 16 bytes versus
24. `Option<Tendril>` is the same size as `Tendril`, thanks to
[`NonZero`][NonZero].

The maximum length of a tendril is 4 GB. The library will panic if you attempt
to go over the limit.

## Formats and encoding

`Tendril` uses [phantom types](http://rustbyexample.com/generics/phantom.html)
to track a buffer's format. This determines at compile time which
operations are available on a given tendril. For example, `Tendril<UTF8>` and
`Tendril<Bytes>` can be borrowed as `&str` and `&[u8]` respectively.

`Tendril` also integrates with
[rust-encoding](https://github.com/lifthrasiir/rust-encoding) and has
preliminary support for [WTF-8][] buffers.

## C interface

`Tendril` provides a C API, which allows Rust to efficiently exchange buffers
with C or any other language.

```c
#include "tendril.h"

int main() {
    tendril t = TENDRIL_INIT;
    tendril_sprintf(&t, "Hello, %d!\n", 2015);
    tendril_fwrite(&t, stdout);
    some_rust_library(t);  // transfer ownership
    return 0;
}
```

See the [API documentation](https://github.com/kmcallister/tendril/blob/master/capi/include/tendril.h#L18)
and the [test program](https://github.com/kmcallister/tendril/blob/master/capi/ctest/test.c).

## Plans for the future

### Ropes

[html5ever][] will use `Tendril` as a zero-copy text representation. It would
be good to preserve this all the way through to Servo's DOM. This would reduce
memory consumption, and possibly speed up text shaping and painting. However,
DOM text may conceivably be larger than 4 GB, and will anyway not be contiguous
in memory around e.g. a character entity reference.

*Solution:* Build a **[rope][] on top of these strings** and use that as
Servo's representation of DOM text. We can perhaps do text shaping and/or
painting in parallel for different chunks of a rope. html5ever can additionally
use this rope type as a replacement for `BufferQueue`.

Because the underlying buffers are reference-counted, the bulk of this rope
is already a [persistent data structure][]. Consider what happens when
appending two ropes to get a "new" rope. A vector-backed rope would copy a
vector of small structs, one for each chunk, and would bump the corresponding
refcounts. But it would not copy any of the string data.

If we want more sharing, then a [2-3 finger tree][] could be a good choice.
We would probably stick with `VecDeque` for ropes under a certain size.

### UTF-16 compatibility

SpiderMonkey expects text to be in UCS-2 format for the most part. The
semantics of JavaScript strings are difficult to implement on UTF-8. This also
applies to HTML parsing via `document.write`. Also, passing SpiderMonkey a
string that isn't contiguous in memory will incur additional overhead and
complexity, if not a full copy.

*Solution:* Use **WTF-8 in parsing** and in the DOM. Servo will **convert to
contiguous UTF-16 when necessary**.  The conversion can easily be parallelized,
if we find a practical need to convert huge chunks of text all at once.

### Sendable

We don't need to share strings between threads, but we do need to move them.

*Solution:* Provide a **separate type for sendable strings**. Converting to
this type entails a copy, unless the refcount is 1.

### Optional atomic refcounting

The above `Send` implementation is not good enough for off-main-thread parsing
in Servo. We will end up copying every small string when we send it to the main
thread.

*Solution:* Use another phantom type to **designate strings which are
atomically refcounted**. You "set" this type variable when you create a string
or promote one from uniquely owned. This statically eliminates the overhead of
atomic refcounting for consumers who don't need strings to have guaranteed
zero-copy `Send`. html5ever will be generic over this choice.

### Source span information

Some html5ever API consumers want to know the originating location in the HTML
source file(s) of each token or parse error. An example application would be a
command-line HTML validator with diagnostic output similar to `rustc`'s.

*Solution:* Accept **some metadata along with each input string**. The type of
metadata is chosen by the API consumer; it defaults to `()`, which has size
zero. For any non-inline string, we can provide the associated metadata as well
as a byte offset.

[NonZero]: http://doc.rust-lang.org/core/nonzero/struct.NonZero.html
[html5ever]: https://github.com/servo/html5ever
[WTF-8]: http://simonsapin.github.io/wtf-8/
[rope]: http://en.wikipedia.org/wiki/Rope_%28data_structure%29
[persistent data structure]: http://en.wikipedia.org/wiki/Persistent_data_structure
[2-3 finger tree]: http://staff.city.ac.uk/~ross/papers/FingerTree.html
