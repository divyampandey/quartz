

Up to now, we’ve treated ReAct agents as a black box.

## Goal of this chapter

In this chapter I want to understand **how the classic ReAct prompt is built by hand**:

- define a simple tool (`get_text_length`)
- create a **ReAct prompt template** with `{tools}`, `{tool_names}`, `{input}`
- use `PromptTemplate.partial` + `render_text_description`
- connect everything with LCEL:  
  `{"input": lambda x: x["input"]} | prompt | llm`

This is a **single ReAct step**, not a full agent loop. But it makes the ReAct pattern very concrete.

---

## 1. Step 1 – Define a tool with `@tool`

```python
from dotenv import load_dotenv
from langchain.tools import tool

load_dotenv()

@tool
def get_text_length(text: str) -> int:
    """Function to get the length of the text."""
    return len(text)

tools = [get_text_length]

```


What `@tool` does here (internally):
- Converts the Python function into a **StructuredTool** object.
- That object has:
    - `name` – `"get_text_length"`
    - `description` – taken from the docstring
    - `func` – the real Python function
    - `schema_args` – Pydantic schema for the arguments (with `model_fields`)

The LLM will never see Python directly.  
It will see the **tool name + description + argument schema** as text.

---

## 2. Step 2 – The classic ReAct prompt template

Now we write the **ReAct prompt** as a big string with placeholders.

```python
from langchain_core.prompts import PromptTemplate
from langchain_core.tools import render_text_description

template = """
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
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:
"""
```

Three important placeholders:

- `{tools}` – human-readable **tool descriptions**
- `{tool_names}` – comma-separated names like `get_text_length, ...`
- `{input}` – the **user question** we want the agent to answer

This template is exactly what implements the ReAct pattern:

- It tells the model to:
    - think (`Thought`)
    - pick a tool (`Action`)
    - feed arguments (`Action Input`)
    - wait for results (`Observation`)
    - and finally give a `Final Answer`.

## 3. Step 3 – Fill known parts: tools and tool names

We know what  tools we have **ahead of time**, so we can plug them in once using `partial`.

```python
prompt = PromptTemplate(
    template=template,
    input_variables=["input", "tools", "tool_names"],
).partial(
    tools=render_text_description(tools),
    tool_names=", ".join([t.name for t in tools]),
)

```

What this does:

1. `PromptTemplate(..., input_variables=[...])`  
    Defines a template that expects three variables:
    - `input`
    - `tools`
    - `tool_names`
2. `.partial(...)`  
    Pre-fills **some** of those variables:
    - `tools=render_text_description(tools)`
        - `render_text_description(tools)` takes your list of tool objects  
            and turns them into a nice **text description**:
			```
			get_text_length: Function to get the length of the text
			```

		* tool_names=", ".join([t.name for t in tools])
			* builds a string like:
				* `get_text_length`

After `.partial(...)`, the only variable that is **not filled yet** is `{input}`.  
That will come from the user question when we call `.invoke()`.

Memory hook:

> `PromptTemplate + partial` =  
> “fix the tools now, fill the question later.”


---

## 4. Step 4 – Define the LLM with a stop token

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0,
    stop=["\nObservation:"],
)
```

Why `stop=["\nObservation:"]`?

- We want the model to **stop generating** as soon as it reaches `\nObservation:`.
- After that point, the **real observation** (tool result) should come from _our code_,  
    not from the model hallucinating a value.
- Without this, the LLM might invent: `Observation: 42` which is fake.
- So the LLM only generates up to:
```
	  Thought:
	  Action Input: "hello"
      Observation:
```

Then it stops, and we can plug in the **real tool result** for `Observation:` in a full agent loop.


## 5. Step 5 – Connect everything with LCEL

Now we build a small chain using **LangChain Expression Language (LCEL)**:

```python
chain = {"input": lambda x: x["input"]} | prompt | llm

```
What this means:

1. `{"input": lambda x: x["input"]}`
    - This is a small “mapping” step.
    - It expects an input dict (e.g. `{"input": "…question…"}`),
    - and says:
	    * To fill `{input}` in the prompt, use `x["input"]`.”
2. `| prompt`
    - Now that we provided the `input` value,
    - the `prompt` object renders the full ReAct prompt text:
        - with `tools`,
        - with `tool_names`,
        - with the current question.
3. `| llm`
    - Sends the rendered prompt text to the LLM.
    - LLM returns a ReAct-style response (`Thought`, `Action`, `Action Input`, etc.).

So calling:
```python
result = chain.invoke(
    {"input": "What is the length of the string 'hello' ? "}
)
print(result)
```
will give you something like:
```text
Thought: I can use the get_text_length tool to compute the length of "hello".
Action: get_text_length
Action Input: "hello"

```


This is **one ReAct reasoning step**:

- The model has:
    - read the tools description,
    - chosen the right tool,
    - built the tool call (`get_text_length("hello")`).

In the full agent implementation, the next step would be:
- parse `Action` and `Action Input`,
- run the tool in Python,
- feed `Observation: <tool_result>` back into another LLM call