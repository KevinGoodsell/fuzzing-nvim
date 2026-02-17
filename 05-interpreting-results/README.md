# Part 5: Interpreting Results

While we have a reliable repro for the bug, it's in the form of a binary blob
that has to be interpreted by our Lua script. The script just turns it into nvim
API calls, so we could update the script to output the API calls instead of
running them, giving us a script that can be run directly. This also provides a
better picture of the steps required for the repro.

Here's a version of the script that prints the API calls instead of running
them:

```lua
local bit = require('bit')

print("vim.opt.swapfile = false\n")
print("local namespace_id = vim.api.nvim_create_namespace('extmark-fuzz')\n\n")

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
    print("vim.api.nvim_buf_clear_namespace(0, namespace_id, 0, -1)\n")
end

local function add_extmark_opts(buf, with_id, with_end)
    local opts = {}
    local print_opts = {}

    if with_id then
        opts.id = read_byte(buf)
        table.insert(print_opts, string.format("  id = %d,\n", opts.id))
    end

    local line = read_byte(buf)
    local col = bit.band(read_byte(buf), 0x7F)
    local flags = read_byte(buf)
    opts.right_gravity = bit.band(flags, 0x01) ~= 0
    opts.undo_restore = bit.band(flags, 0x02) ~= 0
    opts.invalidate = bit.band(flags, 0x04) ~= 0
    if not opts.right_gravity then
        table.insert(print_opts, "  right_gravity = false,\n")
    end
    if not opts.undo_restore then
        table.insert(print_opts, "  undo_restore = false,\n")
    end
    if opts.invalidate then
        table.insert(print_opts, "  invalidate = true,\n")
    end

    if with_end then
        opts.end_row = read_byte(buf)
        opts.end_col = bit.band(read_byte(buf), 0x7F)
        opts.end_right_gravity = bit.band(flags, 0x08) ~= 0
        table.insert(print_opts, string.format(
            "  end_row = %d,\n",
            opts.end_row))
        table.insert(print_opts, string.format(
            "  end_col = %d,\n",
            opts.end_col))
        if opts.end_right_gravity then
            table.insert(print_opts, "  end_right_gravity = true,\n")
        end
    end

    print(string.format(
        "vim.api.nvim_buf_set_extmark(0, namespace_id, %d, %d, {\n", line, col))
    for _, line in ipairs(print_opts) do
        print(line)
    end
    print("})\n")
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

    print(string.format("vim.api.nvim_buf_del_extmark(0, namespace_id, %d)\n", id))
end

local function read_lines(buf, count)
    local print_lines = {}
    for _ = 1, count do
        local len = bit.band(read_byte(buf), 0x7F)
        table.insert(print_lines, string.format("  string.rep('o', %d),\n", len))
    end

    print("lines = {\n")
    for _, line in ipairs(print_lines) do
        print(line)
    end
    print("}\n")
end

local function set_lines(buf)
    local start_line = read_byte(buf)
    local end_line = read_byte(buf)
    local line_count = read_byte(buf)
    read_lines(buf, line_count)

    print(string.format(
        "vim.api.nvim_buf_set_lines(0, %d, %d, false, lines)\n",
        start_line,
        end_line))
end

local function set_text(buf)
    local start_row = read_byte(buf)
    local start_col = bit.band(read_byte(buf), 0x7F)
    local end_row = read_byte(buf)
    local end_col = bit.band(read_byte(buf), 0x7F)
    local line_count = read_byte(buf)
    read_lines(buf, line_count)

    print(string.format(
        "vim.api.nvim_buf_set_text(0, %d, %d, %d, %d, lines)\n",
        start_row,
        start_col,
        end_row,
        end_col))
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
        print("\n")
    end

    print("vim.cmd.qall({ bang = true })\n")
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

Given a crashing input for `extmark-fuzz.lua`, this will print a Lua script that
crashes the same way without needing `extmark-fuzz.lua`. Running this on the
smallest of the minimized input files gives this script:

```
vim.opt.swapfile = false
local namespace_id = vim.api.nvim_create_namespace('extmark-fuzz')

lines = {
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
  string.rep('o', 48),
}
vim.api.nvim_buf_set_lines(0, 48, 48, false, lines)

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 51,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  right_gravity = false,
  undo_restore = false,
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  undo_restore = false,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 48, {
  right_gravity = false,
  undo_restore = false,
  end_row = 97,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 48, {
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 51, 48, {
  right_gravity = false,
  undo_restore = false,
  end_row = 51,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  right_gravity = false,
  invalidate = true,
  end_row = 48,
  end_col = 48,
  end_right_gravity = true,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 51, 48, {
  right_gravity = false,
  undo_restore = false,
  end_row = 51,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  right_gravity = false,
  undo_restore = false,
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  right_gravity = false,
  undo_restore = false,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  right_gravity = false,
  undo_restore = false,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 19, {
  right_gravity = false,
  undo_restore = false,
  end_row = 97,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 48, {
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 19, {
  right_gravity = false,
  undo_restore = false,
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  id = 48,
  right_gravity = false,
  undo_restore = false,
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 48, {
  end_row = 145,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  right_gravity = false,
  undo_restore = false,
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 48, {
  end_row = 147,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 147, 48, {
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 48,
  end_col = 48,
})

vim.api.nvim_buf_set_extmark(0, namespace_id, 48, 48, {
  end_row = 48,
  end_col = 48,
})

lines = {
}
vim.api.nvim_buf_set_lines(0, 48, 161, false, lines)

vim.cmd.qall({ bang = true })
```

This follows a very common pattern in the crash result: Adding several lines of
text, adding a lot of extmarks, then deleting a block of text. There are some
variations that add and remove text at different points, and occasionally an
extmark delete shows up, or a namespace clear.

Another curious pattern is the use of 48-character lines, and extmarks in column
48. This has been extremely common in the results I've examined, and I'm not
sure at this point if it is a required part of the repro or an artifact of some
kind.
