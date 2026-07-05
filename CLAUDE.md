# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

This project uses `uv` for Python dependency management (Python >= 3.13, see `.python-version`). There is no separate lint/test setup configured — no test suite exists in the repo currently.

```bash
# Install dependencies
uv sync

# Run the application (from repo root)
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# Run any one-off python command in the project's environment
uv run <command>
```

App runs at `http://localhost:8000` (web UI) and `http://localhost:8000/docs` (FastAPI Swagger docs).

Requires a `.env` file in the repo root with `ANTHROPIC_API_KEY=...` (see `.env.example`).

**No API key is available in this dev environment** — the developer does not have access to the Claude Console, so `ANTHROPIC_API_KEY` cannot be set here. `/api/query` will fail at the Claude API call (auth error) when tested locally; this is expected and not a regression. Verification of changes should stop at confirming the app starts, static assets load, and non-Claude endpoints (e.g. `/api/courses`) work, or by exercising backend logic (e.g. `VectorStore`/`CourseSearchTool`) directly in a script rather than through a live `/api/query` call.

## Architecture

This is a Retrieval-Augmented Generation (RAG) chatbot for querying course materials. FastAPI backend + vanilla JS frontend, ChromaDB for vector storage, Anthropic Claude for generation, tool-calling (not manual retrieval injection) for search.

### Request flow

`frontend/script.js` → `POST /api/query` (`backend/app.py`) → `RAGSystem.query()` (`backend/rag_system.py`) → `AIGenerator.generate_response()` (`backend/ai_generator.py`), which calls Claude with the `search_course_content` tool available.

Claude decides whether to invoke the tool. If it does, `AIGenerator._handle_tool_execution` runs the tool via `ToolManager.execute_tool`, feeds results back to Claude in a second API call (without tools, to force a final answer), and returns the synthesized text. Sources used in the search are tracked on the tool instance (`CourseSearchTool.last_sources`) and pulled/reset by `RAGSystem` after each query — this is a side-channel, not part of Claude's returned text.

`RAGSystem` is the orchestrator wiring together: `DocumentProcessor`, `VectorStore`, `AIGenerator`, `SessionManager`, and `ToolManager`/`CourseSearchTool`. All are constructed from a single `Config` dataclass (`backend/config.py`).

### Document ingestion

On FastAPI startup (`app.py` `startup_event`), all files in `../docs` are loaded via `RAGSystem.add_course_folder`, which skips courses whose title already exists in the vector store (dedup by title, not by file).

`DocumentProcessor.process_course_document` expects a specific plain-text course script format:
```
Course Title: ...
Course Link: ...
Course Instructor: ...

Lesson 0: <title>
Lesson Link: ...
<lesson content...>

Lesson 1: <title>
...
```
It parses this into a `Course` (with `Lesson`s) plus a list of `CourseChunk`s via sentence-aware chunking (`chunk_text`, configured by `CHUNK_SIZE`/`CHUNK_OVERLAP` in `Config`). The first chunk of each lesson gets a `"Course X Lesson Y content: ..."` prefix injected so embeddings retain context even out of order.

### Vector storage (ChromaDB)

`VectorStore` maintains two collections:
- `course_catalog` — one entry per course, keyed by course title, storing serialized lesson metadata (JSON string) for lookups like course/lesson links and fuzzy course name resolution.
- `course_content` — the actual chunks, filterable by `course_title`/`lesson_number`.

Course name matching is fuzzy: `_resolve_course_name` does a semantic search against `course_catalog` to resolve a partial name (e.g. "MCP") to a canonical course title before filtering `course_content`.

### Models (`backend/models.py`)

Pydantic models `Course`, `Lesson`, `CourseChunk` are the shared data contracts between the document processor and vector store. Course `title` doubles as its unique ID throughout the system (in Chroma IDs, dedup checks, filters).

### Session/history

`SessionManager` keeps an in-memory (non-persistent) map of session ID → message list, capped at `MAX_HISTORY` exchanges. History is serialized to a plain text block and passed into Claude's system prompt on each call, not the structured messages API.

### Frontend

Static HTML/CSS/vanilla JS served directly by FastAPI's `StaticFiles` mount (no build step). `DevStaticFiles` in `app.py` adds no-cache headers so the frontend always reflects the latest saved files without restarting uvicorn.

## Notes for making changes

- Config values (chunk size/overlap, max search results, history length, model names) all live in `backend/config.py`'s `Config` dataclass — check there before hardcoding anything.
- `AIGenerator` currently supports only one round of tool execution (system prompt states "one search per query maximum"); the tool-result follow-up call is made without `tools` in the params, so Claude cannot chain further tool calls.
- Adding a new tool means implementing the `Tool` ABC in `backend/search_tools.py` and registering it via `ToolManager.register_tool` in `RAGSystem.__init__`.
