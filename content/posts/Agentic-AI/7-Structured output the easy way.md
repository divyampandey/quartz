

Real apps rarely want **free-form text**.  
They usually want **structured data**: JSON you can save, validate, and show in a UI.

There are two main ways to do this in LangChain:

1. **Old way** – manual Pydantic parsing  
2. **New way (recommended)** – `with_structured_output`

This post explains both in simple terms and why the new way is better.

### A. Old way recap (manual Pydantic parsing)

- We:
	- Inject _format instructions_ (JSON schema) into the prompt.
    - Ask the LLM: “Please answer in this JSON shape.”
    - Hope the model follows it.
    - Take `result["output"]` (JSON string) and run:
        - `output_parser.parse(json_string)` → `AgentResponse`.


Problems:
- We repeat the **schema text** in _every_ ReAct step (wasting tokens).
- We rely on the model to obey the format just from text (less reliable, esp. older models).
- We manually wire the parser and parsing step.

Memory line:

> Old way = **“prompt + hope + parse”**.

### B. New way: `with_structured_output` (preferred)

**Core idea:**
The new method is much simpler:

> Tell the model **once** what schema you want,  
> and let LangChain + function calling handle the rest.


Example (simplified):
```python
from langchain_openai import ChatOpenAI
from schemas import AgentResponse  # Pydantic model

llm = ChatOpenAI(model="gpt-4o")

structured_llm = llm.with_structured_output(AgentResponse)

```

- `structured_llm` is a **wrapped version** of the model.
- When you call it, it will **return an `AgentResponse` object**, not a plain string.
- Internally it uses:
    - function calling,
    - JSON schema,
    - and `PydanticOutputParser` – but you don’t need to touch any of that.


### 2.2. Calling the structured LLM

Simple example:
```python
result: AgentResponse = structured_llm.invoke(
    "What is the AQI in Gurgaon sector 46 today?"
)

print(result.response)
for s in result.sources:
    print(s.url)
```

You directly get:

- `result.response` – the main answer text
- `result.sources` – list of `Source` objects with `.url`

No manual parsing. No format instructions in your prompt.


## 3. Quick comparison – old vs new

### Old: manual Pydantic parsing

- You:
    - Inject schema text into the prompt.
    - Hope the model respects it.
    - Manually call `output_parser.parse(...)`.

- Downsides:
    - More boilerplate.
    - Schema text repeated many times → more tokens.
    - Less robust on weaker models.

### New: `with_structured_output`
- You:
    - Define your Pydantic model (`AgentResponse`).
    - Call `llm.with_structured_output(AgentResponse)`.
    - Use the returned model instance.
        
- Benefits:
    - Very little code.
    - LangChain sends the schema using **function calling**.
    - LangChain parses the JSON into Pydantic for you.
    - More reliable and cleaner.
        

Memory line:

> New way = **“declare schema once → always get structured data back.”**


---

## 4. When should you use it?

As long as your model supports **function calling** (all modern major models do),  
the recommended approach is:



---
## Complete code example 
Note: It's build up on previous post ([6-Make agents return JSON with Pydantic](6-Make%20agents%20return%20JSON%20with%20Pydantic.md)

```python
from dotenv import load_dotenv
from langchain_classic.agents import AgentExecutor
from langchain_classic.agents.react.agent import create_react_agent
from langchain_classic.prompts import PromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_openai import ChatOpenAI
from langchain_tavily import TavilySearch

  
from prompt import REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS
from schema import AgentResponse


load_dotenv()

  
tavily_search = TavilySearch()
tools = [tavily_search]

  
prompt = PromptTemplate(
template=REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS,
input_variables=[
"input",
"tools",
"tool_names",
"agent_scratchpad",
"format_instructions",
],).partial(format_instructions="")

 

llm = ChatOpenAI(model="gpt-4o", temperature=0)
react_agent = create_react_agent(llm, tools, prompt)
agent_executor = AgentExecutor(
agent=react_agent, tools=tools, verbose=True, handle_parsing_errors=True
)

  

extract_output = RunnableLambda(lambda x: x["output"])
# Structured output LLM for final formatting
llm_structured = llm.with_structured_output(AgentResponse)
agent = agent_executor | extract_output | llm_structured

  
  

def main():
	print("Hello from langchain-course!")

  

	# Run the agent to get the answer with sources
	result = agent.invoke({
	"input": "What is the temperature of Gurgaon sector 46 now? Search for real-time data."})

	print(result)

  
  

if __name__ == "__main__":
	main()
```

Output:

```text
response=
### Weather Overview
- **Location:** Gurgaon Sector 46
- **Temperature:** 67°F (19°C)
- **Condition:** Hazy clouds

### Additional Weather Details
- **Humidity:** Approximately 70%
- **Wind Speed:** 5 mph (8 km/h)
- **Visibility:** Reduced due to haze
- **Feels Like:** 66°F (18°C)

### Tips for the Day
- **Clothing:** Light layers are recommended due to mild temperatures.
- **Outdoor Activities:** Limited visibility might affect outdoor plans; proceed with caution.
- **Health Advisory:** Individuals with respiratory issues should consider staying indoors due to haze.

### Forecast
- **Morning:** Cool and hazy
- **Afternoon:** Slightly warmer with continued haze
- **Evening:** Temperatures dropping, haze persisting

### Sources
- Local weather stations
- Online weather services

sources=[Source(url='https://www.weather.com'), Source(url='https://www.accuweather.com')]

```

## Evolution of ReAct Agents

![](https://divyampandey.github.io/quartz/assets/evolution_of_react_agent.png)

