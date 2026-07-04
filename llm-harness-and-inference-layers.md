# Understanding LLM Harnesses: From Model to Layers to LangChain

**Date:** 2026-07-05
**Context:** A series of questions about how agentic AI tools (Claude Code, LangChain-based agents) actually work, starting from "what is a harness" and building up through statelessness, the API contract, local open-source LLMs, inference-time compute, and finally the LangChain ecosystem of tooling.

---

## Background

This started from one simple question: when people talk about "the harness" for an LLM, what actually is that? I already understood client/server and routing from web dev, so I used that as the scaffolding to build up the rest — what a model actually is (a stateless function), what surrounds it to make it useful, whether that changes when you run an open-source model on your own machine, and where the "spend more compute to think harder" idea fits. It ended with a genuinely useful landing spot: the LangChain/LangGraph/LangSmith/Langfuse ecosystem, which turned out to be a clean real-world instance of everything covered before it.

---

## 1. What Is a Harness

An LLM by itself is just a function: **text in → text out.** One call. It has no memory, can't run code, can't read files, can't loop, can't check its own work. That's the raw model.

A **harness** is all the code *around* that function that turns "a thing that predicts the next token" into "a system that gets tasks done." The name is borrowed from testing/hardware — a test harness is the rig that feeds inputs to a component and captures outputs. Same idea here: it's the scaffolding that drives the model.

**The mental model that makes it click:**

> The model is the **engine**. The harness is the **rest of the car** — steering, transmission, dashboard, brakes. An engine on a workbench revs but goes nowhere. The harness is what makes it *drive*.

Or in one line: **the model reasons; the harness gives it hands, memory, and a steering loop.**

---

## 2. The Three Layers

Splitting "everything around the model" into layers makes it concrete. Using Claude Code as the running example (its client-side code actually leaked in March 2026 via a stray npm source map — which is a separate story, but it's what makes this so easy to point at concretely: the ~513k lines that leaked *were* the harness, layer 3 below):

**Layer 1 — the model (server-side, the "brain")**
The neural network weights. A stateless function. Every API call is independent; the model remembers nothing on its own between calls.

**Layer 2 — inference engine (server-side, around the model)**
Between the network and the raw weights there's real machinery:
- **Tokenization** — turning text into token IDs and back.
- **Sampling** — temperature, top-p, top-k: how a probability distribution becomes an actual chosen token.
- **Tool-call formatting** — parsing the model's raw output into a structured "call this tool with these arguments."
- **KV-cache management** — reusing prior computation so generation doesn't restart from scratch every token.
- **Streaming** — sending tokens back one at a time as they're generated.

**Layer 3 — the agent harness (client-side, the part that leaked from Claude Code)**
This is what people almost always mean by "harness" in the agentic-AI sense:
- **The agent loop** — call the model, execute what it asks for, feed the result back, repeat.
- **The system prompt** — the instructions defining persona, rules, behavior.
- **Tool definitions** — schemas telling the model what tools exist and their arguments.
- **Context management** — deciding what to keep, drop, or summarize as the conversation grows (Claude Code's leaked code showed exactly this: message flags like `isCompactSummary`, `isVisibleInTranscriptOnly`, and multiple compaction strategies).
- **Memory/state** — since the model is stateless, the harness holds conversation history and resends the relevant slice every turn.
- **Guardrails and permissions** — asking before dangerous actions, sandboxing.
- **Parsing and recovery** — interpreting model output, retrying on malformed tool calls.

| Layer | What it does | Where it lives (cloud Claude) |
|---|---|---|
| 1 — Model | Predicts next token | Anthropic's GPUs, secret |
| 2 — Inference engine | Tokenize, sample, KV-cache, tool-call parsing | Anthropic's servers, invisible to you |
| 3 — Agent harness | Loop, system prompt, tools, memory, guardrails | Your machine (e.g. the Claude Code CLI) |

---

## 3. Underneath Layer 2: Where the Actual Arithmetic Happens

This sits right below Layer 2, at the boundary between the inference engine and physical hardware. The weights file is inert data — numbers sitting on disk. The actual multiplication happens on a chip: a GPU (or TPU, or CPU), executed by specialized math libraries that the inference engine calls into.

The chain, concretely:

1. **The inference engine (Layer 2)** — llama.cpp, vLLM, or whatever a cloud provider runs internally — loads the weights into memory and knows the *architecture*: which numbers get multiplied with which, in what order, how many layers, etc. But the engine itself doesn't do arithmetic by hand in a loop; that would be catastrophically slow.

2. **It calls into a specialized math library.** On NVIDIA GPUs, that's libraries like **cuBLAS** and **cuDNN** — vendor-written, extremely optimized routines specifically for matrix multiplication and related linear-algebra operations. On CPUs, it's things like **BLAS/MKL**. These libraries are the actual translators between "multiply this matrix by that matrix" and instructions the chip understands.

3. **The chip itself does the arithmetic.** A GPU has thousands of small cores (and, for this workload specifically, **tensor cores** — circuits purpose-built for the multiply-accumulate operations that matrix multiplication is made of) that all crunch numbers in parallel. This parallelism is *why* GPUs are used at all — a matrix multiply is millions of independent multiply-and-add operations, and a GPU can do huge batches of them simultaneously, where a CPU would do them mostly serially and be far slower.

So, directly:
- **Locally:** *your own GPU* (or CPU, if no GPU) does the actual compute. Ollama/llama.cpp is the engine that hands the work to your hardware via these math libraries.
- **Claude Code / any cloud model:** *the provider's data-center GPUs* (or custom AI accelerators, depending on the provider) do the actual compute. Their inference-serving stack is the engine that hands the work to that hardware.

The pattern is identical in both cases — it's just a question of whose chip is doing the multiplying. Neither the model weights nor the inference engine "compute" anything by themselves in a symbolic sense — they're the data and the orchestration; the actual physical arithmetic (billions of multiply-accumulate operations per token) happens on the silicon, driven by these vendor math libraries.

---

## 4. Statelessness and the API Contract

The core fact underlying everything: **the model is stateless.** One API call, one shot — nothing persists to the next call. If I ask "my name is Nikhil" in one call and "what's my name?" in the next, it won't know, unless *I* resend the whole conversation.

That's not a limitation so much as a deliberate design choice. A stateless API means **any server can handle any request** — nothing is pinned to "the server holding your session." That's how these services scale to millions of users. The cost is pushed onto the caller: you carry the state.

Concretely, the API is a **contract**: an endpoint (`POST /v1/messages`), a request body (JSON — the conversation as a list of `{role, content}` messages), and a response body (JSON — the model's reply, plus a `stop_reason` telling you why it stopped, e.g. `end_turn` vs. `tool_use`). Every "memory" you experience in a chat app is really: *the harness keeps a growing list of messages and resends the entire thing on every single call.* The model never remembers anything — the illusion of memory is a Layer 3 trick, not a Layer 1 property.

This is worth internalizing because it explains something that seems surprising at first: **switching models mid-conversation just works.** Swapping the `model` field in the request body to a different model, while keeping the same `messages` history, hands that history to a *completely different* stateless engine — which picks up mid-conversation without missing a beat, because there was never any state living inside the model to lose in the first place. The continuity was always in the harness's message list.

---

## 5. The Agent Loop, Concretely

Once you accept "stateless model + you resend history," the agent loop falls out naturally. Tool use works like this:

1. You send the conversation plus a list of tool definitions.
2. The model can respond with plain text, **or** with a structured "I want to call tool X with these arguments" (`stop_reason: "tool_use"`).
3. The API does **not** run the tool. It just tells you the model wants to. Running it — touching the filesystem, hitting a database, executing bash — is entirely the harness's job.
4. The harness executes the tool, appends the result to the message list, and calls the model again.
5. Repeat until the model responds with a final answer instead of a tool call.

Stripped to its essence:

```python
messages = [system_prompt, user_task]
while True:
    response = model.call(messages, tools=tool_schemas)   # the model (Layer 1/2)
    if response.wants_tool_call:
        result = execute_tool(response.tool_call)          # the harness does this (Layer 3)
        messages.append(response)
        messages.append(result)                            # feed result back
    else:
        return response.text                               # done
```

Everything interesting an "AI agent" does is that loop plus the surrounding management (context limits, error recovery, permissions). Claude Code is this same idea with a huge amount of robustness, tooling, and polish wrapped around it.

---

## 6. Going Local: Do the Layers Still Hold?

Running an open-source model (Llama, Mistral, Qwen, DeepSeek, Gemma) on your own machine is where the layered picture becomes very concrete, because **you end up owning every layer yourself.**

**What you actually download** is Layer 1 — the weights, usually as a single file (e.g. a `.gguf`). That file is inert. It's just numbers. It does nothing until something loads it and runs the math — which is Layer 2, and now that's *your* problem to solve, not a cloud provider's.

**Layer 2 locally is a real program you install**, called an inference engine/runtime:
- **llama.cpp** — the low-level engine most others build on.
- **Ollama** — the easy wrapper around llama.cpp (`ollama run qwen2.5` and you're generating).
- **LM Studio** — GUI version.
- **vLLM / TGI** — heavier, production-grade, built for serving many concurrent users.

This engine does exactly what cloud Layer 2 does: tokenizes your input, runs the weights forward, samples the next token, manages the KV-cache, streams output — just now on your own CPU/GPU instead of invisibly on someone else's servers.

**Layer 3 (the harness) still has to be built or supplied**, same as ever — a chat UI, a script, LangChain, whatever wraps the raw model calls into something usable.

**Crucially: the model is still stateless, locally, exactly like in the cloud.** Downloading the weights to your own machine doesn't change that at all — there's no hidden memory inside a `.gguf` file. If a local chat app *feels* stateful, that's because the app (Layer 3) is doing the same "keep a message list, resend it every turn" trick as any cloud harness. The statelessness is a property of the weights, not of where they're hosted.

| | Cloud (Claude) | Local (open-source) |
|---|---|---|
| Layer 1 — weights | On Anthropic's GPUs, secret | The weights file, on your disk |
| Layer 2 — inference engine | Anthropic runs it, invisible | You install it (Ollama / llama.cpp / vLLM) |
| Layer 3 — harness | Your app / Claude Code | Your app / a local chat UI |
| Model stateful? | No | No — identical |
| Where's "the server"? | `api.anthropic.com` | `localhost` |

A neat detail: Ollama and LM Studio expose an **OpenAI-compatible API** at `localhost`, so the exact same harness code that talks to a cloud model can talk to your local one by just changing the URL. The client/server split doesn't go away when you go local — you just become the host of both sides.

---

## 7. Spending More Compute to "Think Harder"

This connects the layers to something concrete: what does "reasoning models" or "test-time compute" actually mean, mechanically?

**The atom of compute is one forward pass = one token.** Generating N tokens costs N forward passes through the network. "Spending more compute" literally means causing more forward passes to happen before settling on a final answer.

**Why this helps at all:** a transformer does a *fixed* amount of computation per token — fixed depth, so each forward pass can only "think" so much before it must commit to a token. The trick is to let the model write intermediate tokens and feed them back as its own input — a **scratchpad**. Generating "let me work through this step by step..." spreads a hard computation across *many* forward passes instead of cramming it into one. That's the actual mechanism behind why chain-of-thought reasoning helps: it's not a stylistic quirk, it's literally more computation applied to the problem.

**The forms this takes:**
- **Sequential (longer chains of thought)** — one long run, where the model generates a long internal monologue (thousands of hidden "thinking" tokens) before answering. Same weights, just letting it run longer. Reasoning models (o-series, DeepSeek-R1, Claude's extended thinking) do this.
- **Parallel (sample many, then pick)** — run the model N separate times on the same prompt (sampling gives different attempts each time), then pick the best via majority vote (self-consistency) or a verifier/reward model scoring each attempt (best-of-N).
- **Search (tree of reasoning)** — generate several possible next steps, score partial paths, prune bad branches, expand good ones — interleaving generation with evaluation.
- **Iterative self-refinement** — generate → critique → revise → repeat, each round more forward passes building on the last.

**Why it's called a new scaling axis:** for years, better models came from scaling *training* — more parameters, more data, more training compute, baked once into the weights. Test-time compute is a second, independent axis: spend compute *per query*, at inference, rather than once during training. Plotting accuracy against inference compute gives a clean upward curve, same shape as training scaling laws — you can genuinely trade a smaller model that thinks longer against a bigger model that answers instantly, on reasoning-heavy tasks.

**The catch:** compute = tokens = money and time. A reasoning answer generating 5,000 hidden tokens costs roughly 100× a 50-token answer, and sequential thinking specifically can't be parallelized across more GPUs — token N+1 needs token N first. Diminishing returns set in (each *doubling* of compute buys a roughly constant accuracy bump), and it only helps on problems with actual room to reason — math, code, logic, planning — not "what's the capital of France."

---

## 8. Pre-Built Harnesses: LangChain, LangGraph, LangSmith, Langfuse

This is where the whole "you'd have to build the harness yourself" idea meets the real ecosystem of tools that exist so you don't have to build it *entirely* from scratch. Four names that get thrown around together, but they actually split into two families:

- **LangChain + LangGraph** = frameworks for **building** the harness.
- **LangSmith + Langfuse** = tools for **observing** the harness once it's running.

Build vs. observe.

### LangChain
The original general-purpose framework for building LLM applications, created by **Harrison Chase**, released October 2022, backed by the company **LangChain, Inc.** It gives you standardized abstractions over harness-building: a common interface across model providers (swap Claude for GPT-4 by changing one line), "chains" (sequences of LLM calls + processing steps), memory classes (the message-list pattern, packaged), document loaders/retrievers (for RAG), and tool-calling wrappers. In the terms above, it's largely pre-built plumbing for the exact loop in Section 4. Widely used, though also known for a reputation of over-abstraction in its earlier versions — which is part of why observability tooling (below) became necessary in the first place.

### LangGraph
Also from LangChain, Inc. (2024), built because plain LangChain "chains" are linear pipelines, and real agents need to **loop, branch, and revisit state**. LangGraph models an agent as an explicit graph: nodes are steps (call the model, run a tool, check a condition), edges are transitions (including conditional ones based on model output), and a shared state object persists and updates across the whole run. It also supports checkpointing/persistence, so a long-running agent can pause and resume later — a real production concern once you're past a toy loop.

### LangSmith
LangChain, Inc.'s hosted observability platform (free tier + paid tiers) — the answer to "my agent did something wrong three tool-calls deep, what actually happened?" Since the agent loop is just a sequence of API calls, each with prompts, tool calls, and results feeding into the next, that trace gets long and invisible in production. LangSmith records every step — every prompt, every model response, every tool call and result, latency and token cost per step — and gives you a UI to inspect, replay, and run evaluations. Integrates automatically if you're already using LangChain/LangGraph.

### Langfuse
The open-source alternative to LangSmith — same category, independent company (YC-backed), not affiliated with LangChain, Inc. Fully self-hostable for free, or usable as a managed cloud. Its main distinguishing feature: **framework-agnostic** — it doesn't care whether the harness underneath was built with LangChain, LangGraph, the raw API, or something fully custom. You wrap your own LLM calls with their SDK/decorator and get tracing regardless.

### Who these are actually marketed to
All four are free (or free-tier) at the individual level — that's deliberate, it's the classic **open-core** business model. Developers are the funnel: win them over with a genuinely good free tool, get adoption, get "everyone learns this first" as the default. The actual paying customers are **companies** whose LLM products have scaled to the point where hand-rolled logging/debugging is a real engineering cost — building an internal LangSmith equivalent is a multi-month project a team would rather not do themselves, so they pay a subscription instead. Same shape as Docker (free CLI, paid Hub/enterprise) or Sentry (free/OSS error tracking, paid hosted tiers): free dev tool as the top of the funnel, paid hosted platform as the business, and the fact that the company *needs* the free tool to be genuinely good (since it's their whole funnel) is itself a signal that it's trustworthy to build on.

| Tool | Family | Role | Made by |
|---|---|---|---|
| LangChain | Build | General framework: chains, tools, memory, model-agnostic interface | Harrison Chase / LangChain, Inc. |
| LangGraph | Build | Graph-based agent orchestration — loops, branches, persistence | LangChain, Inc. |
| LangSmith | Observe | Tracing/debugging/evals, tightly integrated with LangChain/LangGraph | LangChain, Inc. |
| Langfuse | Observe | Tracing/debugging/evals, framework-agnostic, open-source, self-hostable | Langfuse (independent, YC) |

None of these touch Layer 1 or Layer 2 — they're purely Layer 3 tooling (and Layer-3-adjacent instrumentation), same territory as Anthropic's own Claude Agent SDK, just an older and more ecosystem-heavy lineage.

---

## Summary Checklist

- **Model = stateless engine.** One call, no memory, forgets everything the instant it responds.
- **Harness = everything that gives the model hands, memory, and a loop** — system prompt, tool definitions, tool execution, context management, guardrails.
- **Three layers:** weights (Layer 1) → inference engine: tokenize/sample/KV-cache/tool-call parsing (Layer 2) → agent harness: loop/prompt/tools/memory (Layer 3).
- **Underneath Layer 2, the actual arithmetic runs on silicon** — the engine calls vendor math libraries (cuBLAS/cuDNN, BLAS/MKL), which drive the GPU's (or CPU's) parallel cores/tensor cores to do the real multiply-accumulate work. Same pattern locally (your GPU) and in the cloud (the provider's data-center GPUs) — only whose chip differs.
- **The API is stateless by design** — you always resend the full conversation; "memory" is the harness carrying a growing message list, never the model itself.
- **The agent loop** is: call model → if it wants a tool, run it and feed the result back → repeat until it gives a final answer. The API only *requests* tool calls; the harness *executes* them.
- **Going local doesn't remove any layer** — it relocates all three onto your own machine. You download the weights (Layer 1), install an inference engine like Ollama/llama.cpp (Layer 2), and still need a harness (Layer 3). The model is exactly as stateless locally as in the cloud.
- **"Thinking harder" = more forward passes** — sequential (long chain-of-thought), parallel (sample-many-and-vote/verify), or search (tree of reasoning), all trading compute/time/money for better answers on genuinely hard reasoning tasks. A second, independent scaling axis alongside training-time scale.
- **LangChain/LangGraph build the harness; LangSmith/Langfuse observe it** — free/open-source at the individual level as the adoption funnel, paid hosted tiers sold to companies who'd rather not build tracing infrastructure in-house. Open-core, same shape as Docker or Sentry.
