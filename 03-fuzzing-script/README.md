# Part 3: The Fuzzing Script

Below is the Lua script I wrote to fuzz nvim, by targeting specific API calls
that seemed relevant. I don't have a lot of Lua experience, and this was written
to get the jobs done, not to be pretty.

```lua
local bit = require('bit')

vim.opt.swapfile = false
local namespace_id = vim.api.nvim_create_namespace('extmark-fuzz')

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

local function buf_empty(buf)
    return buf.index > #buf.bytes
end

local function read_byte(buf)
    local b = buf.bytes

    if buf_empty(buf) then
        return 0
    end

    local byte = b[buf.index]
    buf.index = buf.index + 1

    return byte
end

local function clear_namespace()
    vim.api.nvim_buf_clear_namespace(0, namespace_id, 0, -1)
end

local function add_extmark_opts(buf, with_id, with_end)
    local opts = {}

    if with_id then
        opts.id = read_byte(buf)
    end

    local line = read_byte(buf)
    local col = bit.band(read_byte(buf), 0x7F)
    local flags = read_byte(buf)
    opts.right_gravity = bit.band(flags, 0x01) ~= 0
    opts.undo_restore = bit.band(flags, 0x02) ~= 0
    opts.invalidate = bit.band(flags, 0x04) ~= 0

    if with_end then
        opts.end_row = read_byte(buf)
        opts.end_col = bit.band(read_byte(buf), 0x7F)
        opts.end_right_gravity = bit.band(flags, 0x08) ~= 0
    end

    vim.api.nvim_buf_set_extmark(0, namespace_id, line, col, opts)
end

local function add_extmark(buf)
    add_extmark_opts(buf, false, false)
end

local function add_range_extmark(buf)
    add_extmark_opts(buf, false, true)
end

local function set_extmark(buf)
    add_extmark_opts(buf, true, false)
end

local function set_range_extmark(buf)
    add_extmark_opts(buf, true, true)
end

local function delete_extmark(buf)
    local id = read_byte(buf)

    vim.api.nvim_buf_del_extmark(0, namespace_id, id)
end

local function read_lines(buf, count)
    local lines = {}
    for _ = 1, count do
        local len = bit.band(read_byte(buf), 0x7F)
        table.insert(lines, string.rep("o", len))
    end

    return lines
end

local function set_lines(buf)
    local start_line = read_byte(buf)
    local end_line = read_byte(buf)
    local line_count = read_byte(buf)
    local lines = read_lines(buf, line_count)

    vim.api.nvim_buf_set_lines(0, start_line, end_line, false, lines)
end

local function set_text(buf)
    local start_row = read_byte(buf)
    local start_col = bit.band(read_byte(buf), 0x7F)
    local end_row = read_byte(buf)
    local end_col = bit.band(read_byte(buf), 0x7F)
    local line_count = read_byte(buf)
    local lines = read_lines(buf, line_count)

    vim.api.nvim_buf_set_text(0, start_row, start_col, end_row, end_col, lines)
end

local function handle_input(data)
    local buf = {
        bytes = input_to_bytes(data),
        index = 1,
    }

    while not buf_empty(buf) do
        local op = bit.band(read_byte(buf), 0x0F)

        if op == 1 then
            clear_namespace()
        elseif op == 2 then
            add_extmark(buf)
        elseif op == 3 then
            add_range_extmark(buf)
        elseif op == 4 then
            set_extmark(buf)
        elseif op == 5 then
            set_range_extmark(buf)
        elseif op == 6 then
            delete_extmark(buf)
        elseif op == 7 then
            set_lines(buf)
        elseif op == 8 then
            set_text(buf)
        end
    end
end

local function get_input(chanid, data, name)
    local ok, result = pcall(handle_input, data)
    if not ok then
        vim.print(result)
    end

    vim.cmd.qall({ bang = true })
end

vim.fn.stdioopen({
    on_stdin = get_input,
    stdin_buffered = true,
})
```

Let's break down some of this.

```lua
vim.fn.stdioopen({
    on_stdin = get_input,
    stdin_buffered = true,
})
```

This opens stdin (and stdout) as a channel so we can read stdin. `on_stdin` is a
callback that recieves the data. Making it buffered causes the callback to be
called just once after all the input is available, which should be fine for our
purposes and make things a little bit simpler.

```lua
local function get_input(chanid, data, name)
    local ok, result = pcall(handle_input, data)
    if not ok then
        vim.print(result)
    end

    vim.cmd.qall({ bang = true })
end
```

The input callback is just a wrapper that uses `pcall` for the rest, so that we
can be sure that the last bit runs and quits nvim.

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
Within an input line, a newline character acts as a placeholder for a zero byte.
To turn it into a sequence of bytes, we iterate the lines, iterate over the
line, and convert each character to a byte. If the byte is 10 (newline), replace
it with 0. At the end of each line, add a 10 for the newline. That results in an
extra newline at the very end, so we remove that.

The rest interprets the input bytes, interpreting a byte to an operation, then
subsequent bytes as the parameters for that operation. If we run off the end of
the input array, zeros get substituted until the final operation is complete.
Any sequence of bytes is considered valid. Some bits are discarded to bring
values into the range that we want (lines lengths are limited to less than 128,
for example).

The operations are pretty limited: changing text, adding extmarks, deleting
extmarks, and clearing the namespace. There are also operations for "setting" an
extmark, meaning an ID is supplied so that an existing extmark can be replaced,
or an extmark can be added with a pre-determined ID. Extmarks can also be
provided with an end position, and various flags can be set.

Here is an example command for running the script, which I've named
`extmark-fuzz.lua` and placed in the neovim repository directory:

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

The second nvim instance will show two lines of text, length 16 and 18.
