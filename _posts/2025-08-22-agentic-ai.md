---
layout:         post
title:          "I’ve Got a Crush on AI: Building Smarter Workflows for z/OS”
author:         "Igor Todorovski"
header-img:     "img/in-post/zos_ai_workflow.png"
catalog:        true
hidden:         true
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

Imagine being able to use natural language to automate tasks with familiar tools. For example, what if I could simply ask: **“Show me all of the installed zopen packages”** or **“Install everything I need for web development”**, and have the AI use the zopen package manager to carry it out?

Thanks to Large Language Models (LLMs) and the Model Context Protocol (MCP), this is now possible on z/OS!  

<a href="/blog/img/in-post/crush_zopen.gif" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush_zopen.gif" alt="Crush" title="Crush Image">
</a>  

To make this work, we combine several key technologies:  

* A custom zopen MCP Server that translates AI requests into real commands.  
* Ollama or LLama.cpp, running an open-source language model directly on your workstation or on z/OS.  
* [Crush](https://github.com/charmbracelet/crush), a terminal-native AI agent that ties everything together.  


### What is MCP 

Before we start, let's explain MCP. The **Model Context Protocol (MCP)** is an open standard designed to let AI models safely and reliably interact with external tools (like git, grep and in our case zopen) and data. The way I think of it is as a universal adapter. It defines a standard language that any AI agent can use to "talk" to any tool that also speaks that language.

The design of MCP is heavily inspired by the success of the **Language Server Protocol (LSP)**, which I covered in another [blog](https://igortodorovskiibm.github.io/blog/2024/04/18/vim-autocomplete/). Just as LSP created a standard for code editors to communicate with language analysis tools, MCP creates a standard for AI agents to communicate with any tool or data source. This open-standard approach encourages a rich ecosystem where any tool provider can make their service AI-ready, and any AI agent, like Crush, can consume it.

### Prerequisites

Before we begin, you'll need the following:

- Go 1.23 or later. For z/OS, you can get Go from the [IBM Open Enterprise SDK for Go](https://www.ibm.com/products/open-enterprise-sdk-go-zos).
- An environment with `zopen` installed (either locally or on a remote z/OS system). Use these [instructions](https://zopen.community/#/Guides/QuickStart) if you don't have zopen installed.

### Step 1: Choose Your Language Model

The core of our agent is the Large Language Model. You can leverage a any LLM, but given that the zopen community is all about open-source I am going to leverage an open-source model. Whether you run it remotely or on z/OS is up to you.

**Option A (Recommended): Run the Model remotely (in my case my Workstation) with Ollama**

For the best performance and responsiveness, running the LLM on a local workstation is the ideal choice as long as you have a GPU or Mac M series.

For simplicity, I am going to install the LLM on my Mac using Ollama.

If you use Homebrew, you can install it via the command line.

```bash
brew install ollama
```

I'm going to use qwen3, because it has an 8b parameter model which is not too resource heavy and it does well at coding tasks.

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

#### Step 2: The `zopen` MCP Server - Our Custom Tool Bridge

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

#### Step 3: Crush, Your Terminal Agent - Now on z/OS!

Crush is the AI agent that will run on your z/OS terminal. As of today, **Crush can be installed and run directly on z/OS** via zopen.

```bash
# On z/OS
zopen install crush
```

Before we initialize crush, we need to configure it to use our local LLM.

### Configuring Crush: The `crush.json` File

The `crush.json` file tells Crush how to connect all the pieces. The configuration will differ slightly depending on your chosen deployment model.

#### Configuration for Workstation Model + z/OS Tools

In this setup, Crush runs on z/OS, talks to your `zopen` server on z/OS, but gets its knowledge from the Ollama server running back on your workstation.

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

With your `crush.json` file in place, simply run `crush` in your z/OS terminal. Crush starts, connects to your chosen model server, and automatically launches your `zopen-mcp-server` in the background.

Let's now ask it a question.

> Can you clone the git repo https://github.com/git/git (with a depth of 1) and discover the build dependencies that are needed to build it. Check if zopen has these build dependencies present

<a href="/blog/img/in-post/crush1.png" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush1.png" alt="Crush" title="Crush Image">
</a>  
<a href="/blog/img/in-post/crush2.png" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush2.png" alt="Crush" title="Crush Image">
</a>  
<a href="/blog/img/in-post/crush3.png" class="fancybox" data-fancybox>
  <img src="/blog/img/in-post/crush3.png" alt="Crush" title="Crush Image">
</a>  

As you can see, crush already has support for many tools including git and view to be able to clone and read content. It's also using our zopen mcp server.

It was able to nicely summarize the build depepdencies by inspecting the INSTALL script.

Pretty awesome!

### Available Tools in the zopen MCP server

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

# Next Steps: An AI Assistant for Porting Apps
While managing packages is a great first step, the true power of agentic AI is in automating complex, knowledge-intensive tasks. My next goal is to tackle one of the most time-consuming parts of my workflow: porting new open-source applications to z/OS!
