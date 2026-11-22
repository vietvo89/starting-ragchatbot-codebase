# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Requires Python 3.13+ and uv. On Windows, run shell commands from Git Bash.

- Install dependencies: `uv sync`
- Start the app (from project root): `./run.sh`, which runs `cd backend && uv run uvicorn app:app --reload --port 8000`
- Start manually: `cd backend && uv run uvicorn app:app --reload --port 8000`
- API docs: http://localhost:8000/docs

There is no test suite, and no linter or formatter is configured. Don't assume `pytest` or `ruff` exist; check `pyproject.toml` before adding either.

Environment: `ANTHROPIC_API_KEY` is read from a root `.env` file (see `.env.example`).

## Working directory matters

The backend uses paths relative to the current directory, so it must be started from `backend/`:
- `startup_event` in `backend/app.py` loads courses from `../docs`.
- `Config.CHROMA_PATH` is `./chroma_db`, which resolves to `backend/chroma_db`.
- `app.mount("/", StaticFiles(directory="../frontend"))` also depends on this.

Running `uvicorn` from the project root will silently skip course loading and break static files.

## Architecture

The app is a FastAPI server (`backend/app.py`) that serves the static frontend and two JSON endpoints: `POST /api/query` and `GET /api/courses`. The backend is a flat set of modules in `backend/` with no packages. Imports are bare (`from vector_store import ...`), so run code from `backend/`.

### Ingestion (startup)

`RAGSystem.add_course_folder()` walks `docs/` and processes each `.txt`, `.pdf`, and `.docx` file. Only `.txt` is actually parsed. `DocumentProcessor.read_file()` reads everything as UTF-8 text, and there is no PDF or DOCX extraction library, so those files will produce garbage.

Each file is parsed into a `Course` with lessons and `CourseChunk`s:
- Header lines 1–4 give the title, link, and instructor. The title is the course's unique ID everywhere.
- `Lesson N: Title` lines split the body into lessons.
- `chunk_text()` splits lesson text into sentences and packs them into chunks of up to `CHUNK_SIZE` (800) characters with about `CHUNK_OVERLAP` (100) characters of overlap.

Storage uses two ChromaDB collections in `vector_store.py`:
- `course_catalog` has one document per course, with lesson metadata as a JSON string. It is used for course-name resolution.
- `course_content` has the chunks, filtered by `course_title` and `lesson_number` metadata.

A course is skipped if its title is already in `course_catalog`. The file is still parsed first, so startup re-parses everything.

### Query flow (tool-based RAG)

`RAGSystem.query()` does not search up front. It calls Claude with a single tool, `search_course_content`, and Claude decides whether to use it:
1. `AIGenerator.generate_response()` makes the first Claude call with the system prompt, session history, and tool definitions.
2. If the response has `stop_reason == "tool_use"`, `_handle_tool_execution()` runs `ToolManager.execute_tool()`. That calls `CourseSearchTool.execute()`, which calls `VectorStore.search()`.
3. `search()` resolves a fuzzy `course_name` through `course_catalog`, then queries `course_content` with a metadata filter.
4. A second Claude call is made without `tools`, so Claude cannot search again. This is how the "one search per query" rule is enforced.

Sources are not returned by Claude. `CourseSearchTool` stores them in `last_sources` as a side effect, and `RAGSystem.query()` reads them through `ToolManager` and then resets them. This is shared mutable state, so concurrent requests can interfere with each other's sources.

### Other state

- `SessionManager` holds conversation history in memory, keyed by `session_N`. It is lost on restart. Only the last `MAX_HISTORY * 2` messages are kept (4 messages with the default of 2, which is 2 exchanges). The history is passed into the system prompt as text, not as message turns.
- `Config` in `backend/config.py` holds all tunables. The model is `claude-sonnet-4-20250514` and the embedding model is `all-MiniLM-L6-v2`. Both are set there and not in `.env`.
- Adding a new tool means subclassing `Tool` in `search_tools.py` and registering it with `ToolManager` in `RAGSystem.__init__`. The generator is generic and does not need changes. Note that the second call strips tools, so a multi-round tool loop would need `_handle_tool_execution` changed.

### Gotchas

- The lesson-chunk context prefix is inconsistent. Chunks before the last lesson get `Lesson N content:` only on their first chunk, while the last lesson gets `Course {title} Lesson N content:` on every chunk. Check `document_processor.py` before changing it.
- `DevStaticFiles` in `app.py` is defined but not used. The app mounts plain `StaticFiles`.
- `main.py` is the uv template placeholder and is not the entry point. The server entry point is `backend/app.py`.
- Changing the chunking or embedding model requires rebuilding the store, since `add_course_folder` skips existing courses. Use `add_course_folder(..., clear_existing=True)` or delete `backend/chroma_db`.
