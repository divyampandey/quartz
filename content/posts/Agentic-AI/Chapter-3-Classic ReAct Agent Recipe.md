



## Classic ReAct Agent Recipe (pre-v1 LangChain)

In older LangChain code, ReAct agents were wired in three steps:

1. **Get the ReAct prompt from the Hub**
2. **Build a “reasoning chain” with `create_react_agent`**
3. **Wrap it in an `AgentExecutor` loop and invoke**

### 1. ReAct prompt from the Hub

```python
from langchain import hub

react_prompt = hub.pull("hwchase17/react")
```


This returns a **PromptTemplate** with variables:

- `input` – user question
- `tools`, `tool_names` – auto-filled from our tools list
- `agent_scratchpad` – previous Thoughts / Actions / Observations


The `react_prompt` template (simplified) looks like:

> “Here are the tools you can use…  
> Question: {input}  
> Thought: …  
> Action: …  
> Observation: …  
> (repeat) …”


**Key idea:** the **ReAct prompt** teaches the model how to:
- think step-by-step (**Thought**)
- decide when to call a tool (**Action**)
- use previous steps via the scratchpad (**Observation history**)

### 2. Reasoning engine: `create_react_agent`

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_react_agent

llm = ChatOpenAI(model="gpt-4o")  # reasoning engine

react_agent = create_react_agent(
    llm=llm,
    tools=tools,           # e.g. Tavily search tool
    prompt=react_prompt,   # ReAct prompt from the hub
)

```

- `create_react_agent` returns a **Runnable chain** (often just called “agent” in old code).
- On each call it takes:
    - `input` (user question)
    - `tools`, `tool_names`, `agent_scratchpad` (injected by LangChain)

- It outputs either:
    - a **tool call** (e.g. “use `_search` with these arguments”), or
    - a **final answer**.

💡 **Mental model:** this is the **brain** of the agent – it decides _what to do next_.


## 3. Runtime loop: `AgentExecutor`

```python
from langchain.agents import AgentExecutor

agent = AgentExecutor(
    agent=react_agent,  # reasoning agent
    tools=tools,        # same tools list
    verbose=True,       # show Thoughts / Actions / Observations in logs
)
```


`AgentExecutor` is the **runtime loop**. Conceptually it:

1. Calls the ReAct chain (the brain).
2. Checks the result:
    - If it’s a tool call → run the tool’s Python function / SDK.
    - Add the tool result as a new **Observation** to the `agent_scratchpad`.
3. Calls the ReAct chain again with the updated scratchpad.
4. Repeats until the model returns a **final answer**.

So:

> **ReAct chain** = decides **what** to do.  
> **AgentExecutor** = actually **does it** in a loop.



---

## 4. Invoking the classic ReAct agent

```python
query = (
    "Search for three job postings for an AI engineer "
    "using LangChain in the Bay Area on LinkedIn and list their details."
)

result = agent.invoke({"input": query})

print(result["output"])

```

- We call `agent.invoke(...)` with a dict that contains the `input` key.
- That `input` gets plugged into the ReAct prompt.
- The ReAct loop (LLM ↔ tools) runs until it can answer.
- The final answer is usually available under `result["output"]`  
    (exact key may vary by LangChain version).



---

## 5. Mapping to modern `create_agent` (v1+)

Today, LangChain v1+ recommends `create_agent` instead of `create_react_agent`.

Conceptually:

|Classic LangChain|Modern v1+ LangChain|Role|
|---|---|---|
|`hub.pull("hwchase17/react")`|`system_prompt=...`|ReAct-style instructions|
|`create_react_agent(...)`|`create_agent(...)`|Build the reasoning graph / “brain”|
|`AgentExecutor(...)`|`agent.invoke(...)`|Runtime loop already built-in|

**Bottom line:** this chapter is mainly historical, but the mental model is still useful:

> **Prompt + LLM + tools + loop = agent**  
> Whether you use `create_react_agent` (classic) or `create_agent` (v1+),  
> the core structure is exactly the same.