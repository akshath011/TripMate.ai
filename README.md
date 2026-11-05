<div align="center">

# ✈️ TripMate.ai

### A multi-agent AI travel planner built with LangGraph and MCP

Describe a trip in plain English. A team of specialized agents researches flights, hotels and weather, estimates the budget, drafts a day-wise itinerary, and asks for your approval before delivering the final plan.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Render-46E3B7?style=for-the-badge&logo=render&logoColor=white)](https://tripmate-ai-wdug.onrender.com/)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/LangGraph-multi--agent-1C3C3C?style=flat-square)](https://langchain-ai.github.io/langgraph/)
[![MCP](https://img.shields.io/badge/MCP-Model%20Context%20Protocol-F97316?style=flat-square)](https://modelcontextprotocol.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://www.postgresql.org/)

**[🚀 Try the live app](https://tripmate-ai-wdug.onrender.com/)** · **[📂 Source code](https://github.com/akshath011/TripMate.ai)**

</div>

> **Note:** the live app is on a hosted free tier, so the first request after idle time can take a little while to wake up.

---

## 📸 Demo

<p align="center">
  <img src="docs/images/ui-screenshot.png" alt="TripMate.ai interface showing a generated 7-day Thailand itinerary" width="850">
</p>

**Example prompt:**

> *Plan a 7 days Thailand trip from India with budget hotels and sightseeing.*

The app returns a structured plan (trip summary, highlights, itinerary and estimated costs) that you can **copy** or **download as a PDF**.

---

## 📑 Table of Contents

- [Why this project?](#-why-this-project)
- [Features](#-features)
- [Architecture](#-architecture)
- [How MCP is used](#-how-mcp-is-used)
- [Design decisions](#-design-decisions)
- [Tech stack](#-tech-stack)
- [Project structure](#-project-structure)
- [Getting started](#-getting-started)
- [API](#-api)
- [Deployment](#-deployment)
- [Known limitations](#-known-limitations)
- [Roadmap](#-roadmap)
- [Author](#-author)

---

## 💡 Why this project?

Planning a trip means jumping between flight sites, hotel sites, weather apps and spreadsheets. TripMate.ai brings that into **one conversation**.

It is also an engineering project about a harder question: **how do you make several LLM agents cooperate reliably with real external tools?** That is why it is built on LangGraph for orchestration and MCP for tool integration, instead of one giant prompt.

---

## ✨ Features

| | Feature | Details |
| --- | --- | --- |
| 🧠 | **Natural-language input** | Extracts origin, destination, travellers, duration, budget and preferences from a single sentence |
| 🤖 | **Multi-agent workflow** | Supervisor, flight, hotel, weather, budget, itinerary and final agents, coordinated with LangGraph |
| 🔌 | **MCP tool integration** | Tavily (remote MCP), AviationStack (local MCP) and a custom weather MCP server |
| 💰 | **Budget estimation** | Combines flight, stay and trip costs and compares them with your budget |
| 🗓️ | **Day-wise itinerary** | Structured plan built from all the research |
| 🙋 | **Human-in-the-loop approval** | You review the plan before the final response is produced |
| 🛡️ | **Supervisor and guardrails** | A supervisor step checks and routes the request |
| 💾 | **Persistent state** | PostgreSQL-backed checkpointing keeps workflow state per thread |
| 📄 | **Export** | Copy the plan or download it as a PDF |
| 🔍 | **Observability** | LangSmith tracing for debugging agent runs |

---

## 🏗️ Architecture

<p align="center">
  <img src="docs/images/architecture.png" alt="TripMate.ai agent architecture with MCP tools, PostgreSQL and shared state" width="900">
</p>

Each agent has one job and its own tool. Agents pass information through a **shared state**, and workflow state is persisted in **PostgreSQL**.

### Simplified workflow

```mermaid
flowchart TD
    U([User request]) --> F[Frontend: HTML / CSS / JS]
    F --> API[FastAPI backend]
    API --> S{Supervisor agent<br/>understand + guardrails}
    S --> FL[Flight agent]
    S --> H[Hotel agent]
    S --> W[Weather agent]
    FL -->|AviationStack MCP| B[Budget agent]
    H -->|Tavily MCP| B
    W -->|Weather MCP| B
    B --> I[Itinerary agent]
    I --> A{{Human approval}}
    A -->|approved| FN[Final agent]
    FN --> R([Final travel plan])
    DB[(PostgreSQL<br/>checkpoints)] -.state.- S
    DB -.state.- A
```

> The diagram is a simplified view. The exact nodes and conditional edges are defined in [`backend.py`](backend.py).

### Agents

| Agent | Responsibility | Tool |
| --- | --- | --- |
| `supervisor_agent` | Understands the request, applies guardrails, coordinates the workflow | LLM |
| `flight_agent` | Flight and route research | AviationStack MCP |
| `hotel_agent` | Accommodation research | Tavily MCP |
| `weather_agent` | Current weather and forecast for the destination | Custom weather MCP |
| `budget_agent` | Estimates trip cost against the user's budget | LLM |
| `itinerary_agent` | Builds the day-wise plan from all results | LLM |
| `final_agent` | Produces the consolidated response | LLM |

---

## 🔌 How MCP is used

**MCP (Model Context Protocol)** is a standard way for an AI application to talk to external tools. Instead of hard-wiring every API into every agent, each tool sits behind an **MCP server**, and the app uses one **MCP client** to discover and call those tools.

<p align="center">
  <img src="docs/images/mcp-overview.png" alt="One MCP client connects AI frameworks to reusable MCP servers" width="850">
</p>

TripMate.ai uses all three common setups:

| Tool | MCP type | How it connects |
| --- | --- | --- |
| **Tavily** (web and hotel search) | Remote server | `https://mcp.tavily.com/mcp/` over streamable HTTP |
| **AviationStack** (flight data) | Local server | `uvx aviationstack-mcp` over STDIO |
| **Weather** (OpenWeather-backed) | Custom local server | [`custom_weather_mcp_server.py`](custom_weather_mcp_server.py) |

<p align="center">
  <img src="docs/images/mcp-local-vs-remote.png" alt="Local MCP servers use STDIO; remote MCP servers use streamable HTTP" width="760">
</p>

The MCP client lives in [`mcp_client.py`](mcp_client.py) (built on `mcp` and `langchain-mcp-adapters`) and exposes async helpers that the agents call:

```python
tavily_mcp_search      # hotel / destination search
aviation_mcp_call      # flight-related calls
weather_mcp_search     # current weather
forecast_mcp_search    # forecast
extract_destination    # pull the destination out of the user's request
```

---

## 🧭 Design decisions

| Decision | Why |
| --- | --- |
| **LangGraph instead of one big prompt** | Separate agents give separation of responsibilities, shared state, conditional routing, easier debugging, and a natural place for human approval |
| **MCP instead of direct API calls** | One standard interface for tools. A tool can be swapped or added without rewriting the agents, and the same server can be reused by other apps |
| **Custom weather MCP server** | Shows how to *build* an MCP server, not only consume one, and keeps weather logic isolated from the agents |
| **PostgreSQL checkpointing** | Workflow state survives between steps, which the human approval pause needs |
| **Human approval before the final answer** | Travel plans involve money, so the user stays in control instead of the system being fully autonomous |
| **Tools for facts, LLM for reasoning** | Flights, prices and weather come from tools instead of being guessed by the model |
| **Groq for inference** | Fast hosted inference for a multi-step workflow with several LLM calls |

---

## 🛠️ Tech stack

| Layer | Technology |
| --- | --- |
| Frontend | HTML, CSS, JavaScript (Jinja2 templates) |
| Backend | Python, FastAPI, Uvicorn |
| Agent orchestration | LangGraph, LangChain |
| LLM | Groq, `openai/gpt-oss-120b` |
| Tool protocol | MCP (`mcp`, `langchain-mcp-adapters`) |
| External services | Tavily, AviationStack, OpenWeather |
| Database | PostgreSQL |
| Observability | LangSmith |
| Hosting | Render |

---

## 📁 Project structure

```text
TripMate.ai/
├── app.py                         # FastAPI entry point
├── backend.py                     # LangGraph agents, state and workflow
├── mcp_client.py                  # MCP client + async helper functions
├── custom_weather_mcp_server.py   # Custom weather MCP server
├── mcp_client_test.py             # MCP connectivity / tool-discovery test
├── test.py                        # Runs the MCP test
├── requirements.txt
├── tools/
│   └── tavily_tool.py             # Tavily integration
├── templates/
│   └── index.html
├── static/
│   ├── style.css
│   └── script.js
└── docs/images/                   # README images
```

---

## 🚀 Getting started

### Prerequisites

- Python 3.10 or newer (developed on 3.11)
- A PostgreSQL database
- API keys: **Groq**, **Tavily**, **AviationStack**, **OpenWeather**
- `uvx` available (needed for the local AviationStack MCP server)
- Optional: a LangSmith key for tracing

### Installation

```bash
git clone https://github.com/akshath011/TripMate.ai.git
cd TripMate.ai

# Option A: Conda
conda create -n travel python=3.11 -y
conda activate travel

# Option B: venv
# python -m venv .venv
# source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

### Environment variables

Create a `.env` file in the project root:

```env
# LLM
GROQ_API_KEY=your_groq_key

# Tools
TAVILY_API_KEY=your_tavily_key
AVIATIONSTACK_API_KEY=your_aviationstack_key
OPENWEATHER_API_KEY=your_openweather_key

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/travel_db

# Defaults
DEFAULT_ORIGIN_IATA=BLR            # IATA code of the default departure airport

# Observability (optional)
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://apac.api.smith.langchain.com
LANGSMITH_API_KEY=your_langsmith_key
LANGSMITH_PROJECT=TripMate.ai
```

> 🔐 **Never commit `.env`.** Keep it in `.gitignore`, and keep production secrets in your host's environment settings. API keys must never appear in frontend code.

### Run

```bash
python app.py
```

Open http://127.0.0.1:8000/ in your browser.

### Check MCP connectivity (optional)

```bash
python test.py
```

This connects to the MCP servers and lists the tools they expose.

---

## 🔗 API

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/health` | Health check |
| `POST` | `/api/travel` | Submit a travel request |

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d '{"message":"Plan a 3-day trip to Tokyo with a budget of $1200"}'
```

---

## ☁️ Deployment

Deployed on **Render** and connected to the `main` branch.

| Setting | Value |
| --- | --- |
| Build command | `pip install -r requirements.txt` |
| Start command | `uvicorn app:app --host 0.0.0.0 --port $PORT` |
| Auto-deploy | On push to `main` |

```text
Local change → git push → GitHub (main) → Render build → Live app
```

---

## ⚠️ Known limitations

Being upfront about what is not perfect yet:

- **AviationStack plan limits.** The current subscription returns `function_access_restricted` for some functions (for example `list_airports` and `list_airlines`). A working MCP connection does not guarantee every API function is available, so the flight agent has to handle unavailable functions gracefully.
- **Groq token limits.** The itinerary agent combines output from several agents, which can exceed the model's tokens-per-minute limit (HTTP `413`). The mitigation is to truncate and summarize tool output and cap output tokens.
- **Testing.** The MCP connectivity test exists, but automated end-to-end tests of the full agent workflow are still to be added.
- **Free-tier hosting.** Cold starts can slow the first request.

---

## 🗺️ Roadmap

- [ ] Progress indicator showing which agent is running
- [ ] Structured flight and hotel cards, plus an interactive budget breakdown
- [ ] Prompt-size reduction and summarization between agents
- [ ] Retry logic and clearer separation of tool errors from plan restrictions
- [ ] Automated tests for the agent workflow
- [ ] Saved itineraries, trip history and user accounts
- [ ] Multi-city trips and interactive maps

---

## 👤 Author

**Akshath**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-akshath1718-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/akshath1718)
[![GitHub](https://img.shields.io/badge/GitHub-akshath011-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/akshath011)

---

<div align="center">

If you found this project interesting, consider giving it a ⭐

</div>
