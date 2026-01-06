---
title: "Make Agents Return JSON with Pydantic"
description: "Getting structured data from agents using Pydantic models"
tags:
  - agentic-ai
  - agents
  - pydantic
  - structured-output
date: 2024-11-17
draft: false
---

So far our agents mostly returned **plain text**.  
Real apps usually need **structured data**: JSON objects we can validate, store, and render.

This chapter shows the "***classic***" LangChain way to get structured output:

- Define **Pydantic models** for the response.
- Ask the LLM to follow a **JSON schema** derived from those models.
- Use `PydanticOutputParser` to turn the JSON string into real Python objects.

---

## 1. Pydantic quick overview

- Pydantic = Python library for **data models + validation**.
- You define classes that inherit from `BaseModel`.
- It:
  - checks types,
  - converts data where possible,
  - gives clear errors if data doesn’t match.

LangChain already uses Pydantic internally, so it’s available once you install LangChain.

---

## 2. Define the response schema

`schemas.py`:

```python
from typing import List
from pydantic import BaseModel, Field

class Source(BaseModel):
    """Schema for a source used by the agent."""
    url: str = Field(description="The URL of the source")

class AgentResponse(BaseModel):
    """Schema for the agent response with answer and sources."""
    response: str = Field(description="The agent's answer to the query")
    sources: List[Source] = Field(
        default_factory=list,
        description="List of sources used to generate the answer",
    )
```

Notes:

- `AgentResponse` **contains** `Source` → nested schema.
- `default_factory=list` → each `AgentResponse` gets its own empty `sources` list if none is provided.

## 3. Extend the ReAct prompt with format instructions

We copy the classic ReAct prompt and modify the “final answer” line.

`prompt.py`
```python
REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS = """

Answer the following questions as best you can. You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question formatted according to format_instructions: {format_instructions}

Begin!

Question: {input}
Thought: {agent_scratchpad}

"""

```

Key change:

> “Final Answer … **formatted according to the format instructions** `{format_instructions}`”

This placeholder will be filled with JSON schema text generated from our Pydantic models.


## 4. PydanticOutputParser + PromptTemplate.partial

In `main.py`:
```python
from langchain.output_parsers import PydanticOutputParser
from langchain.prompts import PromptTemplate

from schemas import AgentResponse
from prompt import REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS

# 1) Create the output parser for our model
parser = PydanticOutputParser(pydantic_object=AgentResponse)

# 2) Build a PromptTemplate from our custom prompt
react_prompt = PromptTemplate.from_template(
    REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS
)

# 3) Partially fill in the format_instructions variable
react_prompt = react_prompt.partial(
    format_instructions=parser.get_format_instructions()
)

```

What `get_format_instructions()` returns is basically:

- “The output should be valid JSON that conforms to this JSON schema:”
- plus the full schema for `AgentResponse` and `Source`.

This is injected into the prompt so the LLM knows **exactly** what JSON to produce.

## 5. Running the agent and parsing the output

After wiring the ReAct agent (as in Chapter 3), the agent’s raw output is a **JSON string**:


```python
from langchain_core.runnables import RunnableLambda

tavily_search = TavilySearch()
tools = [tavily_search]

  
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
react_agent = create_react_agent(llm, tools=tools, prompt=react_prompt)
chain = AgentExecutor(agent=react_agent, tools=tools, verbose=True)


```

We turn it into a Pydantic object:
```python
extract_output = RunnableLambda(lambda x: x["output"])
parse_output = RunnableLambda(lambda x: parser.parse(x))
result = chain | extract_output | parse_output
```

Now we have:

- `result.response` → final text answer.
- `result.sources` → list of `Source` objects, each with a `url` property.

We can:
- send this object as JSON over an API,
- store it in a database,
- or render it in a frontend.


## 6. Mental model

- Pydantic defines **what the response should look like**.
- `PydanticOutputParser.get_format_instructions()` turns that into a **JSON schema** string.
- We plug that schema into the ReAct prompt.
- **The LLM**:
    - reads the schema,
    - returns a JSON string that matches the schema.
- **The parser**:
    - turns that JSON into a Pydantic object (`AgentResponse`).

> We don’t manually parse JSON fields.  
> We just design the schema and let the LLM “fill in the form.”


## Complete code example
`schema.py`
```python
from typing import List
from pydantic import BaseModel, Field


class Source(BaseModel):
    """Source class for the input"""
    url: str = Field(description="The url of the source")


class AgentResponse(BaseModel):
    """Agent response class for the output"""
    response: str = Field(description="The response from the agent")
    sources: List[Source] = Field(
        default_factory=list, description="The sources of the response"
    )
```

`prompt.py`
```python
REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS = """
Answer the following questions as best you can. You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question formatted according to format_instructions: {format_instructions}

Begin!

Question: {input}
Thought: {agent_scratchpad}
"""
```

`main.py`
```python
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_classic.agents import AgentExecutor
from langchain_classic.agents.react.agent import create_react_agent 
from langchain_tavily import TavilySearch
from langchain_core.prompts import ChatPromptTemplate
from langchain_classic.output_parsers import PydanticOutputParser
from langchain_classic.prompts import PromptTemplate
from schema import AgentResponse
from langchain_classic import hub
from langchain.agents import create_agent
from prompt import REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS
from langchain_core.runnables import RunnableLambda

load_dotenv()

parser = PydanticOutputParser(pydantic_object=AgentResponse)

react_prompt = PromptTemplate(template=REACT_PROMPT_WITH_FORMAT_INSTRUCTIONS, input_variables=["input", "tools", "tool_names", "agent_scratchpad", "format_instructions"])\
    .partial(format_instructions=parser.get_format_instructions())
print(react_prompt)

tavily_search = TavilySearch()
tools = [tavily_search]

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
react_agent = create_react_agent(llm, tools=tools, prompt=react_prompt)
chain = AgentExecutor(agent=react_agent, tools=tools, verbose=True)
extract_output = RunnableLambda(lambda x: x["output"])
parse_output = RunnableLambda(lambda x: parser.parse(x))

agent = chain | extract_output | parse_output


def main():
    print("Hello from langchain-course!")
    result = agent.invoke({"input": "What is the AQI of Gurgaon sector 46 now?"})
    print(result)

if __name__ == "__main__":
    main()
```

`Output in Langsmith`

![Output ](https://divyampandey.github.io/quartz/assets/langsmith_output.png)




##  Pydantic Parsing vs With Structured output

In subsequent post, we will see how to get the structured output using the function call - `with_structured_output`. Here is the quick comparison sheet between pydantic parsing vs the with_structured_output option.

![Output ](https://divyampandey.github.io/quartz/assets/pydantic_vs_with_structured_output.png)


