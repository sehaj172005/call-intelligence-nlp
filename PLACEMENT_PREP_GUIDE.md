# Placement-Focused Study Guide: Unified AI Customer Care System

**Target:** Final-year student | Placement interviews | Limited time  
**Goal:** Defend this project confidently, use terms correctly, answer "why" without bluffing.

---

## 1. CORE CONCEPTS I MUST MASTER (NON-NEGOTIABLE)

These are the concepts interviewers **will** ask about. Know them at an "I can explain in 2–3 lines" level.

---

### **REST API**
A way for frontend and backend to talk over HTTP. Client sends a **request** (e.g. POST with JSON body); server sends a **response** (e.g. 200 with JSON). In your project: Streamlit sends POST to FastAPI; FastAPI returns JSON.  
**Interview line:** "REST is a style of building APIs over HTTP. My frontend sends POST requests with JSON; the backend returns JSON. No session stored on server—each request is self-contained."

---

### **Client–Server Architecture**
**Client** (Streamlit in browser) asks for something; **Server** (FastAPI on a machine) does the work and replies. Your app: user interacts with Streamlit (client) → Streamlit calls FastAPI (server) → FastAPI calls Groq API (another server).  
**Interview line:** "The frontend is the client that the user sees; the backend is the server that does the logic and talks to external APIs like Groq."

---

### **Service Layer Pattern**
Business logic lives in **services** (e.g. `nlp_service.py`); **routes** only handle HTTP (receive request, validate, call service, return response). Routes don’t contain "how to get sentiment"—they just call `nlp_service.get_sentiment()`.  
**Interview line:** "Routes handle HTTP and validation; services contain the actual logic. This keeps the code testable and lets multiple routes reuse the same service."

---

### **Stateless Backend**
The server doesn’t store conversation or user state between requests. All context (e.g. conversation history) is sent **in the request** each time.  
**Interview line:** "My backend doesn’t store sessions. The frontend sends conversation history in every request, so the server can scale horizontally without shared state."

---

### **Sentiment Analysis**
Determining the **emotional tone** of text (e.g. POSITIVE, NEGATIVE, NEUTRAL, URGENT) and often a **confidence score**. In your project: you send user message to Groq with a prompt; Groq returns a label and score.  
**Interview line:** "I use the LLM to classify the user’s emotional tone and a confidence score so the bot can respond more empathetically and escalation logic can use it."

---

### **Intent Detection**
Identifying **what the user wants** (e.g. technical_support_hardware, billing_query, escalation_request), not just how they feel. You do this via a separate LLM call with a fixed set of intents.  
**Interview line:** "Intent detection tells us the user’s goal—e.g. printer help vs billing—so we can route to the right logic and use the right system prompt for the bot."

---

### **Prompt Engineering**
Designing the **exact text** (prompt) you send to the LLM so it returns the format and content you need (e.g. JSON with specific keys). Your project: you write prompts that ask for JSON and give examples.  
**Interview line:** "I design prompts with clear instructions and JSON examples so the model returns structured output. I also use Groq’s JSON mode to enforce valid JSON."

---

### **Escalation (in support)**
When the bot **hands off** the conversation to a **human agent**. In your project: `predict_escalation()` uses rules (e.g. action_type == "escalate", or high negative sentiment and bot not actively helping).  
**Interview line:** "Escalation means the bot stops and a human takes over. I use rules: explicit user request or safety keywords always escalate; high negative sentiment only escalates if the bot isn’t actively troubleshooting."

---

### **Conversation Context / Multi-turn**
Keeping the **history** of the chat so the bot can refer to earlier messages. You do this by sending `conversation_history` (list of role + content) in every request and storing `context_update` from the LLM.  
**Interview line:** "I send the full conversation history to the backend with each message, and the LLM returns a context_update dict. The frontend stores that and sends it again so the next turn knows what we’re waiting for."

---

### **Request / Response (HTTP)**
**Request:** method (GET/POST), URL, headers, body (e.g. JSON). **Response:** status code (200, 404, 500), body (e.g. JSON). Your chat: POST `/l1_automation/chat` with JSON body → 200 with JSON body.  
**Interview line:** "The frontend sends a POST with a JSON body; the backend validates it, runs the pipeline, and returns a JSON response with the bot reply and metadata."

---

### **Validation (Pydantic)**
Checking that incoming data has the right **types and shape** (e.g. `message` is string, `conversation_history` is list of dicts). Pydantic does this automatically from your model definitions; invalid data → 422.  
**Interview line:** "I use Pydantic models for request bodies. If the client sends wrong types or missing fields, FastAPI returns 422 before my code runs."

---

### **Error Handling & Fallbacks**
When something fails (e.g. LLM returns invalid JSON), you **catch** the error, log it, and return a **safe default** so the app doesn’t crash. Example: `JSONDecodeError` → return `{"label": "ERROR_PARSE", "score": 0.0}`.  
**Interview line:** "I wrap LLM response parsing in try-except. On parse failure I log the raw response and return fallback values so the rest of the pipeline can still run."

---

## 2. TECHNOLOGIES / TOOLS TO LEARN OR REVISE

For each: **what it is**, **why it’s in your project**, **how deep you need to go**.

---

| Item | What it is | Why it's in your project | Depth |
|------|------------|---------------------------|--------|
| **FastAPI** | A Python web framework for building APIs (routes, validation, async). | Backend that exposes REST endpoints and calls the NLP service. | **Working:** Define a route, use Pydantic request/response, call a service. Know: `APIRouter`, `@router.post`, status codes. |
| **Streamlit** | A Python library to build web UIs with widgets (buttons, text inputs, tabs) without HTML/JS. | Frontend: chat UI, grievance form, call analysis tab; all call your backend via `requests`. | **Working:** `st.session_state`, `st.tabs`, `st.form`, `requests.post`, display response. |
| **Pydantic** | Library for data validation using type hints (e.g. `class ChatRequest(BaseModel): message: str`). | Request/response models so FastAPI validates JSON and gives 422 on invalid input. | **Working:** Define a `BaseModel` with fields and types; use it as route body and response_model. |
| **Groq API** | External API that runs LLMs (you use Llama3-8b). Called with OpenAI-compatible client. | All NLP: sentiment, intent, reply generation, summarization, Q&A. | **Basic:** It’s an external API; you send prompts and get text/JSON. Know: API key, base URL, model name, JSON mode. No need to know Groq’s internals. |
| **OpenAI Python client** | `openai` package used with `api_key` and `base_url` to call any OpenAI-compatible API (e.g. Groq). | You use it to call Groq: `client.chat.completions.create(model=..., messages=...)`. | **Basic:** Create client with key and base_url; call `chat.completions.create` with `messages` and optional `response_format`. |
| **Uvicorn** | ASGI server that runs FastAPI (like a “runner” for your app). | You run the backend with `uvicorn main:app --host 0.0.0.0 --port 8000`. | **Basic:** Command to run the app; no need to go deep. |
| **Python `requests`** | Library to send HTTP requests from Python (e.g. POST with JSON). | Streamlit uses `requests.post(BACKEND_URL + "/l1_automation/chat", json=payload)` to call your API. | **Working:** `requests.post(url, json=dict, timeout=...)` and `response.json()`, `response.raise_for_status()`. |
| **JSON** | Text format for data (key-value pairs, arrays). Used in API bodies and LLM responses. | All API payloads and LLM outputs are JSON; you parse with `json.loads()`. | **Working:** Read/write JSON, handle `JSONDecodeError`, know structure of your request/response. |
| **Logging** | Python’s `logging` module to write messages (info, error) to console/file. | You log request info, errors, and raw LLM responses for debugging. | **Basic:** `logging.info()`, `logging.error()`, and that it helps in debugging. |

---

## 3. TERMINOLOGY BREAKDOWN

Crisp definitions + likely follow-ups. Use these in answers; don’t guess beyond this.

---

| Term | Interview-friendly definition | Common follow-up |
|------|-------------------------------|-------------------|
| **L1 (Level 1) support** | First line of support: basic, repetitive queries (password reset, status check, simple troubleshooting). L2/L3 = more complex or specialized. | "L1 is the first line; we automate it so humans handle harder cases." |
| **NLP (Natural Language Processing)** | Making computers understand/generate human language. In your project: sentiment, intent, summarization, Q&A via an LLM. | "I use NLP for sentiment, intent, and generating replies—all through the LLM." |
| **LLM (Large Language Model)** | A model that takes text in and produces text out (e.g. Llama). You use it via Groq’s API, not by training it. | "I use Groq’s Llama3-8b via API for all text understanding and generation." |
| **Endpoint** | A specific URL + method (e.g. POST `/l1_automation/chat`) that the server handles. | "My main endpoints are POST /l1_automation/chat, POST /grievance_management/grievance, POST /call_intelligence/call_nlp and ask_question." |
| **Payload** | The body of an HTTP request (e.g. JSON sent in POST). | "The payload for chat is message, user_id, conversation_history, is_voice_input." |
| **Session state** | Data the frontend keeps **in memory** for the current user/browser session (e.g. chat history). In Streamlit: `st.session_state`. | "I keep messages and conversation_context in session state so the next request can send full history." |
| **Pipeline** | A fixed sequence of steps (e.g. sentiment → intent → response → escalation). | "My chat pipeline is: sentiment, then intent, then generative response, then escalation check." |
| **Structured output** | LLM output in a fixed format (e.g. JSON with keys like label, score) so your code can parse it. | "I ask for JSON with specific keys and use Groq’s JSON mode so I get structured output." |
| **Horizontal scaling** | Adding more **servers** (instances) to handle more load, instead of making one server bigger. | "My backend is stateless, so I can run multiple FastAPI instances behind a load balancer." |
| **Route handler** | The function that runs when a request hits an endpoint (e.g. `chat_with_bot` for POST `/l1_automation/chat`). | "The route handler validates the request, calls the NLP service, and returns the response." |
| **Async** | Code that can wait for I/O (e.g. API call) without blocking other requests. FastAPI supports async; you use it for concurrency. | "FastAPI supports async so multiple requests can be in flight while waiting for Groq." |
| **Context (conversation)** | Information about the current state of the dialogue (e.g. what the bot is waiting for, last topic). You pass history + context_update. | "Context is what we’re waiting for and the current topic; we get it from the LLM’s context_update and store it in session state." |
| **Entity extraction** | Pulling out specific things from text (e.g. order ID, product name). In your project: tags and entities from call transcripts. | "In call intelligence I use the LLM to extract tags and entities from the transcript." |
| **Escalation** | Handing the conversation from bot to human. | "Escalation is when we stop the bot and hand off to a human; I use rules based on action_type and sentiment." |
| **Fallback** | A default value or path when something fails (e.g. default sentiment label when JSON parse fails). | "If JSON parsing fails I use fallbacks like ERROR_PARSE so the rest of the flow still runs." |

---

## 4. "WHY" QUESTIONS I MUST PREPARE FOR

Short, honest answers you can give in an interview.

---

### **Why this architecture? (client–server + service layer)**

**Answer:**  
"I wanted a clear split: frontend for UI and user input, backend for logic and external APIs. The service layer keeps route handlers thin—they only validate and call services—so business logic is in one place and can be reused by multiple endpoints. It also makes testing easier: I can test the NLP service without HTTP."

---

### **Why FastAPI and not Flask/Django?**

**Answer:**  
"FastAPI has built-in async support and automatic request/response validation with Pydantic, and it generates OpenAPI docs. I needed to call an external API (Groq) for every request, so async helps with concurrency. Validation and docs came for free, which saved time."

---

### **Why Streamlit and not React/Vue?**

**Answer:**  
"For a hackathon/demo, Streamlit was faster to build with—pure Python, no separate frontend. I could focus on the backend and NLP. For a production product with many users, I’d consider React or Vue for better control and scalability."

---

### **Why Groq and not OpenAI directly?**

**Answer:**  
"Groq offered a good balance of cost and speed for the hackathon, and it’s OpenAI-compatible so I could use the same client and switch provider later if needed. I didn’t need to host or manage the model myself."

---

### **Why rule-based escalation and not ML?**

**Answer:**  
"Rules are interpretable and easy to tune—I can say exactly when we escalate (e.g. action_type is escalate, or high negative sentiment and bot not helping). I didn’t have labeled data to train an escalation model, and for a demo, rules were sufficient and debuggable."

---

### **Why send full conversation history every time?**

**Answer:**  
"The backend is stateless, so it doesn’t store past messages. Sending history in each request keeps the server simple and scalable—no shared state. The trade-off is a larger payload; for production I might trim to the last N messages or summarize older turns."

---

### **Why not use a database?**

**Answer:**  
"For the scope of the project, I kept it stateless: frontend holds session state, and no persistence was required for the demo. For production I’d add a database to store conversations, grievances, and call analyses for analytics and compliance."

---

## 5. MINIMUM STUDY CHECKLIST (TIME-BOUND)

Only what **directly** helps you defend this project in interviews.

---

### **If you have 1 week**

- [ ] **Project flow:** Draw the flow: User → Streamlit → FastAPI → Groq → back to user. Say it out loud.
- [ ] **Core concepts:** REST API, client–server, service layer, stateless, sentiment, intent, escalation, conversation context. Use the definitions in Section 1.
- [ ] **Your three challenges:** Structured JSON (prompts + JSON mode + try/except + fallbacks), context (history + context_update in session state), escalation (action_type + sentiment rules). Rehearse the "How did you do all this?" answer.
- [ ] **Tech depth:** FastAPI: define a route and a Pydantic model. Streamlit: session state and one POST request. Groq: one call with messages and response_format.
- [ ] **Terminology:** All terms in Section 3—define each in one sentence without looking.
- [ ] **"Why" answers:** Section 4—practice each in 30 seconds.

---

### **If you have 2 weeks**

- Do everything in the 1-week list, then:
- [ ] **Code-level:** Open `chatbot.py` and `nlp_service.py`. Trace one chat request from route → get_sentiment → get_intent_and_specialization → get_generative_response → predict_escalation. Say what each returns.
- [ ] **Error handling:** Where you use try/except, what you log, what fallback you return. Give one example (e.g. sentiment JSON parse fail).
- [ ] **Data structures:** What you send in the request (message, conversation_history shape). What you get back (response, sentiment, options, context, action_type).
- [ ] **Alternatives:** Why not Flask? Why not store state on server? Why not ML for escalation? Use Section 4.
- [ ] **One scaling idea:** "Add more FastAPI instances behind a load balancer; backend is stateless."
- [ ] **One security gap:** "No auth yet; in production I’d add JWT or similar."

---

### **If you have 1 month**

- Do everything in the 2-week list, then:
- [ ] **Run and test:** Run backend and frontend; send a few chats; see logs. Relate what you see to the pipeline.
- [ ] **Read all routes:** What each endpoint expects and returns (request/response models).
- [ ] **Read key service methods:** get_sentiment, get_intent_and_specialization, get_generative_response, predict_escalation. One-sentence purpose of each.
- [ ] **Scaling (high-level):** Load balancer, multiple instances, Redis for cache, queue for API calls. No need to implement.
- [ ] **Testing (high-level):** "I’d unit test the NLP service with mocked Groq calls, and integration test the endpoints with a test client."
- [ ] **Improvements:** Database for persistence, auth, rate limiting, parallelize sentiment + intent. Be ready to name 2–3 and why.

---

## 6. RED FLAGS TO AVOID IN INTERVIEWS

- **Don’t say it if you can’t explain it:** e.g. "microservices," "Kafka," "Kubernetes," "vector DB," "embedding," "fine-tuning," "RAG," "LangChain" (unless you actually used it and can explain). Stick to what your project does: REST API, FastAPI, Streamlit, Groq API, prompts, JSON, rules.
- **Don’t over-claim:**  
  - Don’t say "I built an ML model" — you use an existing LLM via API.  
  - Don’t say "production-ready" — say "demo/hackathon scope; I’d add X and Y for production."  
  - Don’t say "we achieved 90%+ accuracy" as if you measured it rigorously — say "the README mentions 90%+ as a target; I didn’t run a formal evaluation."
- **Don’t badmouth tools:** Don’t say "Streamlit is bad." Say "For this scope Streamlit was practical; for a different scope I’d consider other options."
- **Don’t pretend you did what you didn’t:** No database, no auth, no Kubernetes, no custom model training — say so if asked. Then say what you’d add next.

---

## 7. CONFIDENCE TEST — 10 RAPID-FIRE QUESTIONS

If you can answer these clearly and correctly, you’re in good shape to defend this project.

---

1. **What is the role of the service layer in your project?**  
   *Expected: Routes handle HTTP and validation; services contain business logic (e.g. NLP). Reusable and easier to test.*

2. **What do you send to the backend when the user sends a chat message?**  
   *Expected: Message, user_id, conversation_history (list of role + content), is_voice_input (and simulated_voice_text if voice).*

3. **What are the four main steps in your chat pipeline (in order)?**  
   *Expected: Sentiment analysis → Intent (and specialization) detection → Generative response → Escalation prediction.*

4. **How do you get structured JSON from the LLM?**  
   *Expected: Prompts with explicit JSON examples + Groq’s JSON mode (response_format). Plus try/except and fallbacks when parsing fails.*

5. **Where is conversation history stored, and who sends it to the backend?**  
   *Expected: Stored in Streamlit session state (e.g. messages). Frontend builds conversation_history from it and sends it in every POST.*

6. **When do you escalate to a human?**  
   *Expected: When action_type is "escalate" (e.g. user asked for human), or when sentiment is highly negative/urgent and the bot is not actively helping (e.g. not troubleshoot/provide_solution/ask_for_info).*

7. **What does “stateless backend” mean in your project?**  
   *Expected: The server doesn’t store conversation or user state between requests. Context is sent in the request each time.*

8. **What is Pydantic used for in your API?**  
   *Expected: To define request/response models and validate incoming JSON. Invalid data gets 422 before my code runs.*

9. **Name one place where you use a fallback when something fails.**  
   *Expected: e.g. When parsing LLM JSON fails (e.g. sentiment), we catch JSONDecodeError and return a safe default like ERROR_PARSE / 0.0 so the pipeline can continue.*

10. **What is the difference between sentiment and intent in your project?**  
    *Expected: Sentiment = emotional tone (positive/negative/neutral/urgent) and score. Intent = what the user wants (e.g. technical_support_hardware, billing_query). Both come from separate LLM calls and are used for response and escalation.*

---

**If you can answer 8–10 of these without hesitation, you’re interview-ready for this project.**  
If 5–7: revise Section 1 and 3 and the "How did you do all this?" answer.  
If &lt;5: go through the 1-week checklist and the project flow again, then retry the test.
