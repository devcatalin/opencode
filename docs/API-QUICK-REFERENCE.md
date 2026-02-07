# OpenCode API Quick Reference

Quick reference for integrating with the OpenCode server API.

## Base URL

By default, the server runs on `http://localhost:4096`

## Authentication

If server password is set via `OPENCODE_SERVER_PASSWORD`:
- Use HTTP Basic Auth
- Username: `opencode` (or value of `OPENCODE_SERVER_USERNAME`)
- Password: Value of `OPENCODE_SERVER_PASSWORD`

## Common Headers

```
Content-Type: application/json
x-opencode-directory: /path/to/project  (optional, sets working directory)
```

---

## Quick Start: Create and Use a Session

### 1. Create a Session

```bash
curl -X POST http://localhost:4096/session \
  -H "Content-Type: application/json" \
  -d '{
    "agent": "build",
    "directory": "/path/to/project"
  }'
```

Response:
```json
{
  "id": "session_01234567890",
  "title": "New Session",
  "agent": "build",
  "directory": "/path/to/project",
  "time": {
    "created": 1234567890,
    "updated": 1234567890
  }
}
```

### 2. Subscribe to Events (in separate connection)

```bash
curl -N http://localhost:4096/event
```

This returns Server-Sent Events stream:
```
data: {"type":"server.connected","properties":{}}

data: {"type":"message.created","properties":{...}}

data: {"type":"part.created","properties":{...}}
```

### 3. Send a Message

```bash
curl -X POST http://localhost:4096/session/session_01234567890/message \
  -H "Content-Type: application/json" \
  -d '{
    "parts": [
      {
        "type": "text",
        "text": "Create a hello world function"
      }
    ]
  }'
```

The response streams back the assistant message as JSON.

### 4. Abort if Needed

```bash
curl -X POST http://localhost:4096/session/session_01234567890/abort
```

---

## Essential Endpoints

### Sessions

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/session` | Create new session |
| `GET` | `/session/:sessionID` | Get session details |
| `GET` | `/session` | List sessions (with filters) |
| `PATCH` | `/session/:sessionID` | Update session (title, archived) |
| `DELETE` | `/session/:sessionID` | Delete session |
| `POST` | `/session/:sessionID/message` | Send message to session |
| `POST` | `/session/:sessionID/abort` | Abort running session |
| `POST` | `/session/:sessionID/fork` | Fork session at message point |
| `GET` | `/session/:sessionID/message` | Get all messages |

### Events

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/event` | Subscribe to SSE event stream |

### Permissions

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/permission` | List pending permissions |
| `POST` | `/permission/:requestID/reply` | Approve/deny permission |

### Questions

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/question` | List pending questions |
| `POST` | `/question/:requestID/reply` | Answer question |
| `POST` | `/question/:requestID/reject` | Reject question |

### System Info

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/agent` | List available agents |
| `GET` | `/path` | Get path information |
| `GET` | `/vcs` | Get VCS info (branch, etc.) |
| `GET` | `/lsp` | Get LSP status |
| `GET` | `/doc` | OpenAPI documentation |

---

## Event Types

Events are sent via Server-Sent Events (SSE) on `/event`:

### Server Events
- `server.connected` - Initial connection established
- `server.heartbeat` - Periodic heartbeat (every 30s)

### Session Events
- `session.created` - New session created
- `session.updated` - Session metadata updated
- `session.deleted` - Session deleted
- `session.aborted` - Session aborted

### Message Events
- `message.created` - New message added
- `message.updated` - Message metadata updated

### Part Events
- `part.created` - New message part created (text, tool call, etc.)
- `part.updated` - Part updated (e.g., tool completed)

### Permission Events
- `permission.created` - New permission request
- `permission.resolved` - Permission resolved (approved/denied)

### Question Events
- `question.created` - New question asked
- `question.answered` - Question answered

---

## Request Examples

### List Sessions with Filters

```bash
# Get last 10 root sessions for a directory
curl "http://localhost:4096/session?directory=/my/project&roots=true&limit=10"
```

### Send Message with File Attachment

```bash
curl -X POST http://localhost:4096/session/session_123/message \
  -H "Content-Type: application/json" \
  -d '{
    "parts": [
      {
        "type": "text",
        "text": "Review this file"
      },
      {
        "type": "file",
        "path": "/path/to/file.ts"
      }
    ],
    "agent": "build",
    "model": {
      "providerID": "anthropic",
      "modelID": "claude-3-5-sonnet-20241022"
    }
  }'
```

### Approve a Permission Request

```bash
curl -X POST http://localhost:4096/permission/req_123/reply \
  -H "Content-Type: application/json" \
  -d '{
    "reply": "allow",
    "message": "Approved"
  }'
```

Available replies:
- `allow` - Allow this time and remember for future
- `deny` - Deny this request
- `allow_once` - Allow only this time

### Answer a Question

```bash
curl -X POST http://localhost:4096/question/req_456/reply \
  -H "Content-Type: application/json" \
  -d '{
    "answers": {
      "database_type": "PostgreSQL",
      "port": "5432"
    }
  }'
```

### Fork a Session

```bash
curl -X POST http://localhost:4096/session/session_123/fork \
  -H "Content-Type: application/json" \
  -d '{
    "messageID": "msg_789",
    "agent": "build"
  }'
```

---

## Response Codes

- `200` - Success
- `204` - Success, no content (async operations)
- `400` - Bad request (validation error)
- `404` - Not found (session/message doesn't exist)
- `500` - Server error

---

## Working with Server-Sent Events (SSE)

### JavaScript Example

```javascript
const eventSource = new EventSource('http://localhost:4096/event')

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log('Event:', data.type, data.properties)
  
  switch (data.type) {
    case 'permission.created':
      // Handle permission request
      break
    case 'question.created':
      // Handle question
      break
    case 'part.created':
      // Handle new message part
      break
  }
}

eventSource.onerror = (error) => {
  console.error('SSE Error:', error)
}
```

### Python Example

```python
import requests
import json

response = requests.get('http://localhost:4096/event', stream=True)

for line in response.iter_lines():
    if line.startswith(b'data: '):
        data = json.loads(line[6:])
        print(f"Event: {data['type']}")
        
        if data['type'] == 'permission.created':
            # Handle permission request
            pass
```

---

## Integration Patterns

### Pattern 1: Fire and Forget

Send a message without waiting for completion:

```bash
curl -X POST http://localhost:4096/session/:id/prompt_async \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"type":"text","text":"Run tests"}]}'
```

Use the SSE stream to track progress.

### Pattern 2: Synchronous Request

Send message and wait for response:

```bash
curl -X POST http://localhost:4096/session/:id/message \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"type":"text","text":"Explain this code"}]}'
```

### Pattern 3: Human-in-the-Loop

1. Subscribe to `/event` stream
2. Send message
3. Listen for `permission.created` or `question.created` events
4. Respond with `/permission/:id/reply` or `/question/:id/reply`
5. Agent continues execution

---

## Common Use Cases

### Execute a Shell Command

```json
POST /session/:id/shell
{
  "command": "npm test",
  "cwd": "/path/to/project"
}
```

### Run a Predefined Command

```json
POST /session/:id/command
{
  "command": "commit",
  "args": ["Fix authentication bug"]
}
```

### Get Session History

```bash
curl http://localhost:4096/session/:sessionID/message?limit=50
```

### Check Session Status

```bash
curl http://localhost:4096/session/status
```

Returns:
```json
{
  "session_123": {"type": "busy"},
  "session_456": {"type": "idle"}
}
```

---

## Tips

1. **Always subscribe to `/event`** - Essential for permission/question handling
2. **Use abort endpoint** - Stop runaway sessions
3. **Fork strategically** - Branch conversations at decision points
4. **Set directory header** - Isolate projects with `x-opencode-directory`
5. **Monitor session status** - Check if session is busy before sending messages
6. **Handle permissions gracefully** - Implement UI for approval workflow
7. **Use pagination** - Large histories can be expensive, use `limit` parameter

---

## See Also

- [Full Architecture Documentation](./ARCHITECTURE.md)
- [OpenCode Main README](../README.md)
- OpenAPI spec available at `GET /doc`
