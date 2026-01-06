---
title: "Function Calling vs ReAct Agents: Who Picks the Tool?"
description: Comparing function calling and ReAct approaches to tool selection
tags:
  - agentic-ai
  - llm
  - agents
  - react
date: 2025-01-05
draft: false
---

## Why this comparison matters

In agentic systems, the *hardest* part is often not the tool itself — it’s **tool selection**:

- Which tool should the agent use?
- With what inputs?
- In what order?
- When should it stop calling tools and answer?

A useful mental model is **serverless**:

- In serverless, we shift scalability/availability concerns to the cloud vendor.
- In agents, we can shift **tool selection responsibility** either to:
  - the **LLM vendor** (function/tool calling), or
  - **our own prompting + parsing loop** (classic ReAct prompting).

## Two paradigms of tool selection

### 1) Function calling (Tool calling)

**Idea:** We provide tools with schemas/descriptions → model returns a *structured tool call* (JSON-like) telling us what to invoke.

- Tool selection logic is largely **inside the vendor/model** (fine-tuned for this).
- Output is **structured and parse-friendly**.
- Typically **more reliable + less flaky** than parsing raw text. 

**You (the app) still execute tools.**  
The model only *requests* tool invocations; your backend runs them.

---

### 2) ReAct prompt (Reason + Act via prompting)

**Idea:** We send a carefully crafted ReAct prompt; model prints something like:

- `Action: <tool_name>`
- `Action Input: <args>`

Then our code uses regex/parsers to extract the tool call, executes it, returns the **Observation**, and repeats.

Key characteristics:

- Tool selection reasoning is driven by **prompt engineering**.
- Requires **parsing model text output**, often with regex.
- Gives you **more control / customization**, but more surface area to break.

---

## The core trade-off

### Function calling: “Vendor picks tools”
**Pros**
- More deterministic, structured tool calls  
- Less parsing / boilerplate  
- Often cheaper in tokens (less verbose reasoning printed)

**Cons**
- More “black box”: you usually don’t see *why* the model picked that tool  
- Debugging can feel less transparent (you see the call, not the rationale)
---

### ReAct prompt: “You build the tool-selection brain”
**Pros**
- Maximum flexibility: you can tweak the prompt, format, steps
- Great for learning/teaching how agent loops work
- Can force explicit step-by-step outputs (with caveats)

**Cons**
- Parsing can break (format drift)
- More brittle, more tokens, more glue code
- Reliability depends heavily on prompt quality and model behavior 

---

## A helpful way to think about it

### “Where is the intelligence?”
- **Function calling**: intelligence of tool selection is **baked into the model/vendor**
- **ReAct prompt**: intelligence is **simulated by your prompt + your parser**

So it’s basically:

> **Reliability & simplicity** (function calling)  
> vs  
> **Control & prompt-level custom behavior** (ReAct prompt)

---

## Modern reality: ReAct architecture + function calling

A common “best of both worlds” pattern today is:

- Keep the **ReAct loop** (reason → act → observe → repeat)
- But replace text-parsing with **tool_calls** from function calling

So it’s still *Reason + Act*, but the “Act” step is produced via structured tool calling instead of fragile text.

This evolution removes:
- the giant ReAct prompt template,
- regex parsers,
- “AgentFinish vs AgentAction” text markers,

…and replaces them with:
- `tool_calls` list + `tool_call_id` tracking + tool result messages.

---

## Minimal loop (how tool calling agents work)

### Pseudocode

```python
messages = [HumanMessage(user_query)]

llm = llm.bind_tools(tools)

while True:
    ai_msg = llm.invoke(messages)

    tool_calls = ai_msg.tool_calls  # list (possibly empty)
    messages.append(ai_msg)

    if not tool_calls:
        return ai_msg.content  # final answer

    for call in tool_calls:
        tool_name = call["name"]
        tool_args = call["args"]
        tool_call_id = call["id"]

        result = tools[tool_name].invoke(tool_args)

        messages.append(
            ToolMessage(content=str(result), tool_call_id=tool_call_id)
        )
