---
layout:         post
title:          "I’ve Got a Crush on AI: Automating your z/OS Workflows"
author:         "Igor Todorovski"
header-img:     "img/in-post/zos_ai_workflow.png"
catalog:        true
tags:
    - z/OS
    - zopen
    - AI
    - Agentic
    - Crush
    - MCP
    - Ollama
---

As a developer in the [zopen community](https://zopen.community/), I’m always looking for ways to streamline my workflow on z/OS. With the rise of agentic AI, I started wondering if I could “agentify” my z/OS workflow.  

Imagine being able to use natural language to automate tasks with familiar tools. For example, what if I could simply ask: **“Show me all of the installed zopen packages that need updating”** or **“Install everything I need for web development”**, and have the AI use its knowledge and the zopen package manager to carry out the task?

Thanks to Large Language Models (LLMs) and the Model Context Protocol (MCP), this is now possible on z/OS!  

<div style="text-align: center;">

  <a href="/blog/img/in-post/crush_zopen.gif" class="fancybox" data-fancybox>
    <img src="/blog/img/in-post/crush_zopen.gif" alt="Crush" title="Crush Image" style="display: block; margin: 0 auto;">
    Agentic AI running directly on z/OS!
  </a>
</div>

To make this work, we combine several key technologies:  

* A custom [zopen MCP Server](https://github.com/IgorTodorovskiIBM/zopen-mcp-server) that translates AI requests into real commands.  
* [Ollama](https://github.com/ollama/ollama) or [LLama.cpp](https://github.com/ggml-org/llama.cpp), running an open-source language model directly on your workstation or on z/OS.  
* [Crush](https://github.com/charmbracelet/crush), an open-source terminal-native AI agent that ties everything together.  


### What is MCP 

Before we start, let's explain MCP. The **Model Context Protocol (MCP)** is an open standard designed to let AI models safely and reliably interact with external tools (like git, grep and in our case zopen) and data. The way I think of it is as a universal adapter. It defines a standard language that any AI agent can use to "talk" to any tool that also speaks that language.

The design of MCP is heavily inspired by the success of the **Language Server Protocol (LSP)**, which I covered in another [blog](https://igortodorovskiibm.github.io/blog/2024/04/18/vim-autocomplete/). Just as LSP created a standard for code editors to communicate with language analysis tools, MCP creates a standard for AI agents to communicate with any tool or data source. This open-standard approach encourages a rich ecosystem where any tool provider can make their service AI-ready, and any AI agent, like Crush, can consume it.

### Prerequisites

Before we begin, you'll need the following:

- Go 1.23 or later. For z/OS, you can get Go from the [IBM Open Enterprise SDK for Go product page](https://www.ibm.com/products/open-enterprise-sdk-go-zos).
- An environment with `zopen` installed (either locally or on a remote z/OS system). Use these [instructions](https://zopen.community/#/Guides/QuickStart) if you don't have zopen installed.

### Step 1: Choose Your Language Model

The core of our agent is the Large Language Model. You can leverage any LLM, but given that the zopen community is all about open-source I am going to leverage an open-source model. Whether you run it remotely or on z/OS is up to you.

**Option A (Recommended): Run the Model remotely (in my case my Workstation) with Ollama**

For the best performance and responsiveness, running the LLM on a workstation (as opposed to directly on z/OS) is the ideal choice as long as you have a GPU or Mac M series. Choosing the right model is important and will impact how well the automation workflow behaves.

For simplicity, I am going to install the LLM on my Mac using Ollama.

If you use Homebrew, you can install it via the command line.

```bash
brew install ollama
```

I'm going to use Qwen3, because it's a fairly lightweight model (8b parameter model) and it does well at most coding tasks. If you have enough VRAM, then I would suggest [Qwen-3-Coder](https://huggingface.co/Qwen/Qwen3-Coder-480B-A35B-Instruct) as it is tuned for agentic coding.

In your terminal, run the following command to get the `qwen3:8b` model:

```bash
# On your workstation
ollama serve &
ollama run qwen3:8b
```

Now leave the Ollama application running in the background on your workstation.

**Option B: Run the Model Directly on z/OS with llama.cpp**

For a fully self-contained solution, you can now run a model server directly on z/OS. Follow my [previous blog](https://igortodorovskiibm.github.io/blog/2023/08/22/llama-cpp/) on how to set that up.
One reason for running LLaMa.cpp locally on z/OS is security. Data on z/OS machines is typically sensitive, and as such, many clients choose to air-gap their systems. (Air-gapping isolates a computer or network from external connections). For sensitive industries like finance and healthcare, local AI models are critical for security by limiting exposure to external threats.

### Step 2: The zopen MCP Server - Our Custom Tool Bridge

This is the component we built ourselves. The `zopen-mcp-server` is a Go application that acts as a translator. It listens for MCP requests from our AI agent (Crush) and converts them into `zopen` commands. We use the official [go mcp sdk](https://github.com/modelcontextprotocol/go-sdk) to create the MCP server. Its core components are:

  * **The `mcp.NewServer` Function**: Creates the main server instance.
  * **The `mcp.AddTool` Function**: This is the key. It turns a Go function into a capability that the AI can discover and use. The tool's description is crucial, as it tells the AI what the tool does.
  * **The `main` Function**: This handles startup and configuration, allowing the server to be run as a standalone executable.

The end result is our zopen mcp server, available at [https://github.com/IgorTodorovskiIBM/zopen-mcp-server](https://github.com/IgorTodorovskiIBM/zopen-mcp-server).

To install the server on z/OS, you can use `go install`:

```bash
# On z/OS
go install github.com/IgorTodorovskiIBM/zopen-mcp-server@v1.0.0
```

#### Available Tools in the zopen MCP server

The `zopen-mcp-server` now supports the following commands:

- `zopen_list`: Lists information about zopen community packages.
- `zopen_query`: Lists local or remote info about zopen community packages.
- `zopen_install`: Installs one or more zopen community packages.
- `zopen_remove`: Removes installed zopen community packages.
- `zopen_upgrade`: Upgrades existing zopen community packages.
- `zopen_info`: Displays detailed information about a package.
- `zopen_version`: Displays the installed zopen version.
- `zopen_init`: Initializes the zopen environment.
- `zopen_clean`: Removes unused resources.
- `zopen_alt`: Switches between different versions of a package.

You can also run it remotely if you choose to run your Crush AI agent on your workstation.

## Step 3: Crush, Your Terminal Agent - Now on z/OS!

Crush is an AI agent that runs directly in your z/OS terminal.  

According to the [Crush repository](https://github.com/charmbracelet/crush), it offers a rich set of features:  

**Features**  
- **Multi-Model** – Choose from a wide range of LLMs or add your own.
- **Built-in Tools** – Crush comes with support for a variety of tools out-of-the-box to interact with your system, including bash, download, edit, grep, ls, rg, and view.
- **Flexible** – Switch LLMs mid-session while preserving context.  
- **Session-Based** – Maintain multiple work sessions and contexts per project.  
- **LSP-Enhanced** – Leverages LSPs for additional context, just like your editor.  
- **Extensible** – Add capabilities via MCPs (http, stdio, and sse).  We use the stdio transport for increased security.

And now, thanks to the zopen community, **Crush has been ported to z/OS**. You can install it using zopen:

```bash
# On z/OS
zopen install crush
```

Before we initialize crush, we need to configure it to use our local LLM.

### Configuring Crush: The `crush.json` File

The `crush.json` file tells Crush how to connect all the pieces. The configuration will differ slightly depending on your chosen deployment model.

#### Configuration for Workstation Model + z/OS Tools

In this setup, Crush runs on z/OS, talks to your `zopen` server on z/OS, but gets its knowledge from the Ollama server running back on your workstation.

First, create the file in the standard location:

```bash
mkdir  ~/.local/share/crush/
vim  ~/.local/share/crush/crush.json
```

Copy and adjust these contents appropriately:
```json
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "ollama": {
      "name": "Ollama Workstation",
      "base_url": "http://<your_ip>:11434/v1/",
      "type": "openai",
      "models": [{"name": "Qwen3 8B", "id": "qwen3:8b"}]
    }
  },
  "mcp": {
    "zopen": {
      "type": "stdio",
      "command": "/path/on/zos/to/zopen-mcp-server"
    }
  },
}
```

### The Result: An AI-Powered z/OS Workflow!

With your `crush.json` file in place, simply run `crush` in your z/OS terminal. Crush will start, connect to your chosen LLM server, and launch your `zopen-mcp-server` as a subprocess. 

From here, you can start giving it tasks to automate!  

> Can you clone the git repo https://github.com/git/git (with a depth of 1) and discover the build dependencies needed to compile it? Check if zopen already has these dependencies available.  

<style>
  .fancybox {
    display: block;       /* make each anchor fill full width */
    margin: 0;            /* remove default margins */
    padding: 0;           /* remove any padding */
  }
  .fancybox img {
    display: block;       /* prevents inline spacing */
    margin: 0;            /* remove image margins */
    padding: 0;           /* remove image padding */
  }
</style>

<a href="/blog/img/in-post/crush1.png" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush1.png" alt="Crush" title="Crush Image">
</a>
<a href="/blog/img/in-post/crush2.png" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush2.png" alt="Crush" title="Crush Image">
</a>
<a href="/blog/img/in-post/crush3.png" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush3.png" alt="Crush" title="Crush Image">
</a>

As you can see, Crush already supports many tools, including `git` and `view`, which allow it to clone repositories and inspect their contents. It also leverages our `zopen-mcp-server` to check package availability on z/OS.  

In this example, Crush cloned the Git repository, inspected the `INSTALL` file, and produced a dependency analysis:  

- **Core dependencies found in zopen**: `zlib`, `bash`, `Perl`, `libcurl`, `gettext`, `python`.  
- **Optional dependencies missing or partially available**: `expat`, `openssl`, `tcl/tk`, `xmlto`.  
- **Documentation tools not explicitly packaged in zopen**: `asciidoc`, `makeinfo`, `docbook2X`, `dblatex`, `docbook-xsl`.  

Crush was able to summarize this automatically, saving the manual effort of digging through build scripts.  

Pretty awesome! 

Feel free to experiment. You can ask it anything, from understanding your codebase to making changes to it!

# Next Steps: An AI Assistant for Porting Apps
While package management is a great first step, the real power of agentic AI lies in automating complex, knowledge-intensive tasks. My next goal is to tackle one of the most time-consuming parts of my workflow: **porting new open-source applications to z/OS**.  

Agentic AI can also assist with debugging issues during the porting process—and much more. Exciting times ahead!

Thank you to Mike Fulton, Bill O'Farrell and Andrew Sica for reviewing this post and providing valuable feedback!  
