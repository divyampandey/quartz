---
title: agentic-ai-chapter-0
description: agentic-ai-chapter-0
tags:
  - notes
draft: false
---
### What is an LLM?
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

### Setting Up Your Environment

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