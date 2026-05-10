---
title: Shell 配置文件的组织原则
date: 2026-05-11 01:40:00
tags:
  - shell
  - linux
  - dotfiles
categories: DevOps
---

搞 Linux 的或多或少都踩过 shell 配置文件的坑——环境变量时有时无、alias 在脚本里不生效、SSH 执行远程命令时 PATH 少了一截。归根结底是 `.profile`、`.bash_profile`、`.bashrc` 这三个文件的职责没划清楚。

这篇文章记录一套经过实践验证的组织原则，核心思路就一句话：**每个文件只做一件事**。

---

## 职责划分

### 1. `.profile` — 基础环境层

这个文件只放**所有 login shell 都需要的基础环境**，不管你是交互式还是非交互式，不管是 bash 还是 sh，只要它是 login shell，就应该拿到这些东西。

典型内容包括：

- PATH 的基础设置（追加 `~/bin`、`/usr/local/bin` 等）
- 所有 login shell 都需要的环境变量（如 `EDITOR`、`LANG`）
- CLI 工具所需的 credentials 路径（如 `AWS_CLI_FILE_ENCODING`）

**不该放的东西：** prompt 设置、shell completion、交互专属 alias、任何"让终端更好用"的东西。这些东西在非交互式 login shell（比如 SSH remote command）里只会拖慢速度甚至报错。

```sh
# .profile 示例

# 基础 PATH
export PATH="$HOME/bin:$HOME/.local/bin:/usr/local/bin:$PATH"

# 通用环境变量
export EDITOR=vim
export LANG=en_US.UTF-8
```

### 2. `.bash_profile` — Login 入口分发层

这个文件**只做分发**，不放任何实际配置。它的职责是：

1. 先加载 `.profile`（拿到基础环境）
2. 判断当前 shell 是否为交互式，如果是，再加载 `.bashrc`

推荐结构：

```sh
# .bash_profile

if [ -f "$HOME/.profile" ]; then
    . "$HOME/.profile"
fi

case $- in
    *i*)
        if [ -f "$HOME/.bashrc" ]; then
            . "$HOME/.bashrc"
        fi
        ;;
esac
```

**关键点：** 不要无条件 `source ~/.bashrc`。非交互式 login shell（`ssh host 'command'`）不应该加载交互增强。`case $- in *i*)` 这个判断是标准做法，可靠且轻量。

### 3. `.bashrc` — 交互增强层

这个文件只放**交互式 shell 的增强功能**，也就是只在你打开一个终端、真正坐在键盘前打字时才需要的东西。

典型内容包括：

- Prompt / PS1 设置
- Bash completion
- 交互专属 alias（如 `alias ll='ls -la'`）
- nvm 等工具的交互式初始化（这些通常比较慢，不该阻塞非交互式 shell）

**不该放的东西：** 基础 PATH、所有 login shell 都需要的 credentials/env。如果放在这里，SSH remote command 就拿不到。

```sh
# .bashrc 示例

# Prompt
PS1='\u@\h:\w\$ '

# 常用 alias
alias ll='ls -la'
alias la='ls -A'

# Bash completion
if ! shopt -oq posix; then
    if [ -f /usr/share/bash-completion/bash_completion ]; then
        . /usr/share/bash-completion/bash_completion
    fi
fi
```

---

## 总结

| 文件 | 职责 | 加载时机 |
|---|---|---|
| `.profile` | 基础环境、PATH、env vars | 所有 login shell |
| `.bash_profile` | 分发层：加载 `.profile`，按需加载 `.bashrc` | bash login shell |
| `.bashrc` | 交互增强：prompt、alias、completion | 仅交互式 shell |

记住三个原则：

1. **`.profile` 是地基** — 只放所有场景都通用的东西
2. **`.bash_profile` 是门卫** — 只做判断和分发，不加料
3. **`.bashrc` 是装修** — 只放让你打字更舒服的东西

按这个结构组织，基本上不会再遇到"为什么这个变量在这里有那里没有"的问题。
