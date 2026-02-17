# Fuzzing Neovim

I came across [this Neovim bug
report](https://github.com/neovim/neovim/issues/27196) describing an odd
assertion failure that had been encountered by various people but lacked clear
reproduction steps, and was therefore difficult to make progress on. Eventually
I started to wonder if [fuzzing](https://en.wikipedia.org/wiki/Fuzzing), and
specifically coverage-guided fuzzing could help with something like this.

I had never worked with fuzzing before, and I was interested to try it out.
Fuzzing is usually used in a security research context, but I didn't see why it
couldn't work here. Fuzzing is also usually used to initially discover a bug,
rather than finding a way to reproduce a known bug, but when fuzzing finds an
interesting result it generally saves the input that triggered the result.
Therefore, if it "finds" the assertion failure, the input that caused the
assertion failure will be saved for later use.

This repository describes the steps I took to find a consistent reproduction for
the assertion failure by fuzzing Neovim with [AFL++](https://aflplus.plus/). Be
warned, this was my first time using fuzzing, I'm nowhere near an expert, and
there's a lot here that I don't understand.
