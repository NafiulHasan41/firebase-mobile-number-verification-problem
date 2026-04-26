# Flutter — Required Fixes

---

## Duplicate Messages on Connection Drop

### Problem

When a user sends a message and the connection drops before Flutter receives a response, Flutter does not know if the server received and saved the message. So it retries — sends the same message again. The server has no way to detect it is the same message and saves it again. This results in the same message appearing multiple times in the chat history.

Example from live testing: the user sent a message twice after a connection drop, but the history showed it three times (one successful save + two retries both saved).

### Solution

Flutter needs to generate a UUID for each message when the user hits send. This UUID must be sent with the message as `client_message_id`. On every retry of the same message, Flutter must reuse the exact same UUID — not generate a new one.

The backend already handles the deduplication. If `client_message_id` is received and it matches a message already saved, the duplicate INSERT is silently ignored. Only the first save goes through.

**Request body (both session messages and general chat messages):**
```json
{
  "message": "I want to lose fat",
  "client_message_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Rules:**
- Generate the UUID when the user hits send — not when they start typing
- Store the UUID locally alongside the pending message
- Send the same UUID on every retry attempt for that message
- Once the server confirms success, clear the stored UUID — it is no longer needed
- If the user edits the message before retrying, treat it as a brand new message with a new UUID

**Endpoints this applies to:**
- `POST /api/v1/sessions/{session_id}/message`
- `POST /api/v1/sessions/{session_id}/message/stream`
- `POST /api/v1/chat/threads/{thread_id}/message`
- `POST /api/v1/chat/threads/{thread_id}/message/stream`

**Important:** `client_message_id` is optional — the backend accepts requests without it. Existing behavior is unchanged if Flutter does not send it. No breaking change.

---

## Feedback Popup Appears Before User Reads Final Message

### Problem

When a coaching session ends, the coach streams a closing message. Due to latency, the stream can pause mid-way. The user may think the message is finished and start typing a reply. The stream then resumes, the session ends, and the feedback popup appears immediately — covering the final message before the user has had a chance to read it.

### Solution

Delay showing the feedback popup after the session ends. Do not show it immediately on receipt of the `session_ended` event.

Recommended approach: wait 2–3 seconds after the final message has fully rendered before showing the popup. Alternatively, show it as a non-blocking banner at the bottom of the screen that the user can dismiss at their own pace — not as a modal that covers the chat.

The backend triggers the session end via the `done` SSE event with `message_type: "session_ended"`. Flutter should use this event as the signal to start the delay, not to show the popup immediately.

---

## Wrong Screen After Feedback Submission

### Problem

After the user submits their session feedback rating, the app navigates to the General Chat screen instead of the Home screen.

### Solution

After feedback submission completes successfully, navigate to the Home screen. Check the post-feedback completion navigation handler and update the route.
