



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


## Complete code example for classic ReAct  

Note: You would have to use the `langchain_classic` library if you are on the langchain `1+` version.
```python
from dotenv import load_dotenv
from langchain.tools import tool
from langchain_classic import hub
from langchain_classic.agents import AgentExecutor
from langchain_classic.agents.react.agent import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_tavily import TavilySearch


load_dotenv()
tavily_search = TavilySearch()
tools = [tavily_search]

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
react_agent = create_react_agent(
llm, tools=[tavily_search], prompt=hub.pull("hwchase17/react")
)

chain = AgentExecutor(agent=react_agent, tools=tools, verbose=True)
agent = chain

  
  

def main():
	print("Hello from langchain-course!")
	result = agent.invoke({"input": "What is the AQI of Gurgaon sector 46 now?"})
	print(result)


if __name__ == "__main__":
	main()
```

#### Output:
```json
Hello from langchain-course!


> Entering new AgentExecutor chain...
I need to find the current Air Quality Index (AQI) for Gurgaon sector 46. I'll use the search tool to get the most accurate and up-to-date information.  
Action: tavily_search  
Action Input: "current AQI Gurgaon sector 46"  {'query': 'current AQI Gurgaon sector 46', 'follow_up_questions': None, 'answer': None, 'images': [], 'results': [{'url': 'https://www.aqi.in/us/dashboard/india/haryana/faridabad/sector-46', 'title': 'Sector 46 Air Quality Index (AQI) | Air Pollution', 'content': 'Current Sector 46 Air Quality Index (AQI) is 473 Hazardous level with real-time air pollution PM2.5 (312µg/m³), PM10 (384µg/m³), Temperature (20°C) in', 'score': 0.9999448, 'raw_content': None}, {'url': 'https://www.aqi.in/us/dashboard/india/haryana/faridabad/sector-46/pm', 'title': 'Sector 46 Particulate Matter (PM2.5) Level - AQI.in', 'content': 'The current real-time PM2.5 level in Sector 46 is 19 µg/m³ (Good). This was last updated 11 Sep 2025, 06:24am (Local Time).', 'score': 0.9999335, 'raw_content': None}, {'url': 'https://www.aqi.in/us/dashboard/india/haryana/gurgaon', 'title': 'Gurgaon Air Quality Index (AQI) | Air Pollution', 'content': 'Current Gurgaon Air Quality Index (AQI) is 202 Severe level with real-time air pollution PM2.5 (125µg/m³), PM10 (152µg/m³), Temperature (24°C) in Haryana.', 'score': 0.99972194, 'raw_content': None}, {'url': 'https://air-quality.com/place/india/gurugram/d2853e61?lang=en&standard=aqi_us', 'title': 'Gurugram Real-time Air Quality Index (AQI) & Pollution Report', 'content': 'Gurugram. Gurgaon, Delhi. 2025-11-16 16:10. AQI (US). Very Unhealthy. 259. Pollutants. PM2.5. μg/m³. 183.9. PM10. μg/m³. 249.3. Weather.', 'score': 0.9987551, 'raw_content': None}, {'url': 'https://www.iqair.com/us/india/haryana/gurgaon', 'title': 'Gurgaon Air Quality Index (AQI) and India Air Pollution | IQAir', 'content': 'Gurgaon Air Quality Index (AQI) is now Unhealthy for sensitive groups. Get real-time, historical and forecast PM2.5 and weather data. Read the air pollu', 'score': 0.9978677, 'raw_content': None}], 'response_time': 0.84, 'request_id': 'bc4c3687-eb53-4e9c-bf45-db393dd10d22'}The search results indicate that the current Air Quality Index (AQI) for Gurgaon sector 46 is 473, which is classified as "Hazardous." The PM2.5 level is reported at 312 µg/m³. 

Final Answer: The current AQI of Gurgaon sector 46 is 473 (Hazardous).
```