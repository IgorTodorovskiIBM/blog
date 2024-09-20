---
layout:       post
title:        "Enhancing Vim on z/OS UNIX with Language Server Protocol (LSP) support"
author:       "Igor Todorovski"
header-img:   "img/post-bg-vim.jpg"
catalog:      true
tags:
    - z/OS
    - Porting
    - vim
    - Language Server Protocol
    - Autocomplete
---

As a long-time Vim user who develops directly on z/OS UNIX (yes, call me old school!), one feature that I've always wanted is the support for **Language Server Protocols**.

If you're not aware of Language Servers and the Language Server Protocol (LSP), it's a protocol that standardizes the communication between editors (VS Code, Neovim, Emacs, VIM) and language servers. 

[Microsoft's LSP website](https://microsoft.github.io/language-server-protocol/) summarizes it well:
> The Language Server Protocol (LSP) defines the protocol used between an editor or IDE and a language server that provides language features like auto complete, go to definition, find all references etc. The goal of the Language Server Index Format (LSIF, pronounced like "else if") is to support rich code navigation in development tools or a Web UI without needing a local copy of the source code.

Language servers use JSON-RPC to communicate between the development tool and the language server. Using JSON-RPC allows the server to be implemented in any language.
The protocol itself defines a set of features that can be supported by the server, such as code completion and go-to-definition. 

<p style="text-align: center;">
<img src="/blog/img/in-post/vimgo.gif" alt="tmux.cpp" style="float:center;">
Vim on z/OS with the Go Language Server 
</p>

For more information on the LSP architecture, visit Microsoft's [docs](https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/). To find out what languages implement language servers, check out this [page](https://microsoft.github.io/language-server-protocol/implementors/servers/).

## Why LSP support is great for editors

LSP's enable code editors to support the following features without having to write their own Langauge parsers and analyzers:

* **Code Completion**: Auto-complete suggestions can signifiantly speed up coding because you no longer need to look up the APIs by browsing the manual or system interfaces.
* **Error Diagnostics**: This feature helps you maintain clean code as you are actively writing new code.
* **Code Navigation**: LSPs understand your code and eanbles you to easily navigate to code definitions and call sites.

Ideally, to get the best support for LSPs in a terminal editor, we would want to port Neovim to z/OS. Unfortuantely, the neovim port is still in progress!

So in the meantime, I'm going to describe how we can provide LSP support to Vim on z/OS.

## Adding LSP Support to Vim

### 1. Installing Vim

Begin by installing or updating Vim on your system using the [zopen package manager](https://github.com/zopencommunity/meta):

```bash
zopen install vim
```

**Note:** you'll need the latest Vim release which includes terminal support (required by the plugins we'll install below).

### 2. Installing the vim-plug plugin manager

Before we set up the LSP support, we'll need to install a vim plugin manager. As of this writing, [vim-plug](https://github.com/junegunn/vim-plug) appears to be the most popular. Install it with the following commands:

```bash
# You will need curl for this operation (you can install it via zopen as above)
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

Then, create a `~/.vimrc` file (if it doesn't exist), and add these lines to enable vim-plug:

```vim
call plug#begin()
" Add plugins here
call plug#end()
```

In the next step, we'll install the LSP and auto-complete plugins through vim-plug.

### 3. Installing LSP and Autocompletion Plugins

We'll now use vim-plug to install the following plugins:

- **vim-lsp/vim-lsp-settings**: These plugins simplify LSP configuration in Vim.
- **asyncomplete.vim/asyncomplete-lsp**: These plugins provide the core functionality for autocompletion in Vim.

Update your `~/.vimrc` file with the following changes:

```vim
call plug#begin()
Plug 'prabirshrestha/vim-lsp'
Plug 'IgorTodorovskiIBM/vim-lsp-settings', { 'branch': 'zos' }

Plug 'prabirshrestha/asyncomplete.vim'
Plug 'prabirshrestha/asyncomplete-lsp.vim'
call plug#end()
```
**Note:** We're using my fork of vim-lsp-settings since some of the LSP settings provided in the official repo do not currently work.

Restart Vim and run `:PlugInstall` to install plugins.

### 4. Configuring LSP for Python

Now let's explore how to set up an LSP for Python.

**Note**: The Python LSP requires `python3` and `clang` in the PATH.


First, open a Python file with Vim. vim-lsp-settings associates LSPs with the language extension, so make sure the extension is `.py`.

```bash
vim hello.py
```

Use the following command to install the Python language server (pyls).

```vim
:LspInstallServer
```

After installing, restart vim or reload the file. You should now have auto-complete support via the Python LSP:

<p style="text-align: center;">
<img src="/blog/img/in-post/python_vim.gif" alt="python vim" style="float:center;">
</p>

### 5. Configuring LSP for Go 

Now let's explore how to set up the language server for Go.

**Note**: The Go language server requires the `go` compiler in the PATH. Also make sure to disable CGO with `export CGO_ENABLED=0`.

First, open a Go file with Vim. vim-lsp-settings associates LSPs with the language extension, so make sure the extension is `.go`.

```bash
vim main.go
```

Use the following command to install the Go language server (gopls).

```vim
:LspInstallServer
```

After installing, restart vim or reload the file. You should now have auto-complete support via the Go language server.

<p style="text-align: center;">
<img src="/blog/img/in-post/vimgo.gif" alt="python vim" style="float:center;">
</p>


**Next Steps**
* Test out more languages!
* Neovim! Port LuaJit so that we get all of the Neovim features enabled for z/OS!

# Thank you
Thanks for reading and thanks to Mike Fulton, Haritha D, Peter Haumer and the z/OS Open Tools contributors!
