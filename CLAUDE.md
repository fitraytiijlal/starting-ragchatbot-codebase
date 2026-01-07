# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Running the Application
```bash
# Quick start (from root)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access points:
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

### Environment Setup
```bash
# Install dependencies
uv sync

# Required .env file in root
ANTHROPIC_API_KEY=your_key_here
```

### Development
```bash
# Run from root - server runs with --reload flag for auto-restart on changes
./run.sh

# Rebuild vector database (clear and re-index all courses)
# Edit backend/app.py startup_event: change clear_existing=False to True
# Then restart server
```

## Architecture Overview

### RAG System Flow

This is a **tool-calling RAG system** where Claude decides when to search course content:

```
User Query (via frontend)
    ↓
FastAPI app.py → RAGSystem.query()
    ↓
AIGenerator.generate_response()
    ├─ System prompt instructs Claude to use search tool when needed
    ├─ Claude decides: general knowledge OR search course content
    ├─ If search needed: ToolManager → CourseSearchTool → VectorStore
    ├─ VectorStore performs semantic search in ChromaDB
    ├─ Results returned to Claude
    └─ Claude synthesizes final response
    ↓
SessionManager updates conversation history
    ↓
Response + sources returned to frontend
```

### Component Responsibilities

**RAGSystem (rag_system.py)** - Main orchestrator
- Initializes all components (document processor, vector store, AI generator, session manager)
- Registers search tools with ToolManager
- Coordinates query flow from API to response
- Handles document ingestion from `/docs` folder

**VectorStore (vector_store.py)** - Dual ChromaDB collections
- `course_catalog` collection: Course metadata (titles, instructors, links)
- `course_content` collection: Text chunks with embeddings
- Implements course name resolution: searches catalog first, then filters content
- Supports filtering by course name and lesson number
- Uses sentence-transformers/all-MiniLM-L6-v2 for embeddings

**AIGenerator (ai_generator.py)** - Claude integration
- Implements tool-calling pattern (not direct RAG)
- System prompt instructs Claude to use `search_course_content` tool
- Limits: one search per query, concise responses
- Temperature: 0 (deterministic), Max tokens: 800
- Handles tool execution loop and final response generation

**ToolManager + CourseSearchTool (search_tools.py)** - Tool abstraction
- ToolManager: Registry for tools, provides definitions to Claude, executes tools
- CourseSearchTool: Wraps VectorStore.search(), formats results, tracks sources
- Tool parameters: query (required), course_name (optional), lesson_number (optional)

**DocumentProcessor (document_processor.py)** - Course document parsing
- Expected format: Course metadata header + Lesson N: sections
- Sentence-based chunking (respects sentence boundaries)
- Creates CourseChunk objects with course/lesson context
- Chunk size: 800 chars, overlap: 100 chars (config.py)

**SessionManager (session_manager.py)** - Conversation state
- In-memory session storage (UUID-based)
- Maintains last 2 exchanges (MAX_HISTORY in config.py)
- Formats history for Claude context injection
- Sessions are stateless across restarts

### Key Architectural Decisions

1. **Tool-calling vs Direct RAG**: Claude decides when to search rather than always retrieving context. Enables general questions without forced searches.

2. **Dual Collections**: Separating course metadata from content enables two-stage search - resolve course names semantically, then search within content.

3. **Persistent ChromaDB**: Vector database persists in `backend/chroma_db/` across restarts. Incremental loading prevents re-indexing existing courses.

4. **Startup Document Loading**: `app.py` on_event("startup") auto-loads `/docs` folder. Uses `clear_existing=False` to only add new courses.

5. **Session-based History**: Session IDs enable multi-user/multi-tab support without shared state pollution.

6. **FastAPI + Static Frontend**: Backend serves frontend static files from `/frontend` via mount. Single server process for both API and UI.

### Configuration (backend/config.py)

All system parameters centralized:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2"
- `CHUNK_SIZE`: 800 (text chunk size)
- `CHUNK_OVERLAP`: 100 (overlap for context)
- `MAX_RESULTS`: 5 (search results returned)
- `MAX_HISTORY`: 2 (conversation exchanges remembered)
- `CHROMA_PATH`: "./chroma_db" (vector storage location)

### Course Document Format

Documents in `/docs` must follow this structure:

```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [lesson title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [lesson title]
[lesson content...]
```

DocumentProcessor parses this format to extract metadata and chunk content by lesson.

### Frontend (frontend/)

Vanilla JS application:
- `script.js`: Manages session ID, calls POST /api/query, renders responses with sources
- `index.html`: Two-column layout (sidebar with stats + chat area)
- `style.css`: Dark theme with CSS variables
- Uses marked.js for markdown rendering in AI responses

### API Endpoints

**POST /api/query** - Main query endpoint
- Request: `{query: string, session_id?: string}`
- Response: `{answer: string, sources: string[], session_id: string}`
- Creates new session if not provided
- Returns sources from tool searches

**GET /api/courses** - Course statistics
- Response: `{total_courses: number, course_titles: string[]}`
- Used by frontend sidebar to display catalog

### Adding New Tools

To add a new tool for Claude to use:

1. Create tool class in `search_tools.py` implementing `execute()` method
2. Define tool schema with name, description, input_schema
3. Register with ToolManager in `RAGSystem.__init__()`:
   ```python
   new_tool = NewTool(dependencies)
   self.tool_manager.register_tool(new_tool)
   ```
4. ToolManager automatically provides definition to Claude
5. AIGenerator handles tool execution via ToolManager

### ChromaDB Collections

Both collections use cosine similarity for semantic search:

**course_catalog**
- Documents: Course titles with metadata
- Metadata: instructor, course_link
- Purpose: Resolve course names from natural language

**course_content**
- Documents: Text chunks from lessons
- Metadata: course_title, lesson_number, lesson_title, course_link, lesson_link
- Purpose: Semantic search within course materials

### Common Modifications

**Change AI model**: Edit `ANTHROPIC_MODEL` in `backend/config.py`

**Adjust chunk size**: Edit `CHUNK_SIZE` and `CHUNK_OVERLAP` in `backend/config.py`

**Change search result count**: Edit `MAX_RESULTS` in `backend/config.py`

**Modify system prompt**: Edit `ai_generator.py` system prompt in `generate_response()`

**Add new document types**: Edit `add_course_folder()` in `rag_system.py` file extension check

**Change conversation memory**: Edit `MAX_HISTORY` in `backend/config.py`
