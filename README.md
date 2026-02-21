# Fuzzing Neovim

I came across [this Neovim bug
report](https://github.com/neovim/neovim/issues/27196) describing an odd
assertion failure that had been encountered by various people but lacked clear
reproduction steps, and was therefore difficult to make progress on. Eventually
I started to wonder whether [fuzzing](https://en.wikipedia.org/wiki/Fuzzing),
and specifically coverage-guided fuzzing could help with something like this.

I had never worked with fuzzing before, and I was interested to try it out.
Fuzzing is usually used in a security research context, but I didn't see why it
couldn't work here. Fuzzing is also usually used to initially discover a bug,
rather than finding a way to reproduce a known bug, but if anything knowing that
the bug exists should make it easier.

This repository describes the steps I took to find a consistent reproduction for
the assertion failure by fuzzing Neovim with [AFL++](https://aflplus.plus/). Be
warned, this was my first time using fuzzing and there's a lot here that I don't
understand.
