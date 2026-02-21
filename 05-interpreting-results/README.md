# Part 5: Interpreting Results

While we have a reliable repro for the bug, it's in the form of a binary blob
that has to be interpreted by our Lua script. The script just turns it into nvim
API calls, so we could update the script to output the API calls instead of
running them, giving us a script that can be run directly. This also provides a
better picture of the steps required for the repro.

A new version of the script, this time named `print-calls.lua` can be found in
the same directory as this file.

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

Another curious pattern is the use of 48-character lines, and extmarks in
column 48. This has been extremely common in the results I've examined, and I'm not
sure at this point if it is a required part of the repro or an artifact of some
kind.
