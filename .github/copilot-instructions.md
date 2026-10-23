# Copilot Instructions

## Project overview

This is a Python 3.11+ command-line travel planning agent. The application uses LangGraph to
coordinate a stateful workflow, Pydantic models for validated domain data, Rich for terminal output,
Amadeus for flight/hotel search, configurable LLM providers for conversation and itinerary
generation, and MongoDB for profiles, trips, and audit events.

## Build, run, and test

Use the repository virtual environment:

```bash
.venv/bin/python main.py --provider openrouter
```

Supported providers are `openrouter`, `anthropic`, and `gemini`. Runtime credentials are loaded
from `.env` or the deployment environment; the CLI must not prompt for secrets. Required values
depend on the selected LLM provider and include the selected provider key, `AMADEUS_API_KEY`,
`AMADEUS_API_SECRET`, and `MONGODB_URI`. See `.env.example`.

Install dependencies:

```bash
.venv/bin/pip install -r requirements.txt
```

Run the full test suite:

```bash
.venv/bin/python -m pytest -q
```

Run one test or one test file:

```bash
.venv/bin/python -m pytest tests/test_persistence_and_audit.py -q
.venv/bin/python -m pytest tests/test_persistence_and_audit.py::test_redact_removes_secrets_and_email_addresses -q
```

There is no configured standalone lint command. Before finishing Python changes, run:

```bash
.venv/bin/python -m compileall -q agents config models services ui main.py
git diff --check
```

Tests that use `InMemoryStorage` do not require MongoDB. Production paths require `MONGODB_URI`.
Set `TRAVEL_DATA_MODE=mock` only for explicit local demonstrations/tests; production mode must not
silently substitute mock travel inventory.

## Architecture

`main.py` loads backend-managed credentials, initializes the selected LLM client, creates a
LangGraph graph, seeds a `thread_id`/profile/correlation ID, and resumes graph interrupts from the
Rich CLI.

`agents/graph.py` is the workflow orchestrator. It connects these stages:

```text
gather_preferences
  -> generate_itinerary
  -> present_itinerary
  -> await_approval
  -> extract_items
  -> search_options
  -> present_options
  -> await_decision
  -> booking boundary
  -> trip_summary
```

The shared `TravelAgentState` in `agents/state.py` contains serialized trip/itinerary data,
bookable items, search results, booking records, workflow phase, profile ID, trip ID, and
correlation ID. LangGraph uses SQLite checkpoints under `~/.travel-agent/checkpoints.db` when the
SQLite checkpoint package is available; otherwise it falls back to in-memory checkpoints.

LLM-driven behavior is concentrated in:

- `agents/nodes/preference_node.py`: conversational core-trip and hotel/flight preference intake.
  It uses tool schemas and validates tool output with `TripRequest`, `HotelPreferences`, and
  `FlightPreferences`.
- `agents/nodes/itinerary_node.py`: structured itinerary generation. It requests JSON-only output
  and validates it as an `Itinerary`.
- `services/llm_client.py`: provider-neutral `LLMClient` interface with Anthropic, Gemini, and
  OpenRouter implementations. Use `get_client()` from nodes rather than constructing SDK clients.

Deterministic integration and safety behavior lives in:

- `services/amadeus_client.py`: real flight/hotel search plus explicit opt-in mock adapters.
- `services/confidence_scorer.py`: hard filters and deterministic flight/hotel ranking.
- `services/persistence.py`: MongoDB and test-only in-memory storage adapters.
- `services/audit_logger.py`: structured, redacted audit events with correlation IDs.
- `services/profile_service.py`: local profile identity and profile/audit helpers.
- `services/booking_service.py`: hosted/tokenized checkout boundary. It must not handle raw card
  data and currently fails explicitly until a real provider adapter is configured.

## Repository-specific conventions

- Keep LLM calls behind `services.llm_client.get_client()`. Nodes should not import provider SDKs
  directly or duplicate provider-selection logic.
- Treat LLM output as untrusted input. Use tool schemas for conversational extraction and Pydantic
  validation before writing to graph state, MongoDB, or provider calls.
- Graph nodes should return partial state updates rather than mutating unrelated state. Preserve
  the existing interrupt-based human-in-the-loop pattern for user input and approval.
- Keep external side effects deterministic and explicit. Search, ranking, payment handoff, booking,
  persistence, and recovery belong in services/nodes with clear error states, not in prompts.
- Never silently fall back from real Amadeus data to mock data. Mock mode must be explicitly enabled
  with `TRAVEL_DATA_MODE=mock`.
- Search options are not bookings. Preserve provider offer IDs, source, timestamps, expiry/policy
  metadata, and revalidate offers before any future booking implementation.
- Never collect, log, persist, or send raw payment details to an LLM. Use hosted/tokenized checkout
  and store only checkout/booking references and status.
- Use `AuditLogger`/`profile_service.audit` for important workflow and provider events. Redaction
  covers API keys, tokens, passwords, email addresses, card fields, and related secrets.
- Preserve Pydantic model serialization boundaries: state and persistence commonly carry
  `model_dump()` dictionaries, while UI and business logic may reconstruct typed models.
- Keep production configuration in environment/deployment secrets. Do not add real credentials,
  MongoDB URIs, generated credential files, or local profile/checkpoint files to the repository.
- When changing workflow routing, update both the graph edges and the corresponding state phase
  semantics; approval and booking transitions must remain explicit and user-visible.

## Current booking limitation

The booking node is currently a checkout boundary, not a completed real booking integration.
`services/booking_service.py` raises `BookingUnavailableError` until a concrete provider is
configured. Do not convert this into simulated success. A real implementation must add offer
revalidation, idempotency, provider confirmation verification, partial-booking recovery, and
hosted/tokenized payment handling.
