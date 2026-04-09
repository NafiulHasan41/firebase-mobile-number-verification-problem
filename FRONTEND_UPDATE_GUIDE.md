# Kaya Frontend — Update Guide
_For the Flutter developer. Backend changes that require frontend updates._
_Date: 2026-04-09_

---

## What Changed (Summary)

| # | Change | Impact |
|---|--------|--------|
| 1 | `POST /api/v1/sessions/start` now returns the coach's first message | Frontend must display it — never show a blank chat screen |
| 2 | Intro sessions have no expiry | No countdown timer for intro — hide it entirely |
| 3 | Coach always knows user's name | No change needed — backend handles this |
| 4 | New `POST /api/v1/sessions/start/stream` endpoint | **Recommended** — replaces `/start` for a faster, streaming session open experience |

---

## Change 1 — `POST /api/v1/sessions/start` Returns First Message

### What changed

Previously: `POST /sessions/start` returned only a session ID. Frontend would show a blank chat screen and wait for the user to type first.

Now: The coach's opening message is included directly in the `/start` response. The user arrives to a live conversation, not a blank screen.

### Updated response shape

```json
POST /api/v1/sessions/start
{ "type": "intro" }

→ 200 OK

{
  "session_id": "1909d362-91a0-4ee1-a003-e37ecc2ebde3",
  "type": "intro",
  "status": "active",
  "time_remaining_seconds": null,
  "expires_at": null,
  "first_message": "Hey Nafiul! Welcome — I'm really glad you're here. I'm Kaya, your AI Functional Medicine Health Coach...",
  "first_message_type": "text"
}
```

New fields:

| Field | Type | Description |
|-------|------|-------------|
| `first_message` | `string \| null` | Coach's opening message. Null only if something went wrong (rare). |
| `first_message_type` | `string \| null` | `"text"` or `"options"`. Same as `message_type` on a normal `done` event. |

### What to do

**On `POST /sessions/start` success:**
1. Take the `session_id` from the response.
2. If `first_message` is not null, immediately add it to the message list as an assistant bubble — do **not** send any message to get the opener.
3. Enable the input field so the user can respond.

```dart
final result = await api.startSession(type: 'intro');
final sessionId = result['session_id'];

setState(() {
  // Show the coach's opening immediately
  if (result['first_message'] != null) {
    messageList.add(Message(
      role: 'assistant',
      content: result['first_message'],
      messageType: result['first_message_type'] ?? 'text',
    ));
  }
  sendButtonEnabled = true;
});
scrollToBottom();
```

### Loading history for an existing session

If the user leaves and comes back to a session that was already started, use `GET /api/v1/sessions/{id}/messages` as before. The coach's opening message is already in the history — you do not need to handle it separately. Just render the full history list as normal.

```
GET /api/v1/sessions/{session_id}/messages
```

The first item in `messages` will be the coach's opening (role: `assistant`). Render it like any other assistant bubble.

### What NOT to do

| Wrong | Why |
|-------|-----|
| Show a blank chat and wait for user to type first | User has no idea what to do |
| Send a "Hi" message to trigger the opener | Opening is already done — this creates a duplicate |
| Ignore `first_message` and re-fetch history | Unnecessary request; history already contains the opening |

---

## Change 2 — Intro Sessions Have No Expiry

### What changed

Foundation and follow-up sessions expire after 24 hours. The intro session now has **no expiry** — the user can take as long as they need, leave, come back days later, and continue.

### How to detect: check the type

When `session.type == "intro"`, treat it as open-ended. When `session.type == "foundation"` or `"followup"`, use the timer as before.

### Updated `GET /sessions/active` response for intro

```json
{
  "active_session": {
    "session_id": "1909d362-...",
    "type": "intro",
    "status": "active",
    "time_remaining_seconds": null,
    "expires_at": null,
    "progress": { ... },
    "started_at": "2026-04-08T10:00:00Z"
  }
}
```

`time_remaining_seconds` and `expires_at` are **always null for intro** — this is correct, not an error.

### Updated `POST /sessions/start` response for intro

Same — `time_remaining_seconds: null`, `expires_at: null` for intro.

For foundation and follow-up, these fields are unchanged — they still return the numeric countdown and the expiry timestamp.

### What to do

**Hide the countdown timer entirely for intro sessions.**

```dart
// Show timer only for foundation/followup — not for intro
if (session.type != 'intro' && session.timeRemainingSeconds != null) {
  showCountdownTimer(session.timeRemainingSeconds);
}
```

**Do not show any expiry warning for intro.** Do not show "Session expires in X hours". The intro has no time limit.

**Foundation and follow-up are unchanged** — keep the countdown timer as-is for those.

### Intro ends in only two ways

1. User gives consent → agent calls `end_session` → `message_type: session_ended` arrives in the stream → handle as normal session end.
2. User taps "Leave session" (force end) → same `session_ended` handling.

There is no timer-based expiry for intro. If you were showing a warning like "Your session expires in 3 hours" in the intro, remove it.

---

---

## Change 3 — New Streaming Session Start (`POST /sessions/start/stream`)

### Why this exists

The original `POST /sessions/start` was slow. Here is why: when the user taps "Start Session", the app had to wait for the coach's opening message to be fully generated by the AI before anything appeared on screen. That meant a blank loading screen for 3–5 seconds every time.

This new endpoint solves that. Instead of waiting, the session is created instantly and the coach's opening message streams in word by word — exactly like a normal chat message. The user sees text appearing almost immediately after tapping "Start Session".

**Old flow:**
```
User taps Start → [blank screen for 3–5 seconds] → full opening message appears at once
```

**New flow:**
```
User taps Start → [~50ms] session ready → "Welcome! It's so great..." streams in word by word
```

---

### Endpoint

```
POST /api/v1/sessions/start/stream
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
Content-Type: application/json

{ "type": "intro" }
```

`type` must be one of: `intro`, `foundation`, `followup` — same as before.

**Response:** `text/event-stream` (SSE)

---

### SSE Events

This endpoint produces **one extra event** at the very start compared to normal message streaming. After that, every event is identical to `POST /sessions/{id}/message/stream`.

**Event 1 — Session created (new)**
```
data: {"event": "session_created", "session_id": "uuid", "type": "intro", "status": "active", "time_remaining_seconds": null, "expires_at": null}
```

This is the first thing that arrives, within ~50ms. Grab `session_id` from here — you need it for all subsequent message calls.

**Events 2 onwards — identical to normal message streaming**
```
data: {"event": "status", "data": "thinking"}

data: {"event": "token", "data": "Welcome! It's so great"}
data: {"event": "token", "data": " to have you here today."}

data: {"event": "done", "message_type": "text", "response": "Welcome! It's so great to have you here today..."}
```

| Event | When | What to do |
|-------|------|------------|
| `session_created` | Immediately (~50ms) | Store `session_id`. Navigate to chat screen if not already there. |
| `status` | AI is thinking | Show thinking indicator |
| `token` | Text chunk arriving | Append to the first AI bubble in real-time |
| `done` | Full response ready | Finalize bubble. Enable send button. |
| `error` | Something went wrong | Session was still created — show error but session is usable. |

---

### What to do in Flutter

```dart
void startSession(String type) async {
  final request = http.Request(
    'POST',
    Uri.parse('$baseUrl/api/v1/sessions/start/stream'),
  );
  request.headers['Content-Type'] = 'application/json';
  request.headers['Authorization'] = 'Bearer $firebaseIdToken';
  request.headers['X-Device-Id'] = deviceId;
  request.body = jsonEncode({'type': type});

  setState(() => isLoading = true);
  final response = await http.Client().send(request);
  String sessionId = '';
  String streamBuffer = '';

  response.stream
    .transform(utf8.decoder)
    .transform(const LineSplitter())
    .listen((line) {
      if (!line.startsWith('data: ')) return;
      final json = jsonDecode(line.substring(6));

      switch (json['event']) {
        case 'session_created':
          // Session is ready — store the ID and show the chat screen
          sessionId = json['session_id'];
          setState(() {
            currentSessionId = sessionId;
            isLoading = false;
            // Add an empty AI bubble ready to fill in
            messageList.add(Message(role: 'assistant', content: ''));
          });
          break;

        case 'status':
          // Show "thinking..." indicator if desired
          break;

        case 'token':
          // Stream text into the AI bubble word by word
          streamBuffer += json['data'] as String;
          setState(() {
            messageList.last.content = streamBuffer;
          });
          break;

        case 'done':
          // Finalize the bubble with the clean complete text
          setState(() {
            messageList.last.content = json['response'];
            messageList.last.messageType = json['message_type'];
            sendButtonEnabled = true;  // user can now reply
          });
          break;

        case 'error':
          // Opener failed — session still exists, user can still chat
          setState(() {
            messageList.removeLast();  // remove the empty bubble
            sendButtonEnabled = true;
          });
          showErrorSnackbar('Could not load opening message. You can still start chatting.');
          break;
      }
    });
}
```

---

### Relationship to the old `POST /sessions/start`

| | `POST /sessions/start` | `POST /sessions/start/stream` |
|---|---|---|
| Speed | Slow (3–5s blank wait) | Fast (~50ms to first event) |
| Response format | JSON | SSE stream |
| `session_id` | In JSON body | In `session_created` event |
| Opening message | In `first_message` field | Streams via `token` events |
| Still works? | Yes — unchanged | New, recommended |

**The old `/start` endpoint still works.** You do not have to migrate immediately. But `/start/stream` is the recommended approach going forward — it gives a significantly better experience for the user.

---

### What NOT to do

| Wrong | Why |
|-------|-----|
| Wait for `done` before showing the chat screen | User still sees a blank screen — defeats the purpose |
| Start listening for `token` before `session_created` arrives | You don't have a `session_id` yet — always capture it from `session_created` first |
| Call the old `/start` and then this endpoint | Creates two sessions — only call one |
| Send a message before `session_created` arrives | You don't have a `session_id` yet |

---

| Topic | Reason |
|-------|--------|
| Streaming endpoints | Unchanged — same SSE events, same format |
| Chat threads | Unchanged |
| Session message endpoints | Unchanged |
| Session history (`GET /sessions/{id}/messages`) | Unchanged — opener is just the first message |
| Session feedback | Unchanged |
| Auth flow | Unchanged |
| Payment flow | Unchanged |
| Options buttons (`message_type: options`) | Unchanged |
| Session ended handling (`message_type: session_ended`) | Unchanged |

---

## Quick Checklist for Frontend Developer

### Using new `/start/stream` (recommended)
- [ ] Switch from `POST /sessions/start` to `POST /sessions/start/stream`
- [ ] On `session_created` event — store `session_id`, navigate to chat screen, add empty AI bubble
- [ ] On `token` events — fill the AI bubble in real-time (same as normal message streaming)
- [ ] On `done` — finalize bubble, enable send button
- [ ] On `error` — remove empty bubble, enable send button, show snackbar (session is still usable)

### Session type rules (unchanged)
- [ ] Intro session — hide countdown timer when `session.type == "intro"`
- [ ] Intro session — do not show expiry warning when `session.type == "intro"`
- [ ] Foundation / follow-up — timer and expiry behavior unchanged

### If staying on old `/start` for now
- [ ] Read `first_message` and display it immediately as an assistant bubble
- [ ] Do not show a blank screen or send a trigger message

### General
- [ ] Session history (`GET /sessions/{id}/messages`) — no change, render as before
