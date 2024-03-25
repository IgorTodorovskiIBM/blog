---
layout:       post
title:        "Enhancing Vim on z/OS UNIX with Language Server Protocol (LSP) support"
author:       "Igor Todorovski"
header-img:   "img/in-post/tmux_bg.jpeg"
catalog:      true
hidden:       true
tags:
    - z/OS
    - Porting
    - vim
    - Language Server Protocol
    - Autocomplete
---

As a long-time Vim user who develops directly on z/OS UNIX (call me old school!), one feature that I've always wanted is the support for **Language Server Protocols**.

Language Server Protocol (LSP) is a protocol that standardizes the communication between editors and language servers. 

[Microsoft's LSP website](https://microsoft.github.io/language-server-protocol/) summarizes it well:
> The Language Server Protocol (LSP) defines the protocol used between an editor or IDE and a language server that provides language features like auto complete, go to definition, find all references etc. The goal of the Language Server Index Format (LSIF, pronounced like "else if") is to support rich code navigation in development tools or a Web UI without needing a local copy of the source code.

<p style="text-align: center;">
<img src="/blog/img/in-post/vim_front.gif" alt="cobol vim" style="float:center;">
</p>

## Why LSPs are handy for development

LSP's enable code editors to support the following features:
* **Code Completion**: Auto-complete suggestions can signifiantly speed up coding because you no longer need to look up the APIs by browsing the manual or system interfaces.
* **Error Diagnostics**: This feature helps you maintain clean code as you are actively writing code.
* **Code Navigation**: It understands your code and eanbles you to easily navigate to code definitions!

Ideally, we would want to port Neovim to z/OS as it provides better support for LSP's. But Vim also provides such support through it's plugin ecosystem.

So in the meantime, I'm going to describe how we can bring LSP support into Vim on z/OS. That's right, vim!

## Adding LSP Support to Vim

### 1. Installing Vim

Begin by installing or updating Vim on your system using the [zopen package manager](https://github.com/ZOSOpenTools/meta):

```bash
zopen install vim
```

**Note:** you'll need the latest Vim release which includes terminal support (required by the plugins we'll install below).

### 2. Installing the vim-plug plugin manager

Before we set up the LSP support, we'll need to install a vim plugin manager. As of this writing, vim-plug appears to be the most popular. Install it with the following commands:

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

- **vim-lsp/vim-lsp-settings**: These plugin simplify LSP configuration in Vim.
- **asyncomplete.vim/asyncomplete-lsp**: These plugin provide the core functionality for autocompletion in Vim.

Update your `~/.vimrc` file with the following changes:

```vim
call plug#begin()
Plug 'prabirshrestha/vim-lsp'
Plug 'IgorTodorovskiIBM/vim-lsp-settings', { 'branch': 'zos' }

Plug 'prabirshrestha/asyncomplete.vim'
Plug 'prabirshrestha/asyncomplete-lsp.vim'
call plug#end()
```

Restart Vim and run `:PlugInstall` to install plugins.

### 4. Configuring LSP for Python

Now let's explore how to set up an LSP for Python.

**Note**: The Python LSP requires `python3` and `clang` in the PATH.


- First, open a Python file with Vim. `vim-lsp-settings` associates LSPs with the extension, so make sure the extension is `.py`.

```bash
vim hello.py
```

- Use the following command to install the Python language server (pyls).

```vim
:LspInstallServer
```

After installing, restart vim or reload the file. You should now have auto-complete support via the Python LSP:

<p style="text-align: center;">
<img src="/blog/img/in-post/python_vim.gif" alt="python vim" style="float:center;">
</p>


### 5. What about languages like COBOL?

Thanks to the z Open Editor efforts, COBOL has its own LSP. It's available at [https://github.com/eclipse-che4z/che-che4z-lsp-for-cobol](https://github.com/eclipse-che4z/che-che4z-lsp-for-cobol).

Before installing the COBOL LSP, you will need `unzip` and `java` in your path:
```
zopen install unzip
```

Add Java (min version required is 8) to your PATH:
```
export PATH=/usr/lpp/java/IBM/J11.0_64/:$PATH

```

- Now, open a COBOL file with Vim, with `.cbl` as the extension:

```bash
vim hello.cbl
```

- Use the following command to install the COBOL language server:

```vim
:LspInstallServer
```

Restart vim. You should see something like this

<p style="text-align: center;">
<img src="/blog/img/in-post/cobol_vim.gif" alt="python vim" style="float:center;">
</p>

As you'll notice, there's a few issues with identing which will require additional investigation, but overall the autocomplete support seems to be working!

**Next Steps**
* Get REXX/JCL/Clangd LSPs working
* Neovim: Port LuaJit so that we get all of the Neovim features enabled for z/OS!

# Thank you
Thanks for reading and thanks to Mike Fulton, Haritha D and the z/OS Open Tools contributors!
