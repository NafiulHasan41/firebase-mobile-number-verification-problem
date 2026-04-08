# Kaya Mobile — Streaming, Send Button & History Guide

> For the mobile (Flutter) developer.
> Covers: why input must be locked during streaming, streaming endpoints, session history, and chat history.

---

## 1. Why the Send Button Must Be Disabled During Streaming

**This is the most important rule in this guide.**

Both session and chat endpoints use **Server-Sent Events (SSE)** streaming. The AI takes time to process (tool calls, memory lookups, RAG search) before and during its response. If the user sends a second message while the first is still streaming, **two AI agents run simultaneously on the server for the same session**.

### What happens without input locking

```
User sends "yes"       → Agent A starts processing
User sends "yes again" → Agent B starts (Agent A still running)

Agent A responds: "That's a beautiful vision, would you say..."
Agent B responds: "I love that clarity, would you like to..."

Result: Two AI responses back-to-back, both on the same topic.
```

This was observed in real testing — the AI appeared to repeat itself mid-conversation, and worse, it called `end_session` twice producing **two session-ended messages**.

### The fix — disable the send button on send, re-enable on `done`

```
User taps Send
  → Disable send button immediately
  → Connect to SSE stream
  → Receive status/token events (show thinking indicator, stream text)
  → Receive "done" event
  → Re-enable send button
```

**Disable on:** user taps send (or taps an options button)
**Re-enable on:** `done` event received OR `error` event received

```dart
// Pseudocode
void sendMessage(String text) {
  setState(() => sendButtonEnabled = false);   // disable immediately

  streamMessage(text, onDone: () {
    setState(() => sendButtonEnabled = true); // re-enable on done
  }, onError: () {
    setState(() => sendButtonEnabled = true); // also re-enable on error
  });
}
```

> **Also applies to options buttons.** When the user taps a button in an `options` message, that triggers a send — disable the send button immediately, hide the option buttons, and re-enable when the `done` event arrives.

---

## 2. Streaming Endpoints

### Session message (coaching sessions)

```
POST /api/v1/sessions/{session_id}/message/stream
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
Content-Type: application/json

{ "message": "I've been feeling really tired lately" }
```

**Response:** `text/event-stream`

### Chat message (general chat thread)

```
POST /api/v1/chat/threads/{thread_id}/message/stream
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
Content-Type: application/json

{ "message": "What is functional medicine?" }
```

**Response:** `text/event-stream`

---

## 3. SSE Event Flow

Both endpoints produce the same event format:

```
data: {"event": "status", "data": "thinking"}

data: {"event": "status", "data": "tool_use", "tool": "update_user_memory"}

data: {"event": "token", "data": "That "}
data: {"event": "token", "data": "sounds "}
data: {"event": "token", "data": "tough."}

data: {"event": "done", "message_type": "text", "response": "That sounds tough."}
```

| Event | When | What to do |
|---|---|---|
| `status: thinking` | AI is processing | Show thinking indicator |
| `status: tool_use` | AI calling a tool | Optionally show "searching..." |
| `token` | Text chunk | Append to current bubble in real-time |
| `done` | Final response | Read `message_type`, re-enable send button |
| `error` | Failure | Show error, re-enable send button, allow retry |

### `done` is the source of truth

Always replace streamed text with `done.response` on completion — it is the clean, final version.
Always read `message_type` from `done`, not from tokens.

```dart
String streamBuffer = '';

void handleSSEEvent(Map<String, dynamic> json) {
  switch (json['event']) {
    case 'status':
      showThinkingIndicator();
      break;

    case 'token':
      streamBuffer += json['data'] as String;
      updateCurrentBubble(streamBuffer); // re-render with Markdown widget
      break;

    case 'done':
      hideThinkingIndicator();
      final response = json['response'] as String;
      final messageType = json['message_type'] as String;

      finalizeCurrentBubble(response);        // replace streamed text with clean final
      handleMessageType(messageType, json);
      setState(() => sendButtonEnabled = true); // RE-ENABLE SEND BUTTON HERE
      break;

    case 'error':
      showError(json['data']);
      setState(() => sendButtonEnabled = true); // RE-ENABLE ON ERROR TOO
      break;
  }
}
```

---

## 4. Handling `message_type` on `done`

### `text` — normal response
Render as chat bubble. Unlock input.

### `options` — AI is asking user to choose
```json
{
  "event": "done",
  "message_type": "options",
  "response": "What area would you like to focus on?",
  "options": ["Sleep", "Energy", "Digestion", "Stress"]
}
```
Render text bubble + tappable buttons below it.
When user taps a button → send that text as next message → lock input again.

### `session_ended` — session is over (sessions only)
```json
{
  "event": "done",
  "message_type": "session_ended",
  "response": "Great work today! Your foundation session is complete.",
  "next_session_type": "followup"
}
```
Render final bubble → **permanently disable input for this session** → show "Go to Home" button.
Do not use `next_session_type` to start a session — go to Home screen, which calls `GET /api/v1/me/home` for the correct next step.

### `session_suggestion` — AI suggests starting a session (chat only)
```json
{
  "event": "done",
  "message_type": "session_suggestion",
  "response": "This is best explored in a Foundation session. Want to start one?",
  "session_type": "foundation",
  "is_free": true
}
```
Render bubble + "Start Foundation Session" button with free/paid badge.

---

## 5. Loading Session History

Use this when opening an existing session to restore the conversation.

```
GET /api/v1/sessions/{session_id}/messages
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
```

### Response

```json
{
  "session_id": "847e935b-...",
  "type": "intro",
  "status": "active",
  "messages": [
    {
      "role": "user",
      "content": "hey",
      "message_type": "text",
      "created_at": "2026-04-03T20:29:10.000Z"
    },
    {
      "role": "assistant",
      "content": "Hey there! Welcome — I'm so glad you're here.",
      "message_type": "text",
      "created_at": "2026-04-03T20:29:20.000Z"
    }
  ]
}
```

### What to render

| `role` | `message_type` | What to render |
|---|---|---|
| `user` | `text` | User chat bubble |
| `assistant` | `text` | AI chat bubble (render Markdown) |
| `assistant` | `options` | AI bubble — options buttons are **not** re-shown in history (user already responded) |
| `assistant` | `session_ended` | AI bubble + disabled input + "Go to Home" button |

> **The history only contains real user messages and final AI responses.** No internal messages will ever appear in the response.

### Checking if session is still active

Check `status` in the response:
- `active` → input enabled, user can continue
- `completed` or `expired` → input disabled, show history read-only

Or call `GET /api/v1/sessions/active` first — if the returned `session_id` matches, it is active.

---

## 6. Loading Chat Thread History

Use this when opening a general chat thread.

```
GET /api/v1/chat/threads/{thread_id}
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
```

### Response

```json
{
  "thread_id": "abc123...",
  "title": "Chat with Kaya",
  "messages": [
    {
      "id": "msg-uuid",
      "role": "user",
      "content": "What is functional medicine?",
      "message_type": "text",
      "created_at": "2026-04-03T10:00:00Z"
    },
    {
      "id": "msg-uuid",
      "role": "assistant",
      "content": "Functional Medicine looks at the root cause...",
      "message_type": "text",
      "created_at": "2026-04-03T10:00:08Z"
    },
    {
      "id": "msg-uuid",
      "role": "assistant",
      "content": "This is best explored in a Foundation session. Want to start one?",
      "message_type": "session_suggestion",
      "session_type": "foundation",
      "is_free": true,
      "created_at": "2026-04-03T10:05:00Z"
    }
  ]
}
```

### What to render

| `role` | `message_type` | What to render |
|---|---|---|
| `user` | `text` | User chat bubble |
| `assistant` | `text` | AI chat bubble (render Markdown) |
| `assistant` | `session_suggestion` | AI bubble + "Start Session" button (re-render with `session_type` + `is_free`) |

### Listing all threads

```
GET /api/v1/chat/threads
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
```

```json
{
  "threads": [
    { "id": "abc123", "title": "Chat with Kaya", "created_at": "..." }
  ]
}
```

---

## 7. How History and New Messages Work Together

This section explains how to combine the history loaded from the API with new messages the user sends — so the chat screen feels seamless.

### The concept

Think of the chat screen as a single list of messages. When the screen opens, that list is filled from the API (history). As the user chats, new messages are simply **appended to the same list** — there is no separate "new messages" list to merge. History and live messages are one unified list.

```
┌─────────────────────────────┐
│  [history from API]         │  ← loaded on screen open
│  User: hey                  │
│  AI: Hello! How can I help? │
│  User: I feel tired         │
│  AI: Tell me more...        │
│─────────────────────────────│
│  [new messages appended]    │  ← added as user chats
│  User: yes                  │
│  AI: ▌ (streaming...)       │
└─────────────────────────────┘
```

### Step-by-step flow

**Step 1 — Screen opens: fetch and display history**

Call `GET /api/v1/sessions/{id}/messages` (or `/chat/threads/{id}`).
Put all returned messages into your message list and scroll to the bottom.

```dart
final data = await api.getSessionMessages(sessionId);
setState(() {
  messageList = data.messages;  // fill the list
});
scrollToBottom();
```

If `status` is `completed` or `expired` — show the list as read-only, disable input. The session is over.

---

**Step 2 — User sends a message: add it to the list immediately**

Do not wait for the server. Add the user's bubble to the list the moment they tap Send, then lock the input and start the stream.

```dart
void sendMessage(String text) {
  setState(() {
    messageList.add(Message(role: 'user', content: text));  // show instantly
    sendButtonEnabled = false;
  });
  scrollToBottom();
  startStream(text);
}
```

---

**Step 3 — Stream starts: add one empty AI bubble**

As soon as the stream connection opens, add a single empty AI bubble to the list. This is the bubble that will fill in token by token.

```dart
void onStreamConnected() {
  setState(() {
    messageList.add(Message(role: 'assistant', content: ''));  // empty placeholder
  });
  scrollToBottom();
}
```

---

**Step 4 — Token events: update the last bubble in place**

Each `token` event appends text to the last message in the list (the AI bubble you just added). Do **not** add a new bubble per token — just update the existing one.

```dart
case 'token':
  setState(() {
    messageList.last.content += json['data'];  // update in place
  });
  break;
```

---

**Step 5 — `done` event: finalize and unlock**

Replace the streamed text with the clean final `response`, apply `message_type` logic, and unlock the input.

```dart
case 'done':
  setState(() {
    messageList.last.content = json['response'];         // clean final text
    messageList.last.messageType = json['message_type']; // text / options / session_ended
    sendButtonEnabled = true;                            // re-enable send button
  });
  handleMessageType(json);  // show buttons, disable input, etc.
  break;
```

---

**Step 6 — Error: unlock and let user retry**

```dart
case 'error':
  setState(() {
    messageList.removeLast();       // remove the empty AI bubble
    sendButtonEnabled = true;       // re-enable send button so user can retry
  });
  showErrorSnackbar(json['data']);
  break;
```

---

### What NOT to do

| Wrong | Why |
|---|---|
| Re-fetch history after every message | Causes flicker, duplicates messages already in the list |
| Add a new bubble per `token` event | Creates hundreds of bubbles instead of one filling in |
| Wait for `done` to show the AI bubble | User sees nothing for 10+ seconds — bad UX |
| Keep a separate list for new messages and merge them | Unnecessary complexity — one list is all you need |

---

## 8. Quick Reference

| Action | Endpoint |
|---|---|
| Send session message (streaming) | `POST /api/v1/sessions/{id}/message/stream` |
| Send chat message (streaming) | `POST /api/v1/chat/threads/{id}/message/stream` |
| Load session message history | `GET /api/v1/sessions/{id}/messages` |
| Load chat thread + history | `GET /api/v1/chat/threads/{id}` |
| List all chat threads | `GET /api/v1/chat/threads` |
| Check active session | `GET /api/v1/sessions/active` |

