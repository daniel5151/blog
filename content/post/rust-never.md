+++
title = "Functions that never return should return `never`"
draft = true
tags = ["os", "no_std", "rust", "c", "c++"]
+++

<!--more-->

I've been writing a small microkernel in Rust, based off some of the
[coursework](https://student.cs.uwaterloo.ca/~cs452/W20/) from my last semester
of university.

// come up with a good intro

https://doc.rust-lang.org/stable/rust-by-example/fn/diverging.html

Green threads in Rust:

https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/

Other languages:

https://nshipster.com/never/

https://basarat.gitbook.io/typescript/type-system/never

Can't noreturn through function pointers

https://stackoverflow.com/questions/28739082/how-to-use-noreturn-with-function-pointer

Not a type (yet)

https://github.com/rust-lang/rust/issues/35121

---

The kernel's C API specifies the following interface for creating and destroying
tasks:

```c
// file: <kernel/tasks.h>
int task_spawn(void (*function)());
_Noreturn void task_exit()
```

As the names and signatures would imply, `task_spawn` will run a function as a
task, and `task_exit` will destroy the task it was called from.

But, uh, what's up with that `_Noreturn` attribute on `task_exit`?

Introduced as part of the C11 standard: "The
[`_Noreturn`](https://en.cppreference.com/w/c/language/_Noreturn) keyword
appears in a function declaration and specifies that the function does not
return by executing the return statement or by reaching the end of the function
body". Similarly, the C++11 standard also specified the
[`[[noreturn]]`](https://en.cppreference.com/w/cpp/language/attributes/noreturn)
attribute, which functions identically to its C counterpart.

> Note: Prior to being standardized in C11/C++11, compilers typically supported
> their own custom `noreturn` attributes. e.g: gcc and clang supported
> `__attribute__((noreturn))`, MSVC used `__declspec(noreturn)`, etc...

The `noreturn` attribute is a useful way to give the compiler additional insight
into an application's execution flow, and unlocks some useful optimizations. For
example, any code following a call to a `noreturn` function is likely to get
dead-code-eliminated by the compiler, as it is able to statically determine that
it will never be run. For example:

```c
#include <stdio.h>
#include <stdlib.h>

_Noreturn void my_exit() {
    printf("Goodbye, world!");
    exit(1);
}

int main() {
    printf("Hello, world!");
    my_exit();
    printf("Hello again!"); // never gets run
    return 0;
}
```

```
$ gcc -o noreturn_test noreturn_test.c -Wall
$ strings noreturn_test | grep "Hello"
Hello, world!
<but not "Hello again!">
```

Searching the binary for `"Hello"` only brings up one result - the `"Hello"`
from `"Hello World"`. Looks like the compiler was smart enough to entirely
optimize out the call to `printf("Hello again!")`. Neato!

It should be noted that like many C and C++ constructs, the `noreturn` attribute
must be used with care and caution, as inadvertently returning from an
`noreturn` function _will_ cause undefined behavior. For example, while the
following code compiles just fine (with a warning of course), trying to run it
will most likely end in disaster:

```c
#include <stdio.h>

_Noreturn void my_exit() {
    // oh no, this code doesn't actually call exit!
    return;
}

int main() {
    printf("Hello, world!");
    my_exit();
    printf("Hello again!");
    return 0;
}
```

```
$ gcc --version
gcc (Ubuntu 10.2.0-13ubuntu1) 10.2.0

$ gcc -o noreturn noreturn.c -Wall
noreturn.c: In function ‘fake_exit’:
noreturn.c:5:5: warning: function declared ‘noreturn’ has a ‘return’ statement
    5 |     return;
      |     ^~~~~~
noreturn.c:5:5: warning: ‘noreturn’ function does return
    5 |     return;
      |     ^

$ ./noreturn
Hello, world!
```

Huh. That's weird...

It seems to be working as intended?

Okay... what happens when we crank up the optimization level a bit?

```
$ gcc -o noreturn noreturn.c -Wall -O2 # <-- optimized
noreturn.c: In function ‘fake_exit’:
noreturn.c:5:5: warning: function declared ‘noreturn’ has a ‘return’ statement
    5 |     return;
      |     ^~~~~~
noreturn.c:5:5: warning: ‘noreturn’ function does return
    5 |     return;
      |     ^

$ ./noreturn
<snip>
Hello, world!Hello, world!Hello, world!Hello, world!Hello, world!Hello,
world!Hello, world!Hello, world!Hello, world!Hello, world!Hello, world!Hello,
world!Hello, world!Hello, world!Hello, world!Hello, world!Hello, world!Hello,
world!Hello, world!Hello, world!Hello, world!Hello, world!Hello, world!Hello,
wor[1]    79709 segmentation fault (core dumped)  ./noreturn
```

Ah, yep, that's more like it. Why does it repeatedly print out `"Hello, world!"`
and then segfault? Your guess is as good as mine!

---

```c
// run using `task_spawn(awesome_task);`
void awesome_task() {
    printf("Hello, and goodbye!");
    task_exit();
}
```

Bad things happen if `task_exit` isn't called:

https://stackoverflow.com/questions/49674026/what-happens-if-there-is-no-exit-system-call-in-an-assembly-program

```c
// run using `task_spawn(buggy_task);`
void buggy_task() {
    printf("Calling `task_exit()` is for losers!");
    // task_exit()
}
```

Calling the function from Rust

First pass:

```rust
fn task_spawn(function: extern "C" fn()) -> isize;
```

```rust
// run using `task_spawn(awesome_task);`
fn awesome_task() {
    println!("Hello, and goodbye!");
    task_exit();
}
```

```rust
// run using `task_spawn(buggy_task);`
fn buggy_task() {
    println!("Calling `task_exit()` is _still_ for losers!");
    // task_exit();
}
```

Rust's secret weapon: the `never` (written `!`) type!

Rust's counterpart to the C/C++ `noreturn` attribute is the
[`never`](https://doc.rust-lang.org/std/primitive.never.html) type (written as
`!`), which represents a computation that never resolves to a value. Since it's
part of the type system, `!` doesn't suffer the same limitations as the
`noreturn` attribute in C/C++, and can be used to enforce non-returning
computation through function pointers, and catch errors at compile time!

```rust
fn task_spawn(function: extern "C" fn() -> !) -> isize;
//                                         ^---- returns `never`
```

```rust
// run using `task_spawn(awesome_task);`
fn awesome_task() -> ! {
    println!("Hello, and goodbye!");
    task_exit();
}
```

```rust
// run using `task_spawn(buggy_task);`
fn buggy_task() -> ! {
    println!("Calling `task_exit()` is _still_ for losers!");
    // task_exit();
}
```

<!-- Shoutout to https://github.com/theZiz/aha -->
<pre><code><span style="color:#a6e22e;">   Compiling</span> example v0.1.0
<span style=" color:#f92672;">error[E0308]</span><span>: mismatched types</span>
  <span style=" color:#66d9ef;">--&gt; </span>src/main.rs:1:23
   <span style=" color:#66d9ef;">|</span>
<span style=" color:#66d9ef;"> 1</span> <span style=" color:#66d9ef;">| </span>fn buggy_task() -&gt; ! {
   <span style=" color:#66d9ef;">| </span>   <span style=" color:#66d9ef;">----------</span>      <span style=" color:#f92672;">^</span> <span style=" color:#f92672;">expected `!`, found `()`</span>
   <span style=" color:#66d9ef;">| </span>   <span style=" color:#66d9ef;">|</span>
   <span style=" color:#66d9ef;">| </span>   <span style=" color:#66d9ef;">implicitly returns `()` as its body has no tail or `return` expression</span>
   <span style=" color:#66d9ef;">|</span>
   <span style=" color:#66d9ef;">= </span><span>note</span>:   expected type `<span>!</span>`
           found unit type `<span>()</span>`

<span style=" color:#f92672;">error</span><span>: aborting due to previous error</span>

<span>For more information about this error, try `rustc --explain E0308`.</span>
<span style="color:#f92672;">error</span><span>:</span> could not compile `example`.

To learn more, run the command again with --verbose.
</code></pre>

A ha!

Several common Rust constructs return `never`:

Panicking code:

```rust
fn panicking_task() -> ! {
    panic!("Error: You must construct additional pylons.");
}
```

Infinite loops `loop {}`

```rust
fn event_loop_task() -> ! {
    loop {
        match get_key() {
            Key::Up => handle_up(),
            Key::Down => handle_down(),
            ...
        }
    }
}
```
