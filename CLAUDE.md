# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Retrieval-Augmented Generation (RAG) chatbot system for course materials. It uses semantic search (ChromaDB + Sentence Transformers) combined with Claude AI to answer questions about educational content. The system employs a tool-based approach where Claude decides when to search the vector database.

**Key Architecture Pattern**: The system makes **two Claude API calls per query**:
1. First call determines if search is needed and executes the search tool
2. Second call generates the final answer based on search results

## Development Commands

### Running the Application

```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000

# Access points:
# - Web UI: http://localhost:8000
# - API docs: http://localhost:8000/docs
```

### Dependency Management

**IMPORTANT: Always use `uv`, never use `pip` directly in this project.**

```bash
# Install/sync dependencies
uv sync

# Add a new dependency
uv add <package-name>

# Update dependencies
uv lock --upgrade

# Run Python commands (e.g., scripts, server)
uv run <command>
```

### Environment Setup

The `.env` file must follow this exact format:
```
ANTHROPIC_API_KEY=sk-ant-api03-...
```

Missing the variable name prefix will cause silent failures.

## Core Architecture

### Request Flow (See query-flow-trace.md for detailed diagrams)

```
User Query → FastAPI → RAGSystem → AIGenerator → Claude API (1st call - tool use)
                                         ↓
                                   ToolManager → SearchTool → VectorStore → ChromaDB
                                         ↓
                      Claude API (2nd call - final answer) ← Tool Results
                                         ↓
                              Response + Sources → Frontend
```

### Component Responsibilities

**RAGSystem** (`rag_system.py`):
- Orchestrates all components
- Entry point: `query(query, session_id)` returns `(response, sources)`
- Manages document loading via `add_course_folder()`
- Coordinates session management and source tracking

**AIGenerator** (`ai_generator.py`):
- Wraps Claude API with tool execution logic
- `_handle_tool_execution()` manages the two-call pattern
- System prompt defines when Claude should search vs. use general knowledge
- Temperature=0, max_tokens=800 for deterministic responses

**VectorStore** (`vector_store.py`):
- Manages **two ChromaDB collections**:
  - `course_catalog`: Course metadata (title, instructor, lessons)
  - `course_content`: Text chunks with embeddings
- `search()` is the main interface with semantic course name resolution
- Uses sentence-transformers "all-MiniLM-L6-v2" for embeddings

**ToolManager & SearchTool** (`search_tools.py`):
- Implements the Tool protocol for extensibility
- `CourseSearchTool` tracks sources via `last_sources` attribute
- Sources must be retrieved and reset after each query to avoid stale data

**DocumentProcessor** (`document_processor.py`):
- Chunks text into 800-character pieces with 100-character overlap
- Sentence-based chunking (doesn't break mid-sentence)
- Parses course format:
  ```
  Course Title: [title]
  Course Link: [url]
  Course Instructor: [name]

  Lesson 0: [title]
  Lesson Link: [url]
  [content...]
  ```
- Adds contextual prefixes to chunks: `"Course {title} Lesson {num} content: {chunk}"`

**SessionManager** (`session_manager.py`):
- Maintains conversation history per session
- Limits history to `MAX_HISTORY * 2` messages (user + assistant pairs)
- History is formatted as plain text and injected into system prompt

### Data Models (`models.py`)

- **Course**: Container with title (unique ID), instructor, lessons list
- **Lesson**: Has lesson_number, title, optional link
- **CourseChunk**: Vector store unit with content, course_title, lesson_number, chunk_index

### Configuration (`config.py`)

Critical settings:
- `CHUNK_SIZE`: 800 (must balance context vs. precision)
- `CHUNK_OVERLAP`: 100 (ensures continuity across chunks)
- `MAX_RESULTS`: 5 (search results returned)
- `MAX_HISTORY`: 2 (conversation exchanges to maintain)
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (384 dimensions)

## Important Implementation Details

### Document Loading on Startup

`app.py` has an `@app.on_event("startup")` that loads all `/docs` files. The system:
- Checks existing courses to avoid duplicates (uses course title as unique ID)
- Only processes new courses on restart
- Creates embeddings and stores in persistent ChromaDB

### Course Name Resolution

When filtering by course, the system uses **semantic search** on the course catalog collection to find the best matching course title. This allows partial matches (e.g., "MCP" matches "Introduction to MCP").

### Tool Use Pattern

The `ai_generator.py` system prompt is critical:
- Instructs Claude to use search **only** for course-specific questions
- Limits to **one search per query maximum**
- Tells Claude not to mention "based on search results" (provide direct answers)

### Source Tracking

Sources flow through: `SearchTool.last_sources` → `ToolManager.get_last_sources()` → `RAGSystem.query()` → API response. The `reset_sources()` call after retrieval prevents source leakage between queries.

### Frontend JavaScript (`script.js`)

- Uses `marked.parse()` for markdown rendering
- Session ID maintained in global state (`currentSessionId`)
- Loading state managed by creating/removing DOM elements (not by ID lookups)
- Sources displayed in collapsible `<details>` elements

## Adding New Course Documents

Place `.txt` files in the `/docs` folder with the expected format (see DocumentProcessor section). On next startup:
1. System reads all files in `/docs`
2. Processes only new courses (checks against existing titles)
3. Chunks content with overlap
4. Creates embeddings and stores in ChromaDB

## Extending the System

### Adding New Dependencies

**Always use `uv` to add new packages:**
```bash
uv add package-name
```
This updates both `pyproject.toml` and `uv.lock` automatically. Never manually edit `pyproject.toml` dependencies or use `pip`.

### Adding New Tools

1. Create a class implementing the `Tool` protocol in `search_tools.py`
2. Implement `get_tool_definition()` (Anthropic tool schema)
3. Implement `execute(**kwargs)` (returns string result)
4. Register with `ToolManager.register_tool()`
5. Tool is automatically available to Claude

### Modifying Search Behavior

Key lever: `MAX_RESULTS` in config.py controls precision vs. recall tradeoff
- Lower (3): More precise, may miss context
- Higher (7-10): More context, may dilute relevance

Chunking tradeoff in config.py:
- Larger `CHUNK_SIZE`: More context per chunk, fewer total chunks, coarser search
- Smaller `CHUNK_SIZE`: More precise retrieval, but may break context

## Common Pitfalls

1. **Using pip instead of uv**: This project uses `uv` for ALL dependency management. Never use `pip install`, `pip freeze`, or modify `requirements.txt`. Always use `uv add`, `uv sync`, and `uv run`.
2. **Missing ANTHROPIC_API_KEY prefix in .env**: The key alone without the variable name won't load
3. **Duplicate courses on restart**: System checks titles, so changing title creates a new course
4. **Stale sources**: Must call `reset_sources()` after `get_last_sources()` in RAGSystem
5. **ChromaDB persistence**: Database stored in `./chroma_db` (gitignored), delete to rebuild from scratch
6. **Frontend caching**: `DevStaticFiles` class adds no-cache headers for development

## File Organization

```
backend/           # Python FastAPI backend
  ├── app.py              # FastAPI server + endpoints
  ├── rag_system.py       # Main orchestrator
  ├── ai_generator.py     # Claude API wrapper
  ├── vector_store.py     # ChromaDB interface
  ├── search_tools.py     # Tool definitions
  ├── document_processor.py # Text chunking & parsing
  ├── session_manager.py  # Conversation history
  ├── models.py           # Pydantic data models
  └── config.py           # Configuration settings

frontend/          # Vanilla JS/HTML/CSS
  ├── index.html         # Chat UI structure
  ├── script.js          # API calls & message handling
  └── style.css          # Dark theme styling

docs/              # Course materials (auto-loaded on startup)
```

## Prerequisites

- Python 3.13+
- **uv package manager** (this project uses `uv`, NOT `pip`)
- Anthropic API key
- On Windows: Use Git Bash to run shell scripts
