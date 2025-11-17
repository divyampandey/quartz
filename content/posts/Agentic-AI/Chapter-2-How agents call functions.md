
---

## Tools – How LLMs Talk to the Outside World

Our search agent uses **one tool**: a web search tool.
Under the hood, this tool is not just a random function – it is a **LangChain Tool** object.

### 1. Why do we need Tools?

We already have:
- An SDK / API (Tavily) that can search the web.
- An LLM that can reason.

We want the LLM to be able to **use** that API, but the LLM:
- Doesn't know the function name.
- Doesn't know the arguments.
- Doesn't know the output format.

So we need a way to *describe* this capability to the LLM.

> A **Tool** is that description: it tells the LLM what the function is, what it does, and how to call it.

---

### 2. What is a LangChain Tool?

Conceptually, a Tool is:

> A wrapper around a Python function with:
> - `name`
> - `description`
> - `arguments schema` (parameters + types)
> and some extra metadata.

The LLM sees:
- The **tool name**  
- The **tool description** (natural language)  
- The **argument specification** (JSON schema)

and learns:
- *When* it should use this tool (from description)
- *How* to call it (argument names & types)

---

### 3. Who executes the tool? (LLM vs runtime)

Very important:

- The **LLM does not execute** the tool.
- The LLM only produces a **tool call** like:

```json
{
  "tool": "_search",
  "arguments": {
    "query": "AI engineer jobs in Bay Area",
    "include_domains": ["linkedin.com"],
    "search_depth": "advanced"
  }
}
```
Then:

- **LangChain / AgentExecutor** runs the actual Python function (which uses the Tavily SDK).
- The result (search results) is fed back to the LLM as an **observation**.
- The ReAct loop continues.

> LLM = decides **what** to do.  
> Runtime (AgentExecutor) = actually **does** it.

### 4. Example: Tavily Search as a Tool

Instead of manually wrapping the Tavily SDK, we use the official LangChain integration:

```python
from langchain_tavily import TavilySearch
search_tool = TavilySearch()
tools = [search_tool]
```

If we inspect `search_tool`, we see:

- `name` → `_search`
- `description` → “Search engine optimized for comprehensive, accurate and trusted results…”
- `args` / schema → `{ "query": ..., "include_domains": ..., "exclude_domains": ..., ... }`

This is exactly the information the LLM receives when we create the agent.
Because the Tavily team wrote this integration, we get:
- A rich, precise description
- Correct argument names & types
- Extra advanced options (like `include_domains`, `search_depth`)

This usually leads to **better tool calls** and **better answers** compared to a quick custom wrapper.

### 5. Custom Tools (mental model)

We can also create our own tools by wrapping Python functions (v1-style pseudo-code):

```python
from langchain.tools import tool  

@tool("get_weather", return_direct=False) 
def get_weather(city: str) -> str:
	"""Get the current weather for a given city."""
	# Call a real weather API here     
	return f"The weather in {city} is sunny."  
	# Then include `get_weather` in the tools list we give to `create_agent`.
	
```


The important part is not the decorator syntax, but the **idea**:

- We describe **name**, **docstring/description**, and **arguments**.
- LangChain turns this into a structured Tool that the LLM can call.    
- The agent runtime executes the function and passes the result back.


---

> **Summary:**
> - Tools are how LLMs `“know”` about external functions/APIs.
> - A Tool is a function + name + description + argument schema.
> - LLM _decides_ to call a tool; the runtime _executes_ it.
> - Using official integrations (like `TavilySearchResults`) usually gives better descriptions and better tool calls.
