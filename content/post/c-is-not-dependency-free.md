+++
title = "C Is Not Dependency Free"
date = 2020-11-13
draft = true
tags = ["os", "no_std", "c", "c++", "rust"]
+++

Like any great systems programming language, it doesn't take much of anything to
get C up and running on a new platform.

In fact, a minimal C "runtime" (typically called
[`crt0`](https://en.wikipedia.org/wiki/Crt0)) typically consists of a short
prelude that zeroes out the .bss section and hands off execution to `main`.

That's it!

Compared to other popular programming languages which require substantial
machinery to operate, such as Java's virtual machine, or Go's garbage collector,
C's extremely minimal language "runtime" has allowed it to run on just about
everything.

Unfortunately, while C is effectively dependency free at the _language_ level,
the same can't be said about its _standard library_.

<!--more-->

> To keep things simple, I'll be focusing exclusively on C, though it should be
> noted that everything here applies to C++ as well. While not a strict superset
> of C, C++ inherits most of C's language design and standard library, making it
> equally susceptible to the issues outlined below.

Owing to its strong UNIX heritage, the C standard library includes many
functions and features which assume the existence of some sort of underlying OS.
For example, even the humble and ubiquitous `printf` function relies on the
existence of the `write` syscall, the idea of file descriptors, the `stdout`
stream, etc...

When writing bare-metal C code, beginners often absent-mindedly
`#include <stdio.h>` in an effort to do some quick-and-dirty `printf` debugging.
Unfortunately, while it might appear as though everything is working fine when
compiling individual translation units, things will inevitably come crashing
down at link time. Raise your hand if you've ever been hit with a linker error
that looks something like this:

```bash
/lib/gcc/arm-none-eabi/lib/libc.a(lib_a-writer.o): In function `_write_r':
    writer.c:(.text._write_r+0x20): undefined reference to `_write'
/lib/gcc/arm-none-eabi/lib/libc.a(lib_a-readr.o): In function `_read_r':
    readr.c:(.text._read_r+0x20): undefined reference to `_read'
/lib/gcc/arm-none-eabi/lib/libc.a(lib_a-exit.o): In function `exit':
    exit.c:(.text.exit+0x2c): undefined reference to `_exit'
/lib/gcc/arm-none-eabi/lib/libc.a(lib_a-isattyr.o): In function `_isatty_r':
    isattyr.c:(.text._isatty_r+0x18): undefined reference to `_isatty'
/lib/gcc/arm-none-eabi/lib/libc.a(lib_a-sbrkr.o): In function `_sbrk_r':
    sbrkr.c:(.text._sbrk_r+0x18): undefined reference to `_sbrk'
gcc: ld returned 1 exit status
```

Oops! Turns out those `<stdio.h>` methods rely on the existence of some
underlying syscalls (well, syscall wrappers, if we're being pedantic), and
because this is bare-metal code, they're nowhere to be found!

Now, this certainly isn't the end of the world. It's entirely possible to
appease the linker by stubbing out
[these syscalls](https://wiki.osdev.org/Porting_Newlib#newlib.2Flibc.2Fsys.2Fmyos.2Fsyscalls.c)
with some sort of panicking implementation, and working backwards from a crash
to determine which standard library method can't be used. Heck, if you're lucky,
the toolchain might even include some
[built-in](https://stackoverflow.com/questions/55752998/adding-options-lc-lstdc-or-specs-nosys-specs-allows-arm-none-eabi-gcc-v4-9)
syscall stubs which you can use!

In this particular case, it's very obvious why the `<stdio.h>` functions
wouldn't work when writing bare-metal software, and it's easy to swap them out
with a different, hand-rolled implementation. Unfortunately, things aren't
always so simple.

For example, consider this (admittedly contrived) `greet` function that only
calls `malloc` if the input buffer is too small to fit the resulting string:
[^c-contrived-example]

```c
#include <string.h>
#include <malloc.h> // Uh oh!

#include "myprintf.h" // e.g: a custom printf that writes bytes over a serial port

char* greet(char* name, size_t buf_len) {
    const char* greeting = "Hello ";
    const size_t greeting_len = strlen(greeting);
    const size_t name_len = strlen(name);

    const size_t final_size = name_len + greeting_len + 1;

    char* new_name;

    if (buf_len > final_size) {
        new_name = name;
    } else {
        new_name = malloc(final_size); // This won't work!
    }

    memmove(new_name + greeting_len, name, name_len);
    memcpy(new_name, greeting, greeting_len);
    new_name[final_size - 1] = '\0';

    return new_name;
}

int main() {
    char large_name_buf [128] = "Jimothy";
    char small_name_buf [12] = "Terrance";

    char* greet_1 = greet(large_name_buf, 128);
    char* greet_2 = greet(small_name_buf, 12);

    myprintf("%p\n", large_name_buf);       // 0x7ffcd8854e90
    myprintf("%p: %s\n", greet_1, greet_1); // 0x7ffcd8854e90: Hello Jimothy

    myprintf("%p\n", small_name_buf);       // 0x7fffbe5ef094
    myprintf("%p: %s\n", greet_2, greet_2); // ERROR: calls `malloc`!

    return 0;
}
```

It's entirely possible that the code will work 99% of the time, but then
occasionally fail in the rare case when the buffer is too small, and `malloc`
ends up getting called. `malloc` relies on the `sbrk` syscall, which in this
case, would be stubbed out. What happens next is totally implementation
dependent. The program might crash with an `unimplemented syscall` error, or
`malloc` might end up returning a null pointer, which this code (naively)
doesn't check for (after all, when could `malloc` possibly fail /s).

One classic workaround for these sorts of footguns is to forgo the C standard
library entirely, and rewrite every method from scratch. After all, how hard can
it be to
[implement `memcpy`](https://stackoverflow.com/questions/17591624/understanding-the-source-code-of-memcpy)
anyway? And sure, this approach might work fine for the simpler methods, but are
you really going to want to re-implement something like `sprintf()` yourself?
Probably not.

Things get even hairier when using external C libraries. While many C libraries
proudly market themselves as "dependency-free", quite often they implicitly mean
that they are _external_ dependency-free, and still rely on the C standard
library internally. That's not to say that there aren't any truly dependency
free C libraries as well, there most certainly are, but for a beginner systems
programmer, the distinction between external dependency free and _truly_
dependency free might not be as obvious as they expect.

And so, it's often the case the C programmers are left with one of two options:
use the standard library, but be _very careful_ not to use any methods with
hidden syscall dependencies, or forgo the standard library entirely, and
re-write any bits of functionality they require themselves.

If only the C standard library had a clear "line" between all the
platform-agnostic, dependency free bits of code, and all the platform-specific,
"requires an OS" bits of code...

# Rust is _Truly_ Dependency Free

Like any great systems programming language, it doesn't take much of anything to
get Rust up and running on a new platform.

In fact, a minimal Rust "runtime" (cheekily called
[`r0`](https://github.com/rust-embedded/r0)) typically consists of a short
prelude that zeroes out the .bss section and hands off execution to `main`.

That's it\![^rs-panic-handler]

Just like C, this extremely minimal approach to language design allows Rust to
run on just about any hardware platform (well, assuming there's
[compiler support](https://doc.rust-lang.org/nightly/rustc/platform-support.html)
for it).

Where Rust differs from C is with respect to its standard library. Learning from
the mistakes of C, the Rust language team made the excellent decision to split
the standard library into two parts:

-   [`std`](https://doc.rust-lang.org/std/index.html): The full-fledged Rust
    standard library, "[offering] core types, like `Vec<T>` and `Option<T>`,
    library-defined operations on language primitives, standard macros, I/O and
    multithreading, among many other things."
-   [`core`](https://doc.rust-lang.org/core/): The small _dependency-free_
    foundation of Rust language. `core` "isn't even aware of heap allocation,
    nor does it provide concurrency or I/O. These things require platform
    integration, and [the `core`] library is platform-agnostic."

This seemingly innocuous implementation detail turned out to be absolutely
_incredible_ for bare-metal programmers, as it made it possible to entirely
"opt-out" of all the OS-dependent bits of the standard library. By relying on
the `core` library directly, it's possible to write truly portable bare-metal
code free of any and all "hidden" syscalls!

Unlike C, which allows separate method declarations and implementations (i.e:
declaring a method in a `.h` header file, and implementing it in a `.c` file),
Rust requires that methods are implemented at the same time as they are
declared, making it _impossible_ to get a linker error when writing typical Rust
code. [^rs-link]

So, how does a crate (Rust lingo for library/binary) opt out of the `std`
standard library? Why, using the handy-dandy top-level `#![no_std]` attribute!

As the name implies, the `#![no_std]` attribute signals to the Rust compiler
that the crate shouldn't link with `std`, and that it relies on the `core`
library directly.

`#![no_std]` is also transitive down the dependency graph, so accidentally
including a dependency that relies on `std` within a `no_std` will be rejected
by the compiler. This transitive properly makes it possible to use `cargo`
(Rust's built-in package manager) to integrate external dependencies from
`crate.io`, and be confident that if it compiles, it won't inadvertently panic
at runtime due to a stubbed out "hidden" syscall dependency. With Rust, using
external dependencies with bare-metal code is not only possible, but thanks to
`cargo`'s best-in-class dependency management story, it's incredibly ergonomic
as well!

To really show you the power of `no_std`, lets revisit the `greet` function from
the last section. Consider the following Rust implementation, which goes out of
it's way to be as `unsafe` and un-idiomatic as possible. This could very well be
a "first pass" implementation from a C programmer just getting their feet wet
with Rust:

```rust
// #![no_std] // Un-comment this and see what happens...

use std::alloc::{alloc, Layout};

unsafe fn greet(name: *mut u8, buf_len: usize) -> *mut u8 {
    const GREETING: &'static [u8] = b"Hello ";

    let name_len = strlen(name);
    let final_size = name_len + GREETING.len() + 1;

    let new_name;
    if buf_len > final_size {
        new_name = name;
    } else {
        new_name = alloc(Layout::from_size_align(final_size, 1).unwrap());
    }

    new_name.add(GREETING.len()).copy_from(name, name_len);
    new_name.copy_from_nonoverlapping(GREETING.as_ptr(), GREETING.len());
    *new_name.add(final_size - 1) = b'\0';

    new_name
}
```

(You can play around with this example online using
[this](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ed67db50e68a99e0594e472d87f078ab)
Rust playground link)

Without the `#![no_std]` attribute, this example works just fine. Un-comment the
`#![no_std]` attribute, and all of a sudden, the code won't even compile!

```bash
error[E0433]: failed to resolve: use of undeclared type or module `std`
 --> src/lib.rs:3:5
  |
3 | use std::alloc::{alloc, Layout};
  |     ^^^ use of undeclared type or module `std`
```

Look at that!

Even this gnarly, `unsafe`, un-idiomatic rewrite of the raw C function benefited
from Rust's `no_std` guarantees, which made it _literally impossible_ to
accidentally use the `alloc` function (Rust's equivalent to `malloc`) in a
bare-metal environment\![^alloc]

---

In conclusion, Rust's `no_std` feature makes it an excellent language for
beginners looking to get started with bare-metal programming.

Being able to quickly get up and running on a new platform without having to
implement and/or stub-out a bunch of syscalls is awesome, and being able to
write/publish/consume portable, bare-metal compatible libraries is absolutely
_game-changing_ for the traditionally hermetic world of embedded
development.[^audit-deps]

If you're interested to learn more about using Rust on bare-metal, I'd recommend
checking out the following resources:

-   Philipp Oppermann's excellent series on writing an OS in Rust, particularly
    his post on setting up a
    [Freestanding Rust Binary](https://os.phil-opp.com/freestanding-rust-binary/).
-   Cliff L. Biffle's post on rewriting his C++ project
    [m4vgalib in Rust](http://cliffle.com/blog/m4vga-in-rust/).
-   [The Embedded Rust Book](https://rust-embedded.github.io/book/), which is
    oriented towards embedded systems, such as Microcontrollers.
-   [awesome-embedded-rust](https://github.com/rust-embedded/awesome-embedded-rust)
-   The official Rust [`core`](https://doc.rust-lang.org/core/) docs.

Thanks for reading!

[^c-contrived-example]:
    Yes, this is a contrived example, and would be caught quickly during code
    review by any experienced C programmer. Unfortunately, like many things in
    C, this sort of mistake can easily spiral into a few hours of stressful
    debugging for the inexperienced programmer.

[^rs-panic-handler]:
    Well, technically Rust also requires a
    [`#[panic_handler]`](https://doc.rust-lang.org/nomicon/panic-handler.html)
    routine, which gets called whenever a fatal error occurs (e.g: an assertion
    is broken, a checked array access operation fails, etc...).

[^rs-link]:
    Emphasis on the word _typical_, which in this case, implies writing Rust
    using the `cargo` package manager, and directly linking to other Rust
    libraries. Rust has a very strong FFI story, and can natively link with
    [`extern`](https://doc.rust-lang.org/std/keyword.extern.html) types and
    functions, but doing so will obviously open the door to all sorts of
    fun-to-debug linking errors.

[^alloc]:
    Okay, that's not _exactly_ true, since the `alloc` function is part of the
    [`alloc`](https://doc.rust-lang.org/alloc/index.html) crate, which is a
    built-in `no_std` library that is re-exported as part of `std`. In fact,
    this particular example could be made to work with `no_std` by linking with
    the `alloc` crate and providing a
    [`#[global_allocator]`](https://doc.rust-lang.org/std/alloc/index.html#the-global_allocator-attribute)
    implementation. That's quite a rabbit hole though, and this post is long
    enough as-is.

[^audit-deps]: Just make sure you audit your dependencies!
