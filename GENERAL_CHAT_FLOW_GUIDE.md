# General Chat — Flow Guide for Flutter Developer

> This document explains the correct general chat flow, covering all scenarios the client defined.
> It also explains the root cause of a current bug (new thread created on every app open) and how
> to fix it using the existing API — **no backend changes are needed**.

---

## The Root Cause Bug First

**Problem:** A new general chat thread is being created every time the app opens or the general chat
screen mounts. This means history fills with empty threads and the user can never return to their
previous conversation.

**Cause:** The frontend is calling `POST /api/v1/chat/threads` on screen mount.

**Fix:** Never call `POST /api/v1/chat/threads` on mount. Always call `GET /api/v1/chat/threads`
first and reuse the existing thread. Only create a new thread in two cases:
1. `GET /api/v1/chat/threads` returns an empty list (truly first time ever).
2. The user explicitly taps a **"New Chat"** button.

```
On general chat screen open:
  → GET /api/v1/chat/threads
  → if threads[] is empty  → POST /api/v1/chat/threads  → open new empty chat
  → if threads[] has items → open threads[0] directly    → this is the "active" chat
```

---

## What "Active General Chat" Means

There is no `active` boolean field on general chat threads, and none is needed.

`GET /api/v1/chat/threads` always returns threads sorted by `updated_at DESC` — the most recently
used thread is always `threads[0]`. That is the "active" thread. Every time the user sends a
message in a thread, `updated_at` is bumped, so it naturally floats to the top.

The frontend should treat `threads[0]` as the default thread to open — not create a new one.

---

## Determining App State at Screen Open

Before rendering anything, the frontend needs two pieces of information:

```
1. Is there an active coaching session?
   → GET /api/v1/me/home
   → check if journey[] has any item with status == "active"

2. What general chat threads exist?
   → GET /api/v1/chat/threads
   → threads[0] = last active thread (or empty list = first time)
```

These two calls together drive every scenario below.

---

## Scenario 1 — Active Coaching Session → Views Old Coaching Session from History

**Status:** Already done in app.

**Flow:**
- User is in an active coaching session.
- User navigates to chat history → taps on an old **coaching session**.
- Screen is **read-only**: user can only scroll through messages.
- Footer shows: **"Go to Home"** button + back arrow.
- Back arrow → returns to coaching session screen.
- "Go to Home" → navigates to Home screen.

**API used:** `GET /api/v1/sessions/{session_id}/messages` (read-only, no input).

---

## Scenario 2 — Active Coaching Session → Views Old General Chat from History

**Status:** Already done in app.

**Flow:**
- User is in an active coaching session.
- User navigates to chat history → taps on an old **general chat** thread.
- Screen is **read-only**: user can only scroll through messages.
- Footer shows a note: **"Please complete your current session first"** (no input, no buttons).
- Back arrow only → returns to coaching session screen.

**Note:** The backend also blocks `POST /api/v1/chat/threads` while a coaching session is active
(returns `400 Cannot create a chat thread while a coaching session is active`). This is already
enforced at the API level.

---

## Scenario 3 — No Active Session → General Chat (Empty) → Views Old Coaching Session

**Flow:**
1. No active coaching session exists.
2. User opens general chat → `GET /api/v1/chat/threads` returns empty list (first time ever or
   first time after coaching session completed).
3. Show an **empty chat screen** with an input field — user can start a new conversation.
   - At this point do NOT create a thread yet. Create it only when the user sends the first message.
4. User navigates to chat history → taps on an old **coaching session**.
5. Screen is **read-only**.
6. Footer shows: **"Go to Home"** button + back arrow.
7. Back arrow → returns to the empty general chat screen.
8. "Go to Home" → navigates to Home screen.

**When to create a thread for the empty chat:**
Create the thread (`POST /api/v1/chat/threads`) lazily — only when the user actually sends their
first message, not when the screen opens.

---

## Scenario 4 — No Active Session → General Chat (Empty) → Views Old General Chat

**Flow:**
1. No active coaching session exists.
2. User opens general chat → `GET /api/v1/chat/threads` returns empty list.
3. Show an **empty chat screen** — user can start a new conversation.
4. User navigates to chat history → taps on an old **general chat** thread.
5. Screen shows that thread's messages.
6. User has three options:
   - **Continue this chat** button → activate this thread as the current chat, enable input,
     send messages to this `thread_id`.
   - **Go to Home** button → navigate to Home screen.
   - **Back arrow** → return to the empty general chat screen (the user's new conversation
     entry point is still there).
7. If user taps "Continue this chat" and sends a message → `updated_at` on that thread is bumped
   → it becomes `threads[0]` on the next `GET /api/v1/chat/threads` call → it is now the
   "active" thread naturally.

---

## Scenario 5 — No Active Session → General Chat Has a Previous Thread → Full Options

This is the main general chat experience after the user has used it before.

**Flow:**
1. No active coaching session exists.
2. User opens general chat → `GET /api/v1/chat/threads` returns `threads[0]` (last active thread).
3. Show the **last active chat screen** (`threads[0]`) with its messages loaded and input enabled.
4. User has three options visible on this screen:
   - **(a) Continue the conversation** — input is already enabled, user just types and sends.
   - **(b) View history** — taps history icon → see list of all threads from
     `GET /api/v1/chat/threads`.
   - **(c) Start a new chat** — taps "New Chat" button → `POST /api/v1/chat/threads` →
     navigate to new empty chat screen.

5. From history, user taps on an **old general chat** thread (not `threads[0]`):
   - Screen shows that thread's messages.
   - User has three options:
     - **Continue this chat** button → enable input, send to this `thread_id` → its `updated_at`
       gets bumped → it becomes the new `threads[0]` (new "active" thread).
     - **Go to Home** button → navigate to Home screen.
     - **Back arrow** → return to the previous screen, which was `threads[0]` (the last active
       thread the user was on before going to history).

---

## Navigation Stack Summary

The Flutter navigation stack handles all "back arrow goes to X" behaviour automatically — no
special logic needed, just push/pop correctly.

| User action | Push onto stack |
|---|---|
| Open general chat | GeneralChatScreen(threads[0]) |
| Tap "View History" | ChatHistoryScreen |
| Tap a thread in history | ThreadViewScreen(thread_id) |
| Tap "Back" from ThreadViewScreen | pop → back to ChatHistoryScreen |
| Tap "Back" from ChatHistoryScreen | pop → back to GeneralChatScreen(threads[0]) |
| Tap "Continue this chat" | pop history → replace GeneralChatScreen with this thread_id |

---

## API Call Reference Per Scenario

| Action | API Call |
|---|---|
| Check active coaching session | `GET /api/v1/me/home` — check `journey[].status == "active"` |
| Get last active general chat | `GET /api/v1/chat/threads` → use `threads[0]` |
| Get all threads for history list | `GET /api/v1/chat/threads` (same call, use full list) |
| Load messages in a thread | `GET /api/v1/chat/threads/{thread_id}` |
| Load messages in a coaching session | `GET /api/v1/sessions/{session_id}/messages` |
| Send message in general chat | `POST /api/v1/chat/threads/{thread_id}/message/stream` |
| Create a new chat (user explicit tap) | `POST /api/v1/chat/threads` |

---

## Decision Tree — General Chat Screen on Mount

```
GET /api/v1/chat/threads
│
├── threads[] is empty
│   └── Show empty chat screen
│       User types first message
│       → POST /api/v1/chat/threads (create thread now)
│       → POST /api/v1/chat/threads/{new_id}/message/stream (send message)
│
└── threads[] has items
    └── Open threads[0] as the active chat (load messages, enable input)
        │
        ├── User sends message → POST .../threads[0].thread_id/message/stream
        │
        ├── User taps "View History" → push ChatHistoryScreen
        │   └── User taps old thread → push ThreadViewScreen
        │       ├── "Continue" → pop history, open that thread as active
        │       ├── "Go to Home" → pop all, push HomeScreen
        │       └── Back arrow → pop → back to ChatHistoryScreen
        │
        └── User taps "New Chat" → POST /api/v1/chat/threads → push new empty chat
```

