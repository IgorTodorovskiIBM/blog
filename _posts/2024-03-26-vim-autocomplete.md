---
layout:       post
title:        "Enhancing Vim on z/OS UNIX with Language Server Protocol (LSP)"
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

# Enhancing Vim on z/OS UNIX with Language Server Protocol (LSP)

As a long-time Vim user who has been developing directly on z/OS UNIX (yes, I don't use VS Code!), one feature that I've always wanted is support for Language Server Protocol.

Language Server Protocol (LSP), developed by Microsoft, is a protocol that standardizes communication between editors and language servers. This means language developers can create a single server that works across multiple editors, saving them and editor developers tons of time.

## Why LSPs are awesome:

* Code Completion: LSP's context-aware suggestions can signifiantly speed up coding and reduce errors.
* Inline Error Diagnostics: real-time error highlighting can help maintain clean code as you are developing.
* Code Navigation: LSP's navigation tools simplify exploration of even the most complex codebases. This saves developers time and reduces frustration by offering a more efficient way to traverse code structures.

While Neovim support for LSPs is a foundational, the existing port to z/OS is incomplete. 
In this article, I'm going to describe how we can bring LSP support into Vim on z/OS. Yes, to vim!

## LSP Support in Vim on z/OS

### 1. Installing Vim

Begin by installing Vim on your system using the [zopen package manager](https://github.com/ZOSOpenTools/meta):

```bash
zopen install vim
```

Note: you'll need the latest Vim release which includes terminal support (required by the plugins we'll install below).

### 2. Installing the vim-plug plugin manager

vim-plug is a popular and efficient plugin manager for Vim. Install it with the following commands:

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

Then, create a `~/.vimrc` file (if it doesn't exist), and add this line to enable vim-plug:

```vim
call plug#begin()
" Add plugins here
call plug#end()
```

In the next step, we'll add the LSP plugins.

### 3. Installing LSP and Autocompletion Plugins

Utilize vim-plug to install the following plugins:

- **vim-lsp-settings**: This plugin, as mentioned earlier, simplifies LSP configuration in Vim.
- **asyncomplete.vim**: This plugin provides the core functionality for asynchronous autocompletion in Vim.
- **asyncomplete-lsp.vim**: This companion plugin bridges the gap between asyncomplete.vim and LSP servers, enabling autocompletion suggestions based on the language server you're using (e.g., pyls for Python).

Here's the updated code for your `~/.vimrc` file:

```vim
call plug#begin()
Plug 'prabirshrestha/vim-lsp'
Plug 'IgorTodorovskiIBM/vim-lsp-settings', { 'branch': 'zos' }

Plug 'prabirshrestha/asyncomplete.vim'
Plug 'prabirshrestha/asyncomplete-lsp.vim'
call plug#end()
```

Reload the file or restart Vim and run `:PlugInstall` to install plugins.

**Explanation:**

We've added the necessary plugins for autocompletion: asyncomplete.vim and asyncomplete-lsp.vim.

### 4. Configuring LSP for Python

While vim-lsp-settings can automatically configure LSP for various languages, here's an alternative approach using the `:LspInstallServer` command:

The Python LSP requires python3 so make sure it is in your path. 


- Open a Python file in Vim.

```bash
vim hello.py
```

- Use the following command to install the Python language server (pyls). You will also need Clang installed.

```vim
:LspInstallServer
```

Vim will prompt you to choose a server from a list. Select `pyls` (or the appropriate server for your Python version).

<p style="text-align: center;">
<img src="/blog/img/in-post/python_vim.gif" alt="python vim" style="float:center;">
</p>

**Additional Notes:**

- Remember to restart Vim after installing the plugins.
- vim-lsp-settings offers automatic server management, but this method using `:LspInstallServer` provides more control over server installation.

### 5. Enjoying Python Auto-completion!

With this configuration, you should now benefit from LSP features like auto-completion as you type in your Python files.

### 6. What about COBOL? 

Prereqs: unzip and java

```
zopen install unzip
```

Add Java (min version required is 8) to your PATH:
```
export PATH=/usr/lpp/java/IBM/J11.0_64/:$PATH

```

- Open a COBOL file in Vim.

```bash
vim hello.cbl
```

- Use the following command to install the COBOL language server (pyls):

```vim
:LspInstallServer
```

Restart vim. You should see something like this

<p style="text-align: center;">
<img src="/blog/img/in-post/cobol_vim.gif" alt="python vim" style="float:center;">
</p>

**Additional Notes:**

- This is a basic setup. Explore the documentation for vim-lsp-settings and asyncomplete.vim for more advanced configuration options and customization possibilities.

