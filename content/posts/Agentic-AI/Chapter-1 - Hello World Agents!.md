
## How to create an agent using LangChain

LangChain provides a `create_agent` function in `langchain.agents`.
Before we touch code, we need to understand **what an AI agent is** and **how ReAct agents work**.

---

## 🧠 Quick Snapshot (Revision)

- Agent = **LLM that can decide + act using tools**
- Chains = **fixed flow**, developer decides steps
- Agents = **dynamic flow**, LLM decides next step
- Tools = **APIs / DB / Python / Search / etc.**
- ReAct = **Reasoning → Action → Observation → repeat**
- LangChain / LangGraph = **pre-built ReAct agents + tool interface**

---

## 1. What is an AI Agent?

> An agent is a software system that uses an LLM as a reasoning engine to decide what actions to take, and then executes those actions.

Key ideas:

- Uses an **LLM for reasoning and decision-making**.
- **Executes actions** in the outside world (through tools).
- This is different from simple “prompt → response” usage.

---

## 2. Agents vs. Simple Chains

### Simple Chains

- Sequence of steps is **hard-coded**.
- You, the developer, define:
  - Step 1 → Step 2 → Step 3 → …
- LLM is just **one step** (e.g., summarise, translate).
- **LLM does *not* decide what happens next.**

### Agents

- At each step, the **LLM decides what to do next**:
  - Which tool to call (API, DB, code, etc.)
  - Whether to stop and return an answer.
- The **control flow is dynamic**, driven by the LLM.

---

## 3. Tools = Superpowers for LLMs

Tools are the **capabilities** we give an agent, like:

- Make an **API call** (e.g., weather, stock prices).
- **Search** the web or local documents.
- Query a **database**.
- Run **Python functions** we wrote beforehand.

Because tools can do almost anything, **agents can, in theory, do almost anything** that software can do.

---

## 4. What is a ReAct Agent?
![ReAct Agent Loop](https://divyampandey.github.io/quartz/assets/ReAct_agent_arch.png)
**ReAct = Reasoning + Acting**.

The loop:

1. **Reasoning**  
   LLM thinks about the current state / question (chain-of-thought).

2. **Action**  
   LLM chooses a tool to use (or decides to answer directly).

3. **Observation**  
   Tool returns results (API response, DB rows, code output, etc.).

4. **Repeat**  
   LLM reasons again using the new information, and may act again.

5. **Finish**  
   When it has enough information, it returns a **final answer**.

This iterative loop is the **core pattern** behind many agentic systems.

---

## 5. LangChain / LangGraph and ReAct

- LangChain and LangGraph provide:
  - **Pre-built ReAct agents**.
  - A standard way to **define tools** and pass them to the agent.
  - Support for **complex workflows** and **state** (for long-running tasks).


Now, let's see how to:
  - Define **tools**.
  - Create a **ReAct-style agent**.
  - See how the agent uses tools to solve tasks.



```python
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv # Ensure you have the .env with OPENAI_API_KEY 
from langchain.agents import create_agent
from langchain.tools import tool

load_dotenv()

@tool
def search(query: str) -> str:
	"""Tool that searches the web for information
	Args:
	query: The query to search the web for
	Returns:
	The information for the query
	"""
	print(f"Searching the web for {query}")
	return "Tokyo weather is sunny"


llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
tools = [search]
agent = create_agent(model=llm, tools=tools)

def main():
	print("Hello from langchain-course!")
	result = agent.invoke({"messages": [HumanMessage(content="What is gurgaon sector 46 AQI today?")]})
	for msg in result['messages']:
		print(msg)
		
if __name__ == "__main__":
	main()
```

---


## Search Agents and Web Tools

A **search agent** is just an agent that has a **web search tool** attached.

### What a Search Agent Does

Example query:

> “Search for three job postings for an AI engineer in the Bay Area on LinkedIn and list their details.”

Typical loop:

1. **Decide** → LLM sees the query needs external info.
2. **Search** → calls the web search tool (e.g. Bing, Google, custom API).
3. **Read & Summarise** → LLM digests the results.
4. **Answer** → returns a summary + **links to each source**.

### Grounding with Sources

- Each answer includes **source URLs** (e.g. LinkedIn job pages).
- This is important because:
  - LLMs can **hallucinate**.
  - Users can **open the links**, cross-check, and decide if they trust the source.

> Grounded answer = “Here’s my summary, and here are the pages I used.”


---


# Practical: Turning a Real Search API into a Tool

So far we used a fake tool that always returned `"Tokyo weather is sunny"`.  
Now we plug in a **real search API (Tavily)**.

### 1. Initialising the client

- The Tavily client reads an API key from an environment variable (e.g. `TAVILY_API_KEY`).
- We keep it in a `.env` file so it’s not hard-coded.


```python
from tavily import TavilyClient
import os

tavily = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

```
---
### 2. Wrapping it as a tool function

- Our tool takes a query: str and returns a text summary (or the raw JSON).

```python
@tool
def search(query: str) -> str:
	"""Tool that searches the web for information
	Args:
	query: The query to search the web for
	Returns:
	The information for the query
	"""
	response = tavily.search(query)
	return response

```

- This function can now be registered as a tool in a ReAct agent.


```text
Before: static fake response
After: live web search + real-time information 
```

--- 

#### 3. Using the official LangChain Tavily tool (best practice)

- Instead of writing our own wrapper, we can use the official LangChain integration (which already defines a proper Tool class and parameters):
```python
from langchain_tavily import TavilySearch

tavily_search = TavilySearch()
tools = [tavily_search]

agent = create_agent(llm, tools)
```


---

# Structured Output with Pydantic

LLMs normally return **plain text**.  
For real apps we usually need **structured data**:

- JSON for APIs / DB
- Strongly-typed objects for backend logic
- Clear fields so the UI can render answers + sources


LangChain’s `create_agent` supports this via a `response_format` argument.

### 1. Define Pydantic models

We want the agent to return:

- `answer`: the final text answer
- `sources`: list of URLs it used (for grounding / trust)

```python
from typing import List
from pydantic import BaseModel, Field

class Source(BaseModel):
    """Schema for a source used by the agent."""
    url: str = Field(description="The URL of the source")

class AgentResponse(BaseModel):
    """Schema for the agent response."""
    answer: str = Field(description="The agent's answer to the query")
    sources: List[Source] = Field(
        default_factory=list,
        description="List of sources used to generate the answer",
    )

```

Notes:

- `BaseModel` gives us parsing, validation, and `.model_dump()` / `.json()`.
- `Field(description=...)` helps the LLM understand what to put in each field.
---
### 2. Tell the agent to use this schema

```python
from langchain.agents import create_agent  
agent = create_agent(llm=llm,tools=tools,response_format=AgentResponse)
```

Now, when you call the agent:
```python
result = agent.invoke({"input": "Find 3 AI engineer roles in the Bay Area"})
structured: AgentResponse = result["structured_response"]
```



