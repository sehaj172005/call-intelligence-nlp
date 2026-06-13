# How Context Is Shared Back and Forth

This document explains exactly how **conversation history** and **context** move between the frontend, backend, and LLM in your project.

---

## Two Different Things Called "Context"

In your code there are two related but different concepts—and they use different data structures:

1. **Conversation history** — stored in **lists**. The list of all messages (user + assistant) so far.  
   This is what the LLM actually "sees" to know what was said.  
   Examples: `st.session_state.messages`, `conversation_history`, `messages_for_groq` — all are **lists** of message objects.

2. **Context update** — stored in **dictionaries**. A small dict from the LLM (e.g. `{"awaiting": "cable_check", "device": "printer"}`) describing *state* for the next turn (e.g. what the bot is waiting for).  
   This is optional metadata; the backend returns it and the frontend stores it in a **dict** (e.g. `st.session_state.conversation_context`).

**Summary:** Context is saved in **lists** (conversation history) and **dictionaries** (context update / state).

---

## 1. Conversation history (shared every time)

This is the main way "context" is shared **back and forth**.

### Frontend → Backend (every request)

**Where:** `frontend/streamlit_app.py` — `process_user_input()`

1. The frontend keeps the full chat in **`st.session_state.messages`** (list of user and assistant messages with `role`, `content`, `response_content`, etc.).

2. Before calling the API, it builds **`history_for_backend`** (a list):
   - Loops over `st.session_state.messages`.
   - For each **user** message: appends a **dictionary** `{"role": "user", "content": msg["content"]}` (key-value: `role`, `content`).
   - For each **assistant** message: appends a **dictionary** `{"role": "assistant", "content": msg["response_content"]}` (key-value: `role`, `content`).
   - So the list is a list of dicts, e.g. `[ {"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}, ... ]`.

3. It sends that list in the POST body as **`conversation_history`**:
   ```text
   POST /l1_automation/chat
   Body: { "message": "...", "user_id": "...", "conversation_history": [ ... ] }
   ```

So: **every time the user sends a message, the frontend sends the full conversation history to the backend.**

### Backend → LLM (every request)

**Where:** `app/routes/chatbot.py` and `app/services/nlp_service.py`

1. The route receives **`request.conversation_history`** (same list).

2. It passes it into **`nlp_service.get_generative_response(..., request.conversation_history)`**.

3. Inside **`get_generative_response()`**:
   - It builds **`messages_for_groq`** = system prompt + **conversation history**.
   - For each entry in `conversation_history`:
     - If `role == "user"` and has `content` → append `{"role": "user", "content": entry["content"]}`.
     - If `role == "assistant"` and has `response_content` → append `{"role": "assistant", "content": entry["response_content"]}`.
   - It sends **`messages_for_groq`** to the Groq API.

So: **the backend turns conversation_history into the message list for the LLM. The LLM’s “context” is this full thread of messages.**

### Summary for conversation history

```text
Frontend (st.session_state.messages)
    → build history_for_backend (role + content only)
    → POST conversation_history
Backend (request.conversation_history)
    → get_generative_response(..., conversation_history)
    → build messages_for_groq
    → Groq API (LLM sees full thread)
```

So **conversation history is shared: Frontend → Backend → LLM on every request.** The LLM does not have memory; it gets the full history in the payload each time.

---

## 2. Context update (backend → frontend only)

This is the small **state** dict the LLM can return for the next turn.

### LLM → Backend → Frontend (every response)

**Where:** `app/services/nlp_service.py` and `app/routes/chatbot.py` and `frontend/streamlit_app.py`

1. In **`get_generative_response()`**, the system prompt asks the LLM to return JSON that includes:
   - **`context_update`** (optional dict), e.g. `{"awaiting": "device_model", "troubleshooting_stage": "network_check"}`.

2. The service parses the LLM response and reads:
   - `new_context = parsed_response.get("context_update", {})`
   and returns it as the 4th value of the tuple.

3. The route puts that into the HTTP response as **`context`**:
   - `ChatResponse(..., context=new_context_state, ...)`.

4. The frontend receives **`chat_data["context"]`** and:
   - Stores it in **`st.session_state.conversation_context`**.
   - Also stores a copy in the new assistant message: `"context": new_context_from_backend`.

So: **context_update flows LLM → backend → frontend and is stored in session state (and in the last assistant message).**

### Does context_update go back to the backend?

**In the current code: no.** The next request body only contains:

- `message`
- `user_id`
- `is_voice_input` / `simulated_voice_text`
- **`conversation_history`**

There is no field like `"context": st.session_state.conversation_context` in the payload. So:

- The **backend** never receives the previous turn’s `context_update`.
- The **LLM** only gets context implicitly via the **conversation_history** (the text of the messages). It does not get the structured `context_update` from the previous turn.

So **context_update is “shared” only in one direction: backend → frontend.** It’s useful for the frontend to know “what the bot is waiting for” (e.g. for UI or future features). If you later want the LLM to see the previous `context_update`, you’d add something like `"context": st.session_state.conversation_context` to the request and then inject it into the system or user prompt.

---

## End-to-end flow (one round-trip)

```text
1. User sends message "Yes, cables are secure."

2. Frontend
   - Appends user message to st.session_state.messages.
   - Builds history_for_backend from st.session_state.messages (role + content).
   - POST /l1_automation/chat { message, conversation_history, ... }.
   - Does NOT send st.session_state.conversation_context.

3. Backend
   - Receives request.conversation_history (full thread).
   - Calls get_sentiment(message), get_intent_and_specialization(message).
   - Calls get_generative_response(..., request.conversation_history).
     - Builds messages_for_groq = [system, ...conversation_history].
     - Sends to Groq; LLM returns { reply, options, action_type, context_update }.
     - Returns (reply, ..., new_context, action_type).
   - Returns ChatResponse(..., context=new_context_state).

4. Frontend
   - Receives response; gets context from response["context"].
   - Appends assistant message to st.session_state.messages (including "context": new_context).
   - Sets st.session_state.conversation_context = new_context.
   - Next request will again send only conversation_history (full thread), not the stored context.
```

---

## Short answers for interviews

- **“How is context shared back and forth?”**  
  “The main context is **conversation history**. The frontend keeps all messages in session state, builds a list of role and content, and sends that list as `conversation_history` in every POST. The backend passes it to the LLM so the model sees the full thread. So history is shared frontend → backend → LLM on every request.”

- **“What about the context_update from the LLM?”**  
  “The LLM can return a small dict like `{\"awaiting\": \"cable_check\"}` in its JSON. The backend returns that as `context` in the response, and the frontend stores it in `st.session_state.conversation_context`. Right now we don’t send that stored context back to the backend in the next request—the backend only gets the conversation history. So that context is mainly for the frontend; the LLM’s context each turn is the conversation history we send.”

- **“Where is the source of truth for conversation context?”**  
  “On the frontend: `st.session_state.messages` is the source of truth for the full conversation. The backend is stateless; it only sees what we send in each request, which is the current message plus `conversation_history` built from that list.”
