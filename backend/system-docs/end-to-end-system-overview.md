# End-to-End System Overview

Version: 1.0
Audience: Project Managers, Product Owners, Stakeholders
Purpose: Explain how the chat system works from a user’s first input to the streamed answer, including how conversation history is maintained.

## Executive Summary
- The system helps users ask questions and get focused answers, quickly and reliably.
- Users can either type a question or click a suggested related option. Answers stream in real time for responsiveness.
- The system remembers conversation context so follow‑up questions like "that" or "tell me again" feel natural.

## User Journeys

### 1) Typed Question Flow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Frontend UI
    participant API as Backend API
    participant CF as ChatFlowService
    participant RM as RetrievalManager
    participant CM as ResponseManager
    participant S as Session Store

    U->>UI: Types question "How do I pay fees?"
    UI->>API: POST /chat/text/stream
    API->>CF: stream_chat()
    CF->>RM: search() with question
    RM->>RM: Vector search in knowledge base
    RM-->>CF: Returns top hits + options
    CF->>CM: stream_answer() with hits
    CM->>CM: Generate answer using LLM + context
    CM-->>CF: Stream tokens + faq_meta
    CF-->>UI: SSE: token, faq_meta, options, done
    UI->>U: Shows answer + clickable suggestions
    CF->>S: persist_turn(user_text, assistant_text, faq_id)
    Note over S: Conversation history updated
```

### 2) Click Fast‑Path Flow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Frontend UI
    participant API as Backend API
    participant CF as ChatFlowService
    participant CV as ChoiceValidator
    participant CM as ResponseManager
    participant S as Session Store

    U->>UI: Clicks suggestion "Bank transfer"
    UI->>UI: Shows clicked question as user message
    UI->>API: POST /chat/text/stream?option_faq_id=456
    API->>CF: stream_choice_fast_path()
    CF->>CV: validate_membership(option_faq_id)
    CV->>S: Check last_search_results
    S-->>CV: Returns stored options
    CV-->>CF: Validation success
    CF->>CF: Fetch FAQ record
    CF->>CM: stream_answer() with FAQ-only context
    CM->>CM: Generate focused answer (no retrieval)
    CM-->>CF: Stream tokens + faq_meta
    CF-->>UI: SSE: token, faq_meta, done
    UI->>U: Shows focused answer
    CF->>S: persist_turn(FAQ.question, assistant_text, faq_id)
    Note over S: History shows clicked question as user message
```

### 3) No‑Results Fallback Flow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Frontend UI
    participant API as Backend API
    participant CF as ChatFlowService
    participant RM as RetrievalManager
    participant CM as ResponseManager
    participant S as Session Store

    U->>UI: Types question "Unknown topic"
    UI->>API: POST /chat/text/stream
    API->>CF: stream_chat()
    CF->>RM: search() with question
    RM->>RM: Vector search in knowledge base
    RM-->>CF: Returns empty hits
    CF->>CF: _handle_no_results_fallback()
    CF->>CM: fallback_with_original_question()
    CM->>CM: Try LLM with original question only
    alt Fallback succeeds
        CM-->>CF: Returns answer + faq_id
        CF-->>UI: SSE: token, faq_meta, done
        UI->>U: Shows answer
        CF->>S: persist_turn(user_text, assistant_text, faq_id)
    else Fallback fails
        CF-->>UI: SSE: error, done
        UI->>U: Shows friendly guidance message
        Note over S: No turn persisted
    end
```

## Conversation History Data Flow

```mermaid
graph LR
    subgraph "Session Store (Redis)"
        CT["context:{sid}<br/>Conversation Turns"]
        SR["session:last_search_results:{sid}<br/>Search Results Snapshot"]
    end

    subgraph "Write Operations"
        W1["persist_turn<br/>user_text + assistant_text + faq_id"]
        W2["Store search results<br/>hits + scores + metadata"]
    end

    subgraph "Read Operations"
        R1["Load conversation history<br/>for query reconstruction"]
        R2["Validate option clicks<br/>against stored results"]
        R3["Rebuild suggestions<br/>from stored results"]
    end

    subgraph "Triggers"
        T1["Successful answer streaming"]
        T2["Search results generated"]
        T3["User clicks suggestion"]
        T4["Next question asked"]
    end

    T1 --> W1
    T2 --> W2
    T3 --> R2
    T4 --> R1

    W1 --> CT
    W2 --> SR
    R1 --> CT
    R2 --> SR
    R3 --> SR

    style CT fill:#e3f2fd
    style SR fill:#e8f5e8
    style W1 fill:#fff3e0
    style W2 fill:#fff3e0
    style R1 fill:#f3e5f5
    style R2 fill:#f3e5f5
    style R3 fill:#f3e5f5
```
## How the System Remembers Your Conversation

The chat system keeps track of your conversation so it can provide helpful, contextual responses. Here's how it works:

### What Gets Remembered
- **Your questions and the assistant's answers**: Each exchange is saved so the system can understand what you've already discussed
- **The most recent answer's source**: When the assistant answers from a specific FAQ, it remembers which one so follow-up questions can reference it
- **Available suggestions**: The system remembers what options were presented to you, so clicking them works correctly

### How It Works
- **After successful answers**: When the assistant provides a helpful answer, the conversation is automatically saved
- **For clicked suggestions**: When you click a suggested question, the system saves that clicked question as your message (so the conversation reads naturally)
- **Smart updates**: The system refreshes the available suggestions whenever new options are presented
- **Quality control**: Confusing responses or errors are not saved, keeping the conversation history clean and useful

### When Information Is Saved
- **Typed questions**: Saved immediately after the assistant finishes providing a helpful answer
- **Fallback answers**: Only saved if the system finds a good answer after the initial search fails
- **Clicked suggestions**: Saved after the focused answer is provided (clicking the same suggestion again won't create duplicates)

### Why This Matters
- **Natural conversations**: You can ask follow-up questions like "tell me more about that" or "what was the first option again?"
- **Accurate history**: The conversation shows exactly what you asked and what the assistant answered
- **Reliable suggestions**: Clicked suggestions always work because the system validates them against the latest available options

### Privacy and Freshness
- **Automatic cleanup**: Conversation data expires after periods of inactivity to protect your privacy
- **Fresh suggestions**: If suggestions become outdated, the system automatically provides new ones instead of failing

## High‑Level Architecture

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[Chat UI]
        OPT[Options Component]
        SSE[SSE Client]
    end

    subgraph "Backend API Layer"
        API[FastAPI Router]
        DEP[Dependencies]
    end

    subgraph "Core Services"
        CF[ChatFlowService]
        RM[RetrievalManager]
        CM[ResponseManager]
        CV[ChoiceValidator]
        OB[OptionsBuilder]
        QR[QueryReconstructor]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL<br/>FAQs & Content)]
        CH[(ChromaDB<br/>Vector Search)]
        RD[(Redis<br/>Session Store)]
    end

    subgraph "External Services"
        LLM[OpenAI LLM]
    end

    UI --> API
    OPT --> API
    SSE --> API

    API --> DEP
    DEP --> CF

    CF --> RM
    CF --> CM
    CF --> CV
    CF --> OB
    CF --> QR

    RM --> CH
    RM --> PG
    CM --> LLM
    CV --> RD
    OB --> RD
    QR --> LLM

    CF --> RD

    style UI fill:#e1f5fe
    style API fill:#f3e5f5
    style CF fill:#fff3e0
    style PG fill:#e8f5e8
    style CH fill:#e8f5e8
    style RD fill:#e8f5e8
    style LLM fill:#fff8e1
```

- Frontend (Web): A chat UI that shows messages, renders suggestion chips, and connects to a streaming endpoint.
- Backend (API): A chat endpoint that orchestrates searching, answering, and streaming responses back to the browser.
- Knowledge & State: A structured knowledge base for FAQs and content, plus a session store for conversation history and the latest search results snapshot.

## SSE Streaming Flow

```mermaid
sequenceDiagram
    participant UI as Frontend UI
    participant API as Backend API
    participant CF as ChatFlowService
    participant CM as ResponseManager
    participant LLM as OpenAI LLM

    UI->>API: POST /chat/text/stream
    API->>CF: stream_chat()

    Note over CF: Generate answer using LLM
    CF->>CM: stream_answer()
    CM->>LLM: Generate tokens
    LLM-->>CM: Stream tokens

    loop For each token
        CM-->>CF: token event
        CF-->>UI: {"type":"token","payload":{"text":"..."}}
        UI->>UI: Append to answer display
    end

    Note over CF: Answer complete
    CF-->>UI: {"type":"faq_meta","payload":{"faq_id":123,"question":"..."}}
    UI->>UI: Show FAQ metadata

    alt Options available
        CF-->>UI: {"type":"options","version":"v1","payload":{"options":[...]}}
        UI->>UI: Render clickable suggestions
    end

    CF-->>UI: {"type":"done"}
    UI->>UI: Mark stream complete

    alt Error occurs
        CF-->>UI: {"type":"error","payload":{"message":"..."}}
        UI->>UI: Show error message
        CF-->>UI: {"type":"done"}
    end
```

Example stream (simplified):
```json
{"type":"token","payload":{"text":"To pay your fees,"}}
{"type":"token","payload":{"text":" you can use bank transfer..."}}
{"type":"faq_meta","payload":{"faq_id":123,"question":"How do I pay my fees?","source_url":null}}
{"type":"options","version":"v1","payload":{"options":[{"index":1,"faq_id":456,"question":"Bank transfer","score":0.92}]}}
{"type":"done"}
```

## Retrieval and Suggestions
- Defaults guide how many items are searched and presented:
  - Top‑K search returns up to 4 results; the top result becomes the main answer.
  - Up to 3 additional suggestions are presented (excluding the top result so we don’t duplicate content).
  - A minimum relevance score ensures low‑quality results are filtered out.
- Suggestions are “click‑first”: users should click the suggestion chip/link. If users type “1/2/3” or the option title, it’s treated as a new question and goes through the full pipeline.

## Answer Generation
- Typed questions: the answer is generated using relevant content and the conversation history to keep the reply coherent with prior turns.
- Click fast‑path: the answer is generated directly from the selected FAQ’s content. This skips additional searching and produces a fast, focused answer.

## Clarifier and Fallback Behavior
- The assistant is encouraged to answer whenever possible.
- If there isn’t enough clarity to provide a reliable answer, it may produce a short clarifier. Clarifier messages are not saved as final turns.
- If no results are found, the system tries a last‑chance fallback using the original question (without extra context). If a clear answer is found, it is streamed and saved. Otherwise, a friendly guidance message is returned.

## Error Handling Workflow

```mermaid
flowchart TD
    START[User Action] --> TYPE{Action Type}

    TYPE -->|Typed Question| TYPED[Typed Question Flow]
    TYPE -->|Click Option| CLICK[Click Fast-Path Flow]

    TYPED --> SEARCH[Search Knowledge Base]
    SEARCH --> HITS{Results Found?}

    HITS -->|Yes| ANSWER[Generate Answer]
    HITS -->|No| FALLBACK[Try Fallback]

    FALLBACK --> FALLBACK_RESULT{Fallback Success?}
    FALLBACK_RESULT -->|Yes| ANSWER
    FALLBACK_RESULT -->|No| ERROR_MSG[Show Friendly Error]

    ANSWER --> CLARIFIER{Is Clarifier?}
    CLARIFIER -->|Yes| NO_PERSIST[Don't Persist Turn]
    CLARIFIER -->|No| PERSIST[Persist Turn]

    CLICK --> VALIDATE[Validate Option Click]
    VALIDATE --> VALID{Valid?}

    VALID -->|Yes| FAST_ANSWER[Generate Fast Answer]
    VALID -->|No| STALE_ERROR[Show Stale Options Error]

    FAST_ANSWER --> FAST_PERSIST[Persist Turn]

    ERROR_MSG --> END[End Stream]
    NO_PERSIST --> END
    PERSIST --> END
    STALE_ERROR --> END
    FAST_PERSIST --> END

    style ERROR_MSG fill:#ffebee
    style STALE_ERROR fill:#ffebee
    style NO_PERSIST fill:#fff3e0
    style PERSIST fill:#e8f5e8
    style FAST_PERSIST fill:#e8f5e8
```

## Observability and Success Signals
- We track key operational and product signals, such as:
  - Time to first streamed token and time to final “done”.
  - How often users choose the click fast‑path vs. typing a follow‑up.
  - How often retrieval can be skipped because the system already has enough context.
- These signals guide performance improvements and product decisions.

## Performance and Scalability
- Streaming provides fast perceived responsiveness.
- The click fast‑path reduces extra searching, improving both performance and cost efficiency.

## Security and Privacy
- Conversation history is session‑based and expires after inactivity.
- We store only what is needed to deliver accurate, contextual answers and validate suggestion clicks.

## Configuration Defaults (Business‑Level)
- Sensible defaults ensure consistent behavior: how many results we consider, how many suggestions we show, and the relevance threshold for quality.

## Examples (Appendix)
### Example A — Typed Question with Suggestions
- User: “How do I pay my fees?”
- Assistant streams the answer and presents 3 suggestions (e.g., bank transfer, due dates, international payments).
- The turn is saved after the answer finishes.

### Example B — Click Fast‑Path (Focused Answer)
- User clicks “Bank transfer”.
- The UI shows the clicked question as a user message, then streams the focused answer.
- The turn is saved; re‑clicking does not duplicate the history.

### Example C — No Results → Fallback → Success
- The initial search finds nothing relevant.
- The fallback with the original question finds a clear answer and streams it.
- The turn is saved.

### Example D — No Results → Fallback → Still No Answer
- Neither the initial search nor fallback produces a reliable answer.
- The system returns a friendly message guiding the user to rephrase or try another topic.
- The turn is not saved because there is no conclusive answer.
