# ✈️ AI Travel Booking System

A multi-agent AI travel planner built with **LangGraph**. Four specialized agents cooperate to turn a single natural-language request (e.g. *"7-day Japan trip under ₹2L"*) into a complete travel plan — flights, hotels, itinerary, and a final summary — with conversation state persisted in PostgreSQL and a Streamlit UI on top.

## How it works

The core is a LangGraph `StateGraph` that runs four agents in a fixed pipeline:

```
START → flight_agent → hotel_agent → itinerary_agent → final_agent → END
```

| Agent | Responsibility | Backing service |
|---|---|---|
| **Flight Agent** | Looks up flights matching the query | AviationStack API |
| **Hotel Agent** | Finds hotel recommendations | Tavily Search API |
| **Itinerary Agent** | Drafts a day-by-day itinerary from the flight + hotel results | Groq (LLaMA 3.3 70B) |
| **Final Agent** | Produces the final, user-facing travel plan | Groq (LLaMA 3.3 70B) |

Shared state (`TravelState`) flows through every node and accumulates: `user_query`, `flight_results`, `hotel_results`, `itinerary`, the running `messages` list, and an `llm_calls` counter.

Conversation state is checkpointed to Postgres via `PostgresSaver`, keyed by a `thread_id` — so the same user/session can continue a trip-planning conversation across runs.

## Project structure

```
main.py               # LangGraph pipeline: state, agents, graph wiring, CLI entrypoint
frontend.py            # Streamlit UI (calls into main.app)
tools/
  flight_tool.py        # AviationStack flight search
  tavily_tool.py         # Tavily web search (used for hotels)
requirements.txt
.env                    # API keys / DB URL (not committed)
```

## Tech stack

- **[LangGraph](https://github.com/langchain-ai/langgraph)** — agent orchestration / state graph
- **[Groq](https://groq.com/)** (`llama-3.3-70b-versatile`) — LLM for itinerary + final response generation
- **PostgreSQL** (`psycopg` + `langgraph-checkpoint-postgres`) — persistent conversation checkpointing
- **[Tavily](https://tavily.com/)** — web search for hotel info
- **[AviationStack](https://aviationstack.com/)** — flight data API
- **[Streamlit](https://streamlit.io/)** — web front end

## Setup

1. **Clone and create a virtual environment**
   ```bash
   python -m venv venv
   venv\Scripts\activate        # Windows
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Configure environment variables** — create a `.env` file in the project root:
   ```env
   GROQ_API_KEY=your_groq_api_key
   AVIATIONSTACK_API_KEY=your_aviationstack_api_key
   TAVILY_API_KEY=your_tavily_api_key
   DATABASE_URL=postgresql://user:password@host:port/dbname
   ```

4. **Have a PostgreSQL database available** — `main.py` connects on startup and calls `checkpointer.setup()` to create the required checkpoint tables automatically.

## Running

**CLI:**
```bash
python main.py
```
Prompts for a travel request in the terminal and prints the final plan.

**Streamlit UI:**
```bash
streamlit run frontend.py
```
Provides a hero UI, quick-destination presets, a live view of each agent's output as the pipeline runs, and lets you download the generated plan as a Markdown file (also auto-saved to `travel_plans/`).

## Notes / known limitations

- `search_flights` in [tools/flight_tool.py](tools/flight_tool.py) does not check the AviationStack HTTP response status, so a failed/rate-limited API call returns an empty result rather than raising an error.
- The agent pipeline is linear (no branching/retry logic) — each agent always runs once, in order.
- `.env`, the `langraph_env3/` virtual environment, and generated `travel_plans/` output are excluded via `.gitignore` and should not be committed.
