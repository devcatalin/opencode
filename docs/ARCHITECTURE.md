# OpenCode Server Architecture

This document provides a high-level overview of the OpenCode server architecture, including component diagrams, API endpoints, and integration workflows.

## Table of Contents

1. [System Overview](#system-overview)
2. [Core Components](#core-components)
3. [API Endpoints](#api-endpoints)
4. [Agentic Loop](#agentic-loop)
5. [Integration Workflows](#integration-workflows)
6. [Tool Approval & Human-in-the-Loop](#tool-approval--human-in-the-loop)

---

## System Overview

OpenCode is a client-server architecture where the server runs an agentic loop that processes user requests through AI agents, executes tools, and manages permissions.

```
┌─────────────────────────────────────────────────────────────────┐
│                         OpenCode System                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐         ┌──────────────┐         ┌──────────┐    │
│  │  Client  │◄────────┤ HTTP/WebSocket├────────►│  Server  │    │
│  │  (TUI)   │         │  REST API     │         │  (Hono)  │    │
│  └──────────┘         └──────────────┘         └────┬─────┘    │
│                                                       │          │
│  ┌──────────┐                                       │          │
│  │ Desktop  │                                       │          │
│  │   App    │───────────────────────────────────────┘          │
│  └──────────┘                                                   │
│                                                                   │
│  ┌──────────┐                                                   │
│  │   Web    │                                                   │
│  │  Client  │───────────────────────────────────────────────────┘
│  └──────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### Component Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        OpenCode Server                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐         │
│  │   HTTP Server  │  │  WebSocket/SSE │  │   Bus (Events) │         │
│  │     (Hono)     │  │    Streaming   │  │   Pub/Sub      │         │
│  └────────┬───────┘  └────────┬───────┘  └────────┬───────┘         │
│           │                   │                    │                  │
│           └───────────────────┴────────────────────┘                  │
│                               │                                       │
│           ┌───────────────────┴──────────────────┐                   │
│           │                                       │                   │
│  ┌────────▼─────────┐               ┌────────────▼──────────┐        │
│  │  Session Manager │               │   Instance Manager    │        │
│  │  - Create        │               │   - Per Directory     │        │
│  │  - Get           │               │   - State Isolation   │        │
│  │  - Update        │               │   - VCS Integration   │        │
│  │  - Delete        │               └───────────────────────┘        │
│  │  - Fork          │                                                 │
│  │  - Abort         │                                                 │
│  └────────┬─────────┘                                                 │
│           │                                                            │
│  ┌────────▼────────────────────────────────────────────┐             │
│  │              Agentic Loop Engine                     │             │
│  │  ┌──────────────────────────────────────────────┐   │             │
│  │  │ SessionPrompt.loop()                         │   │             │
│  │  │  - Message Processing                        │   │             │
│  │  │  - Agent Selection                           │   │             │
│  │  │  - Tool Execution                            │   │             │
│  │  │  - Permission Checking                       │   │             │
│  │  │  - Context Management                        │   │             │
│  │  └──────────────────────────────────────────────┘   │             │
│  └──────────┬──────────────┬────────────┬──────────────┘             │
│             │              │            │                             │
│    ┌────────▼────┐  ┌─────▼─────┐  ┌──▼────────────┐                │
│    │   Agent     │  │   Tool    │  │  Permission   │                │
│    │   System    │  │  Registry │  │    System     │                │
│    │             │  │           │  │               │                │
│    │ - build     │  │ - bash    │  │ - Rules       │                │
│    │ - plan      │  │ - edit    │  │ - Ask/Allow   │                │
│    │ - general   │  │ - read    │  │ - Deny        │                │
│    │ - explore   │  │ - task    │  │               │                │
│    │ - custom    │  │ - grep    │  │               │                │
│    └─────────────┘  │ - glob    │  └───────────────┘                │
│                     │ - MCP     │                                    │
│                     │ - LSP     │                                    │
│                     └───────────┘                                    │
│                                                                        │
│  ┌───────────────┐  ┌──────────────┐  ┌─────────────┐               │
│  │   Storage     │  │   Provider   │  │  Question   │               │
│  │   (SQLite)    │  │   (LLM)      │  │   System    │               │
│  │               │  │              │  │             │               │
│  │ - Sessions    │  │ - OpenAI     │  │ - Ask user  │               │
│  │ - Messages    │  │ - Anthropic  │  │ - Collect   │               │
│  │ - Parts       │  │ - Google     │  │   answers   │               │
│  │ - Config      │  │ - Custom     │  └─────────────┘               │
│  └───────────────┘  └──────────────┘                                 │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

#### Session Manager
- Creates and manages agent sessions
- Stores session metadata (title, timestamps, directory)
- Handles session forking, archiving, and deletion
- Tracks parent-child relationships between sessions

#### Agentic Loop Engine
- Processes user messages in a loop until completion
- Manages agent execution flow
- Coordinates tool calls with permission checks
- Handles context overflow and compaction
- Implements step limits and abort logic

#### Agent System
- Defines agent behaviors and permissions
- Built-in agents: `build`, `plan`, `general`, `explore`
- Custom agents via AGENTS.md configuration
- Each agent has its own permission ruleset

#### Tool Registry
- Registers and manages available tools
- Wraps tools with permission checks
- Supports MCP (Model Context Protocol) servers
- LSP integration for code intelligence

#### Permission System
- Rule-based permission evaluation
- Interactive approval/denial flow
- Supports patterns: `allow`, `deny`, `ask`
- Per-tool and per-path permissions

#### Storage Layer
- SQLite database for persistence
- Stores sessions, messages, and parts
- Supports message compaction
- Project-scoped instances

---

## API Endpoints

### Session Management

#### Create Session
```
POST /session
Content-Type: application/json

{
  "agent": "build",
  "directory": "/path/to/project"
}

Response: Session object with ID
```

#### Get Session
```
GET /session/:sessionID

Response: Session metadata and state
```

#### List Sessions
```
GET /session?directory=/path&roots=true&limit=10

Query Parameters:
  - directory: Filter by project directory
  - roots: Only root sessions (no parents)
  - start: Filter by timestamp
  - search: Search by title
  - limit: Max results

Response: Array of session objects
```

#### Send Message (Start/Resume)
```
POST /session/:sessionID/message
Content-Type: application/json

{
  "parts": [
    { "type": "text", "text": "Fix the bug in auth.ts" }
  ],
  "agent": "build",
  "model": {
    "providerID": "anthropic",
    "modelID": "claude-3-5-sonnet-20241022"
  }
}

Response: Streams assistant message parts
```

#### Abort Session
```
POST /session/:sessionID/abort

Response: { success: true }
```

#### Delete Session
```
DELETE /session/:sessionID

Response: { success: true }
```

#### Fork Session
```
POST /session/:sessionID/fork
Content-Type: application/json

{
  "messageID": "msg_123",
  "agent": "build"
}

Response: New session object
```

### Event Streaming

#### Subscribe to Events
```
GET /event
Accept: text/event-stream

Response: Server-Sent Events stream
  - server.connected
  - server.heartbeat
  - session.*
  - message.*
  - part.*
  - permission.*
  - question.*
```

### Permission Management

#### List Pending Permissions
```
GET /permission

Response: Array of pending permission requests
```

#### Reply to Permission
```
POST /permission/:requestID/reply
Content-Type: application/json

{
  "reply": "allow" | "deny" | "allow_once",
  "message": "Optional explanation"
}

Response: { success: true }
```

### Question Handling

#### List Pending Questions
```
GET /question

Response: Array of pending questions
```

#### Answer Question
```
POST /question/:requestID/reply
Content-Type: application/json

{
  "answers": {
    "question1": "answer1",
    "question2": "answer2"
  }
}

Response: { success: true }
```

#### Reject Question
```
POST /question/:requestID/reject

Response: { success: true }
```

### Other Endpoints

- `GET /path` - Get current paths (home, state, config, worktree)
- `GET /agent` - List available agents
- `GET /skill` - List available skills
- `GET /command` - List available commands
- `GET /vcs` - Get VCS info (current branch, etc.)
- `GET /lsp` - Get LSP server status
- `GET /formatter` - Get formatter status
- `GET /doc` - OpenAPI documentation

---

## Agentic Loop

### Loop Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Agentic Loop (SessionPrompt.loop)               │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │  Load Session State  │
                       │  - Get messages      │
                       │  - Find last user    │
                       │  - Find last assist  │
                       └──────────┬───────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │  Check Exit Cond.    │
                  ┌────┤  - Abort signal?     │
                  │    │  - Finished?         │
                  │    │  - No user message?  │
                  │    └──────────┬───────────┘
                  │               │
              Exit│               │Continue
                  │               ▼
                  │    ┌──────────────────────┐
                  │    │  Check for Tasks     │
                  │    │  - Subtask pending?  │
                  │    │  - Compaction needed?│
                  │    └──────────┬───────────┘
                  │               │
                  │         ┌─────┴─────┐
                  │         │           │
                  │    Subtask      Compaction
                  │         │           │
                  │         ▼           ▼
                  │    ┌────────┐  ┌────────┐
                  │    │Execute │  │Compact │
                  │    │Subtask │  │Context │
                  │    └───┬────┘  └────┬───┘
                  │        │            │
                  │        └─────┬──────┘
                  │              │
                  │              ▼
                  │       ┌──────────────────────┐
                  │       │  Normal Processing   │
                  │       │  - Get agent config  │
                  │       │  - Check step limit  │
                  │       └──────────┬───────────┘
                  │                  │
                  │                  ▼
                  │       ┌──────────────────────┐
                  │       │  Call LLM Provider   │
                  │       │  - Build messages    │
                  │       │  - Add tools         │
                  │       │  - Stream response   │
                  │       └──────────┬───────────┘
                  │                  │
                  │            ┌─────┴─────┐
                  │            │           │
                  │       Text Response  Tool Calls
                  │            │           │
                  │            ▼           ▼
                  │       ┌────────┐  ┌────────────┐
                  │       │ Store  │  │  Execute   │
                  │       │Message │  │   Tools    │
                  │       └───┬────┘  └─────┬──────┘
                  │           │             │
                  │           │      ┌──────▼──────────┐
                  │           │      │Check Permission │
                  │           │      │- Allow: Run     │
                  │           │      │- Deny: Error    │
                  │           │      │- Ask: Wait      │
                  │           │      └──────┬──────────┘
                  │           │             │
                  │           │             ▼
                  │           │      ┌──────────────┐
                  │           │      │  Tool Result │
                  │           │      └──────┬───────┘
                  │           │             │
                  │           └─────────────┘
                  │                  │
                  │                  ▼
                  │         ┌────────────────┐
                  │         │ Increment Step │
                  │         └────────┬───────┘
                  │                  │
                  └──────────────────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │Session Complete│
                          │- Notify clients│
                          │- Set to idle   │
                          └───────────────┘
```

### Loop State Machine

```
  START
    │
    ▼
┌─────────┐
│  IDLE   │◄──────────────────────────┐
└────┬────┘                           │
     │ User sends message             │
     ▼                                │
┌─────────┐                           │
│  BUSY   │                           │
│ (Loop)  │                           │
└────┬────┘                           │
     │                                │
     ├──► Process messages            │
     │                                │
     ├──► Execute tools ──► WAITING ─►│
     │    (async)          (approval) │
     │                                │
     ├──► Generate response           │
     │                                │
     └──► Done or max steps ──────────┘
```

---

## Integration Workflows

### Starting a New Session

```
Client                    Server                    Agent Loop
  │                         │                           │
  │  POST /session          │                           │
  ├────────────────────────►│                           │
  │                         │  Create session           │
  │  ◄─────────────────────┤  Return sessionID         │
  │                         │                           │
  │  GET /event (SSE)       │                           │
  ├────────────────────────►│                           │
  │  ◄─────────────────────┤  server.connected         │
  │                         │                           │
  │  POST /session/:id/message                          │
  ├────────────────────────►│                           │
  │                         │  Start loop               │
  │                         ├──────────────────────────►│
  │                         │                           │
  │                         │                     Process message
  │                         │                     Select agent
  │                         │                     Call LLM
  │                         │                           │
  │  ◄─────────────────────┼───────────────────────────┤
  │  message.created (SSE)  │                           │
  │                         │                           │
  │  ◄─────────────────────┼───────────────────────────┤
  │  part.created (SSE)     │                           │
  │  (streaming)            │                           │
  │                         │                           │
  │  ◄─────────────────────┼───────────────────────────┤
  │  message.completed      │                      Loop done
  │                         │                           │
```

### Resuming with Tool Calls

```
Client                    Server                Agent Loop             Tool
  │                         │                       │                  │
  │                         │   (Loop running)      │                  │
  │                         │                       │  Execute tool    │
  │                         │                       ├─────────────────►│
  │                         │                       │                  │
  │  ◄─────────────────────┼───────────────────────┤                  │
  │  part.created           │                       │                  │
  │  type: tool, state: pending                     │                  │
  │                         │                       │                  │
  │                         │                       │  Result          │
  │                         │                       │◄─────────────────┤
  │  ◄─────────────────────┼───────────────────────┤                  │
  │  part.updated           │                       │                  │
  │  state: completed       │                       │                  │
  │                         │                       │                  │
  │                         │                  Call LLM again          │
  │                         │                  (with tool results)     │
  │                         │                       │                  │
  │  ◄─────────────────────┼───────────────────────┤                  │
  │  message.created        │                  Next response           │
  │                         │                       │                  │
```

### Cancelling a Session

```
Client                    Server                    Agent Loop
  │                         │                           │
  │  POST /session/:id/abort│                          │
  ├────────────────────────►│                           │
  │                         │  Set abort signal         │
  │                         ├──────────────────────────►│
  │                         │                      Abort check
  │  ◄─────────────────────┤                      Exit loop
  │  { success: true }      │                           │
  │                         │                           │
  │  ◄─────────────────────┼───────────────────────────┤
  │  session.aborted (SSE)  │                           │
  │                         │                           │
```

---

## Tool Approval & Human-in-the-Loop

### Permission Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Permission Request Flow                           │
└──────────────────────────────────────────────────────────────────────┘

Agent Loop              Permission System          Client
    │                          │                       │
    │  Execute tool            │                       │
    │  (e.g., bash "rm -rf")   │                       │
    ├─────────────────────────►│                       │
    │                          │                       │
    │                    Check ruleset                 │
    │                    - Match patterns              │
    │                    - Evaluate rules              │
    │                          │                       │
    │                    ┌─────▼─────┐                │
    │                    │Rule Result │                │
    │                    └─────┬─────┘                │
    │                          │                       │
    │            ┌─────────────┼─────────────┐         │
    │            │             │             │         │
    │         "allow"       "deny"        "ask"        │
    │            │             │             │         │
    │            ▼             ▼             │         │
    │      ┌─────────┐   ┌─────────┐        │         │
    │      │ Execute │   │  Throw  │        │         │
    │      │  Tool   │   │  Error  │        │         │
    │      └─────────┘   └─────────┘        │         │
    │                                        ▼         │
    │                                 ┌──────────────┐ │
    │                                 │Create Request│ │
    │                                 │- requestID   │ │
    │                                 │- tool name   │ │
    │                                 │- args        │ │
    │                                 └──────┬───────┘ │
    │                                        │         │
    │                                        │Emit event
    │                                        ├─────────────►
    │                                        │         │
    │  Wait for response                    │    permission.created
    │  (async suspension)                   │         │
    │                                        │         │
    │                                        │    User decides
    │                                        │         │
    │                                        │    POST /permission/:id/reply
    │                                        │    { reply: "allow" }
    │                                        │         │
    │                                        │◄────────┤
    │                                 Resume │         │
    │◄───────────────────────────────────────┤         │
    │                                        │         │
    │  Execute tool with approval            │         │
    │                                        │         │
```

### Permission Rules

Permissions are evaluated in order of specificity:
1. Exact path matches (highest priority)
2. Glob pattern matches
3. Tool-level wildcards
4. Default deny (lowest priority)

Example ruleset:
```javascript
{
  "*": "allow",                    // Allow all by default
  "bash": "ask",                   // Ask for bash commands
  "edit": {
    "src/**": "allow",             // Allow edits in src/
    "*.config.js": "deny"          // Deny config file edits
  },
  "external_directory": {
    "/tmp/**": "allow",            // Allow /tmp access
    "*": "ask"                     // Ask for other directories
  }
}
```

### Question Flow

Questions are used when the agent needs information from the user.

```
Agent Loop              Question System           Client
    │                          │                     │
    │  Need user input         │                     │
    │  question() tool call    │                     │
    ├─────────────────────────►│                     │
    │                          │                     │
    │                    Create question             │
    │                    - requestID                 │
    │                    - questions[]               │
    │                          ├─────────────────────►
    │                          │          question.created
    │  Wait for response       │                     │
    │  (async suspension)      │                     │
    │                          │        User answers │
    │                          │                     │
    │                          │    POST /question/:id/reply
    │                          │    { answers: {...} }
    │                          │◄────────────────────┤
    │                          │                     │
    │◄─────────────────────────┤                     │
    │  Resume with answers     │                     │
    │                          │                     │
```

### Permission vs Question

- **Permission**: Used for tool execution authorization
  - Agent wants to DO something (edit, run bash, etc.)
  - Client can approve, deny, or allow once
  - Can be configured with rulesets

- **Question**: Used for information gathering
  - Agent wants to KNOW something from user
  - Client provides answers to questions
  - Always requires explicit user response

---

## Advanced Features

### Session Forking

Create a branch from any point in a conversation:
```
Main Session: msg1 → msg2 → msg3 → msg4
                           ↓
              Fork Session: msg3 → msg5 → msg6
```

### Context Compaction

When context grows too large:
1. Detect overflow based on token count
2. Create compaction task
3. Summarize older messages using LLM
4. Replace original messages with summary
5. Continue with reduced context

### Sub-agents (Task Tool)

Delegate complex tasks to specialized sub-agents:
```
Main Agent: "Refactor the authentication module"
    ↓
    Creates subtask
    ↓
Task Agent: (executes in isolated context)
    - Analyzes code
    - Makes changes
    - Returns summary
    ↓
Main Agent: Receives result, continues
```

### Multi-model Support

OpenCode is model-agnostic:
- Anthropic (Claude)
- OpenAI (GPT)
- Google (Gemini)
- Custom providers
- Local models

Each session can use a different model, and models can be switched mid-conversation.

---

## Security & Sandboxing

### Permission Boundaries

1. **File System**: Restricted by paths and patterns
2. **Shell Commands**: Require approval for dangerous operations
3. **External Directories**: Isolated per-project by default
4. **Network**: No direct network access from tools

### Instance Isolation

Each project directory gets its own isolated instance:
- Separate SQLite database
- Independent configuration
- Isolated agent state
- No cross-project data leakage

---

## Performance Considerations

### Caching

- Message compaction reduces token usage
- LSP provides cached code intelligence
- Provider-level prompt caching where available

### Concurrency

- Multiple sessions can run in parallel
- Tool execution is asynchronous
- Event streaming is non-blocking
- Each session has independent abort control

### Storage

- SQLite for efficient local storage
- Message parts stored separately
- Supports pagination for large histories
- Automatic cleanup of old sessions

---

## Development & Extension

### Adding Custom Agents

Create `.opencode/agent/<name>.md` with:
```markdown
---
mode: primary
permission:
  bash: ask
  edit: allow
---

# Custom Agent Prompt

Your custom instructions here...
```

### Adding Custom Tools

Implement the Tool interface:
```typescript
interface Tool {
  name: string
  description: string
  parameters: z.ZodObject
  execute(args: unknown, context: ToolContext): Promise<unknown>
}
```

### MCP Integration

OpenCode supports Model Context Protocol (MCP) servers for extending tool capabilities.

---

## Glossary

- **Agent**: An AI assistant with specific permissions and behaviors
- **Session**: A conversation thread with history and state
- **Message**: A single exchange (user or assistant)
- **Part**: A component of a message (text, tool call, file, etc.)
- **Tool**: An executable action (bash, edit, read, etc.)
- **Permission**: Authorization rule for tool execution
- **Instance**: Isolated server state per project directory
- **Loop**: The agentic execution cycle processing messages
- **Compaction**: Summarization of conversation history to reduce tokens
- **Fork**: Creating a branch session from an existing one
