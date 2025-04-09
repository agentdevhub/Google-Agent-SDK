# Agent Development Kit (ADK)

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

<img src="docs/assets/agent-development-kit.png" alt="Agent Development Kit Logo" width="150">

**An open-source, code-first Python toolkit for building, evaluating, and deploying sophisticated AI agents with flexibility and control.**

Agent Development Kit (ADK) is designed for developers seeking fine-grained control and flexibility when building advanced AI agents that are tightly integrated with services in Google Cloud. It allows you to define agent behavior, orchestration, and tool use directly in code, enabling robust debugging, versioning, and deployment anywhere – from your laptop to the cloud.

---

## ✨ Key Features

* **Code-First Development:** Define agents, tools, and orchestration logic for maximum control, testability, and versioning.
* **Multi-Agent Architecture:** Build modular and scalable applications by composing multiple specialized agents in flexible hierarchies.
* **Rich Tool Ecosystem:** Equip agents with diverse capabilities using pre-built tools, custom Python functions, API specifications, or integrating existing tools.
* **Flexible Orchestration:** Define workflows using built-in agents for predictable pipelines, or leverage LLM-driven dynamic routing for adaptive behavior.
* **Integrated Developer Experience:** Develop, test, and debug locally with a CLI and interactive dev UI.
* **Built-in Evaluation:** Measure agent performance by evaluating response quality and step-by-step execution trajectory.
* **Deployment Ready:** Containerize and deploy your agents anywhere – scale with Vertex AI Agent Engine, Cloud Run, or Docker.
* **Native Streaming Support:** Build real-time, interactive experiences with native support for bidirectional streaming (text and audio).
* **State, Memory & Artifacts:** Manage short-term conversational context, configure long-term memory, and handle file uploads/downloads.
* **Extensibility:** Customize agent behavior deeply with callbacks and easily integrate third-party tools and services.

## 🚀 Installation

You can install ADK using `pip`:

```bash
pip install google-adk
```

## 🏁 Getting Started

Create your first agent (`my_agent/agent.py`):

```python
# my_agent/agent.py
from google.adk.agents import Agent
from google.adk.tools import google_search

root_agent = Agent(
    name="search_assistant",
    model="gemini-2.0-flash",
    instruction="You are a helpful assistant. Answer user questions using Google Search when needed.",
    description="An assistant that can search the web.",
    tools=[google_search]
)
```

Create `my_agent/__init__.py`:

```python
# my_agent/__init__.py
from . import agent
```

Run it via the CLI (from the directory *containing* `my_agent`):

```bash
adk run my_agent
```

Or launch the dev UI:

```bash
adk web
```

For a full step-by-step guide, check out the quickstart or sample agents.

## 📚 Resources

Explore the full documentation for detailed guides on building, evaluating, and deploying agents:

* **[Get Started](get-started/introduction.md)**
* **[Build Agents](build/agents.md)**
* **[Browse Sample Agents](learn/sample_agents/)**
* **[Evaluate Agents](evaluate/evaluate-agents.md)**
* **[Deploy Agents](deploy/overview.md)**
* **[API Reference](guides/reference.md)**
* **[Troubleshooting](guides/troubleshooting.md)**

## 🤝 Contributing

We welcome contributions from the community! Whether it's bug reports, feature requests, documentation improvements, or code contributions, please see our [**Contributing Guidelines**](./CONTRIBUTING.md) to get started.

## 📄 License

This project is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE) file for details.

---

*Happy Agent Building!*
