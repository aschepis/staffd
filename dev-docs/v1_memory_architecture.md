# Memory Architecture Diagram

This document illustrates the memory architecture for staffd agents, showing how memories are stored, retrieved, and managed.

## Architecture Overview

```mermaid
graph TB
    subgraph "Agent Layer"
        A[Agent] -->|Decides to store/retrieve| MT[Memory Tools]
    end

    subgraph "Memory Tools"
        MT -->|memory_normalize| MN[Normalizer]
        MT -->|memory_store_personal| MR[MemoryRouter]
        MT -->|memory_remember_fact| MR
        MT -->|memory_remember_episode| MR
        MT -->|memory_remember_agent_fact| MR
        MT -->|memory_search| MR
        MT -->|memory_search_personal| MR
    end

    subgraph "Normalization Flow"
        MN -->|Raw Text| LLM[Anthropic LLM]
        LLM -->|normalized, type, tags| MR
    end

    subgraph "Storage Layer"
        MR -->|Store| MS[Memory Store]
        MS -->|Generate Embedding| EMB[Embedder]
        EMB -->|Vector| MS
        MS -->|Insert| DB[(SQLite Database)]
    end

    subgraph "Database Schema"
        DB -->|memory_items| MI[Memory Items Table]
        DB -->|memory_items_fts| FTS[FTS5 Virtual Table]
        DB -->|artifacts| ART[Artifacts Table]

        MI -->|Fields| FIELDS["id, agent_id, thread_id,<br/>scope, type, content,<br/>embedding, metadata,<br/>importance, raw_content,<br/>memory_type, tags_json"]
    end

    subgraph "Retrieval Layer"
        MR -->|Query| MS
        MS -->|Hybrid Search| SEARCH[Search Engine]
        SEARCH -->|Vector Search| VS[Cosine Similarity<br/>on Embeddings]
        SEARCH -->|FTS Search| FTS
        SEARCH -->|Tag Search| TS[Tag Matching<br/>Jaccard Similarity]
        VS -->|Results| MERGE[Score Merging]
        FTS -->|Results| MERGE
        TS -->|Results| MERGE
        MERGE -->|Weighted Scores| MR
    end

    style A fill:#e1f5ff
    style MT fill:#fff4e1
    style MR fill:#e8f5e9
    style MS fill:#f3e5f5
    style DB fill:#ffebee
    style SEARCH fill:#e0f2f1
```

## Memory Types and Scopes

```mermaid
graph LR
    subgraph "Memory Types"
        FACT[Fact<br/>Long-term knowledge]
        EPISODE[Episode<br/>Short-term, thread-linked]
        PROFILE[Profile<br/>Normalized personal memory]
        DOCREF[DocRef<br/>Document reference]
    end

    subgraph "Scopes"
        AGENT[Agent Scope<br/>Private to agent]
        GLOBAL[Global Scope<br/>Shared across agents]
    end

    FACT --> AGENT
    FACT --> GLOBAL
    EPISODE --> AGENT
    PROFILE --> AGENT
    DOCREF --> AGENT
    DOCREF --> GLOBAL

    style FACT fill:#ffcdd2
    style EPISODE fill:#c8e6c9
    style PROFILE fill:#bbdefb
    style DOCREF fill:#fff9c4
    style AGENT fill:#e1bee7
    style GLOBAL fill:#b2dfdb
```

## Storage Flow

```mermaid
sequenceDiagram
    participant A as Agent
    participant MT as Memory Tools
    participant N as Normalizer
    participant MR as MemoryRouter
    participant MS as MemoryStore
    participant E as Embedder
    participant DB as SQLite DB

    Note over A,DB: Personal Memory Storage Flow
    A->>MT: memory_normalize(text)
    MT->>N: Normalize(raw text)
    N->>N: Call Anthropic API
    N-->>MT: normalized, type, tags
    MT-->>A: Return normalized data

    A->>MT: memory_store_personal(normalized, type, tags)
    MT->>MR: StorePersonalMemory(...)
    MR->>MS: StorePersonalMemory(...)
    MS->>E: Embed(normalized text)
    E-->>MS: embedding vector
    MS->>DB: INSERT memory_items
    MS->>DB: INSERT memory_items_fts
    DB-->>MS: Success
    MS-->>MR: MemoryItem
    MR-->>MT: MemoryItem
    MT-->>A: Success

    Note over A,DB: Fact/Episode Storage Flow
    A->>MT: memory_remember_fact/episode(...)
    MT->>MR: AddAgentFact/AddEpisode(...)
    MR->>MS: RememberAgentFact/Episode(...)
    MS->>E: Embed(content)
    E-->>MS: embedding vector
    MS->>DB: INSERT memory_items + FTS
    DB-->>MS: Success
    MS-->>MR: MemoryItem
    MR-->>MT: MemoryItem
    MT-->>A: Success
```

## Retrieval Flow

```mermaid
sequenceDiagram
    participant A as Agent
    participant MT as Memory Tools
    participant MR as MemoryRouter
    participant MS as MemoryStore
    participant E as Embedder
    participant SEARCH as Search Engine
    participant DB as SQLite DB

    A->>MT: memory_search(query, include_global, limit)
    MT->>MR: QueryAgentMemory(agentID, query, ...)
    MR->>E: EmbedText(query) [optional]
    E-->>MR: query_embedding
    MR->>MS: SearchMemory(SearchQuery)

    par Vector Search
        MS->>DB: SELECT with filters<br/>LIMIT 500 candidates
        DB-->>MS: Memory items
        MS->>MS: Cosine similarity<br/>on embeddings
    and FTS Search
        MS->>DB: SELECT rowid FROM<br/>memory_items_fts MATCH query
        DB-->>MS: Row IDs
        MS->>DB: Load items by IDs
        DB-->>MS: Memory items
    and Tag Search
        MS->>DB: SELECT with filters<br/>LIMIT 500 candidates
        DB-->>MS: Memory items
        MS->>MS: Tag intersection<br/>Jaccard similarity
    end

    MS->>MS: Merge results<br/>Weighted scores:<br/>Vector: 0.5<br/>Tags: 0.3<br/>FTS: 0.2
    MS->>MS: Sort by score<br/>Apply limit
    MS-->>MR: []SearchResult
    MR-->>MT: []SearchResult
    MT-->>A: Formatted results
```

## Memory Storage Structure

```mermaid
erDiagram
    MEMORY_ITEMS {
        int64 id PK
        string agent_id "nullable, for agent scope"
        string thread_id "nullable, for episodes"
        string scope "agent | global"
        string type "fact | episode | profile | doc_ref"
        string content "normalized text"
        blob embedding "vector embedding"
        text metadata "JSON"
        int64 created_at
        int64 updated_at
        float64 importance "0.0-1.0"
        string raw_content "original text, for profile"
        string memory_type "preference | biographical | habit | goal | value | project | other"
        text tags_json "JSON array of tags"
    }

    MEMORY_ITEMS_FTS {
        int64 rowid PK "references memory_items.id"
        string content "FTS5 indexed"
    }

    ARTIFACTS {
        int64 id PK
        string agent_id "nullable"
        string thread_id "nullable"
        string scope "agent | global"
        string title
        text body
        text metadata "JSON"
        int64 created_at
        int64 updated_at
    }

    MEMORY_ITEMS ||--o{ MEMORY_ITEMS_FTS : "FTS index"
```

## Decision Flow: When to Store

```mermaid
flowchart TD
    START[Agent receives input] --> DECIDE{Is this<br/>memory-worthy?}

    DECIDE -->|No| SKIP[Skip storage<br/>Keep in conversation only]
    DECIDE -->|Yes| TYPE{What type?}

    TYPE -->|Personal info<br/>preferences, habits, goals| PERSONAL[memory_normalize]
    PERSONAL --> NORM[Get normalized, type, tags]
    NORM --> STORE_P[memory_store_personal]
    STORE_P --> DONE1[Stored as Profile]

    TYPE -->|Long-term fact<br/>about user| FACT[memory_remember_fact]
    FACT --> DONE2[Stored as Global Fact]

    TYPE -->|Agent-specific<br/>operational knowledge| AGENT_FACT[memory_remember_agent_fact]
    AGENT_FACT --> DONE3[Stored as Agent Fact]

    TYPE -->|Short-term<br/>task/thread event| EPISODE[memory_remember_episode]
    EPISODE --> DONE4[Stored as Episode]

    style PERSONAL fill:#bbdefb
    style FACT fill:#ffcdd2
    style AGENT_FACT fill:#c8e6c9
    style EPISODE fill:#fff9c4
```

## Hybrid Search Algorithm

```mermaid
flowchart TD
    START[SearchMemory called] --> CHECK{UseHybrid?}

    CHECK -->|No| SINGLE[Use best available method]
    SINGLE --> PRIORITY{Available methods?}
    PRIORITY -->|Vector| VEC_ONLY[Vector search only]
    PRIORITY -->|Tags| TAG_ONLY[Tag search only]
    PRIORITY -->|FTS| FTS_ONLY[FTS search only]
    VEC_ONLY --> RETURN1[Return results]
    TAG_ONLY --> RETURN1
    FTS_ONLY --> RETURN1

    CHECK -->|Yes| HYBRID[Hybrid Search]
    HYBRID --> PAR[Parallel execution]

    PAR --> VEC[Vector Search<br/>Weight: 0.5]
    PAR --> TAG[Tag Search<br/>Weight: 0.3]
    PAR --> FTS_SEARCH[FTS Search<br/>Weight: 0.2]

    VEC --> MERGE[Merge by ID<br/>Sum weighted scores]
    TAG --> MERGE
    FTS_SEARCH --> MERGE

    MERGE --> SORT[Sort by final score]
    SORT --> LIMIT[Apply limit]
    LIMIT --> RETURN2[Return results]

    style HYBRID fill:#e0f2f1
    style MERGE fill:#fff4e1
```

## Key Components

### Memory Store (`memory/store.go`)

- **Primary storage interface** for all memory operations
- Handles embedding generation via `Embedder`
- Manages SQLite transactions
- Inserts into both `memory_items` and `memory_items_fts` tables

### Memory Router (`memory/router.go`)

- **High-level API** for agents to interact with memory
- Routes between different memory types and scopes
- Provides query methods: `QueryAgentMemory`, `QueryPersonalMemory`, etc.
- Handles reflection (episode → fact consolidation)

### Normalizer (`memory/normalizer.go`)

- **Converts raw text** into structured personal memories
- Uses Anthropic API to generate:
  - Normalized third-person text
  - Memory type (preference, biographical, habit, goal, value, project, other)
  - Tags (3-8 lowercase tokens)

### Search Engine (`memory/search.go`)

- **Hybrid search** combining:
  - **Vector search**: Cosine similarity on embeddings (weight: 0.5)
  - **Tag search**: Jaccard similarity on tag intersections (weight: 0.3)
  - **FTS search**: Full-text search via SQLite FTS5 (weight: 0.2)
- Filters by scope, type, importance, time range
- Merges and ranks results by weighted scores

### Embedder (`memory/embedder.go`)

- **Pluggable interface** for generating embeddings
- Implementations: Ollama, Anthropic, OpenAI
- Encodes/decodes embeddings as binary blobs

## Storage Details

### When Memories Are Stored

1. **Agent-initiated**: Agents call memory tools when they detect memory-worthy information
2. **Tool-based**: Via `memory_store_personal`, `memory_remember_fact`, etc.
3. **Automatic**: Reflection can consolidate episodes into facts

### What Gets Stored

- **Content**: The actual memory text (normalized for profiles)
- **Embedding**: Vector representation for semantic search
- **Metadata**: Optional JSON metadata
- **Importance**: Score (0.0-1.0) for filtering
- **Tags**: For personal memories, enables tag-based search
- **FTS Index**: Full-text search index in separate virtual table

### How Memories Are Stored

1. Content is embedded (if embedder available)
2. Transaction begins
3. Insert into `memory_items` table
4. Insert into `memory_items_fts` virtual table (for FTS)
5. Transaction commits

## Retrieval Details

### When Retrieval Happens

- **Agent-initiated**: Agents call `memory_search` or `memory_search_personal` tools
- **On-demand**: Not automatic; agents decide when to query
- **Context-aware**: Agents can include global memories or search only agent-scoped

### How Retrieval Works

1. **Query preparation**: Generate embedding for query text (optional)
2. **Parallel search**: Execute vector, FTS, and tag searches in parallel
3. **Filtering**: Apply scope, type, importance, time range filters
4. **Scoring**: Calculate relevance scores for each method
5. **Merging**: Combine results with weighted scores (hybrid mode) or return best method
6. **Ranking**: Sort by final score and apply limit

### Search Methods

- **Vector Search**: Cosine similarity on embeddings, scans up to 500 candidates
- **FTS Search**: SQLite FTS5 full-text search, returns row IDs
- **Tag Search**: Intersection matching with Jaccard similarity, scans up to 500 candidates

## Memory Lifecycle

```mermaid
stateDiagram-v2
    [*] --> RawInput: Agent receives input
    RawInput --> Normalized: memory_normalize (if personal)
    Normalized --> Stored: memory_store_personal
    RawInput --> Stored: memory_remember_* (if fact/episode)
    Stored --> Indexed: FTS index created
    Stored --> Embedded: Embedding generated
    Indexed --> Searchable
    Embedded --> Searchable
    Searchable --> Retrieved: memory_search
    Retrieved --> Used: Agent uses in context
    Used --> [*]

    note right of Stored
        Stored in SQLite
        with metadata,
        importance score,
        and optional tags
    end note
```
