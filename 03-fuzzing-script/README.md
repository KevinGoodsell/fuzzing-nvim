# Part 3: The Fuzzing Script

The script I wrote to fuzz nvim by targeting specific, relevant API calls can be
found in `extmark-fuzz.lua`. I don't have a lot of Lua experience, and this was
written to get the jobs done, not to be pretty.

Let's break down some of this.

```lua
vim.fn.stdioopen({
    on_stdin = get_input,
    stdin_buffered = true,
})
```

This opens stdin (and stdout) as a channel so we can read stdin. `on_stdin` is a
callback that receives the data. Making it buffered causes the callback to be
called just once after all the input is available, which should be fine for our
purposes and should make things a little bit simpler.

```lua
local function get_input(chanid, data, name)
    local ok, result = pcall(handle_input, data)
    if not ok then
        vim.print(result)
    end

    vim.cmd.qall({ bang = true })
end
```

The input callback is just a wrapper that uses `pcall` for the rest, so that the
last bit runs and quit neovim even if an exception is thrown.

```lua
local function input_to_bytes(data)
    local b = {}
    for _, s in ipairs(data) do
        for i = 1, #s do
            local val = s:byte(i)
            if val == 10 then
                val = 0
            end
            table.insert(b, val)
        end
        table.insert(b, 10) -- newline
    end

    if #b > 0 then
        -- Remove the extra newline from the end
        table.remove(b)
    end

    return b
end
```

This converts the input data into the format we want to use. The data are
received as an array of strings, each string containing one line of input.
Within an input line, a newline character acts as a stand-in for a zero byte.
To turn it into a sequence of bytes, we iterate the lines, iterate over the
characters in the line, and convert each character to a byte. If the byte is 10
(newline), replace it with 0. At the end of each line, add a 10 for the newline.
That results in an extra newline at the very end, so we remove that.

The rest interprets the input bytes, reading a byte to an operation, then
subsequent bytes as the parameters for that operation. If we run off the end of
the input array, zeros get substituted until the final operation is complete.
Any sequence of bytes is considered valid. Some bytes that don't correspond to a
defined operation will be discarded. Some bits are also discarded to bring
values into the range that we want (text lines are limited to less than 128
characters, for example).

The operations are pretty limited: changing text, adding extmarks, deleting
extmarks, and clearing the namespace. There are also operations for "setting" an
extmark, meaning an ID is supplied so that an existing extmark can be replaced,
or an extmark can be added with a specified ID. Extmarks can also be provided
with an end position, and various flags can be set.

Here is an example command for running the script, which I've placed in the
neovim repository directory:

```
# ./build/bin/nvim -n -u NONE -i NONE --headless --cmd "source extmark-fuzz.lua"
```

It's possible to test the script functionality by passing in specific bytes. To
see the results, it's helpful to remove the `qall` command, then run with
`--listen ./server.sock` and attach another nvim instance with `--server
./server.sock --remote-ui`. For example:

```sh
$ echo -e "\x07\x00\x00\x02\x10\x12" | ./build/bin/nvim -n -u NONE -i NONE --headless --listen ./server.sock --cmd "source extmark-fuzz.lua"
```

```sh
$ ./build/bin/nvim --server ./server.sock --remote-ui
```

The input bytes are:

* 0x07 - set lines
* 0x00 - start line 0
* 0x00 - end line 0
* 0x02 - line count 2
* 0x10 - first line length 16
* 0x12 - second line length 18

The second nvim instance will show the UI with two lines of text, length 16 and
18, as expected.
