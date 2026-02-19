# Part 6: Improvements

## Partial Instrumentation and Higher Stability

Given the narrow scope of the fuzzing we're doing (just a handful of API calls),
we don't need to instrument the entire nvim binary. To instrument just a few
files, add these lines to a new file, `allowlist.txt`:

```
src/nvim/marktree.c
src/nvim/extmark.c
src/nvim/api/buffer.c
src/nvim/api/extmark.c
```

It's possible to instrument specific functions also, but I'm not bothering with
that here.

Now rebuild nvim with an updated command:

```sh
# make clean

# AFL_LLVM_ALLOWLIST=/neovim/allowlist.txt CC=/AFLplusplus/afl-clang-lto CXX=/AFLplusplus/afl-clang-lto++ make CMAKE_BUILD_TYPE=Debug CMAKE_EXTRA_FLAGS="-DSTATIC_BUILD=1"
```

Running the fuzzing campaign again with the new binary gives a stability of
100%, showing that the inconsistent parts of the execution were outside of the
areas we care about. The better stability might make the fuzzing process faster,
and I would expect could give better results.

## Minimization

After finding a repro for the assertion failure, the most important element of
this effort is to minimize the repro to make it more useful. I've found this to
be the most tricky part. Minimizing with `afl-tmin` has worked reasonably well
to shrink the size of the input files, but the end results when converted to a
Lua script are not quite what I would hope for. Some cases add and remove text
multiple times, which seems spurious. Some cases add extmarks that prove to be
unnecessary, or extmark options that don't affect the result. Some cases clear
the buffer namespace, which appears to be unnecessary.

I think there are some ways that this could be improved. For a start, we
probably don't need the fuzzing to be able to add and remove text, because it
seems that all or most test cases add text at the beginning and remove text at
the end. We could make that a fixed part of the fuzzing script. Clearing the
namespace could be removed entirely. These are things that we didn't *know* we
didn't need until we had some test cases, but if we want to keep looking for a
minimal test case we could make these changes and try again.

The next thing I would consider is whether we can update our binary format to
make the job of `afl-tmin` easier. We have extmark flags packed into a byte of
the different extmark adding operations. It would be convenient to use fewer
flags, because unnecessary opts clutter the API calls. This is a kind of
minimization that `afl-tmin` can't do, because it doesn't know that a particular
bit being 0 is "smaller". If I were starting over, I'd consider things like this
when defining the binary format. Maybe the different operations could be
variable-width, more closely reflecting how an actual API call looks.

One of the limitations of this approach is that updating the binary format is
kind of undesirable once you've found some useful inputs. If I started updating
the format expected by `extmark-fuzz.lua` then the pile of crashing inputs that
I had collected up to that point would be useless with the new version. I expect
I would quickly lose track of what inputs went with what version of the script.

## Questions

Here are some questions that I'm left with.

Why was stability low when the entire binary was instrumented? I don't know
enough about the internals of Neovim to make a good guess here. I'm sure there's
some asynchronous parts, so maybe that's the answer?

Was it necessary to build a static binary, particularly after switching to
partial instrumentation? I suspect as long as the parts you want instrumented
are statically linked then dynamic linking of other parts should be fine.

Could one of the other AFL++ compilers have been better for our purposes? The
docs generally suggest LTO if possible, and it seemed to mostly work fine.
