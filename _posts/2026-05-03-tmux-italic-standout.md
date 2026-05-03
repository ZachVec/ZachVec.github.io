---
layout: post
title: "修复 tmux 中斜体显示为高亮的问题"
date: 2026-05-03
categories: terminal
---

在 tmux 中使用 Vim 或 Neovim 时，你可能会遇到一个奇怪的现象：本该显示为**斜体**的文本（如注释、markdown 强调），却显示成了**反色高亮**（黑底白字变白底黑字）。反过来，某些主题的**高亮/反色**区域又变成了斜体。

## 检查问题

先在 tmux 内运行下面这条命令，确认你是否有这个问题：

```bash
echo -e "\e[3mitalic\e[23m"
```

如果看到的是**反色高亮**而不是斜体，说明中招了。

## 修复步骤

### 一、安装修正后的 terminfo

通过 heredoc 直接传给 `tic` 编译安装，无需创建临时文件：

```bash
cat <<'EOF' | tic -x -
tmux-256color|tmux with 256 colors,
  ritm=\E[23m, rmso=\E[27m, sitm=\E[3m, smso=\E[7m, Ms@,
  khome=\E[1~, kend=\E[4~,
  use=xterm-256color, use=screen-256color,
EOF
```

`tic -x -` 中的 `-` 表示从 stdin 读取，`'EOF'` 加引号防止 shell 对 `\E` 做转义插值。

`tic -x` 会将编译后的 terminfo 写入 `~/.terminfo/`，不需要 sudo。

### 二、修改 tmux 配置

在 `~/.tmux.conf` 中添加：

```
set -g default-terminal 'tmux-256color'
set -as terminal-overrides ',xterm*:Tc:sitm=\E[3m'
```

第一条把 tmux 的默认终端类型切换为刚安装的 `tmux-256color`。第二条为 xterm 类终端显式指定斜体序列，`Tc` 表示支持 true color。

### 三、重启 tmux

```bash
tmux kill-server && tmux
```

再次运行验证命令：

```bash
echo -e "\e[3mitalic\e[23m"
```

现在应该能看到正常的斜体文字了。

## 原理

终端的文本属性（粗体、斜体、下划线、反色等）通过 ANSI 转义序列控制。terminfo 数据库为每种终端类型定义了这些序列的映射关系：

| terminfo 字段 | 含义 | 正确序列 |
|--------------|------|---------|
| `sitm` | 开启斜体 | `\E[3m` |
| `ritm` | 关闭斜体 | `\E[23m` |
| `smso` | 开启反色高亮 | `\E[7m` |
| `rmso` | 关闭反色高亮 | `\E[27m` |

tmux 的默认 `$TERM` 是 `screen` 或 `screen-256color`。`screen` 终端本身不支持斜体，所以历史上它把 standout（smso）的序列映射到了 `\E[3m`（斜体），把斜体占为己有。tmux 继承了这个定义。

结果就是：在 tmux 内部，程序查 terminfo 得知要显示反色就发 `\E[3m`，但现代终端把 `\E[3m` 正确理解为斜体——于是反色高亮就变成了斜体。反过来，真正想显示斜体的程序发出来的序列又会被当成反色。

上面创建的 terminfo 文件关键就在于这两行：

```
use=xterm-256color, use=screen-256color,
```

它以 `screen-256color` 为基础，再用 `xterm-256color` 覆盖（后者正确定义了斜体和反色），最后显式指定 `sitm=\E[3m` / `smso=\E[7m` 确保万无一失。`Ms@` 则禁用了 `screen` 对颜色切换的干扰行为。

## 哪些系统会受影响

这个问题本质上是历史遗留，主要出现在 **2017 年之前**的系统上。ncurses 在 2017 年才引入独立的 `tmux-256color` terminfo 条目，正确分离了斜体和反色。

以下场景仍可能遇到：

- **老系统**：CentOS 7、旧版 macOS 等自带的 ncurses 版本过老，没有 `tmux-256color`
- **SSH 到旧服务器**：本地 `$TERM` 可能被 propagate 为 `screen-256color`（尤其当跳板机或远端 tmux 配置陈旧时）
- **tmux 配置显式设了旧 TERM**：如果 `~/.tmux.conf` 里有 `set -g default-terminal "screen-256color"`，问题会被重新引入

现代发行版（Ubuntu 18.04+、Debian 10+、macOS 10.14+）的 ncurses 已自带 `tmux-256color`，一般不会有这个问题。

---

*参考：[tmux/tmux#1202](https://github.com/tmux/tmux/issues/1202)，[ncurses-bug 邮件列表](https://lists.gnu.org/archive/html/bug-ncurses/2017-11/msg00010.html)*
