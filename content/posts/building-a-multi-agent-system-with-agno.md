---
title: "Building a Multi-Agent System with Agno"
subtitle: "How I selected a framework, integrated NVIDIA Nemotron, and verified real agent execution"
description: "An AXIOM-WILD engineering case study covering Agno selection, vendoring, Python integration, AgentOS, multi-agent design, and a real model execution."
slug: building-a-multi-agent-system-with-agno
categories:
  - AI Engineering
  - Engineering Notes
tags:
  - AXIOM
  - AXIOM-WILD
  - Agno
  - multi-agent systems
  - AgentOS
  - NVIDIA Nemotron
  - Python
draft: false
---

# Building a Multi-Agent System with Agno

### How I selected a framework, integrated NVIDIA Nemotron, and verified real agent execution

I am building AXIOM as a larger engineering effort. Its core architecture is being developed independently of any particular AI model, model provider, or agent framework.

But architectural independence does not mean avoiding existing tools.

I also wanted a practical environment where I could experiment with agent-based intelligence, test existing frameworks, connect real models, and observe how agents behave when they are actually running.

I did not want to introduce a Python agent framework into the Rust-based AXIOM core just to conduct those experiments.

Instead, I created a separate experimental project: **AXIOM-WILD**.

The question was straightforward:

**Could I establish a working multi-agent environment, connect it to a real model provider, expose it through an application API, and verify that agents could execute meaningful tasks?**

Rather than spend the initial effort recreating agent infrastructure, I decided to evaluate an existing framework.

That led me to **Agno**.

---

## Why I selected Agno

I selected [Agno](https://github.com/agno-agi) because it provided an existing foundation for agent construction, model integration, teams, and an application-facing environment through AgentOS.

The version used in this experiment was **Agno 3.1.12**.

My intention was to use the framework as an experimental foundation while keeping WILD-specific development separate.

This was not a decision to make Agno the foundation of AXIOM itself.

The distinction mattered:

- AXIOM remains an independent architectural project.
- AXIOM-WILD is a Python-based experimental environment.
- Agno provides the agent framework for this experiment.
- Model providers remain configurable through an adapter.

I wanted to evaluate actual behaviour, not simply compare framework feature lists.

A framework can look excellent in its documentation. The more useful question is what happens when it is installed, configured, connected to a model, and asked to do real work.

---

## Why I chose to vendor Agno

I considered how to keep the framework and WILD-specific implementation separate.

I chose to maintain a local vendored copy of the upstream framework, while keeping the application-specific implementation in a separate source layer.

This gave me control over the framework source and allowed me to investigate or modify it without mixing application code into the upstream project. I also wanted a manageable mechanism for synchronising upstream changes.

There is a trade-off.

Vendoring gives us control, but it also makes us responsible for reviewing upstream updates and checking that local changes remain compatible.

This was a deliberate choice for the experiment, not a claim that vendoring is always better than using a package dependency.

---

## Establishing the Python environment

I used `uv` to manage the Python environment and dependencies.

The intended dependency arrangement was to use Agno's operating-system extras and an editable local package reference to the vendored source.

The setup was not entirely smooth.

During integration, I encountered a parsing issue involving the parent directory's `pyproject.toml`. The obsolete parent configuration was archived so that it would not interfere with the WILD project.

The environment also exposed a missing `openai` dependency and an unset `WILD_MODEL_ID` configuration value. Both were resolved during integration.

I then checked which Agno package Python was actually importing.

This was important. A successful import alone does not prove that the application is using the source tree I intended. Python could accidentally load another installed copy.

I inspected the package import location and confirmed that Python was loading the locally maintained framework source rather than an unrelated system-wide installation.

That gave me confidence that the application was using the vendored Agno source.

---

## Connecting NVIDIA Nemotron

For the initial model integration, I selected **`nvidia/nemotron-3-ultra-550b-a55b`**, using NVIDIA's free API access for experimentation.

The provider endpoint was:

```text
https://integrate.api.nvidia.com/v1
```

I used Agno's `OpenAILike` adapter to connect to the NVIDIA-compatible endpoint.

This allowed me to use an OpenAI-compatible interface without writing a separate agent implementation specifically for NVIDIA.

The configuration needed to align four things:

1. The provider API key.
2. The provider's API endpoint.
3. The model identifier.
4. The adapter used by the Agno agent.

I also corrected the missing Python dependency and model environment setting encountered during the first construction attempt.

The important lesson was that configuring a provider is not the same as verifying a provider.

A model identifier in a configuration file proves very little. I needed to make a real request and inspect the result.

---

## Defining the WILD agents

The initial WILD implementation contained five agents with separate responsibilities.

| Agent | Responsibility |
|---|---|
| Explorer | Explore questions and pursue interesting lines of inquiry |
| Connector | Find relationships across observations and domains |
| Challenger | Test assumptions and preserve dissent |
| Synthesizer | Consolidate findings without hiding disagreements |
| Archivist | Maintain durable knowledge and provenance |

These roles were deliberately different.

For example, Explorer was instructed to distinguish direct observations, sourced claims, hypotheses, and speculation.

Challenger was instructed to question assumptions and identify alternative explanations.

Synthesizer was instructed not to treat agreement between agents as proof.

This was more than giving five agents different names. Each had its own role instructions and a different responsibility in the overall experiment.

I also created a team named **Wild Council**, initially containing Explorer, Connector, and Challenger.

The team used **broadcast mode**, with a delegation tool designed to send one task to all member agents.

The intended interaction was an open discussion among independent perspectives, rather than forcing all agents through a rigid sequence.

The team instructions encouraged the agents to:

- Invite distinct contributions.
- Make connections and challenge assumptions.
- Avoid premature consensus.
- Preserve disagreement.
- Distinguish hypotheses from evidence.
- Conclude with unresolved questions and possible directions.

It was an attempt to make the agents contribute different perspectives instead of simply repeating the same answer.

This experiment also suggested another possible direction: connecting specialised agents to a chatting platform such as Telegram. A Grok-style conversational experience, supported by multiple agents working behind the scenes, is an idea worth exploring in future development.

---

## Inspecting the running AgentOS service

After establishing the agent configuration, I moved from local construction to inspecting the running application.

AgentOS was available on localhost port `7777`.

I first inspected the registered agents:

```bash
curl -s http://127.0.0.1:7777/agents | python3 -m json.tool
```

This command performs an HTTP GET request against the local AgentOS service.

- `curl` sends the HTTP request.
- `127.0.0.1` refers to the local machine.
- Port `7777` is the AgentOS service port.
- `/agents` retrieves the registered agent definitions.
- `python3 -m json.tool` formats the JSON response for readability.

This was not simply an import test. It queried the running application.

The response showed all five agents:

```text
wild-explorer
wild-connector
wild-challenger
wild-synthesizer
wild-archivist
```

It also showed their individual instructions and model configuration.

For each agent, the returned model information included:

```json
{
  "name": "OpenAILike",
  "model": "nvidia/nemotron-3-ultra-550b-a55b",
  "provider": "OpenAI"
}
```

This confirmed that the registered agent definitions were associated with the configured model adapter and model identifier.

The response also showed session configuration, including history being added to the agent context.

This was useful operational evidence. It demonstrated what the service had actually registered, rather than what I merely intended to register in source code.

---

## Inspecting the Wild Council

Next, I queried the team endpoint:

```bash
curl -s http://127.0.0.1:7777/teams | python3 -m json.tool
```

This retrieved the team definitions registered with AgentOS.

The response showed:

- Team ID: `wild-council`
- Name: `Wild Council`
- Mode: `broadcast`
- Members: Explorer, Connector, and Challenger
- Model adapter: `OpenAILike`
- Model identifier: `nvidia/nemotron-3-ultra-550b-a55b`

Most importantly, the response showed a delegation tool:

```text
delegate_task_to_members
```

Its description specified that one invocation sends a task to every member agent. It was not intended to be called separately for each member.

The team configuration therefore represented a particular experimental approach to multi-agent collaboration.

It was not simply a group of agents sharing the same prompt.

There is an important distinction here: inspecting the team registration proves that the team was configured and exposed by AgentOS. It does not, by itself, prove that every member successfully participated in a real team execution. That requires a separate execution test.

---

## Executing a real Explorer task

The most important verification was the actual agent execution.

I submitted a task through the AgentOS HTTP API.

The command was:

```bash
curl -sS -X POST \
  http://127.0.0.1:7777/agents/wild-explorer/runs \
  -F 'message=Investigate an unexpected connection between distributed computing and biological ecosystems. Identify three interesting hypotheses and distinguish evidence from speculation.' \
  -F 'stream=false' \
  | python3 -m json.tool
```

Let's break this down.

### `curl -sS`

This runs curl in silent mode while still displaying errors.

It avoids unnecessary progress output but does not suppress error messages.

### `-X POST`

This sends an HTTP POST request.

Unlike the earlier inspection commands, this request is not just retrieving configuration. It is asking AgentOS to execute an agent.

### `/agents/wild-explorer/runs`

This is the agent execution endpoint.

The target is the registered Explorer agent.

### `-F 'message=...'`

This sends the task as multipart form data.

The actual instruction was:

> Investigate an unexpected connection between distributed computing and biological ecosystems. Identify three interesting hypotheses and distinguish evidence from speculation.

### `-F 'stream=false'`

This requests a non-streaming response.

The application returns the completed run as a JSON response rather than requiring us to consume a sequence of streamed events.

### `| python3 -m json.tool`

This formats the returned JSON so that the generated content, execution status, and metrics can be inspected.

This was the point where the experiment moved beyond configuration and registration.

I had asked a real agent to perform a task using the configured model provider.

---

## What the agent actually produced

The request completed successfully.

The response included:

- A run ID.
- The agent ID and name.
- A session ID.
- The generated research content.
- Model and provider information.
- Token usage and execution metrics.
- A completed execution status.

The Explorer generated a report containing three proposed connections.

### Hypothesis 1: CRDTs and horizontal gene transfer

The agent explored whether conflict-free replicated data types and horizontal gene transfer in bacteria represent related approaches to conflict-free information integration.

It compared concepts such as merging, state transfer, causality, and biological information retention.

It also distinguished observed parallels from speculative extensions.

### Hypothesis 2: CAP theorem and evolutionary information processing

The agent explored whether consistency, availability, and partition tolerance could provide a useful conceptual framework for understanding information processing in fragmented biological populations.

It proposed testable hypotheses about mutation, reproduction, and population fragmentation.

This was an analogy and research hypothesis, not a demonstrated biological theorem.

### Hypothesis 3: Raft consensus and bacterial quorum sensing

The agent compared distributed leader election and quorum-based coordination with bacterial quorum sensing.

It explored autoinducer signalling, population thresholds, coordinated behaviour, and the difference between fixed cluster membership and biological populations with changing membership.

It also proposed a possible synthetic biology experiment.

The response ended with a synthesis and suggested next investigations.

The important point is that the agent did not merely return a short generic acknowledgement. It completed the requested research task and produced a structured response.

However, the scientific claims and references in that response would still require independent source verification before being used as published scientific conclusions.

**The test demonstrated agent execution, not the truth of every generated claim.**

That distinction matters particularly when using agents for research.

---

## Measuring the execution

The returned response contained the following metrics:

| Metric | Recorded result |
|---|---:|
| Model | `nvidia/nemotron-3-ultra-550b-a55b` |
| Status | `COMPLETED` |
| Input tokens | 141 |
| Output tokens | 2,584 |
| Total tokens | 2,725 |
| Time to first token | 48.13 seconds |
| Recorded duration | 48.13 seconds |

These are the metrics returned for this particular run, not a general benchmark of Agno or NVIDIA.

The response also included a completed run ID and a session ID, demonstrating that AgentOS returned execution and session metadata alongside the generated content.

I was interested in more than whether the model returned text. I wanted to see whether the application could expose a completed execution with structured output and measurable runtime information.

That was achieved for this Explorer run.

---

## What I had actually verified

At this stage, the experiment had progressed beyond framework installation.

I had evidence for several separate milestones:

| Layer | Verification | Result |
|---|---|---|
| Python package | Local Agno import | Successful |
| Framework | Agno 3.1.12 | Verified |
| AgentOS | Import and live API inspection | Verified |
| Agent registration | Five agents returned by `/agents` | Verified |
| Team registration | Wild Council returned by `/teams` | Verified |
| Model configuration | NVIDIA model shown in registered agents and team | Verified |
| Provider integration | Actual Explorer inference | Successful |
| Agent execution | HTTP run request | `COMPLETED` |
| Generated output | Structured research response | Returned |
| AgentOS deployment health | Independent health-check test | Not independently verified |

This separation is important.

An agent can be constructed without being able to call a model.

A model can respond without a complete multi-agent team being validated.

A running API can expose configuration without proving that an execution succeeds.

I tested these stages separately.

---

## What remains to be tested

The successful Explorer execution is a meaningful milestone, but it is not the end of the WILD experiment.

Further work should verify:

1. Wild Council execution through the team endpoint.
2. Whether all three members actually contribute to a broadcast task.
3. Whether the returned member events can be inspected and correlated.
4. Session continuity across multiple requests.
5. AgentOS startup and health behaviour across restarts.
6. Error handling for provider failures, timeouts, and invalid model configuration.
7. Behaviour when different agents produce contradictory conclusions.

These are the next validation targets, not results claimed by this experiment.

I wanted to keep this distinction explicit. A successful individual agent run is a real milestone, but it should not be presented as proof that the complete multi-agent collaboration path has already been validated.

---

## What I learned

This experiment reinforced several engineering principles.

**Do not confuse framework installation with integration.**

A package import is only the beginning. I needed to inspect the running service and execute an actual task.

**Keep the experimental boundary clear.**

I could evaluate Agno in Python without changing AXIOM's Rust architectural direction.

**Treat provider routing as a verifiable integration.**

The provider configuration became meaningful only when the Explorer agent successfully completed a real request.

**Inspect runtime state, not just source configuration.**

The `/agents` and `/teams` endpoints showed what AgentOS had actually registered.

**Measure execution rather than relying on impressions.**

The completed run returned token counts, duration, and execution metadata.

**Separate generated hypotheses from established evidence.**

An agent producing a sophisticated research response does not automatically make its conclusions correct.

---

## The larger lesson

AXIOM-WILD began as a controlled experiment to evaluate an existing agent framework.

I selected Agno, chose to vendor its source, established the local Python environment, configured NVIDIA Nemotron through the OpenAI-compatible adapter, defined five agents, and created the Wild Council.

I then inspected the live AgentOS service through its HTTP endpoints and submitted a real research task to Explorer.

The result was a completed agent execution with structured output and recorded metrics.

This was the meaningful outcome: I moved from a framework selection decision to a working, inspectable agent execution path.

AXIOM-WILD remains an experimental environment. Its purpose is to help me learn what existing agent ecosystems can provide, what their operational behaviour looks like, and which ideas are worth carrying forward into the broader AXIOM ecosystem.

The experiment is valuable precisely because it lets me test those ideas without making the larger architecture dependent on the framework being evaluated.
