# Kaya Backend — Session & Chat Message Types

> Companion doc to `API_DOCS.md` and `API_DOCS_V2.md`.
> Covers everything a mobile developer needs to know about `message_type` values, response shapes, and what to render for each.

> **Important — Always Use Streaming:**
> For both coaching sessions and general chat, always use the **streaming (`/stream`) endpoints**:
> - Session messages: `POST /api/v1/sessions/{session_id}/message/stream`
> - Chat messages: `POST /api/v1/chat/threads/{thread_id}/message/stream`
>
> The non-streaming variants block for 10–15 seconds with no feedback to the user — do not use them in production.
> Streaming uses **Server-Sent Events (SSE)**. The `done` event is always the source of truth for `message_type` and the final response.

---

## Overview

Every message response from both **coaching sessions** and **general chat** includes a `message_type` field. This field tells the frontend what kind of message it is and whether any extra UI action is needed.

| `message_type` | Where it appears | What the frontend should do |
|---|---|---|
| `text` | Sessions + Chat | Render as normal chat bubble |
| `options` | Sessions + Chat | Render text + tappable button row below |
| `session_suggestion` | Chat only | Render text + "Start Session" button |
| `session_ended` | Sessions only | Render text + disable input / show next-step UI |

The `done` SSE event (streaming) and the JSON response body (non-streaming) are the **source of truth** — always read `message_type` from there, not from intermediate `token` events.

---

## 1. `message_type: "text"` — Normal Message

The AI responded with plain text. No extra UI needed.

### Streaming (SSE) — `done` event
```
data: {"event": "done", "message_type": "text", "response": "Great, let's explore that together."}
```

### Non-streaming response
```json
{
  "response": "Great, let's explore that together.",
  "message_type": "text",
  "options": null
}
```

### What to do
Render `response` as a normal AI chat bubble. No buttons or special UI.

---

## 2. `message_type: "options"` — Options / Buttons

The AI is asking the user to choose from a list. Display `options` as tappable buttons below the message.

### Streaming (SSE) — `done` event
```
data: {"event": "done", "message_type": "options", "response": "What's your main concern?", "options": ["Sleep", "Energy", "Digestion", "Stress"]}
```

### Non-streaming response
```json
{
  "response": "What's your main concern?",
  "message_type": "options",
  "options": ["Sleep", "Energy", "Digestion", "Stress"]
}
```

### What to do
1. Render `response` as an AI chat bubble.
2. Below the bubble, render `options` as tappable buttons.
3. When the user taps a button, send that button text as the next message:
   ```json
   { "message": "Sleep" }
   ```
4. Hide the buttons after the user taps (the conversation continues normally).

### Flutter/Dart example
```dart
case 'done':
  final response = json['response'] as String;
  final messageType = json['message_type'] as String;

  addAiMessage(response);

  if (messageType == 'options') {
    final options = List<String>.from(json['options'] as List);
    showOptionButtons(options, onTap: (selected) {
      sendMessage(selected); // sends selected text as next user message
    });
  }
  break;
```

---

## 3. `message_type: "session_suggestion"` — AI Suggesting a Session (Chat Only)

Appears **only in general chat** (`/api/v1/chat/threads/{thread_id}/message`).

When the user's question is better answered in a coaching session (e.g. "How do I fix my sleep?"), the AI ends its message with a suggestion to start a session. The backend detects this, strips the internal marker, and adds structured fields to the response.

### Streaming (SSE) — `done` event
```
data: {
  "event": "done",
  "message_type": "session_suggestion",
  "response": "That's something we dig into deeply during a Foundation session. Would you like to start one?",
  "session_type": "foundation",
  "is_free": true
}
```

### Non-streaming response
```json
{
  "response": "That's something we dig into deeply during a Foundation session. Would you like to start one?",
  "message_type": "session_suggestion",
  "options": null,
  "session_type": "foundation",
  "is_free": true
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `session_type` | string | The session type to start: `"intro"`, `"foundation"`, or `"followup"` |
| `is_free` | bool | `true` if this session is free for the user (no credits charged), `false` if it costs a credit |

### `is_free` Logic

| Session Type | Condition | `is_free` |
|---|---|---|
| `intro` | Always free | `true` |
| `foundation` | No prior foundation sessions (first attempt) | `true` |
| `foundation` | User has had at least one foundation session before | `false` |
| `followup` | Always costs a credit | `false` |

### What to do
1. Render `response` as an AI chat bubble.
2. Below the bubble, show a **"Start [session type] Session"** button.
3. `is_free: true` → optionally show "Free" badge on the button.
4. `is_free: false` → optionally show "1 credit" badge on the button.
5. When the user taps: navigate to the session start flow with `session_type` pre-filled.
6. **The frontend creates the session** — call `POST /api/v1/sessions/start` with `{"type": session_type}`.

### Example button UI
```
┌────────────────────────────────────────────────┐
│ AI: That's something we dig into deeply during  │
│ a Foundation session. Would you like to start   │
│ one?                                            │
└────────────────────────────────────────────────┘
           [ Start Foundation Session  FREE ]
```

```
┌────────────────────────────────────────────────┐
│ AI: This is exactly what we cover in a         │
│ follow-up session. Want to book one?            │
└────────────────────────────────────────────────┘
           [ Start Follow-up Session  1 credit ]
```

### Flutter/Dart example
```dart
case 'done':
  final messageType = json['message_type'] as String;
  addAiMessage(json['response'] as String);

  if (messageType == 'session_suggestion') {
    final sessionType = json['session_type'] as String;
    final isFree = json['is_free'] as bool;

    showSessionSuggestionButton(
      sessionType: sessionType,
      isFree: isFree,
      onTap: () {
        // Navigate to session start or directly start session
        Navigator.push(context, SessionStartScreen(type: sessionType));
      },
    );
  }
  break;
```

### Reading from message history

When loading a chat thread (`GET /api/v1/chat/threads/{thread_id}`), suggestion messages include the structured fields:

```json
{
  "id": "...",
  "role": "assistant",
  "content": "That's something we dig into during a Foundation session. Would you like to start one?",
  "message_type": "session_suggestion",
  "session_type": "foundation",
  "is_free": true,
  "created_at": "2026-03-31T10:00:00"
}
```

Use `session_type` and `is_free` to re-render the suggestion button in history. If the user already started that session, you can choose to show the button as disabled/completed.

---

## 4. `message_type: "session_ended"` — Session Completed (Sessions Only)

Appears **only in coaching sessions** (`/api/v1/sessions/{session_id}/message`).

When the AI determines the session is complete, it calls its internal `end_session` tool. The final AI message is delivered with `message_type: "session_ended"`, signalling the frontend to disable further input on that session and guide the user to their next step.

### Streaming (SSE) — `done` event
```
data: {
  "event": "done",
  "message_type": "session_ended",
  "response": "Excellent work today! We've set your health goals and you're ready to start your follow-up sessions.",
  "next_session_type": "followup"
}
```

### Non-streaming response
```json
{
  "response": "Excellent work today! We've set your health goals and you're ready to start your follow-up sessions.",
  "message_type": "session_ended",
  "options": null,
  "next_session_type": "followup"
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `next_session_type` | string or null | Informational only — the recommended next session type. Do not use this to start a session directly. |

> **Do not act on `next_session_type` directly.** It is provided for reference only. Always navigate the user back to the home screen — the home screen calls `GET /me/home` which has the full journey timeline, next session recommendation, credit balance, and everything needed to show the correct next step.

### What to do
1. Render `response` as the final AI chat bubble.
2. **Disable the message input field** — this session is over. Any new message attempt would fail with a `400` error from the server.
3. Show a **"Go to Home"** button.
4. When the user taps — navigate to the home screen.
5. Home screen calls `GET /me/home` → reads `next_session` → shows the correct "Start [next] Session" button with free/credit badge.

> **Why `session_ended` exists:** Previously, the frontend had no signal that a session ended. The AI would call `end_session` internally, the user would send another message, and the backend returned a `400 Session already ended` error. This `message_type` fixes that — the frontend gets the signal in the same response that contains the AI's closing message.

### Example UI
```
┌────────────────────────────────────────────────┐
│ AI: Excellent work today! We've set your        │
│ health goals. You're all set!                   │
└────────────────────────────────────────────────┘
  [ Message input — DISABLED (session ended) ]
                 [ Go to Home ]
```

### Flutter/Dart example
```dart
case 'done':
  final messageType = json['message_type'] as String;
  addAiMessage(json['response'] as String);

  if (messageType == 'session_ended') {
    // Disable input — session is over
    setState(() { inputEnabled = false; });

    // Show Go to Home button — home screen handles what comes next
    showGoHomeButton(
      onTap: () {
        Navigator.pushAndRemoveUntil(
          context,
          MaterialPageRoute(builder: (_) => HomeScreen()),
          (route) => false,
        );
      },
    );
  }
  break;
```

---

## 5. Response Text Formatting (Markdown)

The AI uses **Markdown** in the `response` field. The frontend must render Markdown — not display the raw symbols.

### Formatting elements the AI uses

| Markdown | Renders as | Example raw text |
|---|---|---|
| `**text**` | **Bold** | `**Key insight:**` |
| `*text*` or `_text_` | *Italic* | `*Note:* this is important` |
| `1. item` | Numbered list | `1. Drink water\n2. Sleep early` |
| `- item` or `* item` | Bullet list | `- Reduce sugar\n- Add magnesium` |
| `## Heading` | Section heading | `## Your Goals` |
| `### Sub-heading` | Sub-heading | `### Step 1` |
| `` `code` `` | Inline code | `` `morning routine` `` |

### Real examples from the AI

**Bold for emphasis:**
```
Raw:    "Your **most important** action this week is consistent sleep timing."
Render: Your most important action this week is consistent sleep timing.
                ^^^^^^^^^^^^^^^ bold
```

**Numbered steps:**
```
Raw:
"Here's your plan:\n1. Wake up at 7am\n2. No screens 1 hour before bed\n3. Cut caffeine after 2pm"

Render:
Here's your plan:
1. Wake up at 7am
2. No screens 1 hour before bed
3. Cut caffeine after 2pm
```

**Bullet list:**
```
Raw:
"Common root causes:\n- Chronic stress\n- Poor sleep hygiene\n- Blood sugar spikes"

Render:
Common root causes:
• Chronic stress
• Poor sleep hygiene
• Blood sugar spikes
```

**Mixed bold + list:**
```
Raw:
"**Your goals for this week:**\n1. Sleep by 10pm\n2. *Track* your meals\n3. Morning walk"

Render:
Your goals for this week:   ← bold heading
1. Sleep by 10pm
2. Track your meals         ← "Track" in italic
3. Morning walk
```

### Recommended Flutter packages

Use a Markdown rendering widget. Do **not** display raw asterisks/hyphens to the user.

```dart
// Option 1: flutter_markdown (recommended)
import 'package:flutter_markdown/flutter_markdown.dart';

MarkdownBody(
  data: message.response,
  styleSheet: MarkdownStyleSheet(
    p: TextStyle(fontSize: 15, color: Colors.black87),
    strong: TextStyle(fontWeight: FontWeight.bold),
    em: TextStyle(fontStyle: FontStyle.italic),
    listBullet: TextStyle(fontSize: 15),
  ),
)

// Option 2: markdown_widget
import 'package:markdown_widget/markdown_widget.dart';

MarkdownWidget(data: message.response)
```

### Streaming with Markdown

During streaming, `token` events arrive word-by-word. Markdown symbols may arrive split across multiple tokens:

```
token: "**Your"
token: " goals"
token: ":**"
```

**Recommended approach:** accumulate all tokens into a string, re-render the full string on every token arrival. Most Markdown widgets handle partial/in-progress Markdown gracefully.

```dart
String fullText = '';

case 'token':
  fullText += json['data'] as String;
  setState(() { currentResponse = fullText; }); // re-render with Markdown widget
  break;

case 'done':
  // Replace accumulated text with the clean final response
  setState(() { currentResponse = json['response'] as String; });
  // Now apply message_type logic (options, session_ended, etc.)
  break;
```

### What NOT to do
- Do not strip `**`, `*`, `-`, `1.` — render them as formatting.
- Do not use a plain `Text()` widget for AI responses — bold/italic/lists will show as raw symbols.
- Do not try to parse Markdown manually — use a library.

---

## Full SSE Event Sequence Reference

Both `/sessions/{id}/message/stream` and `/chat/threads/{id}/message/stream` use the same SSE format:

```
data: {"event": "status", "data": "thinking"}

data: {"event": "status", "data": "tool_use", "tool": "update_user_memory"}

data: {"event": "token", "data": "I'd "}

data: {"event": "token", "data": "love "}

data: {"event": "token", "data": "to help!"}

data: {"event": "done", "message_type": "text", "response": "I'd love to help!"}
```

### Event breakdown

| Event | When | What to do |
|---|---|---|
| `status` (`thinking`) | AI is processing | Show "thinking..." indicator |
| `status` (`tool_use`) | AI is using a tool | Show "searching..." or tool name |
| `token` | Text chunk arriving | Append to current chat bubble in real-time |
| `done` | Final response | Source of truth — read `message_type` here |
| `error` | Failure | Show error message, allow retry |

### Important: `done` overrides tokens

The `token` events build the message character by character. The `done` event always contains the **complete, clean response** in `response`. On `done`:
- Replace the streamed text with `response` (in case of any rendering artifacts).
- Read `message_type` and apply extra UI if needed.
- Ignore `token` events after `done` (there should be none, but be safe).

---

## Session Types Reference

| Type | Cost | When | Free condition |
|---|---|---|---|
| `intro` | Free | First session ever | Always free |
| `foundation` | Free (1st) / 1 credit | After intro | Free if no prior foundation sessions |
| `followup` | 1 credit | After foundation (goals set) | Never free |

### Journey progression
```
intro (free)
  └─→ foundation (1st free, then 1 credit each)
        └─→ followup (1 credit each, repeating)
```

Foundation can repeat if goals weren't set. The user must set goals in foundation before they can start followup.

### Starting a session
```
POST /api/v1/sessions/start
Body: {"type": "intro" | "foundation" | "followup"}
```

### Error responses when starting a session

| HTTP | Exact `detail` string | Meaning | What to show the user |
|---|---|---|---|
| `402` | `No credits` | Paid session, balance is 0 | "You have no credits. Purchase credits to continue." |
| `400` | `Introductory session already completed. Please start a Foundation session.` | Trying to start a second intro | "Your Intro is done! Start your Foundation session next." |
| `400` | `Please complete an Introductory session first.` | Trying to start foundation before intro | "Complete your Introductory session first before starting Foundation." |
| `400` | `Foundation already completed. Please start a Follow-up session.` | Trying to start foundation again after it's fully done with goals | "Your Foundation is complete! Start a Follow-up session next." |
| `400` | `Please complete a Foundation session with goals set first.` | Trying to start follow-up before foundation is done (or goals weren't set) | "Complete your Foundation session and set your health goals first." |
| `400` | `Already have an active session` | One active session exists | "You have an active session. Resume it before starting a new one." |

> **Important:** If the home screen uses `/me/home` correctly (see section above), the user should **never** see most of these errors — the recommendation logic prevents showing the wrong button. These errors are a safety net for edge cases.

### Handling progression errors — Flutter/Dart hint

Parse the `detail` string from the error response and redirect the user to the correct session type. The exact UI (dialog, bottom sheet, toast, etc.) follows the client's design:

```dart
Future<void> startSession(String sessionType) async {
  try {
    final resp = await api.post('/sessions/start', body: {'type': sessionType});
    Navigator.push(context, SessionChatScreen(sessionId: resp['session_id']));
  } on ApiException catch (e) {
    final detail = e.body['detail'] as String? ?? '';

    if (e.statusCode == 402) {
      // User has no credits — show purchase flow (per client UI)
      Navigator.push(context, PurchaseScreen());

    } else if (detail.contains('Introductory session first')) {
      // Hint: tell user they need to do intro first, offer to go there
      // e.g. "Complete your intro session before starting Foundation"
      // action → startSession('intro')

    } else if (detail.contains('Foundation already completed')) {
      // Hint: foundation is done, nudge toward follow-up
      // e.g. "Your foundation is done! Time to start a follow-up session"
      // action → startSession('followup')

    } else if (detail.contains('Foundation session with goals set first')) {
      // Hint: foundation not completed with goals — send back to foundation
      // e.g. "Finish your foundation session and set your goals first"
      // action → startSession('foundation')

    } else if (detail.contains('active session')) {
      // An active session already exists — refresh so Resume button appears
      loadHomeScreen();
    }
  }
}
```

---

## General Chat vs Coaching Sessions

| Feature | Coaching Sessions | General Chat |
|---|---|---|
| Endpoint | `/api/v1/sessions/{id}/message` | `/api/v1/chat/threads/{id}/message` |
| `message_type: text` | Yes | Yes |
| `message_type: options` | Yes | Yes |
| `message_type: session_suggestion` | No | **Yes** |
| `message_type: session_ended` | **Yes** | No |
| Billing | Credits charged on start | Free |
| Session expiry | 24-hour timer | No expiry |
| Memory scope | Session-specific + user profile | User profile only |

**General chat** is a free, open-ended conversation. When the AI determines the user's question is better served by a coaching session, it suggests one (`session_suggestion`).

**Coaching sessions** are structured, time-limited, goal-oriented. When the AI completes the session agenda, it ends the session (`session_ended`).

---

## Home Screen — `GET /api/v1/me/home`

Call this single endpoint when the home screen loads. It returns everything needed — journey timeline, next session button, credits, and paywall decision.

```
GET /api/v1/me/home
Headers: Authorization, X-Device-Id
```

See `API_DOCS_V2.md` section 4 for the full response shape and field reference.

### Flutter/Dart example — home screen logic

```dart
Future<void> loadHomeScreen() async {
  final home = await api.get('/me/home');

  final journey = home['journey'] as List;
  final next = home['next_session'];
  final credits = home['credits'];
  final paywallRequired = home['paywall_required'] as bool;

  // Active session check — last item in journey with status "active"
  final activeSession = journey.isNotEmpty && journey.last['status'] == 'active'
      ? journey.last
      : null;

  if (activeSession != null) {
    // Resume existing session
    showResumeButton(
      sessionId: activeSession['session_id'],
      sessionType: activeSession['type'],
      label: 'Resume ${activeSession['label']}',
    );
  } else if (paywallRequired) {
    showPaywall();
  } else {
    // Show start button
    showStartButton(
      label: next['label'],        // e.g. "Follow-up #3"
      isFree: next['is_free'],     // show FREE badge or "1 credit"
      sessionType: next['type'],
    );
  }

  // Render journey timeline
  renderJourney(journey);
}

// When user taps "Start Session"
Future<void> startSession(String sessionType) async {
  try {
    final resp = await api.post('/sessions/start', body: {'type': sessionType});
    final sessionId = resp['session_id'] as String;
    Navigator.push(context, SessionChatScreen(sessionId: sessionId, type: sessionType));
  } on ApiException catch (e) {
    if (e.statusCode == 402) {
      showDialog('No credits', 'Purchase credits to start this session.');
    } else if (e.statusCode == 400) {
      loadHomeScreen(); // refresh and re-evaluate
    }
  }
}
```

### Active session check

The active session is already inside the `journey` list — it appears as the last item with `"status": "active"`. No separate call to `GET /sessions/active` needed for the home screen.

**If an active session exists in journey:** show "Resume [label]" button instead of "Start Session". Take the user to the existing session chat — do NOT call `POST /sessions/start` again.

> `GET /sessions/active` still exists and works — use it inside the session chat screen to verify session state, but not for the home screen render.

### What to display when resuming an active session

When the user taps "Resume [type] Session", navigate to the session chat screen. The session is already running — do not call `POST /sessions/start` again.

**On the session chat screen:**

1. **Load message history** — call `GET /api/v1/sessions/{session_id}/messages` to fetch prior messages.
2. **Show countdown timer** — display `time_remaining_seconds` as a live countdown at the top of the screen. Decrement it locally every second (no need to poll the server — it is accurate at load time).
3. **Enable input** — the session is still active, user can continue chatting.
4. **Handle timer reaching 0** — when the countdown hits 0, the session has expired. Disable the input field and show a message like "Session time is up." The user's next message will return a `400` from the server anyway — handle it gracefully.

**Session chat screen layout:**
```
┌──────────────────────────────────────────┐
│  ← Back    Foundation Session   23:45:10 │  ← countdown
├──────────────────────────────────────────┤
│                                          │
│  [prior messages loaded from history]    │
│                                          │
│  AI: Let's talk about your energy        │
│      levels this week.                   │
│                                          │
├──────────────────────────────────────────┤
│  [ Type your message...          ] [Send] │
└──────────────────────────────────────────┘
```

**Flutter/Dart example — resuming a session:**
```dart
class SessionChatScreen extends StatefulWidget {
  final String sessionId;
  final String sessionType;
  final int timeRemainingSeconds; // from active session response
  ...
}

class _SessionChatScreenState extends State<SessionChatScreen> {
  List<Message> messages = [];
  bool isLoading = true;
  bool inputEnabled = true;
  late int secondsLeft;
  Timer? _countdownTimer;

  @override
  void initState() {
    super.initState();
    secondsLeft = widget.timeRemainingSeconds;
    _startCountdown();
    _loadHistory();
  }

  void _startCountdown() {
    _countdownTimer = Timer.periodic(Duration(seconds: 1), (_) {
      if (secondsLeft <= 0) {
        _countdownTimer?.cancel();
        setState(() { inputEnabled = false; });
        // Optionally show "Session expired" banner
      } else {
        setState(() { secondsLeft--; });
      }
    });
  }

  Future<void> _loadHistory() async {
    setState(() { isLoading = true; });
    try {
      final history = await api.get('/sessions/${widget.sessionId}/messages');
      setState(() {
        messages = (history['messages'] as List)
            .map((m) => Message.fromJson(m))
            .toList();
        isLoading = false;
      });
      // Check last message — if session_ended, disable input immediately
      if (messages.isNotEmpty && messages.last.messageType == 'session_ended') {
        setState(() { inputEnabled = false; });
      }
    } catch (e) {
      setState(() { isLoading = false; });
      // Show error state
    }
  }

  String _formatTime(int seconds) {
    final h = seconds ~/ 3600;
    final m = (seconds % 3600) ~/ 60;
    final s = seconds % 60;
    return '${h.toString().padLeft(2,'0')}:${m.toString().padLeft(2,'0')}:${s.toString().padLeft(2,'0')}';
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('${widget.sessionType.capitalize()} Session'),
        actions: [
          Text(_formatTime(secondsLeft), style: TextStyle(fontSize: 16)),
          SizedBox(width: 16),
        ],
      ),
      body: isLoading
          ? Center(child: CircularProgressIndicator())  // ← loading spinner while fetching history
          : Column(
              children: [
                Expanded(child: MessageList(messages: messages)),
                if (inputEnabled)
                  MessageInput(onSend: _sendMessage)
                else
                  SessionEndedFooter(), // disabled input + Go Home button
              ],
            ),
    );
  }
}
```

**Message history endpoint:**
```
GET /api/v1/sessions/{session_id}/messages
```

Response:
```json
{
  "messages": [
    { "role": "assistant", "content": "Hello! Let's start...", "message_type": "text", "created_at": "..." },
    { "role": "user", "content": "I've been struggling with sleep.", "message_type": "text", "created_at": "..." },
    { "role": "assistant", "content": "Which area concerns you most?", "message_type": "options", "created_at": "..." }
  ]
}
```

> Tool exchange messages (internal AI tool calls/results) are automatically filtered out — you only receive user and assistant messages that are meant to be shown.

---

## Quick Reference — All `done` Event Shapes

### Normal text
```json
{
  "event": "done",
  "message_type": "text",
  "response": "Here is what I found..."
}
```

### With options/buttons
```json
{
  "event": "done",
  "message_type": "options",
  "response": "Which area concerns you most?",
  "options": ["Sleep", "Energy", "Digestion"]
}
```

### Session suggestion (chat only)
```json
{
  "event": "done",
  "message_type": "session_suggestion",
  "response": "That's what we explore in a Foundation session. Want to start one?",
  "session_type": "foundation",
  "is_free": true
}
```

### Session ended (sessions only)
```json
{
  "event": "done",
  "message_type": "session_ended",
  "response": "Great session! You're ready for follow-up.",
  "next_session_type": "followup"
}
```

### Error
```
data: {"event": "error", "data": "Something went wrong. Please try again."}
```

---

## Decision Tree — How to Handle `done`

```
Received "done" event
├── message_type == "text"
│   └── Render response as chat bubble. Done.
│
├── message_type == "options"
│   └── Render response + tappable buttons from `options[]`
│       User taps → send that text as next message
│
├── message_type == "session_suggestion"  (chat only)
│   ├── Render response as chat bubble
│   └── Show "Start [session_type] Session" button
│       is_free: true  → show FREE badge
│       is_free: false → show "1 credit" badge
│       On tap → navigate to session start
│
└── message_type == "session_ended"  (sessions only)
    ├── Render response as chat bubble
    ├── Disable message input (session is over)
    └── Show "Go to Home" button
        On tap → navigate to home screen
        Home screen calls GET /me/home → shows correct next session button
```

---

## Non-Streaming Endpoint Shapes

For `POST /sessions/{id}/message` and `POST /chat/threads/{id}/message` (non-streaming):

### Normal text
```json
{
  "response": "Let's explore that.",
  "message_type": "text",
  "options": null
}
```

### With options
```json
{
  "response": "What matters most to you?",
  "message_type": "options",
  "options": ["Sleep", "Energy", "Stress"]
}
```

### Session suggestion
```json
{
  "response": "That's what Foundation sessions are for. Want to start one?",
  "message_type": "session_suggestion",
  "options": null,
  "session_type": "foundation",
  "is_free": true
}
```

### Session ended
```json
{
  "response": "Great work! See you in the next session.",
  "message_type": "session_ended",
  "options": null,
  "next_session_type": "followup"
}
```

> The non-streaming endpoint waits for the full AI response before returning (10–15 seconds). Always prefer the `/stream` endpoint.

---

## Complete Mobile Flow Guide

This section explains the end-to-end flow a mobile developer needs to implement — from app open to session end. Every screen, every API call, every loading state.

---

### Loading States — When to Show a Spinner

Show a loading indicator any time the app is waiting for a server response. Never leave the user staring at a blank screen.

| Screen | What triggers a loader | What to show |
|---|---|---|
| Home screen | Fetching `GET /me/home` on mount | Full-screen spinner or skeleton cards |
| Session start | `POST /sessions/start` (takes ~1s) | Button loading state / spinner overlay |
| Session chat open | `GET /sessions/{id}/messages` history load | Spinner in message area |
| Sending a message | Waiting for SSE stream to start | Disable send button + show "thinking..." bubble |
| Chat thread open | `GET /chat/threads/{id}` history | Spinner in message area |
| Creating a new thread | `POST /chat/threads` | Button loading state |

**Do not** hide loading states — the AI can take 3–8 seconds to respond. The user must always see feedback that something is happening.

```dart
// Pattern: set isLoading before any await, unset in finally
Future<void> loadHomeScreen() async {
  setState(() { isLoading = true; });
  try {
    final home = await api.get('/me/home');
    // ... apply state
  } catch (e) {
    showErrorBanner('Could not load. Check your connection.');
  } finally {
    setState(() { isLoading = false; });
  }
}
```

---

### Screen 1 — Home Screen

**On screen mount:**
1. Show loading spinner.
2. Call `GET /me/home` (single call — returns everything).
3. Hide spinner when response returns.

**Decision logic (in order):**

```
GET /me/home
│
├── journey has item with status "active"
│   └── show "Resume [label]" button
│       • label from journey item e.g. "Follow-up #2"
│       • Tap → navigate to SessionChatScreen with session_id
│
├── paywall_required == true
│   └── show Paywall screen
│
└── no active session + no paywall
    └── show Start Session button using next_session
        • next_session.label  →  button text  (e.g. "Follow-up #3")
        • next_session.is_free == true   →  show "FREE" badge
        • next_session.is_free == false  →  show "1 credit" label
        • credits.total_usable == 0 AND is_free == false
            →  show button DISABLED with "No credits" label
```

**What to display on home:**
```
┌──────────────────────────────────────────┐
│                                          │
│  Welcome back, [name]                    │
│                                          │
│  [Loading...]                            │  ← spinner while fetching
│                                          │
│  ──── or, when loaded ────               │
│                                          │
│  Active session:                         │
│  [ Resume Foundation Session  23h left ] │  ← if active session
│                                          │
│  ──── or ────                            │
│                                          │
│  [ Start Follow-up Session  1 credit ]   │  ← if no active session
│  [ Start Follow-up Session  No credits ] │  ← disabled, balance = 0
│                                          │
└──────────────────────────────────────────┘
```

---

### Screen 2 — Starting a New Session

When user taps "Start [type] Session":

1. Show button loading state (spinner inside button, disable it).
2. Call `POST /api/v1/sessions/start` with `{"type": session_type}`.
3. On success → navigate to `SessionChatScreen(sessionId, sessionType, timeRemainingSeconds)`.
4. On error → handle per the error table in "Session Types Reference" section.

```dart
Future<void> onStartSession(String sessionType) async {
  setState(() { isStarting = true; });
  try {
    final resp = await api.post('/sessions/start', body: {'type': sessionType});
    Navigator.push(context, MaterialPageRoute(
      builder: (_) => SessionChatScreen(
        sessionId: resp['session_id'],
        sessionType: sessionType,
        timeRemainingSeconds: resp['time_remaining_seconds'],
      ),
    ));
  } on ApiException catch (e) {
    if (e.statusCode == 402) {
      Navigator.push(context, PurchaseScreen());
    } else {
      showDialog('Cannot start session', e.body['detail'] ?? 'Please try again.');
    }
  } finally {
    setState(() { isStarting = false; });
  }
}
```

---

### Screen 3 — Session Chat Screen

This screen is used for both **starting a new session** and **resuming an active one**.

**On screen mount:**
1. Show loading spinner in message area.
2. Call `GET /api/v1/sessions/{session_id}/messages` to load prior messages.
3. Hide spinner, display messages.
4. Start the countdown timer from `time_remaining_seconds`.
5. Check the last message — if `message_type == "session_ended"`, disable input immediately.

**Sending a message:**
1. User types and taps Send.
2. Immediately add user message to the list (optimistic update).
3. Disable the Send button and show a "thinking..." bubble.
4. Open SSE stream to `POST /api/v1/sessions/{session_id}/message/stream`.
5. Handle SSE events:

```dart
Future<void> _sendMessage(String text) async {
  // 1. Optimistic user message
  setState(() {
    messages.add(Message(role: 'user', content: text));
    inputEnabled = false;
    showThinkingBubble = true;
  });

  String accumulated = '';

  try {
    final stream = api.streamPost(
      '/sessions/${widget.sessionId}/message/stream',
      body: {'message': text},
    );

    await for (final event in stream) {
      switch (event['event']) {
        case 'status':
          // Update thinking/tool_use indicator
          setState(() { statusText = event['data']; }); // "thinking" or "tool_use"
          break;

        case 'token':
          // Stream text token by token into live bubble
          accumulated += event['data'] as String;
          setState(() { liveResponse = accumulated; });
          break;

        case 'done':
          final messageType = event['message_type'] as String;
          final response = event['response'] as String;

          // Replace live bubble with final message
          setState(() {
            showThinkingBubble = false;
            liveResponse = '';
            messages.add(Message(
              role: 'assistant',
              content: response,
              messageType: messageType,
              options: event['options'] != null
                  ? List<String>.from(event['options']) : null,
            ));
          });

          // Apply message_type specific UI
          if (messageType == 'options') {
            final options = List<String>.from(event['options'] as List);
            setState(() { activeOptions = options; });

          } else if (messageType == 'session_ended') {
            setState(() { inputEnabled = false; showGoHomeButton = true; });
          }

          // Re-enable input (unless session ended)
          if (messageType != 'session_ended') {
            setState(() { inputEnabled = true; });
          }
          break;

        case 'error':
          setState(() {
            showThinkingBubble = false;
            inputEnabled = true;
          });
          showSnackBar('Something went wrong. Please try again.');
          break;
      }
    }
  } catch (e) {
    setState(() {
      showThinkingBubble = false;
      inputEnabled = true;
    });
    showSnackBar('Connection error. Please try again.');
  }
}
```

**Session screen layout (active):**
```
┌──────────────────────────────────────────┐
│  ← Back    Foundation Session   23:45:10 │  ← live countdown
├──────────────────────────────────────────┤
│                                          │
│  [Loading spinner]                       │  ← while fetching history
│                                          │
│  ──── or, when loaded ────               │
│                                          │
│     AI: Hello! Let's begin.              │
│     You: I've been struggling...         │
│     AI: ●●● (thinking...)               │  ← while waiting
│                                          │
├──────────────────────────────────────────┤
│  [ Type your message...      ] [Send 🚫] │  ← disabled while AI responding
└──────────────────────────────────────────┘

**After AI response completes (`done` event received):**
```
┌──────────────────────────────────────────┐
│  ← Back    Foundation Session   18:45:22 │
├──────────────────────────────────────────┤
│     AI: Hello! Let's begin.              │
│     You: I've been struggling...         │
│     AI: That sounds really difficult...  │
│                                          │
├──────────────────────────────────────────┤
│  [ Type your message...         ] [Send] │  ← re-enabled after done event
└──────────────────────────────────────────┘
```

**Session screen layout (ended):**
```
┌──────────────────────────────────────────┐
│  ← Back    Foundation Session   00:00:00 │
├──────────────────────────────────────────┤
│     AI: Great work today! You've set     │
│         your health goals.               │
├──────────────────────────────────────────┤
│  [ Message input — DISABLED ]            │
│              [ Go to Home ]              │
└──────────────────────────────────────────┘
```

---

### Screen 4 — General Chat Screen

General chat is a free, always-available conversation with Kaya. No billing, no expiry, no active session lock.

**Thread list on mount:**
1. Show loading spinner.
2. Call `GET /api/v1/chat/threads` to list existing threads.
3. Display threads sorted by most recent. Each thread shows its `title` and `updated_at`.

**Opening a thread:**
1. Show loading spinner in message area.
2. Call `GET /api/v1/chat/threads/{thread_id}` for message history.
3. Display messages, enable input.

**Creating a new thread:**
1. Call `POST /api/v1/chat/threads` (returns `thread_id`).
2. Navigate to chat screen with that `thread_id`.
3. No history — start fresh.

**Sending in chat (same SSE flow as sessions):**
- Endpoint: `POST /api/v1/chat/threads/{thread_id}/message/stream`
- `status`, `token`, `done`, `error` events — identical to session streaming.
- `message_type` values: `text`, `options`, `session_suggestion` only.
- `session_ended` never appears in chat.

**Handling `session_suggestion` in chat:**
```dart
case 'done':
  final messageType = event['message_type'] as String;
  addAiMessage(event['response']);

  if (messageType == 'session_suggestion') {
    final sessionType = event['session_type'] as String;
    final isFree = event['is_free'] as bool;

    // Show a button below the AI message
    setState(() {
      suggestionCard = SessionSuggestionCard(
        sessionType: sessionType,
        isFree: isFree,
        onTap: () async {
          // Check for active session first before starting
          final active = await api.get('/sessions/active');
          if (active['active_session'] != null) {
            showDialog('Active session exists',
              'You already have an active ${active['active_session']['type']} session. Resume it first.');
          } else {
            Navigator.push(context, SessionStartScreen(type: sessionType));
          }
        },
      );
    });
  }
  break;
```

**Chat thread screen layout:**
```
┌──────────────────────────────────────────┐
│  ← Back         Kaya Chat           [+]  │  ← [+] creates new thread
├──────────────────────────────────────────┤
│                                          │
│  [Loading spinner]                       │  ← while fetching thread history
│                                          │
│  ──── or when loaded ────                │
│                                          │
│     You: How do I fix my sleep?          │
│     AI: Sleep is deeply connected to     │
│         your circadian rhythm...         │
│                                          │
│     AI: That's what Foundation sessions  │
│         are for. Want to start one?      │
│     [ Start Foundation Session   FREE ]  │  ← session_suggestion card
│                                          │
├──────────────────────────────────────────┤
│  [ Type your message...          ] [Send] │
└──────────────────────────────────────────┘
```

---

### End-to-End Flow Diagram

```
App Opens
│
├── Check auth token (local)
│   └── Not logged in → Login/Register screen
│
└── Logged in → Home Screen
    │
    ├── [LOADING] GET /me/home (single call)
    │
    ├── journey has active item
    │   └── Show "Resume [label]" button
    │       └── Tap → SessionChatScreen
    │           ├── [LOADING] GET /sessions/{id}/messages
    │           ├── Show countdown timer
    │           ├── Enable input
    │           └── Send messages (SSE stream)
    │               └── session_ended → disable input, show Go Home
    │                   └── Go Home → back to Home Screen (re-fetches /me/home)
    │
    └── no active + no paywall
        ├── Show "Start [label]" button (from next_session)
        │   └── Tap → [LOADING] POST /sessions/start
        │       └── Success → SessionChatScreen (fresh session, no history)
        │           └── (same flow as resume above)
        │
        └── Show "Open Chat" button (always available)
            └── Tap → Chat Thread List
                ├── [LOADING] GET /chat/threads
                ├── Select thread → [LOADING] GET /chat/threads/{id}
                │   └── ChatScreen → Send messages (SSE stream)
                │       └── session_suggestion → show Start Session card
                └── [+] New thread → POST /chat/threads → ChatScreen (empty)
```

---

### API Call Summary — What Each Screen Calls

| Screen | API calls | When |
|---|---|---|
| Home screen | `GET /me/home` | On mount |
| Start session | `POST /sessions/start` | On button tap |
| Session chat (new) | `POST /sessions/{id}/message/stream` | On each message send |
| Session chat (resume) | `GET /sessions/{id}/messages`, then stream | On mount, then on send |
| Chat thread list | `GET /chat/threads` | On mount |
| Chat thread open | `GET /chat/threads/{id}` | On mount |
| Create new chat | `POST /chat/threads` | On [+] tap |
| Chat send | `POST /chat/threads/{id}/message/stream` | On each message send |

> All endpoints require `Authorization: Bearer {firebase_id_token}` header. Refresh the token before it expires (Firebase tokens last 1 hour).
