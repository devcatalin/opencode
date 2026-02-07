# Event Streaming with the `/event` Endpoint

This document explains how event streaming works in OpenCode and how to filter events for specific sessions.

## Overview

The `/event` endpoint provides a **Server-Sent Events (SSE)** stream that broadcasts **all events** from sessions running within the same **project directory** (Instance). Clients must filter events on their side to process only the events they care about.

## Key Concepts

### Instance Isolation

OpenCode creates isolated instances per project directory. When you connect to `/event`:

```bash
# Events for /project1
curl -H "x-opencode-directory: /project1" http://localhost:4096/event

# Events for /project2  
curl -H "x-opencode-directory: /project2" http://localhost:4096/event
```

Each connection receives events **only** for sessions in that project directory. Different directories have separate event streams.

### No Session-Level Subscription

**Important:** You cannot subscribe to events from a specific session. The `/event` endpoint streams **all** events from all sessions in the project directory. You must filter on the client side.

## Event Structure

All events follow this structure:

```json
{
  "type": "event.type.name",
  "properties": {
    "sessionID": "session_123",
    ...other properties
  }
}
```

Most events include a `sessionID` property that identifies which session generated the event.

## Event Types

### Server Events
```json
{"type": "server.connected", "properties": {}}
{"type": "server.heartbeat", "properties": {}}
{"type": "server.instance.disposed", "properties": {"directory": "/path"}}
```

### Session Events
```json
{"type": "session.created", "properties": {"info": {...}}}
{"type": "session.updated", "properties": {"info": {...}}}
{"type": "session.deleted", "properties": {"sessionID": "..."}}
```

### Message Events
```json
{"type": "message.created", "properties": {"sessionID": "...", "info": {...}}}
{"type": "message.updated", "properties": {"sessionID": "...", "info": {...}}}
{"type": "message.removed", "properties": {"sessionID": "...", "messageID": "..."}}
```

### Part Events (streaming content)
```json
{"type": "part.created", "properties": {"sessionID": "...", "messageID": "...", "part": {...}}}
{"type": "part.updated", "properties": {"sessionID": "...", "messageID": "...", "part": {...}}}
{"type": "part.removed", "properties": {"sessionID": "...", "messageID": "...", "partID": "..."}}
```

### Permission Events
```json
{"type": "permission.created", "properties": {"sessionID": "...", "requestID": "...", ...}}
{"type": "permission.resolved", "properties": {"sessionID": "...", "requestID": "...", ...}}
```

### Question Events
```json
{"type": "question.created", "properties": {"sessionID": "...", "requestID": "...", ...}}
{"type": "question.answered", "properties": {"sessionID": "...", "requestID": "...", ...}}
```

## Client-Side Filtering

Since all events from all sessions are streamed, you must filter them client-side:

### JavaScript Example: Single Session

```javascript
const sessionID = 'session_123'
const eventSource = new EventSource('http://localhost:4096/event')

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  
  // Filter for specific session
  if (data.properties.sessionID === sessionID) {
    console.log('Event for my session:', data.type)
    
    switch (data.type) {
      case 'part.created':
        // Handle new content streaming
        displayPart(data.properties.part)
        break
      case 'permission.created':
        // Handle permission request
        promptUser(data.properties)
        break
    }
  }
}
```

### JavaScript Example: Multiple Sessions

```javascript
const sessions = new Map() // sessionID -> handler
const eventSource = new EventSource('http://localhost:4096/event')

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  const sessionID = data.properties.sessionID
  
  // Route to appropriate session handler
  const handler = sessions.get(sessionID)
  if (handler) {
    handler.handleEvent(data)
  }
}

// Track multiple sessions
sessions.set('session_123', new SessionHandler('session_123'))
sessions.set('session_456', new SessionHandler('session_456'))
```

### Python Example

```python
import sseclient
import requests
import json

target_session = 'session_123'
response = requests.get('http://localhost:4096/event', stream=True)
client = sseclient.SSEClient(response)

for event in client.events():
    data = json.loads(event.data)
    
    # Filter for specific session
    if data.get('properties', {}).get('sessionID') == target_session:
        print(f"Event: {data['type']}")
        
        if data['type'] == 'permission.created':
            # Handle permission request
            request_id = data['properties']['requestID']
            approve_permission(request_id)
```

## Common Patterns

### Pattern 1: UI Dashboard (All Sessions)

Display all active sessions and their events:

```javascript
const eventSource = new EventSource('http://localhost:4096/event')
const sessionStates = new Map()

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  
  // Update UI for all sessions
  switch (data.type) {
    case 'session.created':
      sessionStates.set(data.properties.info.id, {
        title: data.properties.info.title,
        status: 'idle'
      })
      updateDashboard()
      break
      
    case 'message.created':
      const sid = data.properties.sessionID
      sessionStates.get(sid).status = 'busy'
      updateDashboard()
      break
  }
}
```

### Pattern 2: Single Session View

Focus on one session, ignore others:

```javascript
class SessionView {
  constructor(sessionID) {
    this.sessionID = sessionID
    this.eventSource = new EventSource('http://localhost:4096/event')
    
    this.eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data)
      
      // Only process events for this session
      if (data.properties.sessionID !== this.sessionID) {
        return
      }
      
      this.handleEvent(data)
    }
  }
  
  handleEvent(data) {
    console.log(`Session ${this.sessionID}:`, data.type)
    // ... handle event
  }
}
```

### Pattern 3: Event Type Filtering

Only react to specific event types:

```javascript
const eventSource = new EventSource('http://localhost:4096/event')

// Only care about permissions and questions
const relevantEvents = new Set([
  'permission.created',
  'question.created'
])

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  
  if (relevantEvents.has(data.type)) {
    // Handle human-in-the-loop events
    handleInteraction(data)
  }
}
```

## Connection Management

### Single Connection Per Project

**Best Practice:** Use **one** SSE connection per project directory and multiplex events to different handlers.

```javascript
// ✅ Good: Single connection
const eventBus = new EventSource('http://localhost:4096/event')
sessionA.subscribe(eventBus)
sessionB.subscribe(eventBus)

// ❌ Bad: Multiple connections (wasteful)
const eventSource1 = new EventSource('http://localhost:4096/event')
const eventSource2 = new EventSource('http://localhost:4096/event')
```

### Reconnection

SSE automatically reconnects on disconnect. Handle reconnection:

```javascript
eventSource.addEventListener('error', (e) => {
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('Connection closed, will reconnect automatically')
  }
})
```

### Heartbeat

The server sends `server.heartbeat` every 30 seconds to prevent timeouts. You don't need to respond, but you can use it to detect connection health:

```javascript
let lastHeartbeat = Date.now()

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  
  if (data.type === 'server.heartbeat') {
    lastHeartbeat = Date.now()
  }
}

// Check connection health
setInterval(() => {
  if (Date.now() - lastHeartbeat > 60000) {
    console.warn('No heartbeat for 60s, connection may be stale')
  }
}, 10000)
```

## Multi-Project Setup

To monitor multiple projects simultaneously, create separate connections:

```javascript
const projects = [
  '/home/user/project1',
  '/home/user/project2'
]

const connections = projects.map(dir => {
  const eventSource = new EventSource('http://localhost:4096/event', {
    headers: {
      'x-opencode-directory': dir
    }
  })
  
  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data)
    console.log(`Event from ${dir}:`, data.type)
  }
  
  return { directory: dir, eventSource }
})
```

## Performance Considerations

### Event Volume

A busy session can generate many events:
- One `part.created` per text chunk (streaming)
- One `part.updated` per tool execution state change
- Multiple `message.*` events per conversation turn

For high-volume sessions, consider:
- Debouncing UI updates
- Batching rapid events
- Filtering out events you don't need

### Memory Management

Unsubscribe when done:

```javascript
// Close connection when no longer needed
eventSource.close()

// Or clean up specific handlers
window.addEventListener('beforeunload', () => {
  eventSource.close()
})
```

## Debugging

### Log All Events

```javascript
const eventSource = new EventSource('http://localhost:4096/event')

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log('[EVENT]', data.type, {
    sessionID: data.properties.sessionID,
    timestamp: new Date().toISOString(),
    properties: data.properties
  })
}
```

### Filter by Event Type

```bash
# Using curl and jq to filter specific event types
curl -N http://localhost:4096/event | \
  while read -r line; do
    echo "$line" | grep "^data:" | sed 's/^data: //' | \
      jq 'select(.type == "permission.created")'
  done
```

## FAQ

### Q: Can I subscribe to events from a specific session only?

**A:** No. The `/event` endpoint streams all events from all sessions in the project directory. Filter by `sessionID` on the client side.

### Q: Do I see events from other users' sessions?

**A:** Not if they're in different project directories. Each project directory is isolated. If multiple users work in the same directory (shared server scenario), they will see each other's events.

### Q: What happens when I disconnect?

**A:** Your subscription ends, but sessions continue running. When you reconnect, you won't receive past events—only new events going forward. Use `GET /session/:id/message` to fetch history.

### Q: How do I know when a session is complete?

**A:** Listen for `message.updated` where the message has a `finish` field (e.g., `"finish": "stop"`). This indicates the agent has completed its response.

### Q: Can I filter events server-side?

**A:** Not currently. All events from the project directory are streamed. Client-side filtering is the only option.

## Summary

- **One stream per project directory** - `/event` broadcasts all events for that directory
- **Client-side filtering required** - Filter by `sessionID` or event type in your code
- **Session lifecycle events** - `session.*` events track creation/updates/deletion
- **Real-time content streaming** - `part.*` events deliver content as it's generated
- **Human-in-the-loop** - `permission.*` and `question.*` events require client interaction
- **Single connection pattern** - Reuse one SSE connection and multiplex to multiple handlers

For complete event schemas and API reference, see:
- [ARCHITECTURE.md](./ARCHITECTURE.md) - Full system architecture
- [API-QUICK-REFERENCE.md](./API-QUICK-REFERENCE.md) - Quick API reference
