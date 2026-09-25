# AI Conversation Coach

A context-aware AI agent that uses live transcription, conversation memory, and LLM reasoning to generate relevant follow-up questions and communication feedback — self-hosted end-to-end on a free cloud VPS. Instead of relying on predefined question trees, the agent evaluates the conversation context and determines what response, clarification, or follow-up best fits the discussion.

## The Problem

Interview and conversation practice is often difficult to make realistic and repeatable. A fixed list of questions can provide practice, but it does not necessarily respond to what the person actually says or identify what should be explored next.

A useful practice system needs to do more than generate another question. It should understand the current conversation, maintain context across multiple exchanges, recognize when an answer is incomplete or vague, respond appropriately, and provide specific feedback when feedback is useful.

This project explores how an AI agent can provide that experience — using live transcription, persistent conversation memory, topic tracking, and LLM reasoning to determine what response or follow-up best fits the conversation in real time.

## The Solution

A browser-based voice agent, backed by an always-on n8n workflow, that:
1. Listens through the browser's microphone (built-in Web Speech API where available, falling back to server-side Whisper transcription where it isn't) and transcribes speech in real time
2. Sends the transcript, conversation mode, and topic to an AI Agent with persistent memory of the session
3. Generates a reply that both responds to what was actually said (feedback, when useful) and asks a natural follow-up question — never a generic next question
4. Speaks the reply back out loud via text-to-speech
5. Independently scores every generated reply on six quality dimensions and logs the score with reasoning, separate from the conversation itself

Two modes are supported: **Interview Mode** (behavioral/technical questions, project deep-dives, notices when an answer explains *what* was built but not *why* it mattered) and **Conversation Mode** (casual follow-ups, notices short or vague answers and invites expansion, gives feedback only when it's actually useful).

## Architecture

```
Browser (mic + speaker)
   │  POST { sessionId, mode, topic, message }
   ▼
Webhook
   ▼
Get Topics (Postgres SELECT, COALESCE fallback for a brand-new session)
   ▼
Combine Fields (merges original webhook fields + topics into one flat item)
   ▼
AI Agent  ──ai_languageModel──▶ Gemini Chat Model (primary)
   │       ──ai_languageModel──▶ Groq Chat Model (fallback, on Gemini error/rate limit)
   │       ──ai_memory────────▶ Postgres Chat Memory (session-scoped, own database)
   │       (system message includes the topics from Combine Fields)
   ▼
Edit Fields (isolates the reply text)
   ├──▶ Respond to Webhook  ──▶ back to browser ──▶ spoken via TTS
   ├──▶ Evaluator (LLM Chain) ──▶ Parse Evaluation (JSON.parse) ──▶ Postgres INSERT
   │                                                                  (question_evaluations table)
   └──▶ Topic Summarizer (LLM Chain) ──▶ Save Topics (Postgres UPSERT)
                                                        (session_topics table)
```

### Key design decisions

**Fallback Chat Model instead of a single provider.** Google Gemini's free tier rate-limits aggressively under repeated testing. Rather than the whole agent failing when that happens, the AI Agent node has a second, independent Chat Model (Groq, running an open-weight model) wired in as a fallback — n8n automatically retries against it if the primary call errors. This also means the two providers' rate limits are independent, so a demo session is unlikely to be blocked by either alone.

**A dedicated Postgres database, not n8n's own internal one.** n8n already runs Postgres for its own operational data (from the earlier supplier-routing project's stack). Rather than mixing this project's chat history and evaluation scores into that database, a separate `voice_agent` database was created — same instance, no new service, but cleanly isolated data.

**"First Incoming Item" instead of hand-typed JSON in Respond to Webhook.** The first approach — a manually typed `{{ {"reply": $json.output} }}` expression as the response body — broke the moment the generated text contained an apostrophe or quote that needed escaping, and was fragile to debug because the failure ("Invalid JSON in Response Body") gave no indication *which* character was the problem. Routing the reply through an Edit Fields node first, then setting Respond to Webhook to "First Incoming Item," lets n8n handle JSON serialization and escaping automatically — a pattern reused for the evaluation branch too.

**An explicit JSON.parse step for the Evaluator's output.** The Basic LLM Chain node used for scoring doesn't auto-parse a model's JSON-formatted response into separate fields — it returns the whole thing as one `text` string, even when the prompt explicitly requests JSON. The Postgres insert needs real fields (`relevance`, `contextual`, etc.), so a second Edit Fields node in JSON mode running `JSON.parse($json.text)` sits between the Evaluator and the database insert.

**Production webhook URL over the test URL for all real testing.** The Test URL only stays "armed" for a single call after clicking "Listen for test event" in the editor — any delay switching to a terminal caused silent failures. Once the workflow was saved and set Active, the Production URL removed the need for that arming step entirely and made every test call reliably logged in the Executions list.

**Topic tracking as a separate read/write pair, not baked into memory.** Postgres Chat Memory gives the agent raw conversation history, but raw history isn't the same as "what's been established" — the model still had to infer that from scratch each time. A `session_topics` table (one row per session, upserted after every turn) holds a running plain-text summary instead, read back at the start of the next turn via `COALESCE` (falling back to "nothing covered yet" for a brand-new session so the query always returns exactly one row) and written by a second, independent LLM call after each reply — the same branching pattern as the evaluator. This is also what let the "first message already has real content" bug get fixed cleanly: the topics field ("nothing covered yet" vs. an actual summary) became the real signal for "is this genuinely the start of the conversation," rather than guessing from message count alone.

**A field silently defaulting to "Fixed" instead of "Expression" was the single most repeated bug of this project.** It broke the Postgres Chat Memory node's session key, the original Respond to Webhook body, and later the Combine Fields node feeding topics into the Agent — three separate times, always the same symptom (a field showing the literal `{{ }}` text instead of a resolved value) and the same fix (explicitly clicking the Expression tab, not trusting that pasted text auto-detects as an expression). Worth checking first, before anything else, whenever downstream data looks like it isn't reaching where it should.

**Transcription fallback checked once, at page load, not per recording.** The front end checks whether `window.SpeechRecognition` exists a single time when the page loads, and both the Web Speech path and the `MediaRecorder` + Whisper path converge on the same `sendToCoach()` function afterward — everything downstream (talking to the agent, speaking the reply, saving to the transcript) is identical either way, so adding the fallback didn't require touching any of the existing reply-handling logic.

**Session persistence via localStorage, not a backend change.** The agent's Postgres-backed memory already persisted server-side; what was missing was the browser remembering which session it belonged to. Storing the session ID and a mirror of the visible transcript in `localStorage` (repopulated once on page load, before any new turn can be added) means a page refresh resumes the same conversation with the same visible history, with no changes to the n8n workflow at all.

## Stats Dashboard

A second, independent n8n workflow and a second static page (`stats.html`) that reads the scores the main workflow has been logging all along.

```
Browser (stats.html)
   │  GET
   ▼
Webhook (voice-agent-stats)
   ▼
Postgres (SELECT * FROM question_evaluations ORDER BY created_at)
   ▼
Respond to Webhook ("All Incoming Entries")
   ▼
Chart.js line chart — one line per scoring dimension, plotted over time
```

This workflow never talks to the main `Voice Coach Agent` workflow directly — the only connection between the two systems is that they both point at the same `question_evaluations` table, one writing to it, the other reading from it. The two front-end pages link to each other directly (a plain `<a href>` in each), separate from that shared-table connection.

**Why this exists, not just what it does:**
- **Turns a vague feeling into a real signal.** A conversation is ephemeral — once the tab closes, all that's left is an impression of how it went. The dashboard is the only place that shows whether scores like `naturalness` or `non_repetitive` are actually trending up with practice, or whether a particular mode is consistently weaker.
- **Closes the loop the evaluation layer was built for.** Scoring every reply is only useful if the scores are visible somewhere other than raw SQL output — the chart is what makes that data legible rather than technically-present-but-unused.
- **Evidence over claims, for a portfolio.** "I built an evaluation layer" is a sentence. A chart showing real variation across dozens of scored exchanges is something a reviewer can actually look at.
- **A diagnostic for the system, not just the user.** If a dimension like `grounded_in_answer` scores low consistently across many sessions, that points at the system message or model needing adjustment — separate from the question of whether the user themself is improving.

## Transcription Fallback

A third, independent n8n workflow, used only when a visitor's browser doesn't support the built-in Web Speech API.

```
Browser (MediaRecorder captures raw audio, only when Web Speech API is unavailable)
   │  POST audio file (multipart/form-data)
   ▼
Webhook (transcribe-audio)
   ▼
HTTP Request → Groq's /audio/transcriptions endpoint (whisper-large-v3)
   ▼
Edit Fields (isolates the transcript text)
   ▼
Respond to Webhook → { "transcript": "..." } → fed into the same sendToCoach() flow as Web Speech results
```

Checked once per page load, not per recording — `index.html` picks a code path at load time and never re-checks, so there's no branching logic scattered through the actual recording/sending code.

## Infrastructure

Shares the same self-hosted stack as the supplier-routing project — no new services added, only a new database and two new API credentials.

| Component | Choice | Why |
|---|---|---|
| Hosting | Oracle Cloud Always Free VPS | No time limit, no monthly cost, real dedicated server |
| Runtime | Docker Compose (n8n + Postgres + Caddy) | Isolated, reproducible, one-command redeploy |
| Database | Postgres, separate `voice_agent` database | Session memory (`n8n_chat_histories`) and evaluation scores (`question_evaluations`), isolated from n8n's own internal data |
| LLM (primary) | Google Gemini (free tier) | No cost; used for both the coaching agent and the evaluator |
| LLM (fallback) | Groq (free tier, open-weight model) | Independent rate limit from Gemini; keeps the agent responsive under load |
| Domain | Free DuckDNS subdomain | No ongoing cost |
| HTTPS | Caddy, automatic Let's Encrypt certificates | Browsers require HTTPS (or localhost) for microphone access, so this was a hard requirement, not just a nice-to-have |
| Front end | Two self-contained HTML pages (`index.html`, `stats.html`) | Web Speech API + Whisper (Groq) fallback for transcription/TTS; Chart.js (CDN) for the score dashboard — no build step or framework either way |

## Notes

Built and debugged from scratch, node by node, without importing a pre-built workflow — including working through DNS/domain mix-ups, Windows-specific TLS and terminal quirks, webhook test-mode timing, a deprecated LLM model ID, SQL reserved-keyword and JSON-escaping errors, and a parsing gap between the evaluator and its database insert. This was a deliberate choice: the goal was to come out of it able to maintain and debug an n8n workflow, not just have a working one.
