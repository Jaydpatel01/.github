---
name: claude-api
description: "Build apps with the Claude API or Anthropic SDK. TRIGGER when: code imports `anthropic`/`@anthropic-ai/sdk`/`claude_agent_sdk`, or user asks to use Claude API, Anthropic SDKs, or Agent SDK. DO NOT TRIGGER when: code imports `openai`/other AI SDK, general programming, or ML/data-science tasks."
license: Complete terms in LICENSE.txt
---

# Building LLM-Powered Applications with Claude

This skill helps you build LLM-powered applications with Claude. Choose the right surface based on your needs, detect the project language, then read the relevant language-specific documentation.

## Defaults

Unless the user requests otherwise:

For the Claude model version, please use Claude Opus 4.6, which you can access via the exact model string `claude-opus-4-6`. Please default to using adaptive thinking (`thinking: {type: "adaptive"}`) for anything remotely complicated. And finally, please default to streaming for any request that may involve long input, long output, or high `max_tokens` — it prevents hitting request timeouts. Use the SDK's `.get_final_message()` / `.finalMessage()` helper to get the complete response if you don't need to handle individual stream events

---

## Language Detection

Before reading code examples, determine which language the user is working in:

1. **Look at project files** to infer the language:

   - `*.py`, `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile` → **Python** — read from `python/`
   - `*.ts`, `*.tsx`, `package.json`, `tsconfig.json` → **TypeScript** — read from `typescript/`
   - `*.js`, `*.jsx` (no `.ts` files present) → **TypeScript** — JS uses the same SDK, read from `typescript/`
   - `*.java`, `pom.xml`, `build.gradle` → **Java** — read from `java/`
   - `*.kt`, `*.kts`, `build.gradle.kts` → **Java** — Kotlin uses the Java SDK, read from `java/`
   - `*.scala`, `build.sbt` → **Java** — Scala uses the Java SDK, read from `java/`
   - `*.go`, `go.mod` → **Go** — read from `go/`
   - `*.rb`, `Gemfile` → **Ruby** — read from `ruby/`
   - `*.cs`, `*.csproj` → **C#** — read from `csharp/`
   - `*.php`, `composer.json` → **PHP** — read from `php/`

2. **If multiple languages detected**: Check which language the user's current file or question relates to. If still ambiguous, ask.

3. **If language can't be inferred**: Use AskUserQuestion with options: Python, TypeScript, Java, Go, Ruby, cURL/raw HTTP, C#, PHP. Default to Python examples if unavailable.

4. **If unsupported language detected**: Suggest cURL/raw HTTP examples from `curl/` and note that community SDKs may exist.

---

## Which Surface Should I Use?

> **Start simple.** Default to the simplest tier that meets your needs.

| Use Case | Tier | Recommended Surface | Why |
|------|------|------|------|
| Classification, summarization, extraction, Q&A | Single LLM call | **Claude API** | One request, one response |
| Multi-step pipelines with code-controlled logic | Workflow | **Claude API + tool use** | You orchestrate the loop |
| AI agent with file/web/terminal access | Agent | **Agent SDK** | Built-in tools, safety, and MCP support |

### Decision Tree

```
What does your application need?

1. Single LLM call (classification, summarization, extraction, Q&A)
   └── Claude API — one request, one response

2. Does Claude need to read/write files, browse the web, or run shell commands?
   └── Yes → Agent SDK — built-in tools, don't reimplement them

3. Workflow (multi-step, code-orchestrated, with your own tools)
   └── Claude API with tool use — you control the loop

4. Open-ended agent (model decides its own trajectory, your own tools)
   └── Claude API agentic loop (maximum flexibility)
```

---

## Architecture

Everything goes through `POST /v1/messages`. Tools and output constraints are features of this single endpoint.

---

## Current Models (cached: 2026-02-17)

| Model | Model ID | Context | Input $/1M | Output $/1M |
|---|---|---|---|---|
| Claude Opus 4.6 | `claude-opus-4-6` | 200K (1M beta) | $5.00 | $25.00 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 200K (1M beta) | $3.00 | $15.00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | $1.00 | $5.00 |

**ALWAYS use `claude-opus-4-6` unless the user explicitly names a different model.**

**CRITICAL: Use only the exact model ID strings from the table above. Do not append date suffixes.**

---

## Thinking & Effort (Quick Reference)

**Opus 4.6 — Adaptive thinking (recommended):** Use `thinking: {type: "adaptive"}`. Claude dynamically decides when and how much to think. `budget_tokens` is deprecated on Opus 4.6 and Sonnet 4.6 and must not be used.

**Effort parameter (GA, no beta header):** Controls thinking depth via `output_config: {effort: "low"|"medium"|"high"|"max"}`. Default is `high`. `max` is Opus 4.6 only.

---

## Reading Guide

After detecting the language, read the relevant files based on what the user needs:

### Quick Task Reference

**Single text classification/summarization/extraction/Q&A:**
→ Read only `{lang}/claude-api/README.md`

**Chat UI or real-time response display:**
→ Read `{lang}/claude-api/README.md` + `{lang}/claude-api/streaming.md`

**Function calling / tool use / agents:**
→ Read `{lang}/claude-api/README.md` + `shared/tool-use-concepts.md` + `{lang}/claude-api/tool-use.md`

**Agent with built-in tools (file/web/terminal):**
→ Read `{lang}/agent-sdk/README.md` + `{lang}/agent-sdk/patterns.md`

### Claude API (Full File Reference)

Read the **language-specific Claude API folder** (`{language}/claude-api/`):

1. **`{language}/claude-api/README.md`** — Installation, quick start, common patterns, error handling.
2. **`shared/tool-use-concepts.md`** — Read when the user needs function calling, code execution, memory, or structured outputs.
3. **`{language}/claude-api/tool-use.md`** — Language-specific tool use code examples.
4. **`{language}/claude-api/streaming.md`** — Read when building chat UIs.
5. **`{language}/claude-api/batches.md`** — Read when processing many requests offline.
6. **`shared/error-codes.md`** — Read when debugging HTTP errors.
7. **`shared/live-sources.md`** — WebFetch URLs for fetching the latest official documentation.

---

## Common Pitfalls

- Don't truncate inputs when passing files or content to the API.
- **Opus 4.6 / Sonnet 4.6 thinking:** Use `thinking: {type: "adaptive"}` — do NOT use `budget_tokens`.
- **Opus 4.6 prefill removed:** Assistant message prefills return a 400 error on Opus 4.6.
- **128K output tokens:** Opus 4.6 supports up to 128K `max_tokens`, but SDKs require streaming for large `max_tokens`.
- **Structured outputs:** Use `output_config: {format: {...}}` instead of the deprecated `output_format` parameter.
- **Don't reimplement SDK functionality:** Use SDK helpers instead of building from scratch.
