# Part 4: Running afl-fuzz

We're almost ready to start fuzzing, but one thing that we need first is seeds.
Ideally seeds should be valid inputs to the program being fuzzed. Our particular
program accepts any random byte sequence as valid, so we can get away with a
lazy approach and just use random seeds. It might be useful to formulate some
meaningful inputs to use as seeds, but I just used the lazy approach:

```sh
# mkdir -p fuzz-1/seeds

# for i in `seq 4` ; do dd if=/dev/urandom of=fuzz-1/seeds/seed-$i bs=1k count=1 ; done
```

Now we can start fuzzing:

```sh
# afl-fuzz -i fuzz-1/seeds -o fuzz-1/out -m none -- /neovim/build/bin/nvim -n -u NONE -i NONE --headless --cmd "source extmark-fuzz.lua"
```

The AFL++ [status screen](https://aflplus.plus/docs/status_screen/) should pop
up. Items are color-coded, and red means something might need attention. And we
have some red right away. In the "item geometry" box, "stability" is listed as
something around 77%.

Stability seems to be a measure of how consistently the program behaves. A low
stability suggests that the path through the program depends on something other
than the input. So that's not great, and I'm not sure what causes it in this
case. Poor stability means it will be hard to distinguish meaningful from
non-meaningful input changes. That might make the fuzzing slower, and might make
it hard to determine what crashes out of a set of crashes are unique.

## Processing Results

Eventually some crashes were collected. This took around an hour.

At this point, some sources suggest using `afl-cmin` to eliminate crashes that
are redundant. I'm not sure if this is valuable, and the main AFL++
documentation seems to suggest that `afl-cmin` is used on the input files
*before* using them as seeds for fuzzing. In any case, I haven't had much luck
with `alf-cmin` on the crashing inputs. It always eliminates all the files, as
if none of them produced a crash. I'm not sure I would expect this to be useful
anyway, since I think `afl-fuzz` should already have eliminated crashes that
took the same path through the program.

## Minimizing Test Cases

The next step is to minimize the crashing inputs with `afl-tmin`.

```sh
# mkdir fuzz-1/minimized

# cd fuzz-1/out/default/crashes/

# for f in id* ; do afl-tmin -i "$f" -o "/neovim/fuzz-1/minimized/$f" -- /neovim/build/bin/nvim -n -u NONE -i NONE --headless --cmd "source /neovim/extmark-fuzz.lua" ; done
```

Note the use of full paths in this case. I'm running this from the crashes
directory to keep the shell variable substitutions simple, so the paths have to
be adjusted.

This will chug through each input trying to reduce its size without affecting
the result. It can take awhile. Here are the file sizes before and after in my
case:

```
# wc -c fuzz-1/out/default/crashes/id*
 1638 fuzz-1/out/default/crashes/id:000000,sig:06,src:001948,time:9694759,execs:1945032,op:havoc,rep:12
 1604 fuzz-1/out/default/crashes/id:000001,sig:06,src:001948,time:9696014,execs:1945288,op:havoc,rep:12
 1502 fuzz-1/out/default/crashes/id:000002,sig:06,src:001948,time:9698375,execs:1945770,op:havoc,rep:9
 1495 fuzz-1/out/default/crashes/id:000003,sig:06,src:001948,time:9730280,execs:1951954,op:havoc,rep:4
 1494 fuzz-1/out/default/crashes/id:000004,sig:06,src:001948,time:9741525,execs:1954241,op:havoc,rep:9
 1631 fuzz-1/out/default/crashes/id:000005,sig:06,src:001973,time:9820118,execs:1969943,op:havoc,rep:3
 1578 fuzz-1/out/default/crashes/id:000006,sig:06,src:001973,time:9840285,execs:1974022,op:havoc,rep:10
 1520 fuzz-1/out/default/crashes/id:000007,sig:06,src:001973,time:9872830,execs:1980575,op:havoc,rep:14
 1626 fuzz-1/out/default/crashes/id:000008,sig:06,src:001973,time:9873143,execs:1980638,op:havoc,rep:15
14088 total

# wc -c fuzz-1/minimized/id*          
 362 fuzz-1/minimized/id:000000,sig:06,src:001948,time:9694759,execs:1945032,op:havoc,rep:12
 414 fuzz-1/minimized/id:000001,sig:06,src:001948,time:9696014,execs:1945288,op:havoc,rep:12
 402 fuzz-1/minimized/id:000002,sig:06,src:001948,time:9698375,execs:1945770,op:havoc,rep:9
 315 fuzz-1/minimized/id:000003,sig:06,src:001948,time:9730280,execs:1951954,op:havoc,rep:4
 414 fuzz-1/minimized/id:000004,sig:06,src:001948,time:9741525,execs:1954241,op:havoc,rep:9
 461 fuzz-1/minimized/id:000005,sig:06,src:001973,time:9820118,execs:1969943,op:havoc,rep:3
 313 fuzz-1/minimized/id:000006,sig:06,src:001973,time:9840285,execs:1974022,op:havoc,rep:10
 388 fuzz-1/minimized/id:000007,sig:06,src:001973,time:9872830,execs:1980575,op:havoc,rep:14
 416 fuzz-1/minimized/id:000008,sig:06,src:001973,time:9873143,execs:1980638,op:havoc,rep:15
3485 total
```

## Verifying an Input

To verify that these inputs crash nvim, we can run it manually and feed in the
file:

```
# ./build/bin/nvim -n -u NONE -i NONE --headless --cmd "source extmark-fuzz.lua" < fuzz-1/minimized/id\:000000\,sig\:06\,src\:001948\,time\:9694759\,execs\:1945032\,op\:havoc\,rep\:12 
nvim: /neovim/src/nvim/marktree.c:378: void unintersect_node(MarkTree *, MTNode *, uint64_t, _Bool): Assertion `seen' failed.
Aborted
```

At this point we could do other useful things, like run in a debugger and
collect the stacktrace, inspect values, step through the code up to the point
that it crashes, etc.
