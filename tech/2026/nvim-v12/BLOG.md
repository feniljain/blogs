---
tech: true
draft: false
slug: 'neovim-v12'
title: 'Upgrading to neovim 0.12'
publishedOn: '20-05-2026'
lastEditedOn: '23-05-2026'
---

I recently upgraded my neovim to 0.12.2, I have been chipping away at it slowly using `NVIM_APPNAME` feature so that my day to day activities are not hindered. One of the things I found interesting in using `NVIM_APPNAME` is, I saw a blog where author manually create all the `~/.local/share/nvim-next` etc dirs, I did the same cause I was not aware, but later I realized that one can literally just say `NVIM_APPNAME=nvim-next ~/.local/share/bob/0.12/bin/nvim` and neovim will automatically make all those dirs for you. Also yes, I use awesome [bob-nvim](https://github.com/MordechaiHadad/bob) for managing neovim versions. With that out of the way, lets get started!

## Major changes

Cool, lets talk about major changes, listing them out first:
 - `vim.pack`: Migrating to builtin package manager
 - `lsp`: Ditching nvim-lspconfig etc for `vim.lsp.enable`
 - `ui2`
 - no vimscript in config

### Migrating to builtin package manager

Neovim now comes with an inbuilt package manager called `vim.pack`, its currently experimental but is considered good enough for daily driving. I have used a bunch of package managers over time, vim-plugin, packer, lazy and now `vim.pack`. While I don't care about them a lot, each migration has been inspired by reasons. vim-plug -> packer, lua shift of whole ecosystem. packer -> lazy.nvim, extra features like dependencies, clean config, etc and finally lazy -> vim.pack cause I want to reduce count of external dependencies. With all the changes happening upstream, I am really hopeful that some day my whole config will just fit in a small file!

But I couldn't simply move to using vim.pack, I had to evaluate if it was strictly an upgrade. Factors I looked for:
- Do I depend on dependencies feature of lazy?
- Does it slow down my startup times?
- Does separating config in `vim.pack` make config structure worse?

For `dependencies` section, I went through my plugin list and realized I had these dependencies:
- aerial.nvim on nvim-treesitter and nvim-web-devicons
- telescope on telescope-fzy-native, telescope-live-grep-args and plenary.nvim
- nvim-treesitter-context on nvim-treesitter

These don't seem that bad, all of them are loaded much later in the neovim startup process and I could just keep them in a particular order to make the resolution pass. Well I tried it and that did work out, so ticked this off.

For my startup times, I used `nvim --startuptime startuptime.log .`. If you are not familiar with this command, it instructs neovim to write a log of its startup activities and to get startup time you would look for log `--- NVIM STARTED ---`, first number in that row is the amount of seconds it took to startup your neovim. I measured this and realized I hadn't used "lazy" in lazy.nvim package manager 😭. But I never felt the need to make it go faster, cause I wasn't able to perceive a delay when starting it. Day I start noticing the delay is the day I bring down my hammer. I had done this earlier for my [shell](https://github.com/feniljain/blogs/tree/main/2024/faster-shell-boot) too. But after the migration, timings seemed the same, actually a bit better than lazy.nvim, so I was already happy :)

For config structure, I really liked how lazy forced a dir structure of `plugins` and encouraged keeping config along side plugin installation line itself. It was a clean way. With `vim.pack` I had two ways:
 - keep installation line with config in top level `plugin/` dir
 - keep all installation lines together and config separately

I wanted to maintain order remember? That can easily be done with a single `vim.pack.add` and listing down all the plugins together with their install order. But with `plugin/` dir, I would have to name files `0_`, `1_` etc. This is because files in `plugin/` are loaded automatically by vim and neovim in sorted order. Naming files like that was a turn off for me, so I went with single `vim.pack.add` call and created a new dir called `plugins/` in `lua/` dir and shoved all plugin related config there. I enforced [setup order]((https://github.com/feniljain/dotfiles/blob/9681a844aadc971dae416836e2dc75feb328b8d3/nvim/.config/nvim/lua/fenil/plugins/init.lua#L1-L8)) in `lua/plugins/init.lua`.

With all of this out of the way, migration was really SMOOTH! And I kinda love that I am not pulling a heavy dependency like lazy.nvim in my dep tree :)

If you have advanced usecases or want to understand the feature better, there are two awesome guides: official manual and then https://echasnovski.com/blog/2026-03-13-a-guide-to-vim-pack.html. Read it end-to-end, each section has something you can take away. It is the most comprehensive guide out there right now.

One small trick I learned from the article above is placing `vim.loader.enable` speeds up startup times for free and this is blessed on us by Folke himself!! Ofc I added that and instantly realized 25ms off the loading time xD

### LSP

Neovim v0.12 also brings in more ergonomic LSP usage support. Now, you can place LSP server setting in `lsp/` dir and just call `vim.lsp.enable(<file-name-in-lsp-dir>)'` and this is all the setup you need! `nvim-lspconfig` is now reduced to just maintain settings for upstream LSP servers. As these rarely change, I just copied from upstream and placed in my `lsp/` dir, I also realized I only use two of them now: rust_analyzer and taplo (toml LSP server, also for rust dev :P). With this, I was able to cut down a bunch of lines in my LSP config and also trim down deps of `nvim-lspconfig` and `mason-lspconfig`.

### ui2

This is an experimental feature where command line meets messages meets pager meets dialog windows. Honestly its best explained in [official docs](https://neovim.io/doc/user/lua/#ui2). This was an interesting change which allowed me to trim down a dep I really liked: `fidget.nvim`. It shows LSP progress on the bottom right corner. Now I do it in ui2 itself using [this](https://github.com/feniljain/dotfiles/blob/9681a844aadc971dae416836e2dc75feb328b8d3/nvim/.config/nvim/lua/fenil/config/autocmds.lua#L33-L48) autocommand, that's it, 15 lines are all we need. I get the progress in same place as command line without `Hit Enter` prompts.

This works best with `cmdheight = 0` , which prevents `Hit Enter` prompts. `cmdheight` feature was merged in 0.11 itself, I had tried it then but it felt incomplete and weird, but now with ui2 it has the perfect UX, they have really nailed this! There's just one small hiccup, somehow I am not able to see marco record messages, I could reproduce it on master with minimal config so its definitely an upstream issue, hopefully that gets fixed, but till then I have [two hacky autocmds](https://github.com/feniljain/dotfiles/blob/9681a844aadc971dae416836e2dc75feb328b8d3/nvim/.config/nvim/lua/fenil/config/autocmds.lua#L58-L73) which almost do the same job just slightly worse 🙃.

### No vimscript in config

I finally took the leap and ditched out all the vimscript from my config, its completely lua based now! I know I am very late to the party, but I really wanted to keep it around so that I can use it on VMs where vim is the default. Well what finally prompted this move was making a minimal vimscript based vim config, which I can easily drop anywhere and get productive with vim! [Here's](https://github.com/feniljain/dotfiles/blob/main/vim/minimal-vimrc) the minimal config for the curious. I also have a minimal tmux config in the same lines [here](https://github.com/feniljain/dotfiles/blob/main/tmux/.tmux.conf.minimal).

## Bugs discovered

This is an interesting section cause I usually never come across any hiccups when upgrading, neovim is a super polished, heavily tested software. People are out there doing builds super frequently to test latest and greatest! ( I was one of them till few years ago :P )

But this release I came across two interesting bugs!
- macro recording message display with ui2 and cmdheight=0
- a memory segfault with ui2 + invalid rtp and syntax on!

First one we have already discussed, I hope it gets fixed in an upcoming version or even next release is fine :P

For the second one, this is something severe and I was very astonished to come across it! First, link to bug report: https://github.com/neovim/neovim/issues/39815

minimal repro is just:

```lua
require('vim._core.ui2').enable()
vim.cmd [[
set rtp+=$LMAO " Cannot be a non-existing dir, needs to be an env var which does not exist
set syntax
]]
```

So conditions are:
- ui2 should be enabled
- rtp should be set to an env var which does not exist
- syntax should be on

The reason I came across this is because of this weird rtp setting I had set in [my config](https://github.com/feniljain/dotfiles/blob/8ee7f08e3d5c978d3a667ff3479f4150d10ceeda/nvim/.config/nvim/plugin/sets.vim#L17):
```vim
set rtp+=$GOPATH/src/golang.org/x/lint/misc/vim
```

I have no recollection why I had added this, I have messed around a lot with my config over the years, so there are artifacts I still find in weird corners. But main point being, I don't have golang installed in my system, so `GOPATH` is not set and hence satisfies condition we mentioned above. I am not exactly sure what is causing this crash in neovim internally, it needs some investigation 🧐.

But yeah interesting times! I created a new APPNAME with nvim-debug and managed to track it down to this config and also reproduce on latest `master`. I have reported it, let's see if someone upstream picks it up before I get my hands dirty 🏃.

## Sides

I was checking `:checkhealth vim.lsp` and realized I hadn't seen `:checkhealth` in a while, I did that and boom, it was so much cleaner!! Now we have ✅ and ❌ and ⚠️ to show overall health of the features etc, and in general it looked really really clean!

There are also other features like `:restart`, etc which I didn't delve into much cause I wasn't sure how to make use of those features right now. There are bunch of other features too, do go through all the [release notes](https://neovim.io/doc/user/news-0.12/#news-0.12). Amount of things I have realized by reading the release notes in detail is mind blowing, 100% recommended!

## Conclusion

Overall, I am super happy with this new release, only thing I couldn't change this time is colorscheme, I didn't have any lined up to try out 😓.

Except that, I am already looking forward to more amazing things in upcoming releases (multi-cursor looking at you). Till then, chao!
