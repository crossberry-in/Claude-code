# 12. Data Flow Walkthrough

> Tracing a single user request from keystroke to rendered response -- end to end.

---

## The Complete Flow

This diagram traces exactly what happens when a user types a prompt and presses Enter.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#4a9eff', 'primaryBorderColor': '#4a9eff'}}}%%
flowchart TD
    TYPE["User types in terminal"]:::input
    PASTE["Paste: text, image, file"]:::input
    VOICE["Voice input via STT"]:::input
    IDE_IN["IDE sends prompt"]:::input
    SDK_IN["SDK submitMessage"]:::input

    PARSE["Parse input<br/>slash commands, @mentions"]:::process
    SLASH{"Slash<br/>command?"}:::decision
    LOCAL["Execute locally<br/>clear, compact, model"]:::process
    ATTACH["Build attachments<br/>images, PDFs, CLAUDE.md"]:::process
    UMSG["Create UserMessage"]:::process

    SYS["Build system prompt<br/>tool descriptions +<br/>user rules + CLAUDE.md"]:::context
    GIT["Inject git context<br/>branch, commits, status"]:::context
    USR["Inject user context<br/>CLAUDE.md, date"]:::context
    CMP["Run compaction pipeline"]:::context
    CACHE["Apply cache_control<br/>prompt cache breakpoints"]:::context

    BUILD["Build API request<br/>model, betas, effort,<br/>task_budget, thinking"]:::api
    STREAM["Stream SSE response<br/>from Anthropic API"]:::api
    RETRY["withRetry wrapper<br/>429 backoff, 529 fallback"]:::api

    THINK["Thinking blocks"]:::response
    TEXT["Text blocks"]:::response
    TOOL_USE["tool_use blocks"]:::response

    VALIDATE["Validate input"]:::toolexec
    PERMS["Check permissions"]:::toolexec
    EXEC["Execute tool"]:::toolexec
    RESULT["Map to tool_result"]:::toolexec
    FEEDBACK["Push to messages<br/>LOOP BACK"]:::feedback

    RENDER["Render in terminal"]:::output
    TRANSCRIPT["Persist transcript"]:::output
    SDK_OUT["Yield SDK events"]:::output
    COST["Track cost + usage"]:::output

    TYPE --> PARSE
    PASTE --> PARSE
    VOICE --> PARSE
    IDE_IN --> PARSE
    SDK_IN --> PARSE

    PARSE --> SLASH
    SLASH -->|"yes"| LOCAL --> RENDER
    SLASH -->|"no"| ATTACH --> UMSG

    UMSG --> SYS --> GIT --> USR --> CMP --> CACHE

    CACHE --> BUILD --> STREAM
    STREAM --> RETRY --> STREAM

    STREAM --> THINK --> RENDER
    STREAM --> TEXT --> RENDER
    STREAM --> TOOL_USE

    TOOL_USE --> VALIDATE --> PERMS --> EXEC --> RESULT --> FEEDBACK
    FEEDBACK ==>|"loop back to<br/>compaction"| CMP

    TEXT -->|"end_turn"| TRANSCRIPT --> SDK_OUT --> COST

    classDef input fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef process fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef decision fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef context fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef api fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef response fill:#3d2b00,stroke:#fd7e14,color:#e0e0e0,stroke-width:2px
    classDef toolexec fill:#1a1a4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef feedback fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:3px
    classDef output fill:#333,stroke:#888,color:#e0e0e0,stroke-width:1px
```

---

## Phase by Phase

### Phase 1: Input Capture

The user's input can arrive from **5 different sources**, all converging into `processUserInput()`:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#17a2b8', 'primaryBorderColor': '#17a2b8'}}}%%
flowchart LR
    subgraph Sources["Input Sources"]
        KB["Keyboard input<br/>Direct typing in terminal"]
        PASTE["Clipboard paste<br/>Text, images, files"]
        VOICE["Voice<br/>STT transcription"]
        IDE["IDE Bridge<br/>Prompt from editor"]
        SDK["SDK<br/>Programmatic submitMessage"]
    end

    PUI["processUserInput<br/>Parse slash commands<br/>Resolve @mentions<br/>Build attachments"]:::process

    subgraph Output["Parse Result"]
        MSGS["messages array"]
        SHOULD["shouldQuery boolean"]
    end

    Sources --> PUI --> Output

    classDef process fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
```

Slash commands (`/clear`, `/compact`, `/model`) are intercepted here and handled locally without an API call. Everything else proceeds to the model.

### Phase 2: Context Assembly

Before sending to the API, the system assembles context from multiple sources:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#6f42c1', 'primaryBorderColor': '#6f42c1'}}}%%
flowchart TD
    subgraph SystemPrompt["System Prompt Assembly"]
        DEFAULT["Default prompt<br/>Claude Code identity"]
        TOOL_DESC["Tool descriptions<br/>42+ tool schemas"]
        SECTIONS["System prompt sections<br/>Cached or volatile"]
    end

    subgraph UserContext["User Context"]
        CLAUDE_MD["CLAUDE.md files<br/>Project instructions"]
        DATE["Current date"]
    end

    subgraph SystemContext["System Context"]
        GIT_STATUS["Git snapshot<br/>Branch, status, log"]
    end

    subgraph Compaction["Compaction Pipeline"]
        SNIP["Snip compact"]
        MICRO["Micro compact"]
        AUTO["Auto compact"]
        COLLAPSE["Context collapse"]
    end

    SystemPrompt --> FINAL["Final API request"]:::api
    UserContext --> FINAL
    SystemContext --> FINAL
    Compaction --> FINAL

    classDef api fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

### Phase 3: API Call and Streaming

The request goes through `claude.ts` with retry logic:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#28a745', 'primaryBorderColor': '#28a745'}}}%%
flowchart LR
    REQ["API Request<br/>messages + system + tools"]:::req

    STREAM["SSE Stream"]:::stream

    subgraph Events["Stream Events in Order"]
        E1["message_start<br/>model, id, usage"]
        E2["content_block_start<br/>type + index"]
        E3["content_block_delta<br/>incremental text/tool JSON"]
        E4["content_block_stop"]
        E5["message_delta<br/>stop_reason + final usage"]
        E6["message_stop"]
    end

    REQ --> STREAM --> Events

    classDef req fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef stream fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

Each content block is one of three types: **thinking** (internal reasoning), **text** (user-facing response), or **tool_use** (triggers tool execution).

### Phase 4: Tool Execution Loop

When the model responds with `tool_use`, the agentic loop kicks in:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#e83e8c', 'primaryBorderColor': '#e83e8c'}}}%%
flowchart TD
    TOOL_USE["Model returns tool_use blocks"]:::start

    PAR{"Multiple tools?<br/>isConcurrencySafe?"}:::check

    PARALLEL["Execute in parallel<br/>Read-only tools"]:::exec
    SEQUENTIAL["Execute sequentially<br/>Write tools"]:::exec

    VALIDATE["Zod schema validation"]:::step
    HOOKS_PRE["PreToolUse hooks"]:::step
    PERM_CHECK["Permission check chain<br/>deny - allow - tool -<br/>hooks - classifier - dialog"]:::step
    EXECUTE["tool.call(input, context)"]:::step
    HOOKS_POST["PostToolUse hooks"]:::step
    MAP_RESULT["Map to tool_result message"]:::step

    INJECT["Inject CLAUDE.md attachments<br/>for newly-discovered dirs"]:::step

    PUSH["Push tool_results to messages<br/>Continue loop"]:::feedback

    TOOL_USE --> PAR
    PAR -->|"yes"| PARALLEL
    PAR -->|"no"| SEQUENTIAL

    PARALLEL --> VALIDATE
    SEQUENTIAL --> VALIDATE

    VALIDATE --> HOOKS_PRE --> PERM_CHECK --> EXECUTE --> HOOKS_POST --> MAP_RESULT --> INJECT --> PUSH

    PUSH ==>|"Back to<br/>compaction"| TOOL_USE

    classDef start fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef check fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef exec fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef step fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef feedback fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:3px
```

The loop continues until the model returns `stop_reason: end_turn`, hits `max_turns`, or is cancelled.

### Phase 5: Response Rendering

The final response is rendered and persisted:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#fd7e14', 'primaryBorderColor': '#fd7e14'}}}%%
flowchart LR
    RESPONSE["Model final response<br/>stop_reason: end_turn"]:::input

    subgraph Rendering["Terminal Rendering"]
        MD["Markdown rendering<br/>via marked library"]
        SYNTAX["Syntax highlighting<br/>for code blocks"]
        DIFF["Diff rendering<br/>for file changes"]
    end

    subgraph Persistence["Persistence"]
        TRANSCRIPT["recordTranscript<br/>Persist to disk"]
        USAGE["Accumulate usage<br/>input + output tokens"]
        COST["Track cost<br/>per-model pricing"]
    end

    subgraph SDK_Events["SDK Events"]
        MSG["SDKMessage stream<br/>Normalized events"]
        RESULT_E["Result event<br/>total_cost, usage, duration"]
    end

    RESPONSE --> Rendering
    RESPONSE --> Persistence
    RESPONSE --> SDK_Events

    classDef input fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
```

---

## A Concrete Example

Let's trace what happens when a user types: `fix the bug in utils.ts`

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#4a9eff', 'actorTextColor': '#e0e0e0', 'actorBorder': '#4a9eff', 'signalColor': '#4a9eff', 'noteBkgColor': '#16213e', 'noteTextColor': '#e0e0e0', 'activationBkgColor': '#2d1b4e', 'activationBorderColor': '#e83e8c'}}}%%
sequenceDiagram
    participant U as User
    participant R as REPL.tsx
    participant QE as QueryEngine
    participant Q as query.ts
    participant C as claude.ts
    participant T as Tools

    U->>R: Types "fix the bug in utils.ts"
    R->>QE: submitMessage(prompt)

    Note over QE: Not a slash command, proceed

    QE->>QE: Persist transcript
    QE->>Q: query(messages, systemPrompt, tools)

    Note over Q: Turn 1 - Model reads code

    Q->>Q: Run compaction pipeline
    Q->>C: queryModel(messages)
    C-->>Q: tool_use: FileRead("utils.ts")
    Q->>T: Execute FileRead
    T-->>Q: File contents

    Note over Q: Turn 2 - Model analyzes

    Q->>C: queryModel(messages + file contents)
    C-->>Q: tool_use: Grep("error pattern")
    Q->>T: Execute Grep
    T-->>Q: Search results

    Note over Q: Turn 3 - Model fixes

    Q->>C: queryModel(messages + grep results)
    C-->>Q: tool_use: FileEdit("utils.ts", patch)

    Note over T: Permission check!
    T->>U: Allow FileEdit? (Y/n/always)
    U->>T: Y

    T-->>Q: Edit applied

    Note over Q: Turn 4 - Model confirms

    Q->>C: queryModel(messages + edit result)
    C-->>Q: text: "Fixed the bug" + end_turn

    Q-->>QE: Terminal result
    QE->>QE: Record transcript, track usage
    QE-->>R: Render response
    R->>U: Display formatted answer
```

This shows the typical pattern: **read first, analyze, then write** -- with permission checks only on the write operation.

---

## Key Takeaways

1. **Multiple entry points, single pipeline**: Whether input comes from keyboard, IDE, or SDK, it all converges into the same `processUserInput -> query -> claude.ts` pipeline.

2. **Compaction runs every turn**: The 5-stage compaction pipeline runs *before every single API call*, not just when limits are hit.

3. **Tools loop back**: Tool results are pushed to messages, and the entire pipeline (including compaction) runs again. This is the "agentic" part -- the model keeps going until it's done.

4. **Permissions only interrupt writes**: Read-only tools (FileRead, Grep, Glob) in default mode flow through silently. Only write operations (FileEdit, Bash, FileWrite) trigger permission dialogs.

5. **Everything is streamed**: From SSE events to async generators to React state updates, nothing blocks waiting for a full response. The user sees tokens as they arrive.

---

**Previous:** [<- MCP Deep Dive](./11-mcp-deep-dive.md)
