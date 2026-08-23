# Agentic TripMate

> A modern multi-agent travel planner that combines live flight data, web research, and LLM-powered itinerary planning in one FastAPI application.

Agentic TripMate turns a natural-language travel request into a practical travel plan. The application uses LangGraph to coordinate specialist agents, AviationStack for live flight and status data, Tavily for hotel research, Groq for itinerary and response generation, and PostgreSQL for thread checkpoints.

## What It Does

- Accepts natural-language requests such as `Plan a 7 day Japan trip from India under 2 lakhs`.
- Resolves common cities, countries, airport names, and IATA codes into flight routes.
- Retrieves live flight status data from AviationStack.
- Searches the web for hotel information through Tavily.
- Generates a budget-aware itinerary with Groq.
- Produces a final answer with trip summary, flights, hotels, daily itinerary, budget, and recommendations.
- Keeps a conversation thread ID in the browser and persists LangGraph checkpoints in PostgreSQL.
- Lets users copy the result or download it as a PDF from the browser UI.

## Architecture

```mermaid
flowchart LR
    User[Traveler] --> UI[FastAPI HTML UI]
    UI --> API[POST /api/travel]
    API --> Graph[LangGraph StateGraph]
    Graph --> Flight[Flight Agent]
    Flight --> Hotel[Hotel Agent]
    Hotel --> Itinerary[Itinerary Agent]
    Itinerary --> Final[Final Response Agent]
    Flight --> Aviation[AviationStack API]
    Hotel --> Tavily[Tavily Search API]
    Itinerary --> Groq[Groq LLM]
    Final --> Groq
    Graph <--> DB[(PostgreSQL Checkpointer)]
    Final --> UI
```

## Agent Pipeline

The graph runs sequentially for every request:

1. **Flight Agent**
   - Parses the user request for origin and destination mentions.
   - Supports city names, country names, airport names, and three-letter IATA codes.
   - Uses `DEFAULT_ORIGIN_IATA` when only a destination is detected.
   - Calls AviationStack and formats airline, flight number, status, airport, terminal, gate, schedule, and delay information.
   - Supports route searches, one-sided searches, and global live-flight queries.

2. **Hotel Agent**
   - Builds a query from the original request, prefixed with `Best hotels for`.
   - Calls Tavily and returns up to five web results.
   - Keeps each result's title, URL, and a shortened content snippet for downstream agents.

3. **Itinerary Agent**
   - Sends the user request, flight data, and hotel research to Groq.
   - Asks the model to create a practical, budget-aware, easy-to-follow itinerary.

4. **Final Response Agent**
   - Combines all collected data into the user-facing answer.
   - Formats the answer into six sections: trip summary, flight information, hotel suggestions, day-by-day itinerary, estimated budget, and final recommendations.
   - Explicitly tells the user that AviationStack may provide flight status without ticket prices.

Each node increments `llm_calls` in the shared state, including tool stages in the current implementation. The API returns that counter along with the generated answer and intermediate results.

## Technology Stack

| Area | Technology |
| --- | --- |
| Web framework | FastAPI `0.136.3` |
| ASGI server | Uvicorn `0.48.0` |
| Agent orchestration | LangGraph `1.2.2` |
| LLM framework | LangChain `1.3.2` |
| LLM provider | Groq via `langchain-groq` `1.1.3` |
| Search | Tavily via `tavily-python` `0.7.24` |
| Flight data | AviationStack REST API |
| Persistence | PostgreSQL via Psycopg `3.3.4` |
| Frontend | Jinja2, HTML, CSS, and browser JavaScript |
| Python runtime | Python `3.11` recommended |

All Python package versions are pinned in [`requirements.txt`](requirements.txt).

## Prerequisites

Install or provision the following before running the app:

- Python 3.11
- PostgreSQL 12 or newer, or a hosted PostgreSQL database
- A Groq API key
- A Tavily API key
- An AviationStack API key
- Git, if cloning the project

The backend opens the PostgreSQL connection during import, so all required environment variables must be configured before starting the server or running [`test.py`](test.py).

## Installation

### 1. Clone and enter the project

```bash
git clone <your-repository-url>
cd project_2_agentic
```

### 2. Create a virtual environment

Windows PowerShell:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

macOS or Linux:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

### 3. Install the pinned dependencies

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## Environment Configuration

Create a file named `.env` in the project root. Do not commit it to source control.

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
DATABASE_URL=postgresql://username:password@host:5432/database

# Optional. Defaults to DEL when the request only contains a destination.
DEFAULT_ORIGIN_IATA=DEL
```

### Database notes

- `DATABASE_URL` should be a PostgreSQL connection URL.
- The application automatically appends `sslmode=require` when it is missing.
- The LangGraph PostgreSQL checkpointer creates or updates its required tables during startup with `checkpointer.setup()`.
- The database host must be reachable from the machine or container running the app.

## Run Locally

Start the development server with:

```bash
python app.py
```

Or run Uvicorn directly:

```bash
uvicorn app:app --host 127.0.0.1 --port 8000 --reload
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in a browser.

Useful endpoints:

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/` | Serves the travel planner UI |
| `POST` | `/api/travel` | Runs the agent pipeline |
| `GET` | `/health` | Returns the service health status |

## API Usage

### Request

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"Plan a 7 day Japan trip from India with flights, hotels and sightseeing under 2 lakhs\"}"
```

To continue using an existing thread, include its `thread_id`:

```json
{
  "message": "Add a day trip to Kyoto",
  "thread_id": "user_existing_thread_id"
}
```

### Response shape

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

An empty `message` returns HTTP `400`. Unexpected provider, database, or application errors return HTTP `500` with an `error` field.

## Example Prompts

```text
Plan a complete 7 days Japan trip from India including flights, hotels and sightseeing under 2 lakhs.

Plan a 5 days Dubai trip from Delhi with flights, hotels and sightseeing.

Plan a 7 days Thailand trip from India with budget hotels and sightseeing.

Give me all country flight info.
```

For the most reliable route detection, use clear phrasing such as `from Delhi to Tokyo` or include IATA codes such as `DEL to NRT`.

## Run with Docker

Build the image:

```bash
docker build -t agentic-tripmate .
```

Run it with your environment file:

```bash
docker run --rm -p 8000:8000 --env-file .env agentic-tripmate
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000). The included [`Dockerfile`](Dockerfile) uses Python 3.11, installs the pinned dependencies, exposes port `8000`, and binds Uvicorn to `0.0.0.0` inside the container.

## Testing and Diagnostics

The current [`test.py`](test.py) is an interactive smoke test. It prompts for a travel request and invokes the backend with the fixed thread ID `test_user`.

```bash
python test.py
```

For a basic service check after startup:

```bash
curl http://127.0.0.1:8000/health
```

Expected response:

```json
{
  "status": "ok",
  "message": "AI Travel Planner API is running"
}
```

## Project Structure

```text
project_2_agentic/
├── app.py                 # FastAPI application and HTTP routes
├── backend.py             # LangGraph state, agents, LLM, and PostgreSQL checkpointer
├── requirements.txt       # Pinned Python dependencies
├── Dockerfile             # Production-style container entry point
├── test.py                # Interactive backend smoke test
├── tools/
│   ├── flight_tool.py     # Route parsing and AviationStack integration
│   └── tavily_tool.py     # Tavily hotel research integration
├── templates/
│   └── index.html         # Planner page markup
└── static/
    ├── script.js          # API calls, thread storage, copy, and PDF download
    └── style.css          # Application styling
```

## Important Limitations

- AviationStack provides live flight and status information; it does not guarantee ticket fare prices. The final response calls this out when pricing is unavailable.
- Hotel suggestions are web search results, not verified bookings or live room availability.
- The generated itinerary and estimated budget should be reviewed before booking.
- Provider availability, rate limits, network access, and API quotas can affect results.
- The frontend stores the current `thread_id` in browser `localStorage`; clearing site storage starts a new client-side thread.
- The app uses CDN-hosted `marked` and `html2pdf.js` assets in the browser, so PDF and Markdown rendering need network access unless those assets are self-hosted.

## Troubleshooting

**`GROQ_API_KEY is missing`**

Confirm that `.env` exists in the project root and contains a valid `GROQ_API_KEY`.

**`DATABASE_URL is missing` or PostgreSQL connection errors**

Check the connection URL, credentials, firewall rules, and whether the database is reachable. For hosted PostgreSQL, use the provider's externally reachable connection URL.

**Flight API errors**

Verify `AVIATIONSTACK_API_KEY`, check the AviationStack quota, and use a clear origin and destination. A missing key is returned as a readable flight result error rather than stopping the graph immediately.

**Tavily search errors**

Verify `TAVILY_API_KEY`, network access, and the provider quota.

**The browser shows a server error**

Inspect the Uvicorn console output. The backend prints the original exception and traceback, which usually identifies whether the failure came from PostgreSQL, an external API, or the LLM call.

## Security Checklist

- Keep `.env` and all API keys out of Git.
- Use a restricted PostgreSQL user for the application.
- Add authentication and request rate limiting before exposing the API publicly.
- Validate and constrain user input when deploying to untrusted users.
- Review generated travel information and provider responses before treating them as booking advice.

## License

No license file is currently included. Add a license before distributing the project publicly.