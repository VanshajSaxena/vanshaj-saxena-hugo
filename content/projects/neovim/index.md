---
title: 'Neovim'
#date: 2024-02-10T03:36:02+05:30
draft: false
showtoc: false
weight: 97
---
---

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
*tables*. **A single data structure to rule them all**.

Since I first started using Neovim I maintained my own Neovim configuration,
building it from scratch and rewriting it, several times. This was in itself a
spiritual process in my programming journey.

Here's the structure of my Neovim configuration:

### Structure
``` shell
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

Here is the [source code](https://github.com/VanshajSaxena/nvim) for anyone interested in diving more deep into the
Neovim Ecosystem.

