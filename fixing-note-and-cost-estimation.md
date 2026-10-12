# Fixing Note & Cost Estimation
**Course Inquiry Auto-Triage Bot**

---

## Fixing Note

### Issue 1 — Shared credential caused webhook secret and Anthropic/Groq key to change together
**Problem:** Updating the API key value from inside one node also changed it in the other.
**Root Cause:** Both `01_Webhook` and `06_Claude_API` were pointed at the same generic **Header Auth** credential object rather than two separate ones — Header Auth is a generic credential type, so n8n allowed it to be reused across nodes without warning.
**Fix:** Created two distinct Header Auth credentials — `Webhook Header Auth` (`Authorization: Bearer <webhook secret>`) and a separate key for the AI provider — and re-attached each node to its own credential individually.
**Result:** Editing one no longer affects the other; each secret is independently rotatable.

### Issue 2 — Workflow halted when no duplicate rows were found
**Problem:** Execution stopped at `04b_Duplicate_Lookup` with "No output data returned," even though this was expected behavior on a clean sheet.
**Root Cause:** n8n's default behavior treats zero output items as a reason to stop the branch rather than pass an empty result downstream.
**Fix:** Enabled **Always Output Data** in `04b_Duplicate_Lookup`'s node settings.
**Result:** A lookup with no matches now correctly passes through as "not a duplicate" instead of halting the run.

### Issue 3 — AI API failures (bad credential, provider errors) crashed the execution instead of falling back
**Problem:** A wrong API key or a transient provider error stopped the workflow entirely rather than triggering the retry/fallback path the spec requires.
**Root Cause:** By default, an HTTP Request node throws and halts the workflow on a non-2xx response; the original design only handled *business-level* invalidity (bad JSON), not *transport-level* failure (401, 5xx, timeout).
**Fix:** Set **On Error → Continue (using error output)** on both `06_Claude_API` and `09_Retry_Claude`. Their error outputs feed dedicated handler nodes — `AI_API_Error_Handler` and `Retry_AI_API_Error_Handler` — which inspect the error message for auth-related keywords (`unauthorized`, `invalid api key`, `401`, etc.) and set `ai_auth_error`. An `If` node then routes an auth failure straight to `10_Fallback` (no point retrying against the same broken credential), while any other transient error still gets one retry attempt via `09_Retry_Claude`. A retry that also fails at the transport level routes through `Retry_AI_API_Error_Handler` directly into `10_Fallback` with `needs_human=true`.
**Result:** No credential or provider failure can crash the workflow — every path terminates in either a successful classification or a logged, human-flagged fallback, matching the "graceful degradation" requirement.

---

## Cost Estimation

**Mechanism:** `13a_Compute_Route_And_Cost` and `13_Log_Execution` compute cost from the AI provider's **actual reported token usage** (`ai_input_tokens`, `ai_output_tokens`), not an estimate from character count.

**Formula** (in `13_Log_Execution`, field `estimated_token_cost`):
```
{{ ($json.ai_input_tokens / 1000000 * 1.00) + ($json.ai_output_tokens / 1000000 * 5.00) }}
```
`1.00` / `5.00` = input/output price per million tokens, held in `11_Config` (`claude_input_price_per_m`, `claude_output_price_per_m`) — Claude Haiku 4.5's published rates.

**Example, one classification call:**

| | Tokens | Rate/MTok | Cost |
|---|---|---|---|
| Input | ~150 | $1.00 | $0.00015 |
| Output | ~60 | $5.00 | $0.00030 |
| **Total** | | | **≈ $0.00045/call** |

At 10,000 inquiries/month with an assumed 15% retry rate: **≈ $5.18/month**.

**Two caveats worth stating plainly, not glossing over:**
1. **Provider mismatch:** the workflow currently calls Groq (`openai/gpt-oss-20b` / `qwen/qwen3.8-27b`), not Anthropic — the $1.00/$5.00 rates above are Claude Haiku 4.5's pricing, used as the documented baseline the design was built against. They do not reflect actual Groq billing; update `claude_input_price_per_m`/`claude_output_price_per_m` to Groq's published per-model rates for a cost figure that matches current real spend.
2. **Retry cost undercount:** `07b_Parse_Retry_JSON` overwrites `ai_input_tokens`/`ai_output_tokens` with the retry call's usage only. When a retry occurs, the logged `estimated_token_cost` reflects just the second call — the first (failed) call's tokens were also billed by the provider but aren't added to the total. Actual spend on retried inquiries is therefore modestly higher than what `Execution Audit_Log` shows.
