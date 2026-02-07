# OpenCode Documentation

Welcome to the OpenCode architecture and API documentation.

## 📚 Documentation Index

### [ARCHITECTURE.md](./ARCHITECTURE.md)
**Comprehensive architecture overview with ASCII diagrams**

Complete guide to understanding the OpenCode server architecture, including:
- System overview and component diagrams
- Core components (Session Manager, Agentic Loop, Agent System, Tool Registry, etc.)
- Detailed API endpoint reference
- Agentic loop flow and state machine
- Integration workflow diagrams (starting, resuming, canceling sessions)
- Tool approval and human-in-the-loop flows
- Permission system and question handling
- Advanced features (forking, compaction, sub-agents)
- Security and performance considerations

**Best for:** Understanding the system design, learning how components interact, contributing to the codebase

### [API-QUICK-REFERENCE.md](./API-QUICK-REFERENCE.md)
**Quick reference guide for API integration**

Practical guide for developers integrating with OpenCode:
- Quick start examples (curl commands)
- Essential endpoints table
- Event types and SSE integration
- Request/response examples
- Common integration patterns
- Use case examples
- Tips and best practices

**Best for:** Building integrations, quick API lookups, example code snippets

### [EVENT-STREAMING.md](./EVENT-STREAMING.md)
**Deep dive into the `/event` endpoint and SSE streaming**

Focused guide on how event streaming works:
- How the `/event` endpoint broadcasts events
- Instance isolation (per project directory)
- Client-side filtering by sessionID or event type
- Event structure and all event types
- Connection management and reconnection
- JavaScript and Python filtering examples
- Multi-project and multi-session patterns
- Performance and debugging tips

**Best for:** Understanding event flow, implementing real-time UI updates, handling permissions/questions

---

## 🚀 Quick Start

To integrate with OpenCode server:

1. **Start the server** (default: `http://localhost:4096`)
   ```bash
   opencode --server
   ```

2. **Create a session**
   ```bash
   curl -X POST http://localhost:4096/session \
     -H "Content-Type: application/json" \
     -d '{"agent": "build"}'
   ```

3. **Subscribe to events** (in separate terminal)
   ```bash
   curl -N http://localhost:4096/event
   ```

4. **Send a message**
   ```bash
   curl -X POST http://localhost:4096/session/SESSION_ID/message \
     -H "Content-Type: application/json" \
     -d '{"parts": [{"type": "text", "text": "Hello!"}]}'
   ```

---

## 🎯 Key Concepts

### Sessions
- A conversation thread with an AI agent
- Has unique ID, title, and directory context
- Can be forked to create branches
- Persists across server restarts

### Agents
- AI assistants with specific behaviors and permissions
- Built-in: `build`, `plan`, `general`, `explore`
- Customizable via `.opencode/agent/` directory
- Each has its own permission ruleset

### Messages & Parts
- **Message**: Single exchange (user or assistant)
- **Part**: Component of a message (text, tool call, file, etc.)
- Messages stream as they're generated
- Parts can be updated (e.g., tool execution progress)

### Tools
- Actions the agent can execute (bash, edit, read, etc.)
- Wrapped with permission checks
- Can be synchronous or asynchronous
- Extensible via MCP (Model Context Protocol)

### Permissions
- Control what tools can do
- Three modes: `allow`, `deny`, `ask`
- Pattern-based matching (exact, glob, wildcards)
- Interactive approval workflow via API

### Questions
- Agent requests information from user
- Separate from permissions (information vs. action)
- Answered via API endpoint
- Resumes agent execution when answered

---

## 📊 Architecture Overview

```
┌──────────┐     HTTP/SSE      ┌──────────────┐     Agentic      ┌──────────┐
│  Client  │ ◄──────────────► │   Server     │ ◄──────────────► │  Agent   │
│  (TUI)   │    REST API       │   (Hono)     │      Loop        │  System  │
└──────────┘                   └──────┬───────┘                   └──────────┘
                                      │
                                      ▼
                               ┌──────────────┐
                               │   Storage    │
                               │  (SQLite)    │
                               └──────────────┘
```

The server runs an **agentic loop** that:
1. Receives user messages
2. Selects and configures an agent
3. Calls LLM provider with tools
4. Executes tools (with permission checks)
5. Streams results back to client
6. Repeats until task complete

---

## 🔗 Related Resources

- **Main README**: [../README.md](../README.md)
- **Contributing Guide**: [../CONTRIBUTING.md](../CONTRIBUTING.md)
- **OpenAPI Spec**: Available at `GET http://localhost:4096/doc`
- **Website**: https://opencode.ai
- **Discord**: https://opencode.ai/discord

---

## 💡 Common Questions

### How do I handle permissions?
Subscribe to `/event` SSE stream and listen for `permission.created` events. Respond with `POST /permission/:id/reply` with `allow`, `deny`, or `allow_once`.

### Can I use multiple sessions simultaneously?
Yes! Each session runs independently. The server handles concurrent sessions.

### How do I cancel a running session?
Use `POST /session/:sessionID/abort` to stop execution.

### What's the difference between permissions and questions?
- **Permissions**: Agent wants to DO something (needs authorization)
- **Questions**: Agent wants to KNOW something (needs information)

### How do I customize agent behavior?
Create `.opencode/agent/<name>.md` files with custom prompts and permission rules.

### Can I use my own LLM provider?
Yes! OpenCode supports OpenAI, Anthropic, Google, and custom providers. Configure via API or config files.

---

## 📝 Examples

### Complete Integration Example (JavaScript)

```javascript
// 1. Create session
const session = await fetch('http://localhost:4096/session', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ agent: 'build' })
}).then(r => r.json())

// 2. Subscribe to events
const events = new EventSource('http://localhost:4096/event')
events.onmessage = (e) => {
  const event = JSON.parse(e.data)
  
  if (event.type === 'permission.created') {
    // Auto-approve all permissions (be careful!)
    fetch(`http://localhost:4096/permission/${event.properties.requestID}/reply`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ reply: 'allow' })
    })
  }
}

// 3. Send message
await fetch(`http://localhost:4096/session/${session.id}/message`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    parts: [{ type: 'text', text: 'Create a hello world function' }]
  })
})
```

---

## 🤝 Contributing

To add or improve documentation:

1. Fork the repository
2. Make your changes
3. Ensure diagrams are clear and ASCII art is properly formatted
4. Submit a pull request

For questions or suggestions, open an issue or ask in our [Discord](https://opencode.ai/discord).
