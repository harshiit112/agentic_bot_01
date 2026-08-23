# ✈️ Agentic TripMate: Multi-Agent AI Travel Planner

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11%2B-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python 3.11+" />
  <img src="https://img.shields.io/badge/FastAPI-0.136.3-009688?style=for-the-badge&logo=fastapi&logoColor=white" alt="FastAPI" />
  <img src="https://img.shields.io/badge/LangGraph-1.2.2-1C3C3C?style=for-the-badge" alt="LangGraph" />
  <img src="https://img.shields.io/badge/Groq-LLM-F55036?style=for-the-badge" alt="Groq LLM" />
  <img src="https://img.shields.io/badge/PostgreSQL-Checkpointing-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL" />
</p>

<p align="center">
  <strong>From one natural-language request to a flight-aware, hotel-researched, day-by-day travel plan.</strong>
</p>

---

## 🌍 Project Overview

**Agentic TripMate** is a multi-agent travel planning system built with FastAPI and LangGraph. It coordinates specialist agents to interpret a travel request, find live flight information, research hotel options, generate an itinerary, and return a polished final recommendation.

The application combines deterministic tools with LLM reasoning:

- **AviationStack** supplies live flight and status information.
- **Tavily** researches hotel and destination information from the web.
- **Groq** generates the itinerary and final travel response.
- **PostgreSQL** stores LangGraph checkpoints for thread-based execution.
- **FastAPI** exposes the browser interface and JSON API.

> Agentic TripMate is a planning assistant, not a booking engine. Always verify availability, prices, schedules, visa requirements, and travel advisories before booking.

## 🚀 Key Features

- **Sequential multi-agent pipeline:** Four specialized LangGraph nodes pass shared travel state from research to response generation.
- **Natural-language route parsing:** Understands common cities, countries, airport names, and IATA codes such as `DEL` and `NRT`.
- **Live flight lookup:** Queries AviationStack for airline, flight number, status, airport, terminal, gate, schedule, and delay data.
- **Hotel discovery:** Uses Tavily to return up to five web research results with titles, URLs, and concise snippets.
- **Budget-aware itinerary generation:** Groq receives the original request and collected research to create a practical daily plan.
- **Structured final answer:** Produces trip summary, flight information, hotel suggestions, daily itinerary, estimated budget, and final recommendations.
- **Thread-aware execution:** Browser `localStorage` keeps the returned `thread_id`, while PostgreSQL stores LangGraph checkpoints.
- **Useful browser actions:** Render Markdown, copy the final plan, and download a PDF version.
- **Docker-ready deployment:** Includes a Python 3.11 slim image and a Uvicorn production-style entry point.

## 🧠 System Architecture

Agentic TripMate uses a directed, sequential graph. Each agent enriches the shared `TravelState` before handing it to the next node.

```mermaid
graph LR
    U([Traveler]) --> W[FastAPI Web UI]
    W --> R[POST /api/travel]
    R --> G[LangGraph StateGraph]

    G --> F[Flight Agent]
    F --> H[Hotel Agent]
    H --> I[Itinerary Agent]
    I --> A[Final Response Agent]

    F -. live flight data .-> AS[(AviationStack)]
    H -. hotel research .-> TV[(Tavily)]
    I -. itinerary generation .-> GR[(Groq)]
    A -. response synthesis .-> GR
    G <--> PG[(PostgreSQL Checkpointer)]
    A --> W
```

### The Agent Roster

1. **✈️ Flight Agent**
   - Cleans and interprets the user's travel request.
   - Detects explicit routes such as `from Delhi to Tokyo` and IATA routes such as `DEL to NRT`.
   - Resolves supported city and country names using `airportsdata` and `pycountry`.
   - Falls back to `DEFAULT_ORIGIN_IATA` when only a destination is found.
   - Queries AviationStack and formats the returned live flight records.

2. **🏨 Hotel Agent**
   - Builds a destination-focused search query from the original request.
   - Calls Tavily with a maximum of five results.
   - Trims long snippets before passing the research into the generation stages.

3. **🗺️ Itinerary Agent**
   - Uses Groq model `openai/gpt-oss-120b`.
   - Receives the user request, flight results, and hotel results.
   - Creates a practical, budget-aware itinerary that is easy to follow.

4. **🧾 Final Response Agent**
   - Combines the collected state into a user-facing answer.
   - Requests six consistent sections: trip summary, flight information, hotel suggestions, day-by-day itinerary, estimated budget, and final recommendations.
   - Reminds users that AviationStack may provide status data without ticket prices.

### State and Persistence

The shared `TravelState` contains:

```text
messages       Conversation and agent messages
user_query     Original travel request
flight_results Formatted AviationStack output
hotel_results  Formatted Tavily output
itinerary      Groq-generated itinerary
llm_calls      Per-run call counter
```

`PostgresSaver` is configured as the LangGraph checkpointer. Every request gets a generated thread ID when one is not supplied. The frontend stores this ID locally and sends it with subsequent requests.

## 📂 Project Directory Structure

```text
project_2_agentic/
├── app.py                 # FastAPI app, HTML route, travel API, and health endpoint
├── backend.py             # LangGraph state, agents, Groq model, and PostgreSQL checkpointer
├── requirements.txt       # Pinned Python dependencies
├── Dockerfile             # Python 3.11 container configuration
├── test.py                # Interactive backend smoke test
├── tools/
│   ├── _init__.py         # Tools package marker
│   ├── flight_tool.py     # Route parsing and AviationStack integration
│   └── tavily_tool.py     # Tavily hotel research integration
├── templates/
│   └── index.html         # Travel planner page
├── static/
│   ├── script.js          # API calls, thread storage, copy, and PDF download
│   └── style.css          # Responsive visual styling
├── LICENSE                # Apache License 2.0
└── README.md              # Project documentation
```

## ⚡ Getting Started

### Prerequisites

- Python 3.11 or newer
- A reachable PostgreSQL database
- Groq API key
- Tavily API key
- AviationStack API key
- Docker, if using the container workflow

The backend connects to PostgreSQL and initializes the LangGraph checkpointer while `backend.py` is imported. Configure the environment before starting the server or running the smoke test.

### 1. Clone the repository

```bash
git clone <your-repository-url>
cd project_2_agentic
```

### 2. Create a virtual environment

**Windows PowerShell**

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**macOS / Linux**

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

The project uses pinned package versions from [`requirements.txt`](requirements.txt).

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### 4. Configure environment variables

Create `.env` in the project root:

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
DATABASE_URL=postgresql://username:password@host:5432/database

# Optional; used when a request contains a destination but no origin.
DEFAULT_ORIGIN_IATA=DEL
```

The backend adds `sslmode=require` to `DATABASE_URL` when no SSL mode is present.

### 5. Launch the web application

```bash
python app.py
```

Or use Uvicorn directly:

```bash
uvicorn app:app --host 127.0.0.1 --port 8000 --reload
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser.

## 💬 Example Travel Requests

```text
Plan a complete 7 days Japan trip from India including flights, hotels and sightseeing under 2 lakhs.

Plan a 5 days Dubai trip from Delhi with flights, hotels and sightseeing.

Plan a 7 days Thailand trip from India with budget hotels and sightseeing.

Give me all country flight info.
```

For the most reliable flight route detection, use one of these patterns:

```text
from Delhi to Tokyo
to Tokyo from Delhi
DEL to NRT
flights from Delhi
flights to Tokyo
```

## 🔌 API Reference

### `GET /`

Serves the Jinja2-powered travel planner interface.

### `GET /health`

Returns:

```json
{
  "status": "ok",
  "message": "AI Travel Planner API is running"
}
```

### `POST /api/travel`

Request body:

```json
{
  "message": "Plan a 7 day Japan trip from India",
  "thread_id": null
}
```

`thread_id` is optional. Omit it to create a new thread, or provide an existing ID to continue that thread.

Example request:

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"Plan a 7 day Japan trip from India with flights, hotels and sightseeing under 2 lakhs\"}"
```

Successful response shape:

```json
{
  "success": true,
  "thread_id": "user_generated_thread_id",
  "answer": "...",
  "flight_results": "...",
  "hotel_results": "...",
  "itinerary": "...",
  "llm_calls": 4
}
```

An empty message returns `400`. Unexpected database, provider, or application failures return `500` with an `error` field.

## 🐳 Docker Deployment

Build the image:

```bash
docker build -t agentic-tripmate .
```

Run the container:

```bash
docker run --rm -p 8000:8000 --env-file .env agentic-tripmate
```

The included [`Dockerfile`](Dockerfile) uses `python:3.11-slim`, installs the pinned dependencies, exposes port `8000`, and starts Uvicorn on `0.0.0.0`.

## 🧪 Testing and Diagnostics

The current [`test.py`](test.py) is an interactive end-to-end smoke test:

```bash
python test.py
```

It prompts for a travel request and runs the graph with the fixed thread ID `test_user`.

For a service-level check:

```bash
curl http://127.0.0.1:8000/health
```

For syntax validation without calling external providers:

```bash
python -m compileall -q app.py backend.py tools
```

## 🧰 Technology Stack

| Layer | Technology | Role |
| --- | --- | --- |
| Runtime | Python 3.11 | Application runtime and container base |
| API | FastAPI 0.136.3 | HTTP routes and request validation |
| Server | Uvicorn 0.48.0 | ASGI development and container server |
| Orchestration | LangGraph 1.2.2 | Stateful agent graph execution |
| LLM framework | LangChain 1.3.2 | Message and model integration |
| Model provider | Groq | Itinerary and response generation |
| Web research | Tavily 0.7.24 | Hotel and destination search |
| Flight provider | AviationStack | Live flight/status data |
| Persistence | PostgreSQL + Psycopg | LangGraph checkpoints |
| Templates | Jinja2 3.1.6 | Server-rendered HTML |
| Frontend | HTML, CSS, JavaScript | Planner UI and result actions |

## 📈 Engineering Highlights

- **Clear orchestration boundary:** `backend.py` owns graph state, node behavior, model calls, and checkpointing, while `app.py` owns HTTP concerns.
- **Tool isolation:** Flight route parsing and external flight calls live in `tools/flight_tool.py`; web research lives in `tools/tavily_tool.py`.
- **Shared typed state:** `TravelState` makes the data passed between graph nodes explicit.
- **Provider-aware output:** The final prompt distinguishes live flight status from ticket pricing, reducing misleading booking expectations.
- **Graceful flight failures:** Missing AviationStack keys, invalid JSON, request failures, provider errors, and empty results become readable flight result messages.
- **Browser-friendly delivery:** The UI renders Markdown and supports copy and PDF export without requiring a separate frontend build system.

## ⚠️ Limitations and Operational Notes

- AviationStack data is live/status-oriented and may not include ticket prices.
- Tavily hotel results are web research suggestions, not verified availability or reservations.
- Generated itineraries and budgets are recommendations and should be independently verified.
- Provider quotas, network availability, API latency, and database reachability affect response time and reliability.
- The browser stores `thread_id` in `localStorage`; clearing browser site data removes the client-side reference.
- Markdown rendering and PDF export rely on CDN-hosted `marked` and `html2pdf.js` assets.
- Add authentication, rate limiting, structured logging, and production monitoring before public deployment.

## 🔐 Security Checklist

- Never commit `.env` or provider credentials.
- Use a least-privilege PostgreSQL account.
- Restrict database network access where possible.
- Add authentication and rate limiting before exposing `/api/travel` publicly.
- Treat provider output and generated travel advice as untrusted external data.

## 📄 License

Agentic TripMate is licensed under the [Apache License 2.0](LICENSE).

See [`LICENSE`](LICENSE) for the complete terms.

---