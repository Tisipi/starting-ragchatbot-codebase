# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend<br/>(script.js)
    participant API as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant TM as Tool Manager<br/>(search_tools.py)
    participant ST as Search Tool<br/>(CourseSearchTool)
    participant VS as Vector Store<br/>(vector_store.py)
    participant DB as ChromaDB<br/>(Embeddings)
    participant Claude as Anthropic<br/>Claude API

    U->>F: Types query & clicks send
    F->>F: Display loading animation
    F->>API: POST /api/query<br/>{query, session_id}
    
    API->>RAG: rag_system.query(query, session_id)
    RAG->>RAG: Build prompt:<br/>"Answer this question about course materials: {query}"
    RAG->>AI: generate_response(prompt, history, tools, tool_manager)
    
    AI->>AI: Build system prompt + conversation context
    AI->>Claude: messages.create()<br/>with tools enabled
    Claude-->>AI: Response with tool_use
    
    AI->>AI: _handle_tool_execution()
    AI->>TM: execute_tool("search_course_content", **params)
    TM->>ST: CourseSearchTool.execute(query, course_name, lesson_number)
    
    ST->>VS: search(query, course_name, lesson_number)
    VS->>VS: _resolve_course_name() if needed
    VS->>DB: course_catalog.query() for course matching
    DB-->>VS: Best matching course
    VS->>VS: _build_filter() for search constraints
    VS->>DB: course_content.query() with embeddings
    DB-->>VS: Ranked search results
    VS-->>ST: SearchResults(documents, metadata, distances)
    
    ST->>ST: _format_results()<br/>Add course/lesson context
    ST->>ST: Store sources for UI
    ST-->>TM: Formatted search results
    TM-->>AI: Tool execution results
    
    AI->>Claude: Second API call with tool results
    Claude-->>AI: Final synthesized answer
    AI-->>RAG: Generated response text
    
    RAG->>TM: get_last_sources()
    TM-->>RAG: Sources list from search
    RAG->>RAG: Update conversation history
    RAG-->>API: (response, sources)
    
    API-->>F: QueryResponse<br/>{answer, sources, session_id}
    F->>F: Remove loading animation
    F->>F: addMessage() with markdown parsing
    F->>U: Display answer + sources
```

## Key Components

### Data Flow Layers:
1. **Presentation**: Frontend JavaScript handles UI interactions
2. **API**: FastAPI routes and validates requests  
3. **Orchestration**: RAG System coordinates components
4. **Intelligence**: AI Generator manages Claude interactions
5. **Search**: Tool system executes semantic search
6. **Storage**: Vector Store queries ChromaDB embeddings

### Critical Transformations:
- **User Text** → **API JSON** → **Vector Search** → **Tool Results** → **AI Synthesis** → **HTML Display**

### Parallel Operations:
- Course name resolution and content search happen in sequence
- Tool execution and AI response generation are tightly coupled
- Frontend shows loading state while backend processes

### Error Handling:
- Each layer can return errors that propagate back to user
- Vector store handles missing courses gracefully
- Frontend displays error messages in chat interface