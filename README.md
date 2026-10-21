# A/B Split Testing – Chat Prompts

> n8n workflow that randomly (but consistently per session) assigns a chat session to one of two system-prompt variants and runs the conversation through AI agents backed by different LLMs.

| | |
|---|---|
| **Workflow name** | A/B Split Testing - Chat Prompts |
| **Platform** | n8n (execution order `v1`) |
| **Status** | Inactive (`active: false`) |
| **Trigger** | n8n Chat Trigger |

---

## 1. Overview

This workflow demonstrates a lightweight way to A/B test system prompts for a conversational assistant. When a chat message arrives, a Code node derives a prompt variant from the chat **session ID**, so the same session always receives the same variant. The selected prompt is passed to AI Agent nodes as their system message.

The design is intentionally stateless: no database or external store is used for session assignment or analytics..

The workflow currently contains **two parallel agent branches** fed by the same variant assignment:

| Branch | Node | Model | Provider |
|---|---|---|---|
| 1 | `AI Agent1` | `gpt-4o-mini` | OpenAI |
| 2 | `AI Agent` | `allam-2-7b` | Groq |

This means that, in addition to the prompt A/B comparison, the same input and prompt can be compared across two different models.

## 2. Objectives

- Compare two system-prompt styles on real chat traffic:
  - **Baseline** – clear, structured answers with a brief explanation and actionable steps.
  - **Alternative** – short answers.
- Guarantee **session stickiness**: a user never switches variants mid-conversation.
- Allow side-by-side evaluation of different LLM providers using the same prompt.

## 3. Workflow Architecture

### Nodes

| # | Node | Type | Role |
|---|---|---|---|
| 1 | **When chat message received1** | `chatTrigger` (v1.1) | Entry point; receives `chatInput` and `sessionId` |
| 2 | **Assign A/B Prompt Variant** | `code` (v2) | Hashes the session ID, picks the variant, attaches `variant`, `show_alternative`, and `prompt` |
| 3 | **AI Agent1** | `agent` (v1.7) | Answers using the assigned prompt; user text = `{{ $json.chatInput }}` |
| 4 | **OpenAI Chat Model1** | `lmChatOpenAi` (v1.2) | Language model for `AI Agent1` (`gpt-4o-mini`) |
| 5 | **AI Agent** | `agent` (v3.1) | Second agent using the same assigned prompt |
| 6 | **Groq Chat Model** | `lmChatGroq` (v1) | Language model for `AI Agent` (`allam-2-7b`) |
| 7 | **Setup Notes** | `stickyNote` | In-canvas documentation (not executed) |





### Output fields added by the Code node

| Field | Description |
|---|---|
| `sessionId` | Resolved session identifier |
| `variant` | `"baseline"` or `"alternative"` |
| `show_alternative` | Boolean flag |
| `prompt` | The system prompt text for the selected variant |

## 4. Execution Flow

1. A user sends a message through the n8n chat interface.
2. The **Chat Trigger** emits `chatInput` and `sessionId`.
3. The **Code node** resolves the session ID (falling back to `chatSessionId`, then `'default-session'`), computes the hash, and selects the variant.
4. The item, enriched with `variant` and `prompt`, is sent to **both** agents in parallel.
5. Each agent uses `$json.prompt` as its system message and `$json.chatInput` as the user message.
6. Each agent calls its connected model (OpenAI or Groq) and generates a response.
7. Subsequent messages in the same session repeat steps 2–6 and receive the same variant.

## 5. Technology Stack

| Layer | Technology |
|---|---|
| Orchestration | n8n |
| Trigger | `@n8n/n8n-nodes-langchain.chatTrigger` |
| Logic | JavaScript (n8n Code node) |
| Agents | `@n8n/n8n-nodes-langchain.agent` (v1.7 and v3.1) |
| LLM provider 1 | OpenAI – `gpt-4o-mini` |
| LLM provider 2 | Groq – `allam-2-7b` |

**Credentials required**

- **OpenAI** – the exported file has no credential selected (`id: null`), so you must choose your own in *OpenAI Chat Model1* before testing.
- **Groq** – a credential named "Groq account" is referenced; it must exist in the target n8n instance or be re-selected.

## 6. Testing and Validation


**Behavioral validation**

- **Baseline** responses should be structured, include a brief explanation, and list actionable steps.
- **Alternative** responses should be noticeably shorter.
- Compare the OpenAI and Groq outputs for the same prompt to evaluate model differences.


## 7. Error Handling

The workflow has **no explicit error handling**. Current behavior:

- **Missing session ID** – the Code node falls back to `'default-session'`, so all such requests share one variant.
- **Missing or invalid credentials** – the corresponding agent node fails and the execution reports an error.
- **Model/API failures** (rate limits, timeouts, provider outage) – the failing branch errors; there are no retries, fallbacks, or error workflows configured.
- **Parallel branches** – a failure in one branch does not remove the other branch's result, but no logic reconciles them.

## 8. Limitations

- **No analytics** – the assigned variant and responses are not stored, so there is no built-in way to measure which prompt performs better.
- **Parallel agents, one chat reply** – both agents are wired to the same input, but the chat interface will only surface a single response; the other branch's output is visible only in the execution log. The sticky note describes a single-agent design and is out of date.
- **No conversation memory** – neither agent has a memory node, so each message is handled without prior context.
- **Fixed 50/50 split** – ratios cannot be changed without editing code, and small sample sizes may be unbalanced.
- **Hash-based assignment** – deterministic but not truly random; sessions with the same ID can never be re-bucketed.
- **Variants differ in length and structure** – the two prompts differ in more than one way, so results cannot isolate a single factor.
- **Model comparison confounds the test** – with two different models running, differences in output may come from the model rather than the prompt.
- **Agent configuration inconsistencies** – `AI Agent` (v3.1) does not explicitly define its user-message source as `AI Agent1` does; verify that it receives `chatInput` as expected.
- **Workflow inactive** – it must be activated for production use.

