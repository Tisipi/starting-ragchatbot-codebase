# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start - runs the complete application
./run.sh

# Manual start (from backend directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Package Management
```bash
# Install/sync dependencies
uv sync

# Add new dependency
uv add <package-name>
```

### Environment Setup
```bash
# Copy and configure environment variables
cp .env.example .env
# Edit .env to add your ANTHROPIC_API_KEY
```

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** for querying course materials using semantic search and AI-powered responses.

### Core Architecture Pattern
The system follows a **Tool-Enhanced RAG** pattern where Claude uses search tools to retrieve relevant course content before generating responses:

```
User Query → FastAPI → RAG System → AI Generator → Claude API (with tools)
                                        ↓
                                   Tool Manager → Course Search Tool → Vector Store → ChromaDB
```

### Key Components

**RAG System (`rag_system.py`)**: Central orchestrator that coordinates all components. Manages the query lifecycle from initial request to final response with sources.

**AI Generator (`ai_generator.py`)**: Handles dual Claude API interactions - initial query with tools enabled, then tool execution, then final synthesis. Critical pattern: `tool_use` → tool execution → second API call.

**Tool System (`search_tools.py`)**: Implements Anthropic's tool calling interface. The `CourseSearchTool` performs semantic search and tracks sources for UI display.

**Vector Store (`vector_store.py`)**: Manages two ChromaDB collections:
- `course_catalog`: Course metadata for title/instructor matching  
- `course_content`: Text chunks with embeddings for semantic search

**Document Processor (`document_processor.py`)**: Parses structured course documents with format:
```
Course Title: [title]
Course Link: [url] 
Course Instructor: [instructor]
Lesson X: [lesson title]
[lesson content]
```

### Data Flow Patterns

**Document Loading Pipeline**:
1. Parse course metadata and lessons from structured text
2. Chunk lesson content with sentence-aware splitting (800 chars, 100 char overlap)
3. Add contextual prefixes: `"Course [title] Lesson [X] content: [chunk]"`
4. Store in ChromaDB with embeddings via sentence-transformers

**Query Processing Pipeline**:
1. User query → FastAPI validates and creates session if needed
2. RAG System builds prompt and retrieves conversation history
3. AI Generator calls Claude with available tools
4. If Claude requests tool use → Tool Manager executes search → Second Claude call synthesizes results
5. Sources tracked and returned to frontend for display

### Configuration (`config.py`)
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" 
- `CHUNK_SIZE`: 800, `CHUNK_OVERLAP`: 100
- `MAX_RESULTS`: 5, `MAX_HISTORY`: 2 (conversation messages)

### Session Management
Conversations are tracked in-memory with configurable history limits. Sessions auto-created on first query, conversation context passed to Claude for coherent multi-turn interactions.

### Frontend Integration
Vanilla JS frontend at `/frontend/` served by FastAPI. Uses `/api/query` POST endpoint with JSON `{query, session_id}`. Displays responses with markdown parsing and collapsible sources.

## Working with the Codebase

### Adding New Document Types
Extend `DocumentProcessor.process_course_document()` to handle new formats. The current parser expects specific header patterns - modify the regex matching in the method.

### Modifying Search Behavior  
The search system resolves course names semantically via the `course_catalog` collection before filtering `course_content`. Adjust `VectorStore._resolve_course_name()` for different matching strategies.

### Extending Tool Capabilities
Implement the `Tool` abstract base class in `search_tools.py`. Register new tools with `ToolManager.register_tool()`. Tools must provide Anthropic-compatible definitions and execution methods.

### AI Response Customization
Modify the system prompt in `AIGenerator.SYSTEM_PROMPT` to change Claude's behavior. The current prompt emphasizes brevity and educational value while suppressing meta-commentary about search processes.

## Key Design Decisions

**Tool-First Architecture**: Claude decides when to search rather than always performing retrieval, enabling both general knowledge and specific course queries in the same interface.

**Dual Collection Strategy**: Separate storage for course metadata vs content enables semantic course name matching while maintaining efficient content search.

**Contextual Chunking**: Adding course/lesson prefixes to chunks improves search relevance and provides context for AI synthesis.

**Session-Based History**: Limited conversation memory (2 exchanges) balances context awareness with API costs and performance.