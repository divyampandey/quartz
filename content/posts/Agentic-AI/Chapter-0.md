---
title: Chapter-0
description: Chapter-0
tags:
  - notes
draft: false
---
## What is an LLM?
 Large Language Model - which is trained on massive amounts of data. It could understand and generate text like human-like text. Can follow task like translation, summarization, code-generation etc.

##### Key things to remember
1. LLM are `stateless` by default.
	* Each call to LLM is independent.
	* It doesn't remember previous conversations unless you explicitly provide context.
	* You must send the entire conversation history each time.
 2. `Token` based
	 * LLM process text in tokens
	 * You pay per token (input + output)
	 * Each model has a maximum token limit. (`context window`)

3.  Probabilistic
    * LLM generate text by predicting the next most likely token.
    * Same input can produce different output (controlled by `temperature`)
    * Not deterministic (unless `temperature=0`)


##### Popular LLM Providers:

| Provider  | Popular Models                   | Key Features                       |
| --------- | -------------------------------- | ---------------------------------- |
| OpenAI    | GPT-5, GPT-4o, GPT-4o-mini, o1   | Strong reasoning, function calling |
| Anthropic | Claude 3.5 Sonnet, Claude 3 Opus | Long context, safety-focused       |
| Google    | Gemini Pro, Gemini Flash         | Multimodal, fast                   |
| Meta      | Llama 3.1, 3.2                   | Open source, can self-host         |
|           |                                  |                                    |

## Setting Up Your Environment

##### Step 1: Install Python (if not already)
You need Python 3.9 or higher. Check your version:
```
python --version
Python 3.10.16
```

##### Step 2: Create a Virtual Environment
```
python -m venv langchain_env

# On Mac/Linux:
source langchain_env/bin/activate

  

# On Windows:
langchain_env\Scripts\activate
```


##### Step 3: Install LangChain
```
# Install core LangChain
pip install langchain

# Install a provider package (we'll start with OpenAI)
pip install langchain-openai

# Install other useful packages
pip install python-dotenv
```

Ensure you have the env variable `OPENAI_API_KEY` setup in `.env` file.

## Invocation
##### LLMs can be invoked with simple strings or structured messages

###### Example one:
```
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
import os

# Load environment variables
load_dotenv()


# Simple Text String

# Initialize the model
llm = ChatOpenAI(
model="gpt-4o",
temperature=0.7

)

# Simple prompt as a string
prompt = "Explain what a Large Language Model is in one sentence."

  
# Invoke the model
response = llm.invoke(prompt)

  
print(f"Prompt: {prompt}")
print(f"\nResponse: {response.content}")
print(f"\nResponse type: {type(response)}")
print(f"Response attributes: {dir(response)}")
```

The response type is:
`Response type: <class 'langchain_core.messages.ai.AIMessage'>`

##### SystemMessage sets the AI's behavior/context
##### HumanMessage represents user input

```
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()
from langchain_core.messages import AIMessage, HumanMessage, SystemMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

messages = [
SystemMessage(content="Your are a helpful math tutor and give logical explanation in 1 line."),
HumanMessage(content="what is 2+2?"),
AIMessage(content="2+2 equals 4."),
HumanMessage(content="what is 2*2?"),
AIMessage(content="2*2 equals 4."),
HumanMessage(content="so they are the same mathematically?"),
]
 
response = llm.invoke(messages)
print(response.content)
print(type(response))
```

**Result:**
```
Yes, both 2+2 and 2*2 result in the same value, 4, but they represent different operations.
<class 'langchain_core.messages.ai.AIMessage'>
```

##### The response object contains content + metadata
1. `response.content`
2. `response.response_metadata`

**example:**
```
response.response_metadata.get('model_name')
'gpt-4o-2024-08-06'

response.response_metadata.get('token_usage')

{'completion_tokens': 34, 'prompt_tokens': 17, 'total_tokens': 51, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}

```


## Parameters
##### Temperature Controls Creativity:
- `0.0`: Deterministic, same output every time
- `0.7`: Balanced, good for most use cases
- `1.5+`: Very creative, unpredictable

**example with temperature=0:**
```
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()

llm = ChatOpenAI(model="gpt-4o", temperature=0)
for i in range(4):
	response = llm.invoke("Give me the title of a sci-fi movie")
	print(f"Try {i+1}: {response.content}")

```

**Response:**
```
Try 1: "Inception" is a popular sci-fi movie that explores the concept of shared dreams and subconscious manipulation.
Try 2: "Inception" is a popular sci-fi movie that explores the concept of shared dreams and subconscious manipulation.
Try 3: "Inception" is a popular sci-fi movie that explores the concept of shared dreams and subconscious manipulation.
Try 4: "Inception" is a popular sci-fi movie.
```


**example with temperature=0.7 (balanced):**
```
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()

llm = ChatOpenAI(model="gpt-4o", temperature=0.7)
for i in range(4):
	response = llm.invoke("Give me the title of a sci-fi movie")
	print(f"Try {i+1}: {response.content}")
```

**Response:**
```
Try 1: "Inception"
Try 2: "Blade Runner 2049"
Try 3: "Inception"
Try 4: "Blade Runner 2049"
```

**Response with temperature=1.7 (creative):**

```
Try 1: "Echoes of Infinity: A Journey Beyond the Stars"
Try 2: "Galactic Echoes: The Lost Voice of Cyberspace"
Try 3: "Echoes of Tomorrow: Shadows in the Stars"
Try 4: "Eclipse Reign: Dawn of Ethereal Realms"
```



##### Max Tokens Limits Response Length:
- Set it to control costs and response size
- Response may be cut off mid-sentence if too low

If you setup a very small value for the `max_tokens` then the output response might be cutoff.
**example:**
```
llm = ChatOpenAI(model="gpt-4o", temperature=1.1)

for i in range(4):
response = llm.invoke("Give me the title of a sci-fi movie",max_tokens=5)

print(f"Try {i+1}: {response.content}")
```

**response:**
```
Try 1: "Inception"
Try 2: "Inception" is a
Try 3: "The Matrix" is a
Try 4: "Inception" is a
```





##### Using chain and prompts to write your application code

you can use prompts and chain to modularise the llm calling part. For e.g Langchain library provides `ChatPromptTemplate` and `PromptTemplate` for creating prompts.

**A simple example:**
```python
from langchain_core import ChatPromptTemplate, PromptTemplate
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI


load_dotenv()

INFO = """
Mango is the king of the fruits. It has more than 10 varities and it is very popular in different parts of the world.

"""
def main():
	prompt_template = """
	Given the {information} about topic. Give me following
	1. short summary of the topic
	2. Fun facts about the topic.

	"""
	
	prompt = ChatPromptTemplate.from_template(prompt_template) 
	
	llm = ChatOpenAI(model="gpt-4o", temperature=0.7)
	
	chain = prompt | llm 
	
	response = chain.invoke({"information": INFO})
	
	print(response.content)
```
