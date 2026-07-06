# Running Claude Fable 5 on the Raw API: A Field Guide from One Very Long Monday

*Notes from operating a self-hosted chat frontend (Node/Express, BYOK) against `claude-fable-5`, June–July 2026. Everything below was hit in production or verified against live API responses — error messages are quoted verbatim.*

## TL;DR

Fable 5 is not "Opus but bigger." It ships with different API semantics: sampling params are gone, thinking output is hidden by default, safety refusals arrive as HTTP 200, and the fallback system only lets you fall to one specific model. If you swap the model string and change nothing else, you will hit at least three of the issues below.

Pricing context: $10 / $50 per million input/output tokens (2× Opus 4.8), 1M context, 128k max output, prompt caching with the standard 90% read discount. Mandatory 30-day data retention (no ZDR).

## 1. `temperature` is dead

First request after switching the model string:

```
{"type":"error","error":{"type":"invalid_request_error",
 "message":"`temperature` is deprecated for this model."}}
```

Fable 5 manages sampling internally. Strip `temperature` (and don't send `top_k`; `top_p` is heavily restricted). If your request builder passes user sampling settings through unconditionally, gate it by model:

```js
if (body.temperature !== undefined && !/fable/i.test(body.model)) {
  rb.temperature = body.temperature;
}
```

Note that some gateway providers still silently accept `temperature` for Fable — which means the bug only surfaces when you switch to the native API. Gate it everywhere.

## 2. Thinking blocks are empty unless you ask nicely

Adaptive thinking is the only mode and is always on (`thinking: {"type":"disabled"}` errors out; manual `budget_tokens` is unsupported). But the raw chain of thought is never returned, and `thinking.display` **defaults to `omitted`** — you get thinking blocks with an empty `thinking` field and only the encrypted `signature`.

If you want anything human-readable:

```json
"thinking": { "type": "adaptive", "display": "summarized" }
```

Two expectations to reset:

- **Easy prompts produce no thinking block at all.** Adaptive means the model decides. "Which is bigger, 9.11 or 9.9" got zero thinking events in our streams; a Ramsey-theory proof produced a full `thinking_delta` stream. If your UI shows a thinking panel, make it optional per turn, not expected.
- **The summary is a paraphrase, not the reasoning.** It reads like a press briefing ("I should keep my response concise"), and its language is chosen by the summarizer, not by you — in our logs it stayed English through hours of Chinese conversation and only switched to Chinese when the reasoning itself was *about* Chinese words. There is no documented control for summary language, length, or style.

Billing note: you pay for the full internal thinking tokens, not the summary tokens you see.

## 3. Refusals are HTTP 200, and your UI will render them as silence

Fable 5 runs safety classifiers (categories: `cyber`, `bio`, `reasoning_extraction`). A flagged request returns a **successful** response:

- HTTP 200
- `stop_reason: "refusal"`
- `stop_details.category` naming the classifier
- empty content (refusals before any output are not billed)

If your stream handler only surfaces text deltas and transport errors, a refusal renders as an empty assistant message with no explanation — the most confusing possible failure mode. Detect `stop_reason === "refusal"` at end of stream and tell the user explicitly.

The classifiers are deliberately conservative and false-positive on benign technical content. Our reliable repro: a Chinese test prompt containing the words for "rollback verification" tripped `cyber` twice in a row; the same innocuous sentence without those words passed. If your users talk about servers, auth, or security at all, they will meet this. Two more gotchas:

- A refused turn left in history keeps getting refused. Remove or rewrite it before retrying.
- In long agent sessions the *first* request can be refused, because it carries workspace context (docs note this for Claude Code).

## 4. Fallback exists, is optional on the API, and only goes to one place

In the Claude apps, flagged requests silently fall back to Opus 4.8. On the raw API, nothing happens unless you configure it. Three official options: the server-side `fallbacks` parameter (beta header `server-side-fallback-2026-06-01`; native API and Claude Platform on AWS only — not Bedrock/Vertex/Foundry), SDK middleware, or a manual retry using the `fallback_credit_token` from `stop_details`.

We tried pointing the fallback at a different model and got a definitive answer:

```
'claude-opus-4-6' is not a valid fallback target for 'claude-fable-5'.
Valid targets: claude-opus-4-8.
```

So the fallback list is a whitelist of exactly one. If your product (or your users) don't want silent model substitution, skip the fallback entirely and surface the refusal — that's a legitimate design choice, arguably the more honest one for a chat product where users care *which* model answered. Billing sweetener if you do use it: fallback input is billed as a cache read (10% of base input) instead of a cache write.

One more cross-model trap: when you retry a refused conversation on another model, strip the `thinking`/`redacted_thinking` blocks from prior assistant turns first, or the target model rejects them.

## 5. Prompt caching still carries the economics (numbers)

Nothing Fable-specific broke our caching setup, and it's the only reason the model is affordable interactively. Our layout: system prompt + a frozen "stable memory" block on a `cache_control` breakpoint with `ttl: "1h"`, per-turn dynamic content attached to the current user message *beyond* the last breakpoint so it never invalidates the prefix.

A real morning of chat (33 requests):

| metric | value |
|---|---|
| typical per-turn `input_tokens` | **2** (everything else read from cache) |
| cache reads per turn | 5,151 → 8,840, monotonically growing |
| cache writes per turn | ~300–1,500 (new turns only) |
| cold start (new conversation) | one write of ~6–8.5k tokens |
| per-turn cost, steady state | $0.010–0.020 |
| 13-minute idle gap | survived on the 1h TTL, full cache hit after |

The lesson that outranks all cache tuning: at $50/M output, **response length dominates steady-state cost**. A single max-effort thinking run on a proof-style question produced 8,183 output tokens — $0.205, ten times a normal turn. Keep interactive effort at `low`/`medium`; the docs themselves say lower Fable effort often beats prior models at max.

Also nice: the prompt-cache minimum dropped to 512 tokens on the native API.

## 6. Bonus lesson (not Fable-specific): kill your silent fallbacks

Unrelated to Anthropic but it ate half the Monday: our semantic memory-recall pipeline had a TF-IDF keyword-search fallback for when the primary (embedding-based) path failed. The primary path had been failing for **three months** — a model cold-load took 19.7s against a 20s timeout — and nobody noticed, because the fallback kept quietly serving garbage that was just plausible enough. The user experienced it as "recall keeps injecting five irrelevant memories."

We removed the fallback entirely: failure now returns nothing plus a loud log line. A visible gap gets fixed in a day; a silent degradation survives a quarter. If a component of yours has a "worse but never empty" fallback, ask what outages it is currently hiding.

## Checklist for migrating an existing harness to `claude-fable-5`

- [ ] Strip `temperature` / `top_k` / thinking `budget_tokens` / assistant prefill for this model
- [ ] Set `thinking: {type:"adaptive", display:"summarized"}` if you display thinking
- [ ] Handle `stop_reason: "refusal"` as a normal outcome, not an error; show the user something
- [ ] Decide your fallback policy explicitly (server-side beta / SDK middleware / manual / none) — target can only be `claude-opus-4-8`
- [ ] On cross-model retry, strip prior thinking blocks
- [ ] Remove refused turns from history before retrying
- [ ] Re-check your cost model: 2× Opus pricing, thinking billed at full internal tokens, output length is the lever
- [ ] Confirm 30-day retention is acceptable (no ZDR on this model)

*Sources: platform.claude.com docs (extended/adaptive thinking, Fable 5 introduction, refusals & fallback), anthropic.com/claude/fable, plus live API responses quoted above.*
