---
layout:         post
title:          "An AI Workflow for z/OS: Integrating a Tool Server with Crush"
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

As a developer in the [zopen community](https://zopen.community/), I’m always looking for ways to streamline my workflow on z/OS. With agentic AI now at the forefront, I was extremely curious if I could "agentify" parts of my workflow. 
As an experiment, I wanted to see if I could use agentic AI to control `zopen` with natural language. For example, what if you could simply ask your terminal: **"Show me all the installed zopen packages,"** or **"Install git and less for me,"** and it would invoke the zopen tools to perform those operations in a secure way. Thanks to LLM models and the MCP protocol, this is no longer a pipe dream! 


As of this writing, the tools to build a powerful AI assistant for z/OS are here! And thanks to the technologies I will cover today, you now have the flexibility to run the LLMs either on your local workstation or **directly on z/OS itself**.

Our custom zopen MCP Server, which translates AI requests into real commands.

Ollama running the powerful Qwen2 language model directly on my machine.

Crush from Charmbracelet, a terminal-native AI agent that ties everything together.

In this blog, we'll cover the flexible architecture that makes this possible, using our custom `zopen` MCP Server, an AI model served by Ollama or llama.cpp, and the Crush terminal agent from Charmbracelet.

### What is MCP and Where Did It Come From?

The **Model Context Protocol (MCP)** is an open standard designed to let AI models safely and reliably interact with external tools and data. The way I think of it is as a universal adapter. It defines a standard language that any AI agent can use to "talk" to any tool that also speaks that language.

The design of MCP is heavily inspired by the success of the **Language Server Protocol (LSP)**, which I covered in another blog. Just as LSP created a standard for code editors to communicate with language analysis tools, MCP creates a standard for AI agents to communicate with any tool or data source. This open-standard approach encourages a rich ecosystem where any tool provider can make their service AI-ready, and any AI agent, like Crush, can consume it.

### The Setup

You have two primary ways to deploy this solution, depending on your needs, performance, security, etc.

#### Step 1: The Language Model - Your AI's Brain

The core of our agent is the Large Language Model. You can leverage a cloud LLM, but given that we're all about open source I am going to leverage an open-source model. Whether you run it remotely or on z/OS is up to you.

**Option A (Recommended): Run the Model remotely (in my case my Workstation) with Ollama**

For the best performance and responsiveness, running the LLM on a local workstation is the ideal choice as long as you have a GPU or Mac M series.

For simplicity, I am going to install the LLM on my Mac using Ollama.

If you use Homebrew, you can install it via the command line.

CODEBLOCK BASH
brew install ollama
CODEBLOCK

We're going to use qwen3, because it has an 8b parameter model which is not too resource heavy. 

In your terminal, run the following command to get the `qwen3:8b` model:

CODEBLOCK BASH
ollama run qwen3:8b
CODEBLOCK

Now leave the Ollama application running in the background on your workstation.

**Option B: Run the Model Directly on z/OS with llama.cpp**

For a fully self-contained solution, you can now run a model server directly on z/OS. Follow my previous blog on how to set that up - https://igortodorovskiibm.github.io/blog/2023/08/22/llama-cpp/.
One other reason for running LLaMa.cpp locally on z/OS is security. Data on z/OS machines is typically sensitive, and as such, many clients choose to air-gap their systems. (Air-gapping isolates a computer or network from external connections). For sensitive industries like finance and healthcare, local AI models are critical for security by limiting exposure to external threats.

#### Step 2: The `zopen` MCP Server - Our Custom Tool Bridge

This is the component we build ourselves. The `zopen_server.py` script is a simple Python application that acts as a translator. It listens for MCP requests from our AI agent (Crush) and converts them into `zopen` commands. We use **FastMCP**, a high-level Python library that makes building MCP servers simple. Its core components are:

  * **The `FastMCP` Object**: Creates the main server instance (e.g., `mcp = FastMCP("Zopen Tools")`).
  * **The `@mcp.tool` Decorator**: This is the key. It turns a standard Python function into a capability that the AI can discover and use. The function's docstring is crucial, as it tells the AI what the tool does.
  * **The `__main__` Block**: This handles startup and configuration, allowing the server to be run as a standalone script.

When running this server on z/OS, it no longer needs SSH details, as it can execute `zopen` commands directly in its local shell.

#### Step 3: Crush, Your Terminal Agent - Now on z/OS!

Crush is the AI agent that will live in your terminal. As of today, **Crush can be installed and run directly on z/OS** via zopen.

CODEBLOCK BASH
zopen install crush
CODEBLOCK

### Configuring Crush: The `crush.json` File

The `crush.json` file tells Crush how to connect all the pieces. The configuration will differ slightly depending on your chosen deployment model.

#### Configuration for Workstation Model + z/OS Tools

In this setup, Crush runs on z/OS, talks to your `zopen` server on z/OS, but gets its intelligence from the Ollama server running back on your workstation.

CODEBLOCK JSON
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "ollama": {
      "name": "Ollama Workstation",
      "base_url": "http://<your_workstation_ip>:11434/v1/",
      "type": "openai",
      "models": [{"name": "Qwen3 8B", "id": "qwen3:8b"}]
    }
  },
  "mcp": {
    "zopen": {
      "type": "stdio",
      "command": "python",
      "args": ["/path/on/zos/to/zopen_server.py"]
    }
  },
  "permissions": { "allowed_tools": ["mcp_zopen_zopen_list", "..."] }
}
CODEBLOCK

#### Configuration for Everything on z/OS

Here, Crush, the `zopen` server, and the `llama.cpp` server all run on z/OS.

CODEBLOCK JSON
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "llama_cpp_zos": {
      "name": "Llama.cpp on z/OS",
      "base_url": "http://localhost:8080/v1/",
      "type": "openai",
      "models": [{"name": "Qwen3 8B", "id": "qwen3:8b"}]
    }
  },
  "mcp": {
    "zopen": {
      "type": "stdio",
      "command": "python",
      "args": ["/path/on/zos/to/zopen_server.py"]
    }
  },
  "permissions": { "allowed_tools": ["mcp_zopen_zopen_list", "..."] }
}
CODEBLOCK

### The Result: An AI-Powered z/OS Workflow!

With your `crush.json` file in place, simply run `crush` in your z/OS terminal. Crush starts, connects to your chosen model server, and automatically launches your `zopen_server.py` in the background.

> **Me:** "List the zopen packages installed on the mainframe."
>
> **Crush:** *(finds the `mcp_zopen_zopen_list` tool and runs it)*
>
> CODEBLOCK
> ✅ Command successful: zopen list
> 
> Package      Version
> zopen-build  2.0.0
> ...
> 
> CODEBLOCK

# Next Steps: An AI Assistant for Porting Apps
While managing packages is a great first step, the true power of this architecture is in automating complex, knowledge-intensive tasks. My next goal is to tackle one of the most time-consuming parts of my workflow: porting new open-source applications to z/OS.

This process often begins by creating a meta.json file, a task that requires careful analysis of a project's build system. To automate this, I'm developing a new zopen_generate tool for the MCP server.

Unlike the other tools, this one won't just wrap a command. It will use our local LLM to generate code. When I ask Crush to "analyze this Makefile and generate a zopen meta.json file," the zopen_generate tool will be called. Inside the tool, it will construct a specialized prompt, send it to our local Ollama model, and return the generated meta.json file.
