
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





