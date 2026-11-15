---
title: How to install gruvbox colorscheme in nvim
description: "How to install gruvbox colorscheme without a plugin manager"
date: 2026-11-15
tags: ["howto", "nvim", "tutorial"]
---

## INTRODUCTION

In this blog we are going to be installing the gruvbox color scheme without using a plugin manager of any kind in nvim

We are going to install this repo: [github](https://github.com/ellisonleao/gruvbox.nvim)

## REQUIREMENTS

>NOTE: This blog assumes you have neovim 9.5 or higher, haven't tested in older versions.

```bash
mkdir -p ~/.config/nvim
touch ~/.config/nvim/init.lua
```

Run the above command to create a config file for nvim.

>NOTE: You can skip this if you already have a config file.

## INSTALLATION

```bash
git clone https://github.com/ellisonleao/gruvbox.nvim ~/.local/share/nvim/site/pack/themes/start/gruvbox.nvim
```

This command automatically installs the gruvbox manually, now you should be able to select it using `:colo gruvbox`

We can 2 lines to our init.lua to automatically select the theme each time we open a file

```lua
Inside init.lua
vim.o.background = "dark" -- or "light" for light mode
vim.cmd([[colorscheme gruvbox]])
```

And your done, enjoy your new colorscheme!!!

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
