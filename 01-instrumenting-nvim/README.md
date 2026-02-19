# Part 1: The Problem, and Instrumenting nvim

[The issue](https://github.com/neovim/neovim/issues/27196) in question is an
assertion failure in Neovim's `marktree.c` code. The code is not very easy to
understand, but I gather it's a B-tree implementation used for implementing
**extmarks**. Extmarks are locations in the buffer that adjust as text is
changed. The assertion that fails is "seen", which is a boolean value that gets
set when a loop finds the thing it is looking for. The particular function has a
`strict` boolean argument that seems to mean "the item is expected to be
present." So when `strict` is true, the loop needs to find the item it's looking
for and set `seen`. Otherwise, some invariant has been violated.

While a number of people reported encountering this assertion failure, there was
very little known about how to reproduce it. It occurred randomly while people
were working in Neovim.

Issues like this can be a nightmare for software developers. It's rare enough
that developers don't see it, the resulting error messages or stack traces
clearly show that a problem exists but don't tell you where or when the problem
occurred, and it happens deep into a user session without a clear action that
triggers it. It's possible that the tree structure got corrupted twenty minutes
before the assertion finally detected it. Developers don't have many useful
avenues of investigation: review the code looking for issues that have been
missed, add additional diagnostic gathering to the code and hope something new
is revealed the next time the assertion fails, or just hope that some user
stumbles across a reliable way to reproduce the issue.

Rather than waiting for users to randomly encounter the issue, maybe we can
substitute a software fuzzer, essentially doing the same thing but much faster
and with a recorded history that will make the input reproducible.

Initially I thought that I might be able to feed the fuzzer input into nvim as
if the input were keyboard key presses. I'll get into the various problems with
this later (some of them might be obvious already).

## Fuzzing Basics

I used this [fuzzing tutorial](https://github.com/alex-maleno/Fuzzing-Module) to
get started. A lot of what I do later will be based on this.

## Getting AFL++

I opted to use the [Docker
image](https://hub.docker.com/r/aflplusplus/aflplusplus) of AFL++. I generally
use containers under Podman rather than Docker, but I don't think there are any
differences that are significant for our purposes here. If you are following
along you may have to make minor adjustments.

Start the container with something like this:

```sh
$ podman run -it -v ./neovim:/neovim --name fuzz-neovim docker.io/aflplusplus/aflplusplus
```

In this case I have the neovim git repository cloned to `neovim` in the current
directory, so it will be available as `/neovim` in the container.

## Preparing to Build Neovim

First, add the dependencies from the Neovim `BUILD.md` files. Since the AFL++
container uses Ubuntu, follow the instructions for Ubuntu (though there's no
need to use `sudo` since we're already root inside the container):

```sh
# apt update

# apt install ninja-build gettext cmake curl build-essential git
```

It's strongly recommended to build a statically-linked executable for fuzzing.
To build a statically linked `nvim`, also install `musl-dev`:

```sh
# apt install musl-dev
```

Now we're ready to build.

## Building with AFL++

To build the instrumented `nvim` binary we will run the usual `nvim` build, but
with environment variables set to make it use the AFL++ compiler wrapper. We'll
also need to specify that we want a statically linked binary, and that we want a
debug build. A debug build will be more useful if we want to run in a debugger
and it will make the `assert` actually terminate the program.

If your Neovim repository isn't fresh, you should probably run `make distclean`
first, so that `cmake` will rebuild the build system with the new options
selected.

We also need to choose what version of the AFL++ compiler wrapper to use. [The
AFL++ documentation](https://aflplus.plus/docs/fuzzing_in_depth/) gives some
guidance, which starts with "use LTO if possible", so we'll use that. Here's the
build command:

```sh
# CC=/AFLplusplus/afl-clang-lto CXX=/AFLplusplus/afl-clang-lto++ make CMAKE_BUILD_TYPE=Debug CMAKE_EXTRA_FLAGS="-DSTATIC_BUILD=1"
```

I don't know if the C++ compiler is ever used in building Neovim, I added it
just in case.

But this build fails:

```
/AFLplusplus/afl-clang-lto -fPIC -g -O2 -fomit-frame-pointer -Wall  -fPIC -DLUA_USE_APICHECK -funwind-tables -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -U_FORTIFY_SOURCE  -DLUA_ROOT=\"/neovim/.deps/usr\" -DLUA_MULTILIB=\"lib\" -DLUA_LJDIR=\"/neovim/.deps/usr/share/luajit-2.1\" -fno-stack-protector   -c -o lib_buffer_dyn.o lib_buffer.c
lj_err.c:492:2: error: "Broken build system -- only use the provided Makefiles!"
  492 | #error "Broken build system -- only use the provided Makefiles!"
      |  ^
1 error generated.
gmake[2]: *** [Makefile:719: lj_err.o] Error 1
```

## The LuaJIT Build Error

This error comes from the LuaJIT dependency. If you check the `lj_err.c` file,
there's a lengthy comment at the top describing the options for frame unwinding,
and the error comes from a section of related conditionally-compiled code.
Reading the comment you can discover that external frame unwinding is the
default on toolchains that produce unwind tables by default, and that the POSIX
and Windows ABIs for x64 mandate unwind tables. Given that, we should be able to
specify that we want external frame unwinding and bypass the issue.

The specific problem comes from `Makefile`, where it tries to check for frame
unwinding by compiling a small test program and examining the resulting object
file. But if you check the `.o` files that have been built, you'll see that they
aren't standard object files:

```
# file .deps/build/src/luajit/src/lj_alloc.o
.deps/build/src/luajit/src/lj_alloc.o: LLVM IR bitcode
```

I don't know exactly what is happening here, but it looks like the LTO
(link-time optimization) mode uses this different output format, and the LuaJIT
Makefile doesn't handle it.

This isn't a big deal, we don't need to rely on the Makefile's autodetection for
unwind tables, we already know that the toolchain should produce them and
external frame unwinding should work. We can just request external frame
unwinding. To do that, apply this patch:

```
diff --git a/cmake.deps/cmake/BuildLuajit.cmake b/cmake.deps/cmake/BuildLuajit.cmake
index 070eeafc00..7d3a19858e 100644
--- a/cmake.deps/cmake/BuildLuajit.cmake
+++ b/cmake.deps/cmake/BuildLuajit.cmake
@@ -71,7 +71,7 @@ if(CYGWIN)
 elseif(UNIX)
   BuildLuajit(INSTALL_COMMAND ${BUILDCMD_UNIX}
     CC=${DEPS_C_COMPILER} PREFIX=${DEPS_INSTALL_DIR}
-    ${DEPLOYMENT_TARGET} install)
+    ${DEPLOYMENT_TARGET} TARGET_XCFLAGS=-DLUAJIT_UNWIND_EXTERNAL install)
 
 elseif(MINGW)
```

Then rerun the build command. This time it should succeed.
