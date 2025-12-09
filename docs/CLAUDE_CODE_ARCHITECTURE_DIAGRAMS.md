# Claude Code Integration Architecture Diagrams

> **Rendering**: These diagrams use Mermaid syntax. View in GitHub, VS Code with Mermaid extension, or any Mermaid-compatible renderer.

---

## 1. Complete System Architecture

```mermaid
graph TB
    subgraph UserInterface["User Interface Layer"]
        Desktop["Desktop App<br/>(Electron Renderer)"]
        Web["Web Interface<br/>(Fastify + WebSocket)"]
        Mobile["Mobile Interface<br/>(React PWA)"]
    end

    subgraph MainProcess["Main Process Layer"]
        IPC["IPC Handlers<br/>(index.ts)"]
        PM["Process Manager<br/>(process-manager.ts)"]
        AD["Agent Detector<br/>(agent-detector.ts)"]
        WS["Web Server<br/>(web-server.ts)"]
    end

    subgraph External["External Processes"]
        Claude["Claude Code CLI<br/>(claude)"]
        Terminal["Shell<br/>(bash/zsh)"]
        FS["Session Storage<br/>(~/.claude/projects/)"]
    end

    Desktop <-->|"IPC"| IPC
    Web <-->|"WebSocket"| WS
    Mobile <-->|"WebSocket"| WS

    IPC --> PM
    IPC --> AD
    IPC <--> WS

    PM -->|"spawn"| Claude
    PM -->|"spawn"| Terminal
    Claude -->|"stdout (stream-json)"| PM
    Claude -->|"write"| FS

    AD -->|"which claude"| Claude
```

---

## 2. Process Spawn Decision Flow

```mermaid
flowchart TD
    Start([User Sends Message]) --> HasImages{Has Images?}

    HasImages -->|Yes| ImageMode["Add --input-format stream-json"]
    HasImages -->|No| NoImageMode["Prompt as CLI arg"]

    ImageMode --> HasResume{Has Session ID?}
    NoImageMode --> HasResume

    HasResume -->|Yes| AddResume["Add --resume <session-id>"]
    HasResume -->|No| NewSession["New Session"]

    AddResume --> ReadOnly{Read-Only Mode?}
    NewSession --> ReadOnly

    ReadOnly -->|Yes| AddPlan["Add --permission-mode plan"]
    ReadOnly -->|No| SpawnReady["Ready to Spawn"]

    AddPlan --> SpawnReady

    SpawnReady --> Spawn["spawn('claude', args)"]

    Spawn --> ImageInput{Image Mode?}
    ImageInput -->|Yes| WriteStdin["Write JSON to stdin<br/>Close stdin"]
    ImageInput -->|No| CloseStdin["Close stdin immediately"]

    WriteStdin --> ProcessOutput
    CloseStdin --> ProcessOutput["Process stdout/stderr"]
```

---

## 3. Stream-JSON Output Processing Pipeline

```mermaid
flowchart LR
    subgraph Input["Claude stdout"]
        Raw["Raw JSONL bytes"]
    end

    subgraph Buffer["Line Buffer"]
        Acc["Accumulate chunks"]
        Split["Split by newline"]
        Parse["JSON.parse each line"]
    end

    subgraph Router["Message Router"]
        Check{"Message Type?"}
        SysInit["system:init"]
        Result["result"]
        Usage["modelUsage"]
        Other["assistant/other"]
    end

    subgraph Output["Event Emitter"]
        SlashCmd["emit('slash-commands')"]
        Data["emit('data')"]
        SessionId["emit('session-id')"]
        UsageEvt["emit('usage')"]
    end

    Raw --> Acc --> Split --> Parse --> Check

    Check -->|"type=system, subtype=init"| SysInit --> SlashCmd
    Check -->|"type=result"| Result --> Data
    Check -->|"has session_id (first)"| Result --> SessionId
    Check -->|"has modelUsage"| Usage --> UsageEvt
    Check -->|"type=assistant"| Other -->|"Skip"| X((Discard))
```

---

## 4. Multi-Tab Session State

```mermaid
stateDiagram-v2
    state Session {
        [*] --> Tab1: Create first tab

        state Tab1 {
            [*] --> T1_Idle
            T1_Idle --> T1_Busy: Send message
            T1_Busy --> T1_Idle: Response complete
            T1_Busy: claudeSessionId = "abc..."
        }

        state Tab2 {
            [*] --> T2_Idle
            T2_Idle --> T2_Busy: Send message
            T2_Busy --> T2_Idle: Response complete
            T2_Busy: claudeSessionId = "def..."
        }

        state Tab3 {
            [*] --> T3_Idle
            T3_Idle --> T3_AwaitingId: First message
            T3_AwaitingId --> T3_Busy: Received session_id
            T3_Busy --> T3_Idle: Response complete
            T3_AwaitingId: claudeSessionId = null
            T3_Busy: claudeSessionId = "ghi..."
        }
    }

    Session --> ExecutionQueue: Queue messages when busy
    ExecutionQueue --> Session: Process next on idle
```

---

## 5. Session ID Lifecycle

```mermaid
sequenceDiagram
    participant UI as User Interface
    participant App as App.tsx
    participant PM as ProcessManager
    participant CC as Claude Code
    participant Store as Session Store

    Note over UI,Store: New Tab Created (claudeSessionId = null)

    UI->>App: Send first message
    App->>App: Check tab.claudeSessionId
    Note over App: null → don't add --resume

    App->>PM: spawn(args without --resume)
    PM->>CC: claude --print ... -- "prompt"

    CC-->>PM: {"session_id": "new-uuid-123", ...}
    PM-->>App: emit('session-id', 'new-uuid-123')

    App->>Store: registerSessionOrigin(cwd, 'new-uuid-123', 'user')
    App->>App: tab.claudeSessionId = 'new-uuid-123'
    App->>App: tab.awaitingSessionId = false

    Note over UI,Store: Subsequent Messages

    UI->>App: Send another message
    App->>App: Check tab.claudeSessionId
    Note over App: 'new-uuid-123' → add --resume

    App->>PM: spawn(args with --resume new-uuid-123)
    PM->>CC: claude --print --resume new-uuid-123 -- "prompt"
    Note over CC: Continues same conversation
```

---

## 6. Image Attachment Flow

```mermaid
flowchart TD
    subgraph UserAction["User Action"]
        Paste["Paste Image<br/>(Cmd+V)"]
        Drag["Drag & Drop"]
        Attach["Click Attach"]
    end

    subgraph Processing["Image Processing"]
        Validate["Validate Type<br/>(png/jpg/gif/webp)"]
        Convert["Convert to Base64<br/>Data URL"]
        Stage["Add to<br/>tab.stagedImages[]"]
    end

    subgraph Display["UI Display"]
        Preview["Show Thumbnails<br/>in Input Area"]
        Remove["Allow Remove<br/>Before Send"]
    end

    subgraph Send["Send Message"]
        Build["Build stream-json<br/>message"]
        Args["Add --input-format<br/>stream-json to args"]
        Write["Write to stdin"]
        Close["Close stdin"]
    end

    Paste --> Validate
    Drag --> Validate
    Attach --> Validate

    Validate --> Convert --> Stage --> Preview
    Preview --> Remove
    Remove -.->|"User removes"| Stage

    Preview -->|"User sends"| Build
    Build --> Args --> Write --> Close
```

---

## 7. Web/Remote Session Broadcasting

```mermaid
flowchart TB
    subgraph Desktop["Desktop Process"]
        PM["ProcessManager"]
        App["App.tsx"]
    end

    subgraph MainProcess["Main Process"]
        IPC["IPC Handler"]
        WS["WebServer"]
    end

    subgraph Clients["Remote Clients"]
        Web1["Web Client 1"]
        Web2["Web Client 2"]
        Mobile["Mobile Client"]
    end

    PM -->|"process:data"| IPC
    PM -->|"process:exit"| IPC
    PM -->|"process:usage"| IPC

    IPC -->|"Update UI"| App
    IPC -->|"broadcastToSessionClients"| WS

    WS -->|"WebSocket"| Web1
    WS -->|"WebSocket"| Web2
    WS -->|"WebSocket"| Mobile

    Web1 -->|"executeCommand"| WS
    WS -->|"remote:executeCommand"| IPC
    IPC -->|"Trigger spawn"| PM
```

---

## 8. Execution Queue Flow

```mermaid
flowchart TD
    subgraph Input["User Input"]
        Msg1["Message 1"]
        Msg2["Message 2"]
        Msg3["Message 3"]
    end

    subgraph Check["State Check"]
        Busy{Session Busy?}
    end

    subgraph Queue["Execution Queue"]
        Q["Queue Array"]
        Q1["Item 1"]
        Q2["Item 2"]
    end

    subgraph Process["Processing"]
        Spawn["Spawn Claude"]
        Wait["Wait for Exit"]
        Next{Queue Empty?}
    end

    Msg1 --> Busy
    Msg2 --> Busy
    Msg3 --> Busy

    Busy -->|"No"| Spawn
    Busy -->|"Yes"| Q

    Q --> Q1 & Q2

    Spawn --> Wait
    Wait --> Next

    Next -->|"No"| Pop["Pop from Queue"]
    Pop --> Spawn

    Next -->|"Yes"| Idle["Set State: Idle"]
```

---

## 9. Agent Detection Sequence

```mermaid
sequenceDiagram
    participant App as Application
    participant AD as AgentDetector
    participant FS as File System
    participant Shell as Shell (which)

    App->>AD: detectAgents()

    Note over AD: Check cache first
    AD->>AD: cachedAgents exists?

    alt Cache Hit
        AD-->>App: Return cached agents
    else Cache Miss
        AD->>AD: Build expanded PATH

        Note over AD: Add /opt/homebrew/bin,<br/>~/.local/bin, etc.

        loop For each agent definition
            alt Custom path configured
                AD->>FS: stat(customPath)
                FS-->>AD: File info
                AD->>FS: access(customPath, X_OK)
                FS-->>AD: Executable check
            else Use PATH
                AD->>Shell: which claude
                Shell-->>AD: /opt/homebrew/bin/claude
            end
        end

        AD->>AD: Cache results
        AD-->>App: Return agent configs
    end
```

---

## 10. Cost Tracking Data Flow

```mermaid
flowchart LR
    subgraph Claude["Claude Code Output"]
        MU["modelUsage: {<br/>  'claude-sonnet-4': {<br/>    inputTokens: 1500,<br/>    outputTokens: 500,<br/>    cacheReadInputTokens: 1000<br/>  }<br/>}"]
        Cost["total_cost_usd: 0.0234"]
    end

    subgraph Parse["Parser"]
        Agg["Aggregate per-model<br/>token counts"]
    end

    subgraph State["Session State"]
        TabUsage["tab.usageStats = {<br/>  inputTokens: 1500,<br/>  outputTokens: 500,<br/>  totalCostUsd: 0.0234<br/>}"]
    end

    subgraph Display["UI Display"]
        Tokens["Token Counter"]
        CostBadge["Cost Badge"]
        ContextBar["Context Window Bar"]
    end

    MU --> Agg
    Cost --> Agg
    Agg --> TabUsage
    TabUsage --> Tokens & CostBadge & ContextBar
```

---

## 11. Agentic Orchestration Stack

```mermaid
graph TB
    subgraph History["HISTORY LAYER"]
        HE["HistoryEntry"]
        HE --> |"Links to"| CS["Claude Session"]
        HE --> |"Contains"| Syn["Synopsis Summary"]
    end

    subgraph Batch["BATCH LAYER (Auto Run)"]
        Doc["Document Loop"]
        Task["Task Loop"]
        Spawn["Agent Spawn"]
        Doc --> Task --> Spawn
        Spawn --> |"Synopsis"| History
    end

    subgraph Queue["QUEUE LAYER"]
        EQ["Execution Queue"]
        FIFO["FIFO Processing"]
        Tab["Tab Dispatch"]
        EQ --> FIFO --> Tab
    end

    subgraph Session["SESSION LAYER"]
        PM["ProcessManager"]
        Parse["Stream-JSON Parser"]
        Origin["Session Origin Tracker"]
        PM --> Parse --> Origin
    end

    Queue --> |"Spawn"| Batch
    Batch --> |"Enqueue"| Queue
    Session --> |"Events"| Queue
```

---

## 12. Batch Processing State Machine

```mermaid
stateDiagram-v2
    [*] --> Idle: Initialize

    Idle --> Running: Start Batch
    Running --> Stopping: Stop Requested
    Stopping --> Idle: Task Complete

    state Running {
        [*] --> DocumentLoop

        state DocumentLoop {
            [*] --> ReadDoc
            ReadDoc --> TaskLoop: Has tasks
            ReadDoc --> NextDoc: No tasks

            state TaskLoop {
                [*] --> SpawnAgent
                SpawnAgent --> WaitExit
                WaitExit --> Synopsis
                Synopsis --> AddHistory
                AddHistory --> RereadDoc
                RereadDoc --> SpawnAgent: More tasks
                RereadDoc --> [*]: No tasks
            }

            TaskLoop --> ResetDoc: Reset enabled
            ResetDoc --> NextDoc
            TaskLoop --> NextDoc: No reset
        }

        NextDoc --> DocumentLoop: More docs
        NextDoc --> LoopCheck: All docs done

        LoopCheck --> DocumentLoop: Loop enabled
        LoopCheck --> [*]: Loop disabled
    }
```

---

## 13. Synopsis Generation Flow

```mermaid
sequenceDiagram
    participant Main as Main Session
    participant BG as Background Process
    participant Claude as Claude Code
    participant History as History Store

    Note over Main: Task completes
    Main->>Main: Check saveToHistory flag

    alt saveToHistory enabled
        Main->>BG: spawnBackgroundSynopsis(claudeSessionId)

        BG->>Claude: claude --resume <id> -- "synopsis prompt"
        Claude-->>BG: **Summary:** ... **Details:** ...

        BG->>BG: parseSynopsis(response)
        BG-->>Main: { shortSummary, fullSynopsis }

        Main->>History: addHistoryEntry({<br/>  type: 'USER',<br/>  summary,<br/>  claudeSessionId<br/>})
    end
```

---

## 14. Multi-Tab Parallel Execution

```mermaid
gantt
    title Multi-Tab Execution Timeline
    dateFormat X
    axisFormat %s

    section Tab 1 (Write)
    Task A (Write Mode)    :active, t1a, 0, 20
    Idle                   :t1b, 20, 40
    Task D (Write Mode)    :active, t1d, 40, 60

    section Tab 2 (Read)
    Waiting                :t2a, 0, 5
    Task B (Read-Only)     :active, t2b, 5, 15
    Task E (Read-Only)     :active, t2e, 25, 35

    section Tab 3 (Read)
    Waiting                :t3a, 0, 8
    Task C (Read-Only)     :active, t3c, 8, 18
    Task F (Read-Only)     :active, t3f, 30, 45
```

---

## 15. Session Origin Classification

```mermaid
flowchart LR
    subgraph Sources["Session Sources"]
        UI["User Types in Maestro"]
        Batch["Auto Run / Batch"]
        CLI["CLI Playbook"]
    end

    subgraph Registration["Origin Registration"]
        RegUser["registerSessionOrigin(<br/>path, id, 'user')"]
        RegAuto["registerSessionOrigin(<br/>path, id, 'auto')"]
    end

    subgraph Storage["Session Origins Store"]
        Store["{ projectPath: {<br/>  sessionId: {<br/>    origin: 'user'|'auto',<br/>    sessionName?,<br/>    starred?<br/>  }<br/>}}"]
    end

    subgraph Usage["Benefits"]
        Filter["UI Filtering"]
        HistType["History Type"]
        Naming["Session Naming"]
        Star["Starring"]
    end

    UI --> RegUser
    Batch --> RegAuto
    CLI --> RegAuto

    RegUser --> Store
    RegAuto --> Store

    Store --> Filter
    Store --> HistType
    Store --> Naming
    Store --> Star
```

---

## 16. Complete Agentic Flow

```mermaid
flowchart TB
    subgraph User["User Action"]
        Input["Send Message"]
        StartBatch["Start Auto Run"]
    end

    subgraph Queue["Execution Queue"]
        Check{Busy?}
        Add["Add to Queue"]
        Process["Process Item"]
    end

    subgraph Spawn["Process Spawn"]
        BuildArgs["Build CLI Args"]
        AddResume["Add --resume?"]
        AddPlan["Add --permission-mode plan?"]
        SpawnClaude["spawn('claude', args)"]
    end

    subgraph Parse["Output Processing"]
        StreamJSON["Parse JSONL"]
        EmitData["emit('data')"]
        EmitSessionId["emit('session-id')"]
        EmitUsage["emit('usage')"]
    end

    subgraph Complete["Completion"]
        MarkIdle["Mark Tab Idle"]
        CheckQueue{Queue Empty?}
        Synopsis["Generate Synopsis?"]
        AddHistory["Add History Entry"]
        NextItem["Process Next"]
    end

    Input --> Check
    StartBatch --> Check

    Check -->|No| BuildArgs
    Check -->|Yes| Add
    Add -.-> Process

    BuildArgs --> AddResume
    AddResume --> AddPlan
    AddPlan --> SpawnClaude

    SpawnClaude --> StreamJSON
    StreamJSON --> EmitData
    StreamJSON --> EmitSessionId
    StreamJSON --> EmitUsage

    EmitData --> MarkIdle
    MarkIdle --> CheckQueue

    CheckQueue -->|No| NextItem
    NextItem --> Process
    Process --> BuildArgs

    CheckQueue -->|Yes| Synopsis
    Synopsis --> AddHistory
    AddHistory --> Done((Done))
```

---

## Visual Legend

| Symbol | Meaning |
|--------|---------|
| `[Rectangle]` | Process/Component |
| `{Diamond}` | Decision Point |
| `((Circle))` | Terminal/End State |
| `-->` | Data Flow |
| `-.->` | Optional/Conditional Flow |
| `<-->` | Bidirectional Communication |

---

*Diagrams optimized for Mermaid rendering. Use GitHub preview or Mermaid Live Editor.*
