# Part 2: First Fuzzing Attempt, and Reconsidering the Approach

Now that we have the instrumented nvim binary, we should be able to try fuzzing
it. But it's not very clear how to go about doing this, and the more I
considered it the more problems became apparent.

Fuzzing with AFL++ usually expects the program being fuzzed to accept input on
stdin. Neovim typically accepts input from a TTY. If stdin isn't a TTY, Neovim
takes the data on stdin as file content to load into a buffer, not as commands
to execute. There may be some combination of command line arguments to make this
work, but it's only the first of many problems.

Neovim also generally takes over the terminal screen when it runs, but AFL++
does the same, and uses it to display its status report. I don't know exactly
what would happen here, but it doesn't seem like it would be good. Again, this
can probably be fixed with options like `--headless`, but there are more
problems to consider.

Neovim has a lot of capabilities. Running completely arbitrary commands through
it seems both ill-advised and unlikely to reach the specific assertion failure
in a reasonable amount of time. While being fuzzed it could write or overwrite
arbitrary files, create and run arbitrary scripts, run arbitrary system
commands, etc. This would need some kind of sandboxing to reign it in, and
getting meaningful results would probably take a very long time.

Also, Neovim would need to be told to exit at the end of each fuzzing input,
otherwise AFL++ would see it as a hung process and record that as a finding
(which would be rather meaningless).

I had not thought this through sufficiently, and needed to reconsider the
approach.

For the record, I did *try* a fuzzing attempt in spite of these problems (before
all of them had occurred to me), but `afl-fuzz` gave up immediately because nvim
didn't exit on the first input. AFL++ really wants valid inputs for its initial
seeds, so if they crash or hang it won't continue. In hindsight it's probably a
good thing it stopped.

I thought about modifying the nvim code to prevent file writing. As for running
scripts and system commands, it would be running inside the AFL++ container so
the potential damage would be limited. But this only begins to address the
problems noted, and it's not a simple change to make in the code.

Another thought I had was to take the relevant source file, `marktree.c`, and
build a small test program around it. Maybe that would work, but it would
require significant understanding of that code and the results might not be that
helpful. E.g., if there's some way you can use marktree that produces the
assertion failure, but nvim itself doesn't appear to use it in that way, then
what progress have we made?

After some consideration, the idea I came up with was to write a script that
runs in nvim, reads from stdin, executes nvim API calls based on the input, and
exits once the input is exhausted. It could be restricted to a few specific API
calls, and it could interpret input however I wanted. The next part will go over
this approach.
