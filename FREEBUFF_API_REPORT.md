# Freebuff API Structure & "Thinking" Mechanisms Report

## 1. Overview
This report details the API structure of Freebuff/Codebuff, specifically focusing on the wire-level session management, chat completion endpoints, payload structures, and the comprehensive handling of "thinking" and reasoning across different AI models.

---

## 2. API Endpoints & Protocol Structure

### 2.1 Session Management (`/api/v1/freebuff/session`)
The Freebuff CLI orchestrates a waiting room and active session state via this endpoint.

*   **Methods & Flow**:
    *   `GET`: Probes the current session state. Uses the `x-freebuff-instance-id` header to detect if another CLI instance has taken over the session (returns `superseded`).
    *   `POST`: Joins or switches queues. Uses the `x-freebuff-model` header to declare the requested model queue.
    *   `DELETE`: Explicitly ends the current session and releases the slot.
*   **Headers Required**:
    *   `Authorization: Bearer <token>`
    *   `x-freebuff-instance-id` (GET only)
    *   `x-freebuff-model` (POST only)
*   **Response Payload (`FreebuffSessionServerResponse`)**:
    *   Responses are heavily typed and dictate the CLI state. Key statuses include:
        *   `none`: User has no active/queued session. Includes quota snapshots.
        *   `queued`: User is waiting. Provides `position`, `estimatedWaitMs`, and queue depths.
        *   `active`: User is admitted. Provides `expiresAt` and `remainingMs`.
        *   `ended`: Session grace period.
        *   `rate_limited`: User hit daily limits for limited/premium models.
        *   `country_blocked`: Geo-fencing rejection.

### 2.2 Chat Completions (`/api/v1/chat/completions`)
This is the primary endpoint for executing agent steps and generating AI responses.

*   **Method**: `POST`
*   **Headers Required**:
    *   `Authorization: Bearer <token>`
    *   `x-codebuff-api-key: <token>`
    *   `Content-Type: application/json`
*   **Response & Errors**:
    *   Returns a stream of text, reasoning, or tool-call deltas.
    *   **401 Unauthorized**: Missing or invalid tokens.
    *   **402 Payment Required**: Returned when the user runs out of credits/sessions. The backend formats a user-friendly message predicting when free credits reset (e.g., "Your free credits reset in 3 hours").

---

## 3. "Thinking" & Reasoning Implementations

Freebuff handles reasoning in four distinct ways depending on the model provider and agent definition.

### 3.1 Inline `<think>` Tag Parsing (Streaming Models)
Models like **DeepSeek V4 Pro/Flash** stream their thoughts in line using `<think>...</think>` tags.
*   **Implementation**: Handled in `cli/src/utils/think-tag-parser.ts`.
*   **Mechanics**: The parser (`parseThinkTags`) buffers stream chunks and looks for partial tag prefixes (`<t`, `<th`, etc.). Once a valid block of text is trapped between `<think>` tags, it emits segments of `type: 'thinking'`.
*   **UI Integration**: The CLI groups these thinking segments into blocks and manages their visibility using a `thinkingCollapseState` ('hidden', 'expanded', 'preview').

### 3.2 OpenRouter `reasoningOptions`
For models served via OpenRouter, reasoning is configured at the API request level.
*   **Implementation**: Agents define `reasoningOptions` inside their `AgentTemplate`.
*   **Parameters**:
    *   `effort`: Can be set to `'high'`, `'medium'`, `'low'`, `'minimal'`, or `'none'`.
    *   `max_tokens`: A strict token limit for thinking.
    *   `exclude`: A boolean flag. If `true`, the `reasoning-delta` chunks returned by the stream are completely dropped by the SDK and not shown to the user or injected into the message history.

### 3.3 Gemini `thinkingBudget`
For Google Gemini models, thinking is passed directly to the AI SDK provider options.
*   **Implementation**: Specifically configured as `thinkingConfig: { thinkingBudget: number }`. In some backend routes, it defaults to 128 tokens unless overridden by the agent's parameters.

### 3.4 Multi-Agent "Thinker" Pattern
Beyond native model reasoning, Freebuff implements a programmatic, multi-agent orchestration pattern to force deep analysis.
*   **Implementation**: Found under `.agents/deep-thinking/`.
*   **Mechanics**: A master orchestrator agent (`deep-thinker` or `deepest-thinker`) executes a `spawn_agents` tool to spin up specialized sub-agents in parallel.
*   **Sub-agents include**:
    *   `gpt5-thinker`: For quick, focused insights.
    *   `sonnet-thinker`: For balanced, nuanced analysis.
    *   `gemini-thinker`: For creative, innovative exploration.
*   **Integration**: Parent models like Kimi, DeepSeek V4 Pro, MiMo Pro, and MiniMax M3 are explicitly allowlisted (via `FREEBUFF_GEMINI_THINKER_PARENT_MODELS`) to spawn these thinker agents to synthesize complex solutions before responding.

---



### 3.5 MiniMax Reasoning
The MiniMax models (specifically `MiniMax M3`) run inside the standard Freebuff orchestrator (`base2`).
*   **Routing**: Under the hood, they route through OpenRouter (using model ID `minimax/minimax-m3`), meaning their reasoning can be configured using the same `reasoningOptions` (`effort`, `max_tokens`, `exclude`) as other OpenRouter-backed models.
*   **Sub-agent Offloading**: As one of the core Freebuff models, `MiniMax M3` is explicitly allowlisted in `FREEBUFF_GEMINI_THINKER_PARENT_MODELS`. This allows the root orchestrator to offload deeper reasoning to the `gemini-thinker` sub-agent when complex problems arise, bridging native inference with the multi-agent "Thinker" paradigm.
*   **Cost & Privacy**: In free mode, the orchestrator overrides the provider options to explicitly set `data_collection: 'deny'`.

## 4. Model Capabilities Overview

*   **DeepSeek V4 Pro & Flash**: Utilizes inline `<think>` tags. Highly capable models used in both full and limited Freebuff tiers.
*   **MiMo 2.5 / Pro**: Multimodal models supported in the UI.
*   **Kimi K2.6 & MiniMax M3**: Core full-access models. MiniMax M3 is the default recommended model for new users as it does not burn premium quotas.
*   **Data Collection Tracing**: Freebuff tracks if a model requires data collection warnings via `FREEBUFF_DATA_COLLECTION_WARNING`. Models like DeepSeek V4 Pro are flagged and their traces are explicitly handled.

---
*Generated by Jules.*
