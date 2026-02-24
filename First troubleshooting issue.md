# 🧠 Agentic AI SOC Analyst – Architecture Progress Summary

This document summarizes where we are in the project, what we changed, and why.

It is meant to help me quickly understand the architecture decisions and debugging progress when I return to this later.

---

# ✅ Clean Beginner-Friendly Summary of What We Built

---

## 1️⃣ Why We Don’t Want n8n to Query Log Analytics Directly

If n8n talks directly to:

```

[https://api.loganalytics.azure.com/](https://api.loganalytics.azure.com/)...

```

Then:

- n8n would need Azure AD authentication
- I would have to configure OAuth inside n8n
- I would need client IDs, secrets, and token handling
- It becomes messy and unnecessary

### ✅ Better Approach

👉 Let the **Python script** handle authentication to Azure  
👉 Let **n8n only talk to the Python API**

This keeps the system clean and easier to maintain.

---

## 2️⃣ What the Python Script Originally Was

The original `_main.py` is a **CLI (Command Line Interface) program**.

That means:

- It runs from the terminal
- It uses `input()`
- It prints results to the console
- It is **NOT** a web server

Originally, the flow looked like:

```

Terminal → python _main.py

```

This is not something n8n can call.

n8n needs an HTTP endpoint like:

```

POST [http://localhost:8000/run](http://localhost:8000/run)

```

But the script did not provide any HTTP endpoints.

---

## 3️⃣ What We Did With FastAPI

We created a new file:

```

api.py

```

This file wraps the existing logic and exposes it as a web API.

It now provides endpoints such as:

```

POST /query-context
POST /log-analytics-query
POST /hunt
POST /run

```

Now the architecture looks like:

```

n8n → FastAPI (localhost:8000) → EXECUTOR.py → Azure Log Analytics

````

This is the correct architecture.

- n8n = Orchestrator (visual workflow tool)
- FastAPI = Web interface to Python logic
- EXECUTOR.py = Core logic engine
- Azure Log Analytics = Data source

---

## 4️⃣ About API Keys and Secrets

I created a separate file for API keys to avoid mixing secrets with logic.

That is good thinking.

### ✅ Even Better Practice

Use **environment variables** instead of storing secrets in files.

Do NOT:

- Hardcode API keys in Python files
- Commit secrets to GitHub

Instead, use:

```python
os.getenv("OPENAI_API_KEY")
````

This is secure and production-friendly architecture.

---

## 5️⃣ The “Strings vs Objects” Error (What Actually Happened)

The major debugging issue we hit was NOT related to:

* Azure
* FastAPI
* n8n

It was related to how the code called the **OpenAI Responses API**.

Originally, the code passed something like:

```python
input=[system_message, user_message]
```

But the Responses API does NOT accept raw strings in a list like that.

It expects structured input objects.

After adjusting the structure, another error appeared:

```
expected a string, but got an object instead
```

That means:

* `system_message` (or `user_message`)
* was actually a `dict` or `list`
* not a plain string

So the real issue was simply:

👉 The wrong format being passed to:

```python
openai_client.responses.create()
```

It was a formatting problem — not an Azure or FastAPI problem.

---

# 🎯 Current Understanding

At this point, I understand:

* A CLI script is NOT a web API.
* FastAPI turns Python into an HTTP server.
* n8n should call my API — not Azure directly.
* OpenAI Responses API requires properly structured input.
* Environment variables are the correct way to manage secrets.

This is no longer just a script — it is a small distributed system.

---

# 🚀 Where We Left Off

* FastAPI server running locally via `uvicorn`
* Endpoints created and testable via `/docs`
* Debugging OpenAI input formatting issues
* Preparing to connect n8n to the `/run` endpoint

Next step: stabilize API responses and integrate cleanly with n8n.
