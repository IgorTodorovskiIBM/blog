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

As a long-time Vim user who develops directly on z/OS UNIX, one feature I've always wanted is comprehensive Language Server Protocol (LSP) support.

Language Server Protocol (LSP), initially developed by Microsoft, standardizes communication between editors and language servers. This means that languages can develop one language server that can be sued across multiple editors, saving time for language maintainers and editors.

* Code Completion: LSP's context-aware suggestions have significantly sped up my coding and reduced errors, enhancing productivity.
* Inline Error Diagnostics: LSP's real-time error highlighting is invaluable, helping me maintain high code quality by catching issues as I type.
* Code Navigation: LSP's navigation tools simplify exploring complex codebases, saving me time and reducing frustration.
* Code Refactoring: LSP's automated refactoring tools have improved code readability and efficiency, making my projects easier to maintain.

In this article, I'm going to describe how we can bring LSP support into Vim on z/OS.

## LSP Support in Vim on z/OS

### 1. Installing Vim

Begin by ensuring Vim is installed on your system. If not, run the following command:

```
zopen install vim
```

Note: you'll need the latest Vim release which includes terminal support (required by the plugins we'll install below).

### 2. Installing the vim-plug plugin manager

vim-plug is a popular and efficient plugin manager for Vim. Install it with the following commands:

```
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

Then, create a `~/.vimrc` file (if it doesn't exist), and add this line to enable vim-plug:

```
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
Plug 'mattn/vim-lsp-settings'

Plug 'prabirshrestha/asyncomplete.vim'
Plug 'prabirshrestha/asyncomplete-lsp.vim'
call plug#end()
```

Reload the file or restart Vim and run `:PlugInstall` to install plugins.

**Explanation:**

We've added the necessary plugins for autocompletion: asyncomplete.vim and asyncomplete-lsp.vim.

### 4. Configuring LSP for Python

While vim-lsp-settings can automatically configure LSP for various languages, here's an alternative approach using the `:LspInstallServer` command:

- Open a Python file in Vim.
```
vim hello.py
```

- Use the following command to install the Python language server (pyls):

```vim
:LspInstallServer
```

Vim will prompt you to choose a server from a list. Select `pyls` (or the appropriate server for your Python version).

<insert image>

**Additional Notes:**

- Remember to restart Vim after installing the plugins.
- vim-lsp-settings offers automatic server management, but this method using `:LspInstallServer` provides more control over server installation.

### 5. Enjoying Python Auto-completion!

With this configuration, you should now benefit from LSP features like auto-completion as you type in your Python files.

### 6. What about COBOL? 

- Open a COBOL file in Vim.
```
vim hello.cbl
```

- Use the following command to install the COBOL language server (pyls):

```vim
:LspInstallServer
```

<insert image>

**Additional Notes:**

- This is a basic setup. Explore the documentation for vim-lsp-settings and asyncomplete.vim for more advanced configuration options and customization possibilities.

