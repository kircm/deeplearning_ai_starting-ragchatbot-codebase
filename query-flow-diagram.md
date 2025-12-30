# RAG Chatbot Query Flow Diagram

```mermaid
sequenceDiagram
    actor User
    participant Frontend as Frontend<br/>(script.js)
    participant API as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant Session as Session Manager<br/>(session_manager.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant Claude as Claude API<br/>(Anthropic)
    participant Tools as Tool Manager<br/>(search_tools.py)
    participant Vector as Vector Store<br/>(vector_store.py)
    participant Chroma as ChromaDB<br/>(Database)

    %% User Input
    User->>Frontend: Types query & clicks send
    activate Frontend
    Frontend->>Frontend: Disable input, show loading

    %% API Call
    Frontend->>+API: POST /api/query<br/>{query, session_id}

    %% Session Management
    API->>+Session: Get/create session
    Session-->>-API: session_id

    %% RAG Processing
    API->>+RAG: query(query, session_id)
    RAG->>+Session: get_conversation_history()
    Session-->>-RAG: Previous messages

    %% First Claude Call
    RAG->>+AI: generate_response()<br/>(with tools, history)
    AI->>+Claude: API Call #1<br/>System: "Use search tool"<br/>Tools: [search_course_content]
    Claude-->>-AI: Response: tool_use<br/>{search_course_content}

    %% Tool Execution
    AI->>+Tools: execute_tool()<br/>(query, course_name?, lesson_number?)

    %% Vector Search
    Tools->>+Vector: search(query, filters)

    rect rgb(220, 240, 255)
        Note over Vector,Chroma: Vector Search Process
        alt If course_name provided
            Vector->>+Chroma: Query course_catalog<br/>(resolve course name)
            Chroma-->>-Vector: Best matching course
        end

        Vector->>Vector: Build filters<br/>(course + lesson)
        Vector->>+Chroma: Query course_content<br/>(semantic search)
        Chroma-->>-Vector: Top 5 similar chunks
    end

    Vector-->>-Tools: SearchResults<br/>(docs + metadata)
    Tools->>Tools: Format results<br/>[Course - Lesson N]<br/>content...
    Tools->>Tools: Track sources
    Tools-->>-AI: Formatted search results

    %% Second Claude Call
    AI->>+Claude: API Call #2<br/>Messages: [user, assistant+tool_use, tool_results]
    Claude->>Claude: Synthesize answer<br/>from search results
    Claude-->>-AI: Final answer text
    AI-->>-RAG: Generated response

    %% Finalization
    RAG->>+Tools: get_last_sources()
    Tools-->>-RAG: [sources list]
    RAG->>Tools: reset_sources()
    RAG->>+Session: add_exchange()<br/>(query, answer)
    Session->>Session: Store in history<br/>(max 5 exchanges)
    Session-->>-RAG: ✓

    RAG-->>-API: (answer, sources)

    %% API Response
    API-->>-Frontend: QueryResponse<br/>{answer, sources, session_id}

    %% Display
    Frontend->>Frontend: Remove loading
    Frontend->>Frontend: Render markdown answer
    Frontend->>Frontend: Display sources (collapsible)
    Frontend->>Frontend: Enable input
    deactivate Frontend
    Frontend-->>User: Show answer & sources

    %% Legend
    Note over User,Chroma: Key: Blue box = Vector/Embedding operations
```

## Component Responsibilities

### Frontend Layer
- **script.js**: User interaction, API calls, UI updates
- **index.html**: UI structure, input/output display

### API Layer
- **app.py**: FastAPI endpoints, request/response handling
- **Session Manager**: Conversation history tracking (max 5 exchanges)

### RAG Core
- **RAG System**: Main orchestrator, coordinates all components
- **AI Generator**: Claude API interaction, tool execution loop

### Search Layer
- **Tool Manager**: Tool registration and execution
- **Course Search Tool**: Search interface, result formatting
- **Vector Store**: Dual collections (catalog + content)
- **ChromaDB**: Persistent vector storage with embeddings

## Key Features

1. **Agentic RAG**: Claude decides when to search (not always)
2. **Two-Step Search**:
   - Course catalog (fuzzy name matching)
   - Content search (semantic + filters)
3. **Tool Calling**: Claude API native tool use
4. **Session Context**: Multi-turn conversations with history
5. **Source Tracking**: Separate sources for UI display

## Data Flow

```
Query Text
    ↓
[Sentence Transformer Embedding]
    ↓
[ChromaDB Similarity Search]
    ↓
Retrieved Content Chunks
    ↓
[Claude Synthesis]
    ↓
Natural Language Answer
```
