# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Retrieval-Augmented Generation (RAG) chatbot system for querying course materials. It uses an **agentic tool-based approach** where Claude decides when to search rather than automatically retrieving documents for every query.

## Development Commands

**IMPORTANT**: This project uses `uv` for package management. Always use `uv` commands - **never use `pip` directly**.

### Setup
```bash
# Install dependencies (use uv, not pip!)
uv sync

# Set up environment (required)
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
```

### Running the Application
```bash
# Quick start
./run.sh

# Manual start (from project root)
cd backend && uv run uvicorn app:app --reload --port 8000

# Access points:
# - Web UI: http://localhost:8000
# - API docs: http://localhost:8000/docs
```

### Package Management
```bash
# Add a new dependency (use uv, not pip!)
uv add <package-name>

# Install dependencies
uv sync

# Run Python commands
uv run <command>
```

## Architecture

### Core Design Pattern: Agentic RAG with Two-Phase API Calls

Unlike traditional RAG systems that always retrieve documents, this uses Claude's tool calling:

1. **First API call**: Claude receives the query with search tool available, decides whether to search
2. **Tool execution**: If Claude chooses to search, the search tool executes
3. **Second API call**: Claude receives search results and synthesizes the final answer

This is implemented in [ai_generator.py:43-87](backend/ai_generator.py#L43-L87).

### Dual ChromaDB Collection Strategy

The vector store uses two separate collections for different purposes:

- **`course_catalog`**: Stores course metadata (titles, instructors, lessons)
  - Used for fuzzy course name resolution via semantic search
  - Course title is used as the collection ID
  - Lesson metadata stored as JSON string in metadata field

- **`course_content`**: Stores chunked course content
  - Contains actual searchable text chunks
  - Metadata includes course_title, lesson_number, chunk_index
  - Enables filtered search by course and/or lesson

See [vector_store.py:50-59](backend/vector_store.py#L50-L59) for collection setup.

### Search Resolution Flow

When a search is executed:

1. **Course name resolution** (if course_name provided): Vector search on `course_catalog` to find best matching course title ([vector_store.py:102-116](backend/vector_store.py#L102-L116))
2. **Filter building**: Create ChromaDB filters for course and/or lesson ([vector_store.py:118-133](backend/vector_store.py#L118-L133))
3. **Content search**: Semantic search on `course_content` with filters applied ([vector_store.py:93-97](backend/vector_store.py#L93-L97))
4. **Results formatting**: Add course/lesson context headers to each result ([search_tools.py:88-114](backend/search_tools.py#L88-L114))

### Document Processing Format

Course documents must follow this structure:

```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: [lesson title]
Lesson Link: [lesson url]
[lesson content]

Lesson 1: [lesson title]
Lesson Link: [lesson url]
[lesson content]
```

Documents are processed in [document_processor.py:97-259](backend/document_processor.py#L97-L259) with:
- Sentence-based chunking (800 chars with 100 char overlap)
- First chunk of each lesson prefixed with "Lesson N content:" for context
- Metadata extraction from structured headers

### Session Management

Conversations maintain context through session IDs:
- Maximum 2 conversation exchanges kept in history (4 messages total: 2 user + 2 assistant)
- History automatically trimmed when exceeded ([session_manager.py:34-35](backend/session_manager.py#L34-L35))
- Session history passed to Claude in system prompt ([ai_generator.py:61-65](backend/ai_generator.py#L61-L65))

### Tool Registration Pattern

Tools implement the `Tool` abstract base class ([search_tools.py:6-17](backend/search_tools.py#L6-L17)):
- `get_tool_definition()`: Returns Anthropic-compatible tool schema
- `execute(**kwargs)`: Executes the tool logic

Tools are registered with `ToolManager` which handles execution and tracks sources for UI display.

## Configuration

Key settings in [backend/config.py](backend/config.py):

- **Model**: `claude-sonnet-4-20250514`
- **Embedding**: `all-MiniLM-L6-v2` (Sentence Transformers)
- **Chunk size**: 800 characters
- **Chunk overlap**: 100 characters
- **Max results**: 5 per search
- **Max history**: 2 conversation exchanges
- **ChromaDB path**: `./chroma_db` (relative to backend directory)

## Important Implementation Details

### Why One Search Per Query

The AI Generator system prompt ([ai_generator.py:8-30](backend/ai_generator.py#L8-L30)) explicitly limits searches to **one per query maximum**. This is intentional to:
- Keep responses focused and concise
- Prevent excessive API calls
- Maintain fast response times

### Context Enrichment in Chunks

Chunks are enriched with contextual prefixes during processing:
- First chunk of lesson: `"Lesson {N} content: {chunk}"`
- Last lesson chunks: `"Course {title} Lesson {N} content: {chunk}"`

This helps the embedding model understand the context even when chunks are retrieved in isolation.

### Source Tracking

Sources are tracked separately from the main data flow:
- `CourseSearchTool` stores `last_sources` during result formatting
- `ToolManager.get_last_sources()` retrieves them after AI response
- Sources are reset after each query to prevent carryover

This pattern keeps source metadata separate from the content passed to Claude.

## Adding New Courses

Place course documents in the `docs/` folder (supports `.txt`, `.pdf`, `.docx`). The application:
- Automatically loads documents on startup ([app.py:88-98](backend/app.py#L88-L98))
- Checks existing course titles to avoid duplicates ([rag_system.py:76-96](backend/rag_system.py#L76-L96))
- Skips already-processed courses (idempotent)

## Key Files by Function

- **API layer**: [backend/app.py](backend/app.py)
- **Orchestration**: [backend/rag_system.py](backend/rag_system.py)
- **Claude interaction**: [backend/ai_generator.py](backend/ai_generator.py)
- **Vector operations**: [backend/vector_store.py](backend/vector_store.py)
- **Search tools**: [backend/search_tools.py](backend/search_tools.py)
- **Document parsing**: [backend/document_processor.py](backend/document_processor.py)
- **Data models**: [backend/models.py](backend/models.py)
- **Session state**: [backend/session_manager.py](backend/session_manager.py)
- **Configuration**: [backend/config.py](backend/config.py)
- **Frontend**: [frontend/script.js](frontend/script.js), [frontend/index.html](frontend/index.html)
