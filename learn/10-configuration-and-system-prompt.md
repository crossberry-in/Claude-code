# 10. Configuration and System Prompt

> How settings cascade from enterprise policy down to project CLAUDE.md, and how the system prompt is assembled.

---

## Configuration Hierarchy

Claude Code loads settings from **5 layers**, each with strict priority. Higher layers override lower ones.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#4a9eff', 'primaryBorderColor': '#4a9eff'}}}%%
flowchart TD
    subgraph Sources["Configuration Sources — Highest to Lowest Priority"]
        direction TB
        ENT["Enterprise / MDM Policy<br/>managed-settings.json<br/>Org-level, cannot override"]:::ent
        USER["User Settings<br/>~/.claude/settings.json<br/>Per-user global defaults"]:::user
        PROJ["Project Settings<br/>.claude/settings.json<br/>Per-repo configuration"]:::proj
        MCP_CFG[".mcp.json<br/>Project MCP servers<br/>Committed to repo"]:::proj
        CLAUDE_MD["CLAUDE.md files<br/>Instructions, rules,<br/>memory for the model"]:::md
    end

    ENT --> MERGE
    USER --> MERGE
    PROJ --> MERGE
    MCP_CFG --> MERGE
    CLAUDE_MD --> MERGE

    MERGE["getInitialSettings<br/>Merge all layers<br/>Higher priority wins"]:::merge

    APPSTATE["AppState.settings<br/>Runtime configuration"]:::state

    MERGE --> APPSTATE

    classDef ent fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
    classDef user fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef proj fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef md fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef merge fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef state fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

### Enterprise / MDM Policy

Highest priority. Set by organization admins via Mobile Device Management (macOS profiles). Stored at a managed file path. Users **cannot** override these settings. Controls things like:
- Allowed/denied MCP servers
- Permission mode restrictions
- Feature availability
- Analytics opt-out

### User Settings (`~/.claude/settings.json`)

Per-user defaults. Controls preferences like allowed tools, custom deny rules, and hooks.

### Project Settings (`.claude/settings.json`)

Per-repository settings. Committed to the repo so all collaborators share the same configuration.

### `.mcp.json`

MCP server definitions for the project. Lives at the repo root. Defines which MCP servers are available. Validated via Zod schema (`McpServerConfigSchema`).

### CLAUDE.md

Markdown instruction files that the model reads as part of its system prompt. These are the "memory" files that tell Claude about this specific project.

---

## CLAUDE.md Discovery

CLAUDE.md files are discovered from multiple locations and merged:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#6f42c1', 'primaryBorderColor': '#6f42c1'}}}%%
flowchart TD
    subgraph Discovery["CLAUDE.md Discovery — getMemoryFiles"]
        GLOBAL["~/.claude/CLAUDE.md<br/>Global instructions"]:::global
        CWD_WALK["Walk from cwd up to git root<br/>Check each dir for CLAUDE.md<br/>and .claude/CLAUDE.md"]:::walk
        ADDITIONAL["--add-dir paths<br/>Extra directories to scan"]:::additional
    end

    FILTER["filterInjectedMemoryFiles<br/>Remove duplicates,<br/>check setting sources"]:::filter

    PARSE["getClaudeMds<br/>Read and concatenate<br/>all discovered files"]:::parse

    CACHE["setCachedClaudeMdContent<br/>Cache for auto-mode<br/>classifier to read"]:::cache

    INJECT["Injected into getUserContext<br/>becomes part of system prompt"]:::inject

    Discovery --> FILTER --> PARSE --> CACHE --> INJECT

    classDef global fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef walk fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef additional fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef filter fill:#3d2b00,stroke:#fd7e14,color:#e0e0e0,stroke-width:2px
    classDef parse fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef cache fill:#333,stroke:#888,color:#e0e0e0,stroke-width:1px
    classDef inject fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

The `--bare` flag skips auto-discovery but still honors explicit `--add-dir` paths.

---

## System Prompt Assembly

The system prompt is what the model "sees" before any conversation messages. It's assembled from multiple pieces:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#e83e8c', 'primaryBorderColor': '#e83e8c'}}}%%
flowchart TD
    subgraph Priority["System Prompt Priority — buildEffectiveSystemPrompt"]
        OVERRIDE["1. Override System Prompt<br/>Set via loop mode<br/>REPLACES everything"]:::override
        COORD["2. Coordinator System Prompt<br/>If coordinator mode active"]:::coord
        AGENT["3. Agent System Prompt<br/>If mainThreadAgentDefinition set<br/>In proactive mode: APPENDED<br/>Otherwise: REPLACES default"]:::agent
        CUSTOM["4. Custom System Prompt<br/>Via --system-prompt flag"]:::custom
        DEFAULT["5. Default System Prompt<br/>The standard Claude Code prompt"]:::default
    end

    APPEND["appendSystemPrompt<br/>Always appended at end<br/>except when override is set"]:::append

    EFFECTIVE["Effective System Prompt<br/>Sent as system param<br/>in API request"]:::result

    OVERRIDE -->|"if set"| EFFECTIVE
    COORD -->|"if coordinator"| EFFECTIVE
    AGENT -->|"if agent defined"| EFFECTIVE
    CUSTOM -->|"if --system-prompt"| EFFECTIVE
    DEFAULT -->|"fallback"| EFFECTIVE
    APPEND --> EFFECTIVE

    classDef override fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
    classDef coord fill:#3d2b00,stroke:#fd7e14,color:#e0e0e0,stroke-width:2px
    classDef agent fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef custom fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef default fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef append fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef result fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

### System Prompt Sections

Individual sections of the system prompt are defined via `systemPromptSection()`:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#4a9eff', 'primaryBorderColor': '#4a9eff'}}}%%
flowchart LR
    subgraph Sections["System Prompt Sections"]
        CACHED["Cached Sections<br/>Computed once at start<br/>Reused every turn<br/>Cleared on /clear or /compact"]:::cached
        VOLATILE["DANGEROUS Uncached Sections<br/>Recomputed every turn<br/>BREAKS prompt cache"]:::volatile
    end

    RESOLVE["resolveSystemPromptSections<br/>Check cache, compute if needed"]:::resolve

    PROMPT["Final system prompt string"]:::result

    CACHED --> RESOLVE
    VOLATILE --> RESOLVE
    RESOLVE --> PROMPT

    classDef cached fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef volatile fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
    classDef resolve fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef result fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
```

Most sections are **cached** (computed once, reused every turn) to preserve prompt cache hits. Volatile sections that recompute every turn are explicitly named `DANGEROUS_uncachedSystemPromptSection` as a warning.

---

## Context Injection

Beyond the system prompt, two context objects are injected into every conversation:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#28a745', 'primaryBorderColor': '#28a745'}}}%%
flowchart LR
    subgraph SystemCtx["getSystemContext — memoized"]
        GIT["Git status snapshot<br/>branch, status, log,<br/>default branch, user"]
        INJECT_SYS["System prompt injection<br/>internal debugging only"]
    end

    subgraph UserCtx["getUserContext — memoized"]
        CMD["CLAUDE.md content<br/>all discovered files<br/>concatenated"]
        DATE["Current date<br/>ISO format"]
    end

    PREPEND["prependUserContext<br/>Prepended to messages<br/>before API call"]:::merge

    SystemCtx --> PREPEND
    UserCtx --> PREPEND

    classDef merge fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

Both are **memoized** and cached for the duration of the conversation. They're cleared on `/clear` and `/compact`.

---

## Markdown Config Discovery

Commands, agents, skills, and workflows are all loaded via `markdownConfigLoader.ts`:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#fd7e14', 'primaryBorderColor': '#fd7e14'}}}%%
flowchart TD
    subgraph Dirs["Search Directories — Priority Order"]
        MANAGED["Managed (enterprise)<br/>Policy-level configs"]:::ent
        USER_DIR["User (~/.claude/)<br/>Personal configs"]:::user
        PROJ_DIRS["Project (.claude/)<br/>Walk cwd up to git root<br/>Check each ancestor"]:::proj
    end

    LOADER["loadMarkdownFilesForSubdir<br/>Discover + parse YAML frontmatter<br/>+ markdown content"]:::loader

    DEDUP["Deduplicate by inode<br/>Handle symlinks gracefully"]:::dedup

    FILES["MarkdownFile array<br/>filePath, frontmatter,<br/>content, source"]:::result

    MANAGED --> LOADER
    USER_DIR --> LOADER
    PROJ_DIRS --> LOADER
    LOADER --> DEDUP --> FILES

    classDef ent fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
    classDef user fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef proj fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef loader fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef dedup fill:#3d2b00,stroke:#fd7e14,color:#e0e0e0,stroke-width:2px
    classDef result fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

This same loader powers:
- Slash commands (`.claude/commands/`)
- Agents (`.claude/agents/`)
- Skills (`.claude/skills/`)
- Workflows (`.claude/workflows/`)
- Output styles (`.claude/output-styles/`)

Each entry is parsed with YAML frontmatter for metadata (description, tools, etc.) and the markdown body becomes the instructions.

---

## Key Files

- `src/utils/systemPrompt.ts` — `buildEffectiveSystemPrompt()` logic
- `src/constants/systemPromptSections.ts` — Section caching system
- `src/context.ts` — `getSystemContext()`, `getUserContext()`, git status
- `src/utils/markdownConfigLoader.ts` — CLAUDE.md and `.claude/` directory discovery
- `src/utils/claudemd.ts` — Memory file reading and concatenation
- `src/utils/settings/settings.ts` — Settings merge logic

---

**Previous:** [<- UI Architecture](./09-ui-architecture.md) | **Next:** [MCP Deep Dive ->](./11-mcp-deep-dive.md)
