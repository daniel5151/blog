+++
title = "CS 452 OS in Rust"
description = "todo"
draft = true
tags = ["os", "no_std", "rust", "c", "c++"]
+++

<!--more-->

Back in
[the before times](https://www.urbandictionary.com/define.php?term=The%20Before%20Time&defid=15158728)
(read: early 2020), I was wrapping up my final semester as an undergraduate
student, taking the [in]famous
[CS 452 - Real-time Programming](https://student.cs.uwaterloo.ca/~cs452/W20/)
course at the University of Waterloo. To give a quick summary of what the course
is all about: students work in groups of two, spending a semester
[~~isolated from society~~](https://www.reddit.com/r/uwaterloo/comments/bn6sd2/is_cs_452_real_time_programmingtrains_and_anti/en2uwya/)
writing an ARM-based real-time microkernel entirely from scratch, and using it
to control a [model train set](https://www.youtube.com/watch?v=4COS47Ox-FM). Fun
fact, the QNX operating system[^1] was written by two UW students after taking
CS 452!

Historically, the course required that the kernel and userland had to be written
in plain C (with a light dusting of ARM assembly). Thanks to a much overdue
toolchain update, our class was was also given the option to use C++.

Now, I've done my fair share of C and C++ over the years, but after discovering
Rust, I knew that I'd never want to work with C++ (and to a lesser extent, plain
C) ever again.

And so, when it came time to decide between option A (writing the OS in plain C)
and option B (writing the OS in C++), I thought I'd go for secret option C,
which was to write the OS in Rust!

Unfortunately, while my enthusiasm for Rust undeniable (heck, I'd even slapped
together a POC implementation of the first assignment that ran on actual
hardware), the instructors didn't grant me permission to write the OS in Rust,
citing concerns that I would be entering uncharted territory, and that they
wouldn't be able to help out if I got stuck (which in hindsight were perfectly
reasonable - the course is hard enough as is!).

And that's where the story ended for a while.

[My partner](https://jameshageman.com/) and I ended up writing our OS in C++
(which in hindsight was a _terrible_ idea, as the kernel was simple enough to be
written in C, and using C++ introduced a bunch of unnecessary complexity - but
that's a story for another time), and barreled forwards in the course.

I'd initially wanted to continue working on the Rust kernel alongside the C++
kernel, but the semester proved to be a difficult one, and I left my POC
implementation by the wayside...

---

Fast forward a few months, and I suddenly found myself with a lot of free time
on my hands. Instead of traveling the world with my friends post-graduation, I
ended up moving back into my childhood bedroom and settling in for [what we all
though would only be] a few months of quarantine.

In between the TV, video-games, and mass hysteria, I did find some time to sit
down and work on some passion projects:

-   Polishing up [`ts7200`](https://github.com/daniel5151/ts7200), an emulator
    my [friend and fellow classmate](https://github.com/iburinoc/gba-rs) and I
    wrote for the TS-7200 single-board-computer that was used in CS 452.
-   [`clicky`](https://github.com/daniel5151/clicky), an experimental emulator
    for classic clickwheel iPods
-   [`gdbstub`](https://github.com/daniel5151/gdbstub), an ergonomic and
    easy-to-integrate implementation of the GDB Remote Serial Protocol.
    -   Cool side-note, `gdbstub` was recently integrated into
        [`crosvm`](https://chromium-review.googlesource.com/c/chromiumos/platform/crosvm/+/2440221),
        which is super cool, since I worked on `crosvm` as an intern at Google!

[^1]: https://en.wikipedia.org/wiki/QNX#History
