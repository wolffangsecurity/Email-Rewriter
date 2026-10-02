<img width="1280" height="720"  src="https://github.com/user-attachments/assets/5257bc03-fcce-418e-947c-caacbb37d852" />

# Polished Email Rewriter

Polished Email Rewriter is a web application that converts informal drafts into structured, professional emails using large language models. The system routes user text through a backend security proxy to OpenRouter or a local deterministic engine, enforcing tone constraints and factual preservation without storing user drafts. The client provides real-time diff visualization, inline text editing, and formatted clipboard export.

<img width="1591" height="763" alt="image" src="https://github.com/user-attachments/assets/7bf117d6-b82f-477d-a6bd-8e8b931bd90f" />


## Features

- **Tone Selection**: Transforms text into one of four communication registers: Executive Formal, Warm and Professional, Concise and Direct, or Diplomatic and De-escalating ([`src/components/ToneSelector.tsx`](src/components/ToneSelector.tsx)).
- **Fact Preservation Constraints**: Instructs the model via system prompt to maintain original facts, dates, commitments, and names without hallucinating additions ([`server/services/llm.ts`](server/services/llm.ts)).
- **Word-Level Diff View**: Computes additions and deletions against the original draft to verify edits prior to sending ([`src/components/DiffView.tsx`](src/components/DiffView.tsx)).
- **Inline Output Editing**: Allows direct editing of both subject line and email body with a reset option to revert to generated output ([`src/components/PolishedOutput.tsx`](src/components/PolishedOutput.tsx)).
- **Rich Clipboard Export**: Copies text as both HTML and plain text for pasting with intact paragraph and list formatting into email clients ([`src/components/PolishedOutput.tsx`](src/components/PolishedOutput.tsx)).
- **Session History**: Retains up to eight previous revisions within browser session storage with instant restoration and clear controls ([`src/App.tsx`](src/App.tsx)).
- **Zero Server Data Retention**: Processes drafts in memory without saving message bodies or completions to disk or persistent databases ([`server/services/logger.ts`](server/services/logger.ts)).
- **Local Fallback Engine**: Provides deterministic template-based generation when an external API key is not configured ([`server/services/llm.ts`](server/services/llm.ts)).

## Try it Out
- https://email-rewriter-three.vercel.app/

## How It Works

### System Overview

```
+------------------------------------------------------+
| Browser Client (React / Vite on Port 5173)           |
| UI state, Diff visualizer, Ephemeral session storage |
+------------------------------------------------------+
                           |
                           | HTTP POST /api/rewrite
                           v
+------------------------------------------------------+
| Backend API Proxy (Express on Port 3002)             |
| Rate limiter, Helmet CSP, Security middleware chain  |
+------------------------------------------------------+
                           |
                           | Upstream HTTPS request
                           v
+------------------------------------------------------+
| LLM Provider (OpenRouter API / Local Fallback)       |
| Chat completions endpoint with system guardrails     |
+------------------------------------------------------+
```

### Execution Loop

```
+------------------------------------------------------+
| 1. Input: "delay launch to Tue, db index bug"        |
| 2. Policy Engine: Validates rewrite-email intent     |
| 3. Firewall: Checks regex patterns for injection     |
| 4. Sanitizer: Normalizes Unicode and XML escapes     |
| 5. LLM Call: Applies system prompt + tone register   |
| 6. Sanitizer: Strips script/iframe tags from output  |
| 7. Metrics: Calculates word counts and reading time  |
| 8. Client Render: Displays subject, body, and diff   |
+------------------------------------------------------+
```

### Request Lifecycle Steps

1. The user inputs text, selects a tone, and submits the form from the browser interface.
2. The browser sends a `POST` request with JSON payload `{ draft, tone }` to `/api/rewrite`.
3. Express security middleware validates identity headers, monitors anomaly thresholds, verifies intent policy, blocks unauthorized tool headers, rejects memory write intents, and blocks multi-agent flags.
4. The prompt firewall and input sanitizer normalize Unicode, strip unprintable control characters, enforce the 4,000 character limit, and scan for prompt injection indicators.
5. The request schema is parsed and validated using Zod in [`server/routes/rewrite.ts`](server/routes/rewrite.ts).
6. The LLM provider formats the system prompt, embeds the XML-escaped user text inside `<user_draft>` tags, and sends a request to the OpenRouter chat completions endpoint (or dispatches to the local deterministic engine).
7. If an upstream 429 rate limit is returned, the provider iterates across fallback candidate models defined in [`server/services/llm.ts`](server/services/llm.ts).
8. The raw completion is parsed into subject and body components and passed through output sanitization.
9. Privacy-preserving metadata (latency, status code, character counts, tone, model) is logged to server stdout and audit logs without draft text.
10. The client receives the JSON payload, updates the diff computation, and stores the item in local session history.

## Key Modules

| File | Purpose |
| :--- | :--- |
| [`server/index.ts`](server/index.ts) | Express server bootstrap, Helmet CSP headers, CORS, rate limiting, and route mounting. |
| [`server/routes/rewrite.ts`](server/routes/rewrite.ts) | HTTP route handler, Zod schema validation, metrics computation, and response formatting. |
| [`server/services/llm.ts`](server/services/llm.ts) | OpenRouter client, fallback model cascade, system prompt contracts, and local fallback engine. |
| [`server/services/sanitizer.ts`](server/services/sanitizer.ts) | Input normalization, injection pattern detection, XML escaping, output tag stripping, and reading time calculation. |
| [`server/services/logger.ts`](server/services/logger.ts) | Metadata logger omitting user draft content and completions. |
| [`server/security/prompt_firewall.ts`](server/security/prompt_firewall.ts) | Middleware blocking requests matching injection heuristics and dispatching security events. |
| [`server/security/anomaly_detector.ts`](server/security/anomaly_detector.ts) | In-memory IP anomaly counter that triggers a 429 circuit breaker upon threshold breach. |
| [`server/security/policy_engine.ts`](server/security/policy_engine.ts) | Validates allowed intent headers and checks HTTPS transport in production. |
| [`server/security/identity_manager.ts`](server/security/identity_manager.ts) | Strips authorization tokens and sets static guest identity context. |
| [`server/security/tool_guard.ts`](server/security/tool_guard.ts) | Enforces an empty tool execution allowlist. |
| [`server/security/memory_guard.ts`](server/security/memory_guard.ts) | Rejects memory and vector write intent headers. |
| [`server/security/inter_agent.ts`](server/security/inter_agent.ts) | Blocks multi-agent coordination headers. |
| [`server/security/sandbox_executor.ts`](server/security/sandbox_executor.ts) | Blocks dynamic code execution flags. |
| [`server/security/audit_logger.ts`](server/security/audit_logger.ts) | Append-only structured JSON security event writer for local environments. |
| [`server/security/error_handler.ts`](server/security/error_handler.ts) | Catch-all error handler returning sanitized status responses without stack traces. |
| [`src/App.tsx`](src/App.tsx) | Main React application state, session history drawer, and layout coordination. |
| [`src/components/DraftInput.tsx`](src/components/DraftInput.tsx) | Textarea input component with character counts, presets, and keyboard shortcuts. |
| [`src/components/PolishedOutput.tsx`](src/components/PolishedOutput.tsx) | Output container supporting inline editing, dual-format clipboard copying, and view switching. |
| [`src/components/DiffView.tsx`](src/components/DiffView.tsx) | Word-level diff visualizer showing added and removed phrasing. |
| [`src/components/ToneSelector.tsx`](src/components/ToneSelector.tsx) | Register selector component with four tone options. |

## Security Design

The application implements defense-in-depth controls designed by the author for this personal project:

- **Strict Proxy Boundary**: Upstream API keys are held exclusively server-side and are never bundled into client assets.
- **Input and Output Sanitization**: User drafts are XML-escaped and bound inside tags in [`server/services/llm.ts`](server/services/llm.ts); output is stripped of script and iframe tags in [`server/services/sanitizer.ts`](server/services/sanitizer.ts).
- **Stateless Operation**: No message drafts, rewritten bodies, or prompt payloads are persisted in persistent databases or file logs.
- **Default-Deny Middleware**: Middleware rejects unauthorized tool invocations, code execution flags, memory write requests, and non-whitelisted intents.
- **Rate and Anomaly Limiting**: Standard IP-based rate limiting via express-rate-limit and a multi-strike anomaly circuit breaker in [`server/security/anomaly_detector.ts`](server/security/anomaly_detector.ts).

Trust boundary: The browser client and user input are untrusted. The Express proxy acts as the single trust boundary mediating communication with external LLM APIs.

