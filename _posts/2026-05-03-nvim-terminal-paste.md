---
layout: post
title: "修复 Neovim :terminal 中粘贴长文本丢失字符的问题"
date: 2026-05-03
categories: terminal
---

在 Neovim 的 `:terminal` 中使用 Claude Code 等交互式 TUI 程序时，通过 `Shift+Insert` 粘贴文本会出现非确定性的**字符丢失或损坏**：中间片段消失、行首字符被替换，且每次结果不同。

## 现象

在 Neovim 中打开 `:terminal`，启动 Claude Code，粘贴一段多行文本。期望看到完整内容，实际却出现：

```
Et -g default-terminal      # 首字符 's' 丢失且多出 'E'
et -g def                   # 后半段被截断
```

不同尝试之间结果不一致（非确定性）。这个问题的特殊之处在于：

- **不经过 Neovim** 直接粘贴到 Claude Code 正常
- **Vim 的 `:terminal`** 粘贴正常
- **`nvim --clean`** 也能复现（排除配置/插件）
- **macOS 和 WSL2** 均受影响

## 解决方案

### Neovim ≥ 0.12.2

升级即可，社区已在 [#39152](https://github.com/neovim/neovim/pull/39152) 修复。

```bash
# 使用你喜欢的包管理器升级
brew upgrade neovim       # macOS
winget upgrade Neovim     # Windows
```

### Neovim < 0.12.2

在不修改 Neovim 二进制的情况下，可以通过覆盖 `vim.paste` 来绕过有 bug 的 `terminal_paste()` 实现：

```lua
-- 放到 init.lua 中
if vim.fn.has("nvim-0.12.2") == 0 then
  local paste_streaming = false

  vim.paste = (function(overridden)
    return function(lines, phase)
      if vim.bo.buftype ~= "terminal" then
        return overridden(lines, phase)
      end
      local chan = vim.bo.channel
      if not chan or chan == 0 then
        return overridden(lines, phase)
      end

      local text = table.concat(lines, "\n")
      local is_first = (phase == 1 or phase == -1)
      local is_last = (phase == 3 or phase == -1)

      if is_first and not paste_streaming then
        paste_streaming = true
        vim.api.nvim_chan_send(chan, "\27[200~")
      end
      vim.api.nvim_chan_send(chan, text)
      if is_last then
        vim.api.nvim_chan_send(chan, "\27[201~")
        paste_streaming = false
      end

      return true
    end
  end)(vim.paste)
end
```

这个 workaround 只在 terminal buffer 中接管 paste 处理：直接用 `nvim_chan_send` 把文本发送到终端 job，并为整个 paste stream 只生成一对 bracketed paste 标记，而不是每 chunk 一对。

升级到 0.12.2 后，`vim.fn.has("nvim-0.12.2")` 检查自动跳过覆盖代码，不会干扰已修复的行为。

### Workaround 的风险

这个 workaround 在大多数场景下工作正常，但有几个值得注意的局限：

1. **dot-repeat 不工作**：覆盖函数直接 `return true`，没有像原生 `vim.paste()` 那样调用 `paste_store()` 记录 paste 历史。在 terminal 中粘贴无法通过 `.` 重复。这是有意为之——terminal buffer 中的 dot-repeat 应该由终端 job 自己处理，Neovim 的粘贴重放本来就不适合 terminal mode。

2. **绕过 `tpf_flags` 过滤**：原生的 `terminal_paste()` 会根据 `tpf_flags`（`'paste'` 选项）过滤控制字符（如 ESC、退格等）。`nvim_chan_send` 不会做这个过滤。但现代终端程序（bash ≥ 4.4、Claude Code、vim 等）都已启用 bracketed paste，通过 `\e[200~` / `\e[201~` 标记来识别粘贴内容，对控制字符的处理更安全，因此这在实际使用中很少造成问题。

3. **仅在 `buftype=terminal` 生效**：覆盖函数在非 terminal buffer 中直接调用原生的 `overridden(lines, phase)`，不影响普通 buffer 的粘贴行为。

4. **跨 chunk 状态用闭包变量管理**：`paste_streaming` 是一个模块级闭包变量。如果两个 terminal 同时收到 streamed paste（极端边缘场景，同一个 Neovim 实例中不同 terminal 同时被粘贴），状态会冲突。实际使用中几乎不会遇到，但理论上存在。

## 原因

Neovim TUI 层在处理外部 paste 时，通过 bracketed paste 判断 paste 的起止位置。Paste 内容被缓冲在 **4096 字节**（`KEY_BUFFER_SIZE = 0x1000`）的内置 buffer 中。当内容超过这个值时，会被分割成多个 chunk 发送：

```
Chunk 1 (phase=1): \e[200~ + 前4096字节 + \e[201~
Chunk 2 (phase=2): \e[200~ + 中间4096字节 + \e[201~   ← 不应出现
Chunk 3 (phase=3): \e[200~ + 剩余部分 + \e[201~       ← 不应出现
```

核心问题在于 `src/nvim/terminal.c` 中的 `terminal_paste()` 函数——每次调用都**无条件**添加一对 `vterm_keyboard_start_paste()` / `vterm_keyboard_end_paste()`。当 chunk 机制多次调用 `terminal_paste()` 时，每个 chunk 都被包装成了独立的 bracketed paste。子进程（Claude Code、vim 等）收到了多个被错误分割的 paste 事件。

对于 Claude Code 这类交互式 TUI 程序，每次收到"独立 paste"后都会立即处理并重新渲染界面。多个独立 paste 之间的 UI redraw 互相覆盖，最终只有部分内容被保留。

即使内容短于 4096 字节（单 chunk），`terminal_paste()` 内部的 `uv_write()` 异步写和 vterm 输出回调之间的时序也在特定系统上引入了竞态。这也解释了为什么 Vim 的 `:terminal` 没有此问题——Vim 使用同步 `write()` 而非 libuv 的异步 `uv_write()`。

修复的具体策略是将 bracketed paste 标记的生成从 `terminal_paste()` 提升到 `nvim_paste()` 层级，引入 `streamed_paste` 标记确保整个 paste stream 只有一对 `\e[200~` / `\e[201~`：

```c
// nvim_paste() — 修复后
if (phase == 1 || phase == -1)
    terminal_set_streamed_paste(term, true);   // 只在开始时输出 \e[200~
// … vim.paste() → do_put() → terminal_paste() — 内部不再加标记 …
if (phase == 3 || phase == -1)
    terminal_set_streamed_paste(term, false);  // 只在结束时输出 \e[201~
```

## 相关资料

- [neovim/neovim#39152](https://github.com/neovim/neovim/pull/39152) — 修复 PR：forward streamed bracketed paste properly
- [folke/sidekick.nvim#207](https://github.com/folke/sidekick.nvim/issues/207) — sidekick 用户报告的同一问题（已自动关闭）
- [coder/claudecode.nvim#161](https://github.com/coder/claudecode.nvim/issues/161) — claudecode.nvim 的同类问题报告
- [Neovim `:terminal` paste 早期讨论](https://github.com/neovim/neovim/issues/3476) — bracketed paste 在 `:terminal` 中的历史演进
