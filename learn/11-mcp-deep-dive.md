# 11. MCP Deep Dive

> The Model Context Protocol subsystem -- transports, tool wrapping, authentication, and server lifecycle.

---

## What is MCP?

MCP (Model Context Protocol) is an open protocol for connecting AI models to external tools and data sources. Claude Code is both an **MCP client** (connecting to external MCP servers) and can act as an **MCP server** (exposing its own tools to other systems).

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#4a9eff', 'primaryBorderColor': '#4a9eff'}}}%%
graph TB
    subgraph ClaudeCode["Claude Code"]
        CLIENT["MCP Client<br/>Connects to external servers"]:::client
        SERVER["MCP Server Mode<br/>Exposes tools to other systems"]:::server
        TOOLS["42 Built-in Tools<br/>+ MCP tools merged in"]:::tools
    end

    subgraph External["External MCP Servers"]
        STDIO_S["stdio servers<br/>Local processes"]:::external
        SSE_S["SSE/HTTP servers<br/>Remote endpoints"]:::external
        WS_S["WebSocket servers<br/>Persistent connections"]:::external
        CLAUDE_AI["claude.ai proxy<br/>Managed connectors"]:::external
        IDE_S["IDE servers<br/>Editor integration"]:::external
    end

    CLIENT --> STDIO_S
    CLIENT --> SSE_S
    CLIENT --> WS_S
    CLIENT --> CLAUDE_AI
    CLIENT --> IDE_S

    STDIO_S --> TOOLS
    SSE_S --> TOOLS
    WS_S --> TOOLS
    CLAUDE_AI --> TOOLS
    IDE_S --> TOOLS

    classDef client fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef server fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef tools fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef external fill:#333,stroke:#888,color:#aaa,stroke-width:1px,stroke-dasharray: 5 5
```

---

## Transport Types

Claude Code supports **6 transport types** for connecting to MCP servers:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#17a2b8', 'primaryBorderColor': '#17a2b8'}}}%%
flowchart TD
    subgraph Local["Local Transports"]
        STDIO["stdio<br/>Spawn local process<br/>Communicate via stdin/stdout<br/>Most common type"]:::local
        SDK_T["sdk<br/>In-process transport<br/>SDK-managed placeholder<br/>Tool calls route to SDK"]:::local
    end

    subgraph Remote["Remote Transports"]
        SSE["sse<br/>Server-Sent Events<br/>Legacy remote protocol<br/>OAuth authentication"]:::remote
        HTTP["http<br/>Streamable HTTP<br/>Modern remote protocol<br/>Per-request timeouts"]:::remote
        WS["ws<br/>WebSocket<br/>Persistent bidirectional<br/>TLS/proxy support"]:::remote
        CLAUDEAI["claudeai-proxy<br/>Via claude.ai OAuth<br/>Managed connectors<br/>Auto token refresh"]:::remote
    end

    subgraph IDE["IDE Transports"]
        SSE_IDE["sse-ide<br/>IDE SSE connection<br/>No authentication"]:::ide
        WS_IDE["ws-ide<br/>IDE WebSocket<br/>Auth token in header"]:::ide
    end

    classDef local fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef remote fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef ide fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
```

---

## Connection Lifecycle

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#28a745', 'actorTextColor': '#e0e0e0', 'actorBorder': '#28a745', 'signalColor': '#28a745', 'noteBkgColor': '#16213e', 'noteTextColor': '#e0e0e0', 'activationBkgColor': '#1b3a1b', 'activationBorderColor': '#28a745'}}}%%
sequenceDiagram
    participant H as useManageMCPConnections
    participant C as connectToServer (memoized)
    participant T as Transport Layer
    participant S as External Server

    H->>H: getAllMcpConfigs - merge all sources
    H->>H: Batch servers (local=3, remote=20)

    loop For each server
        H->>C: connectToServer(name, config)
        activate C

        alt Auth cached as needed
            C-->>H: return needs-auth
        else Server disabled
            C-->>H: return disabled
        else Policy blocked
            C-->>H: return not allowed
        end

        C->>T: Create transport (stdio/sse/http/ws)
        T->>S: Connect
        activate S

        alt auth required (401/403)
            S-->>C: Auth error
            C-->>H: return needs-auth
        else connection timeout
            S-->>C: Timeout (30s default)
            C-->>H: return error
        else success
            S-->>T: Connected
            deactivate S
            C->>S: tools/list
            S-->>C: Available tools
            C->>C: Wrap as MCPTool objects
            C-->>H: return connected + tools
        end
        deactivate C
    end

    H->>H: Update AppState.mcp
```

### Batched Connections

Servers are connected in parallel batches to avoid overwhelming the system:
- **Local (stdio) servers**: batch size of 3 (spawning processes is heavy)
- **Remote servers**: batch size of 20 (network connections are lighter)

The batch sizes are configurable via `MCP_SERVER_CONNECTION_BATCH_SIZE` and `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` environment variables.

---

## Tool Wrapping

Each MCP tool is wrapped in a `MCPTool` object that implements the standard `Tool` interface:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#ffc107', 'primaryBorderColor': '#ffc107'}}}%%
flowchart LR
    MCP_TOOL["MCP Server Tool<br/>name, inputSchema,<br/>description"]:::raw

    WRAP["MCPTool wrapper<br/>Implements Tool interface<br/>name: mcp__server__tool"]:::wrap

    subgraph Lifecycle["Tool Call Lifecycle"]
        VALIDATE["Validate input<br/>via JSON Schema"]:::step
        PERM["Check permissions<br/>Same system as built-in"]:::step
        CALL["Forward to MCP server<br/>tools/call RPC"]:::step
        RESULT["Process result<br/>Truncate if needed<br/>Handle images"]:::step
    end

    MCP_TOOL --> WRAP --> VALIDATE --> PERM --> CALL --> RESULT

    classDef raw fill:#333,stroke:#888,color:#e0e0e0,stroke-width:1px
    classDef wrap fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef step fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
```

### Naming Convention

MCP tools are named `mcp__<server>__<tool>`:
- `mcp__slack__send_message`
- `mcp__github__create_issue`
- `mcp__ide__getDiagnostics`

The double-underscore separators prevent ambiguity. In SDK no-prefix mode, tools keep their original names.

### Description Capping

MCP tool descriptions are capped at **2,048 characters**. OpenAPI-generated servers have been observed dumping 15-60KB of endpoint docs into descriptions, which wastes context tokens.

---

## Configuration Sources

MCP server definitions come from multiple sources, merged with priority rules:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#dc3545', 'primaryBorderColor': '#dc3545'}}}%%
flowchart TD
    subgraph Sources["MCP Config Sources"]
        ENT["Enterprise managed-mcp.json<br/>Exclusive control if present"]:::ent
        GLOBAL["Global ~/.claude/settings.json<br/>mcpServers field"]:::user
        LOCAL[".claude/settings.local.json<br/>Local project overrides"]:::local
        PROJECT[".mcp.json at project root<br/>Committed to repo"]:::proj
        PLUGIN["Plugin MCP servers<br/>From installed plugins"]:::plugin
        CLAUDEAI["claude.ai connectors<br/>Fetched at startup"]:::claudeai
        CLI["--mcp-config flag<br/>Runtime override"]:::cli
        SDK_CFG["SDK mcp_set_servers<br/>Programmatic control"]:::sdk
    end

    MERGE["getAllMcpConfigs<br/>Merge + dedup + policy filter"]:::merge

    POLICY{"Enterprise policy<br/>allowedMcpServers?<br/>deniedMcpServers?"}:::check

    ALLOWED["Allowed servers<br/>proceed to connect"]:::allow
    BLOCKED["Blocked servers<br/>silently dropped"]:::deny

    Sources --> MERGE --> POLICY
    POLICY -->|"allowed"| ALLOWED
    POLICY -->|"denied"| BLOCKED

    classDef ent fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
    classDef user fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef local fill:#3d2b00,stroke:#fd7e14,color:#e0e0e0,stroke-width:2px
    classDef proj fill:#0d4f4f,stroke:#17a2b8,color:#e0e0e0,stroke-width:2px
    classDef plugin fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef claudeai fill:#1a1a4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef cli fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef sdk fill:#333,stroke:#888,color:#e0e0e0,stroke-width:1px
    classDef merge fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef check fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef allow fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef deny fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
```

### Deduplication

Plugin and claude.ai connector servers are deduplicated against manually-configured servers:
- **Signature-based**: Same `command+args` (stdio) or same `url` (remote) = same server
- **Manual wins**: If a user manually configured a server, the plugin/connector duplicate is suppressed
- **First-loaded wins**: Between plugins, the first one loaded takes priority

---

## Authentication

Remote MCP servers may require OAuth authentication:

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a2e', 'primaryTextColor': '#e0e0e0', 'lineColor': '#dc3545', 'primaryBorderColor': '#dc3545'}}}%%
flowchart TD
    CONNECT["Connect to remote server"]:::start

    AUTH{"Auth required?"}:::check

    PROVIDER["ClaudeAuthProvider<br/>OAuth flow per server"]:::auth
    TOKENS["Retrieve/refresh tokens"]:::auth

    OK["Connection established"]:::success
    NEEDS["Status: needs-auth<br/>Cached for 15 min"]:::needsauth

    CALL["MCP tool call"]:::start
    CALL_AUTH{"401 on call?"}:::check

    RETRY["Refresh token<br/>retry once"]:::auth
    MCE["McpAuthError<br/>Update status to needs-auth"]:::error

    CONNECT --> AUTH
    AUTH -->|"no"| OK
    AUTH -->|"yes"| PROVIDER --> TOKENS
    TOKENS -->|"success"| OK
    TOKENS -->|"failure"| NEEDS

    CALL --> CALL_AUTH
    CALL_AUTH -->|"no"| OK
    CALL_AUTH -->|"yes"| RETRY
    RETRY -->|"success"| OK
    RETRY -->|"failure"| MCE

    classDef start fill:#1a2d4a,stroke:#4a9eff,color:#e0e0e0,stroke-width:2px
    classDef check fill:#2d2d0d,stroke:#ffc107,color:#e0e0e0,stroke-width:2px
    classDef auth fill:#2d1b4e,stroke:#6f42c1,color:#e0e0e0,stroke-width:2px
    classDef success fill:#1b3a1b,stroke:#28a745,color:#e0e0e0,stroke-width:2px
    classDef needsauth fill:#3d2b00,stroke:#fd7e14,color:#e0e0e0,stroke-width:2px
    classDef error fill:#4a1a1a,stroke:#dc3545,color:#e0e0e0,stroke-width:2px
```

### Session Expiry Handling

MCP servers return HTTP 404 + JSON-RPC error code `-32001` when a session expires. Claude Code detects this, clears the connection cache, and reconnects automatically (`McpSessionExpiredError`).

---

## Key Files

- `src/services/mcp/client.ts` (3,349 lines) -- Connection manager, tool wrapping, transport creation
- `src/services/mcp/config.ts` (1,579 lines) -- Config merging, policy filtering, deduplication
- `src/services/mcp/types.ts` -- Type definitions for server configs and connections
- `src/services/mcp/auth.ts` -- OAuth provider and step-up authentication
- `src/services/mcp/useManageMCPConnections.ts` -- React hook managing connection lifecycle
- `src/tools/MCPTool/MCPTool.ts` -- MCPTool wrapper implementing the Tool interface

---

**Previous:** [<- Configuration](./10-configuration-and-system-prompt.md) | **Next:** [Data Flow Walkthrough ->](./12-data-flow-walkthrough.md)
