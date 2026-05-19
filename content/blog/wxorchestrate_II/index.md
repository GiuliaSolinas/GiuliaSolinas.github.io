---
title: 'Mastering Multi-Agent Orchestration and Tool Selection in watsonx Orchestrate'
summary: Some notes about multi-agent orchestration and tool selection.
date: 2026-05-14

image:
  caption: 'Photo by Ryno Marais on Unsplash'
  

authors:
  - admin
  

tags:
  - agentic AI
  - coordination


content_meta:
  trending: true
---

# Mastering Multi-Agent Orchestration and Tool Selection in watsonx Orchestrate

In the first blog, we explored the foundational architectural choices: when to use the Agent Development Kit (ADK) for pro-code development and when to leverage the Agent Builder for speed. Now, let's dive deeper into two critical best practices that separate agentic design from AI assistants and more traditional chatbot solution: **Multi-Agent Orchestration** and **Agent Descriptions**.

---

## Multi-Agent Orchestration: The Supervisor/Manager Pattern

If there's one architectural pattern I had to internalize for watsonx Orchestrate, it's the **Supervisor/Manager agent topology**. This approach leverages the concept of Multi-Agent Systems—a core capability that transforms how complex workflows are handled.

### Why Single Agents Fail at Scale

Do not get me wrong, single agents have been there for a while and can still be deployed with success. Furthermore, single agents that handle a sequence of tasks in parallel can be more performing that a crew of agents accomplishing the same tasks in parallel but without real coordination --see for example, [research](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/) from Google.

However a single agent hits a wall when performing a very complex job that would require parallel tasks and multiple tools, even if they are supported by very powerful, state-of-the-art models. When multiple tools have similar names or work with the same data, the LLM inside a single, large agent can struggle with **tool routing**—the process of selecting the correct tool for a given step. The more tools you pile into one agent, the more overwhelmed the model's reasoning becomes. The result? Incorrect routing, failed tasks, and frustrated users.

There is now quite a lot of evidence on this topic, suggesting that the maximum number of tools an agent should handle should be larger than eight. Allen Chen, distinguished engineer at IBM, has written a great article about it 
on [Medium](https://achan2013.medium.com/how-many-tools-functions-can-an-ai-agent-has-21e0a82b7847).

The solution to this problem is thinking modular and designing a multi-agent framework, in which each agent covers a specialized task with its associated tools and connects loosely with the other agents. The idea is to turn our implementations from monolith agents to lightweight, decentralized yet connected agents. I find multi-agent coordination a very fascinating topic and I recommend reading this [research paper](https://arxiv.org/abs/2507.08616v1) by Grötschla and co-authors to know more about forms of coordination design. 

One of those modes is indeed the "Supervisor/Manager" patter that is at the core of watsonx Orchestrate. Its foundational idea is to have an orchestrating agent taking a overview of the tasks and coordination among its collaborators. Collaborators can pass information among each other and run tasks in parallel or in sequence. Yet, the ultimate check resides to the orchestrator.    

### The Modular Solution

By decomposing a complex task into smaller, specialized agents (e.g., a "Leave Balance Agent" and a "Submission Agent"), you dramatically simplify the decision space. Each sub-agent is given a small, specific set of tools, making its reasoning and selection process highly accurate.

This architecture assigns **specialized roles** to agents, allowing them to excel at their dedicated tasks. A supervising agent (the manager) then handles the overall complexity by **delegating** the user's request to the correct sequence of specialized agents—the orchestration step.

This modular, collaborative approach is the **best practice for complex workflows** in watsonx Orchestrate.

---

## Agent Descriptions: Writing for the Model

Here's a truth that surprises many developers: **the agent description isn't for humans—it's primarily for the AI**.

The description is critically important because it is used by the underlying Large Language Model (LLM) within watsonx Orchestrate to perform reasoning and routing. The LLM reads the description in a `YAML` file to determine the agent's capabilities and decide if it should be invoked to handle a user's request.

### What Makes a Great Description

A well-crafted description should clearly state:

- **Purpose**: What the agent does
- **Inputs**: The type of information it needs
- **Outputs**: What it returns
- **Constraints**: Any key limitations or special instructions

This information is packed in the `YAML` file with this structure,for a customer service agent running on ServiceNow.

```
spec_version: v1
style: default 
name: service_now_agent 
llm: groq/openai/gpt-oss-120b
description: |
    You are an agent who specializes in customer care for a large healthcare institution. You should be compassionate
    to the user.
    
    You are able to help help a user create tickets in service now for processing by a human later. Examples of when to do
    do this include for adding members to plans or helping users with documentation.
instructions: |
    If a user is having difficulty either generating benefits documents or adding additional members to their plan,
    create a new incident for our support team using service_now_create_incident tool. Be compassionate about the user
    facing difficulty.
    
    The output of get_service_now_incidents should be formatted as a github style formatted markdown table.
collaborators: []
tools:
    - create_service_now_incident
    - get_my_service_now_incidents
    - get_service_now_incident_by_number

```
Here some notes about this example:

- The agent's style is set as default, which is the baseline agentic reasoning mode for native agents in wx orchestrate. It can be changed into the ReAct style. 
- The agent is a stand alone one and does not have collaborators. They can be added with additional YAML files stored in the project's workspace and loaded in the environment. You do not need to setup an A2A protocol to let them sync and collaborate, unless we have imported third-party agents that need to coordinate with native ones. 
- There is a list of tools, typically Python files or MCP integration.  

### Clarity Over Complexity

Using clear, non-technical language ensures the LLM interprets the agent's role correctly, which in turn leads to a better user experience. While developers also read the description, think of it primarily as a **prompt for the AI's logic**—and AI needs unambiguous instructions.

Avoid:
- Technical jargon that the LLM won't recognize
- Internal acronyms without context
- Vague or overly brief descriptions

These introduce ambiguity and **increase the risk of incorrect routing decisions**. When in doubt, err on the side of clarity. Ambiguity is poison.

---

## Choosing the Right Tool: Python vs. MCP

We've touched on the role of tools before before, but this topic deserves a deeper explanation. In watsonx Orchestrate, as in the other agentic design frameworks, tools can be developed through Python code or MCP integrations. The question is **when do you use Python tools versus MCP tools?**

### Python Tools (via ADK)

The Agent Development Kit (ADK) allows developers to create custom Python tools. Python is the ideal choice when you need:

- Complex conditional logic (business rules)
- Data manipulation and calculations
- Handling accrual rates, rollovers, and time-based calculations
- Any custom computational logic unique to your business

For example, calculating remaining vacation days based on custom seniority rules? That requires Python.

### MCP Tools (Model Context Protocol)

MCP tools are excellent for **integrating with existing systems**—typically via OpenAPI specifications. They're best suited for:

- Calling standard REST APIs
- Simple, form-based workflows
- Connecting to external services

What they lack is the ability to embed complex, custom computational logic. If you need to transform data in unique ways or implement business rules that don't exist in an external service, reach for Python.

---

## Key Takeaways

- **Decompose complex workflows** into specialized agents to reduce the LLM's cognitive load and improve routing accuracy.
- **Write descriptions for the AI first**—clear, unambiguous descriptions are the difference between reliable routing and failed execution.
- **Choose Python for logic, MCP for integration**—understand the strength of each tool type and use them appropriately.

These are my learning in trying to discover and put in practice the best practices to build multi-agents. The [wx Orchestrate documentation](https://developer.watson-orchestrate.ibm.com/) offers several tips on how to write good agent's descriptions, and offers an extensive list of dos and don'ts. Check it and have fun.  