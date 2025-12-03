<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Give me the final code where the jmeter script for load testing created by langchain or langgraph framework agent.

Excellent — let’s put everything together into a **final working prototype** where a LangChain (or LangGraph-compatible) agent dynamically creates a JMeter script (.jmx), executes it, and analyzes results.

The architecture includes one backend (FastAPI) and optional Streamlit UI, but you can run the backend alone to test the automation loop.

***

## 1. Project Structure

```
performance_agent/
├── backend/
│   ├── main.py              # FastAPI API server
│   ├── agent_runner.py      # LangChain agent definition
│   ├── agent_tools.py       # Tools to create, execute, and analyze tests
│   ├── template_base.jmx    # Base placeholder JMeter file
│   ├── results/             # Output folder for .jmx and .jtl
│   └── requirements.txt
└── frontend/
    └── app.py (Streamlit UI, optional)
```


***

## 2. Install Required Packages

```bash
pip install fastapi uvicorn langchain openai langchain-community requests
```

You must have **Apache JMeter CLI** set up and accessible in PATH.
For LLMs, use `OPENAI_API_KEY` or any local LLM that is LangChain-compatible.

***

## 3. Template JMeter File (backend/template_base.jmx)

This acts as a base for placeholders dynamically filled by the agent.

```xml
<TestPlan>
  <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Thread Group">
    <stringProp name="ThreadGroup.num_threads">{{USERS}}</stringProp>
    <stringProp name="ThreadGroup.ramp_time">{{RAMP_UP}}</stringProp>
    <stringProp name="ThreadGroup.duration">{{DURATION}}</stringProp>
    <elementProp name="HTTPRequest" elementType="HTTPSamplerProxy">
      <stringProp name="HTTPSampler.domain">{{DOMAIN}}</stringProp>
      <stringProp name="HTTPSampler.path">{{PATH}}</stringProp>
      <stringProp name="HTTPSampler.method">{{METHOD}}</stringProp>
    </elementProp>
  </ThreadGroup>
</TestPlan>
```


***

## 4. Agent Tools (backend/agent_tools.py)

```python
import csv
import os
import subprocess
import re
from langchain.tools import tool

@tool("create_jmeter_script")
def create_jmeter_script(spec: str) -> str:
    """
    Reads a natural language spec like 
    'Test https://example.com/api/login for 60 seconds with 100 users using POST'
    and generates a test plan file.
    """
    # Extract parameters via regex (basic demo)
    url_match = re.search(r'(https?://[^\s]+)', spec)
    url = url_match.group(1) if url_match else "https://example.com"
    domain = url.replace("https://", "").replace("http://", "").split("/")[0]
    path = "/" + "/".join(url.split("/")[1:]) if "/" in url else "/"
    users_match = re.search(r'(\d+)\s*(users|threads)', spec)
    users = users_match.group(1) if users_match else "10"
    duration_match = re.search(r'(\d+)\s*(seconds|second|sec|s)', spec)
    duration = duration_match.group(1) if duration_match else "30"
    method = "POST" if "post" in spec.lower() else "GET"
    ramp_up = "10"

    with open("backend/template_base.jmx") as f:
        template = f.read()

    final = template.replace("{{USERS}}", users)\
                    .replace("{{DURATION}}", duration)\
                    .replace("{{DOMAIN}}", domain)\
                    .replace("{{PATH}}", path)\
                    .replace("{{METHOD}}", method)\
                    .replace("{{RAMP_UP}}", ramp_up)

    os.makedirs("backend/results", exist_ok=True)
    jmx_path = "backend/results/generated_test.jmx"
    with open(jmx_path, "w") as f:
        f.write(final)

    return f"Created test plan: {jmx_path}"

@tool("run_jmeter_script")
def run_jmeter_script(jmx_path: str) -> str:
    """Runs JMeter using the given .jmx file, returns the results path."""
    result_path = jmx_path.replace(".jmx", ".jtl")
    command = f"jmeter -n -t {jmx_path} -l {result_path}"
    subprocess.run(command, shell=True)
    return f"Test executed, results file: {result_path}"

@tool("analyze_jmeter_results")
def analyze_jmeter_results(result_path: str) -> str:
    """Reads .jtl result and returns brief metrics."""
    with open(result_path, "r") as f:
        reader = csv.DictReader(f)
        resp_times, total, success = [], 0, 0
        for row in reader:
            total += 1
            if row.get("elapsed"): resp_times.append(float(row["elapsed"]))
            if row.get("success") == "true": success += 1
    avg = sum(resp_times)/len(resp_times) if resp_times else 0
    success_rate = success / total * 100 if total else 0
    return f"Average: {avg:.2f} ms, Success rate: {success_rate:.1f}%"
```


***

## 5. LangChain Agent (backend/agent_runner.py)

```python
from langchain.agents import initialize_agent, AgentType
from langchain.llms import OpenAI
from agent_tools import create_jmeter_script, run_jmeter_script, analyze_jmeter_results

def create_agent():
    llm = OpenAI(model="gpt-4-turbo", temperature=0.2)

    tools = [create_jmeter_script, run_jmeter_script, analyze_jmeter_results]

    agent = initialize_agent(
        tools=tools,
        llm=llm,
        agent_type=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
        verbose=True,
    )
    return agent
```


***

## 6. FastAPI Backend (backend/main.py)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from agent_runner import create_agent

app = FastAPI()
agent = create_agent()

class AgentPrompt(BaseModel):
    message: str

@app.post("/execute")
def execute_agent(req: AgentPrompt):
    result = agent.run(req.message)
    return {"agent_output": result}
```

Run backend with:

```bash
uvicorn backend.main:app --reload
```


***

## 7. Optional Streamlit Frontend (frontend/app.py)

```python
import streamlit as st
import requests

st.title("LangChain JMeter Load Testing Agent")
query = st.text_area("Enter test request:", 
    "Test https://example.com/api/login for 60 seconds with 50 users using POST")

if st.button("Run Test"):
    resp = requests.post("http://localhost:8000/execute", json={"message": query})
    if resp.status_code == 200:
        st.write(resp.json()["agent_output"])
    else:
        st.error("Agent call failed.")
```

Run with:

```bash
streamlit run frontend/app.py
```


***

## 8. How the Agent Works

1. The user prompt in English (“Test https://example.com/api/login for 50 users for 2 min”) enters FastAPI.
2. LangChain’s LLM interprets the task and plans which tools to invoke.
3. It calls:
    - `create_jmeter_script` → builds `.jmx` configuration dynamically
    - `run_jmeter_script` → executes performance test via JMeter CLI
    - `analyze_jmeter_results` → computes summary from results
4. The agent composes reasoning + final analysis → returns to UI or API client.

***

## 9. Optional: Make it LangGraph-Compatible

If you prefer **LangGraph**, simply replace `initialize_agent` usage with a custom directed graph node where:

- Node 1 → Prompt → Generate Config
- Node 2 → Execute Test
- Node 3 → Analyze Results
Nodes can communicate messages through graph memory.

***

This setup already creates, executes, and analyzes JMeter load tests **autonomously** from natural language input.

Would you like the **LangGraph version** (with explicit nodes and state transitions for better control and observability)? It’s ideal if you plan to scale this agent into a more production-ready orchestrator.

