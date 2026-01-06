---
title: "Modern LangChain Agents with 'create_agent'"
description: "Building modern agents using LangChain's create_agent function"
tags:
  - agentic-ai
  - agents
  - langchain
  - tutorial
date: 2024-11-17
draft: false
---

So far:

- We met **LLMs**.
- We met **agents** and **tools**.
- We saw the **classic ReAct recipe**.
- We learned how to get **structured output** (Pydantic + `with_structured_output`).

Now we put it all together using the **new LangChain v1 API**:

> One call to `create_agent` gives us:  
> **reasoning + tools + loop + structured output**.

---

## 1. Minimal example

```python
from dotenv import load_dotenv
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langchain_openai import ChatOpenAI
from langchain_tavily import TavilySearch

from schema import AgentResponse  # Pydantic model: answer + sources

load_dotenv()

# 1) Tools
tavily_search = TavilySearch()
tools = [tavily_search]

# 2) Base LLM
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 3) Agent = LLM + tools + structured output
agent = create_agent(
    model=llm,
    tools=tools,
    response_format=AgentResponse,  # <- Pydantic schema
)

# 4) Invoke
result: AgentResponse = agent.invoke(
    {
        "messages": [
            HumanMessage(
                content="What is the temperature in Gurgaon sector 46 right now?"
            )
        ]
    }
)

# result is a dict:
# {
#   "messages": [...],
#   "structured_response": AgentResponse(...)
# }

response: AgentResponse = result["structured_response"]

print(response.answer)
for s in response.sources:
    print(s.url)
    


```


**What’s happening:**

- `TavilySearch()` = tool for web search.
- `ChatOpenAI(...)` = base model.
- `create_agent(...)`:
    - builds the **reasoning graph** (ReAct-style),
    - wires in **tool use**,
    - applies **structured output** using `response_format=AgentResponse`.

- `agent.invoke(...)` returns a **dictionary**:
    - `"messages"` → internal conversation history,
    - `"structured_response"` → your **Pydantic object**.

You normally care about `structured_response`.



## 2. What changed compared to the classic recipe?

In the classic ReAct agent we had to think about:

- `hub.pull("hwchase17/react")` – custom ReAct prompt
- `create_react_agent(...)` – reasoning chain
- `AgentExecutor(...)` – while-loop runtime
- manual structured output plumbing

With `create_agent` we no longer need:
- **Hub prompt** – ReAct prompt is handled for us.
- **AgentExecutor** – the loop runs inside a LangGraph graph.
- **Manual structured output setup** – we just pass `response_format=AgentResponse`.

Mental model:

> `create_agent(model, tools, response_format)`  
> = **ReAct brain + tool loop + Pydantic output** in one call.



## 3. Why messages instead of a single `input` string?

Old version (< 1.0) calls looked like:
```python
agent.invoke({"input": "your question"})

```

New style uses **messages**:
```python
from langchain_core.messages import HumanMessage

agent.invoke({
    "messages": [
        HumanMessage(content="your question here")
    ]
})

```

This is closer to how chat models actually work:

- easy to add a **system message** later,
- easy to add **previous user / assistant turns**,
- same pattern works for simple chats and more complex conversations.

For now, just remember:

> With `create_agent`, send a **list of messages**,  
> not a single `input` string.


## 4. When should you use `create_agent`?

Use this pattern when:
- You are on **LangChain v1+**.
- You want a **simple agent** with:
    - tools (search, DB, APIs, code),
    - built-in reasoning,
    - and **structured output** as a Pydantic model.

Default recipe:
```python
agent = create_agent(
    model=llm,
    tools=tools,
    response_format=MyPydanticModel,
)

result = agent.invoke({"messages": [...]})
parsed: MyPydanticModel = result["structured_response"]

```

That’s the modern, “batteries included” way to build LangChain agents.
