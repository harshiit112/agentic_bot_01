# ✈️ Agentic TripMate

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11%2B-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python 3.11+" />
  <img src="https://img.shields.io/badge/FastAPI-0.136.3-009688?style=for-the-badge&logo=fastapi&logoColor=white" alt="FastAPI" />
  <img src="https://img.shields.io/badge/LangGraph-1.2.2-1C3C3C?style=for-the-badge" alt="LangGraph" />
  <img src="https://img.shields.io/badge/Groq-LLM-F55036?style=for-the-badge" alt="Groq" />
  <img src="https://img.shields.io/badge/License-Apache%202.0-blue?style=for-the-badge" alt="Apache 2.0" />
</p>

> A multi-agent AI travel planner that finds live flight data, researches hotels, and creates a practical day-by-day itinerary from one natural-language request.

## Overview

Agentic TripMate is a FastAPI application powered by LangGraph. It combines external travel data with LLM reasoning to generate a complete travel plan.

**Built with:** FastAPI, LangGraph, Groq, AviationStack, Tavily, PostgreSQL, Jinja2, HTML, CSS, and JavaScript.

## Features

- Natural-language travel planning
- City, country, airport, and IATA route detection
- Live flight and flight-status lookup through AviationStack
- Hotel and destination research through Tavily
- Budget-aware itinerary generation with Groq
- Thread IDs and PostgreSQL LangGraph checkpoints
- Markdown results, copy-to-clipboard, and PDF download
- Local and Docker-based execution

## Architecture

```mermaid
graph LR
    U([Traveler]) --> UI[FastAPI UI]
    UI --> API[POST /api/travel]
    API --> F[Flight Agent]
    F --> H[Hotel Agent]
    H --> I[Itinerary Agent]
    I --> R[Final Response Agent]
    F --> A[(AviationStack)]
    H --> T[(Tavily)]
    I --> G[(Groq)]
    R --> G
    API <--> P[(PostgreSQL Checkpointer)]
    R --> UI
```

### Agent pipeline

1. **Flight Agent:** Parses the request, resolves locations to IATA codes, and fetches live flight data.
2. **Hotel Agent:** Searches Tavily for up to five relevant hotel results.
3. **Itinerary Agent:** Uses the request, flight data, and hotel research to create a budget-aware itinerary.
4. **Final Response Agent:** Formats the result into trip summary, flights, hotels, daily itinerary, estimated budget, and recommendations.

## Project Structure

```text
project_2_agentic/
├── app.py                 # FastAPI routes and web application
├── backend.py             # LangGraph agents and PostgreSQL checkpointer
├── requirements.txt       # Pinned dependencies
├── Dockerfile             # Docker deployment configuration
├── test.py                # Interactive smoke test
├── tools/
│   ├── _init__.py
│   ├── flight_tool.py     # Route parsing and AviationStack client
│   └── tavily_tool.py     # Tavily search client
├── templates/index.html    # Web UI markup
├── static/                 # Browser JavaScript and CSS
├── LICENSE                 # Apache License 2.0
└── README.md
```

## Setup

### Requirements

- Python 3.11+
- PostgreSQL database
- Groq API key
- Tavily API key
- AviationStack API key

### Install

```bash
git clone <your-repository-url>
cd project_2_agentic
python -m venv .venv
```

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

macOS/Linux:

```bash
source .venv/bin/activate
```

Install the pinned dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### Environment variables

Create `.env` in the project root:

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
DATABASE_URL=postgresql://username:password@host:5432/database
DEFAULT_ORIGIN_IATA=DEL
```

`DEFAULT_ORIGIN_IATA` is optional and defaults to `DEL`. The backend adds `sslmode=require` to `DATABASE_URL` when it is not already present.

## Run the App

```bash
python app.py
```

Or:

```bash
uvicorn app:app --host 127.0.0.1 --port 8000 --reload
```

Open <http://127.0.0.1:8000>.

### Docker

```bash
docker build -t agentic-tripmate .
docker run --rm -p 8000:8000 --env-file .env agentic-tripmate
```

## API

### `GET /health`

Returns the service status.

### `POST /api/travel`

Request:

```json
{
  "message": "Plan a 7 day Japan trip from India under 2 lakhs",
  "thread_id": null
}
```

Response:

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

Use the returned `thread_id` to continue a thread. Empty messages return `400`; unexpected failures return `500`.

## Example Prompts

```text
Plan a complete 7 days Japan trip from India including flights, hotels and sightseeing under 2 lakhs.
Plan a 5 days Dubai trip from Delhi with flights, hotels and sightseeing.
Plan a 7 days Thailand trip from India with budget hotels and sightseeing.
Give me all country flight info.
```

Clear routes such as `from Delhi to Tokyo` or `DEL to NRT` provide the best flight matching.

## Test and Troubleshoot

Run the interactive smoke test:

```bash
python test.py
```

Check Python syntax:

```bash
python -m compileall -q app.py backend.py tools
```

The application requires all API keys and a reachable PostgreSQL database before `app.py`, `backend.py`, or `test.py` can run. AviationStack provides flight status data and may not provide ticket prices. Tavily results are research suggestions, not confirmed hotel bookings.

## License

Licensed under the [Apache License 2.0](LICENSE).
