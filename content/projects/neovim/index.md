---
title: "Neovim"
#date: 2024-02-10T03:36:02+05:30
draft: true
showtoc: false
weight: 98
---

> [**Source Code**](https://github.com/VanshajSaxena/nvim)

[**Neovim**](https://neovim.io/) is a hyperextensible modal text editor based
on Vim.

I started using Vim during the first year of my engineering and instantly knew
it is an incredible tool. I new use vim keybinding in everything that I do,
from text editing to browsing the web.

Neovim is highly configurable and extensible, this is one of its qualities
among others. It is configured using a simple language called
[Lua](https://www.lua.org/about.html).

> Lua is a powerful, efficient, lightweight, embeddable scripting language. It
> supports procedural programming, object-oriented programming, functional
> programming, data-driven programming, and data description.

I started to learn Lua in my free time and became quite familiar with it very
quickly. The language is so simple that it has only a single data structure,
_tables_. **A single data structure to rule them all**.

Since I first started using Neovim I maintained my own Neovim configuration,
building it from scratch and rewriting it, several times. This was in itself a
spiritual process in my programming journey.

---

### Motivation

Neovim puts you in a place where you control every aspect of your development
process and environment, moreover it gives you the opportunity to learn about
the tools and dependencies essential for your project. This is the primary
reason I use Neovim as my main text editor.

---

### Structure

Here's the structure of my Neovim configuration:

```shell
nvim
├── hyperfine.out
├── init.lua
├── lazy-lock.json
├── lazyvim.json
├── LICENSE
├── lua
│   ├── config
│   │   ├── autocmds.lua
│   │   ├── keymaps.lua
│   │   ├── options.lua
│   │   └── startup.lua
│   └── plugins
│       ├── coding.lua
│       ├── colorscheme.lua
│       ├── disabled.lua
│       ├── editor.lua
│       ├── extras.lua
│       ├── lsp
│       │   ├── core.lua
│       │   └── utils.lua
│       ├── overrides.lua
│       ├── tools.lua
│       └── ui.lua
└── stylua.toml
```

This has evolved so much since I first started.

I now have moved to using Neovim Distributions as they provide structured API
through which I can extend further and save some time for other projects.
Maintaining a Neovim configurable is sometimes so time taking that you get to
have very little time for other projects that you work upon.
