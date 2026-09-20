**Status:** 🟢 Active

# History-trim cache hysteresis — stable cut points at the context ceiling

**Topic:** provider / history-trim / prompt-cache
**Updated:** 2026-09-20
**Tags:** #cache #provider #history-trim #deepseek #go
**Related:** Upstream PR #212 (cache-key parity + deterministic prefix), issue [#232](https://github.com/ltmoerdani/opencode-copilot-chat/issues/232) (cache-rate monitoring)

---

## Problem

A long session that reaches the input budget loses its provider prompt-cache
hits almost entirely. Observed on a 614K-token OpenCode Go session
(`deepseek-v4.1-flash`, chat-completions): the cache-hit rate alternated
between ~99.8% and **11.4%** in the same conversation, minutes apart, and the
whole session averaged 49% with ~45% token efficiency (20.1M prompt tokens,
9.1M cached). The 11.4% misses always reported the same
`cachedTokens=69888` — exactly the fixed head (system prompt + tools).

## Analysis

The trimmer (`trimOldMessagesToFitContext`) drops the oldest droppable units
until the payload fits **and then stops at the minimal fit**. At the ceiling,
the landing sits just under the budget with almost no slack, so the next
turn's growth (one user turn is easily 2K+ tokens) exceeds the budget again —
and the next trim moves the cut point to a **different position**.

The provider's prefix cache only reuses the bytes before the first changed
message. A moved cut therefore invalidates everything after the anchor:
only system prompt + tools stay cached (69,888 tokens ≈ 11.4% of 614K).
Every moved cut re-bills the conversation body at full input price.

Log correlation from the affected session (trim drop count → next hit rate):

```text
05:35  trim=245  → 99.8%   (cut unchanged)
05:36  trim=249  → 11.4%   (cut moved)
05:37  trim=251  → 11.4%   (cut moved)
05:37  trim=253  → 100.0%  (retry of the same cut re-primes)
06:00  trim=317  → 99.8% ×4 (cut stable again)
06:02  trim=319  → 11.3%   (cut moved)
```

207 trims fired on that session over ~15 hours. The fix is not about the
cache key (that layer was verified working): it is about **keeping the cut
point still** once a trim is unavoidable.

## Fix

When the trimmer drops anything, it now keeps dropping (same unit
granularity, same tool-group safety rules) until the payload is at or below
a **low-water mark**: `budget − headroom`, with the headroom from new config
constants `HISTORY_TRIM_HEADROOM_RATIO` (3%), floored by
`HISTORY_TRIM_HEADROOM_MIN_TOKENS` (8,192 — capped at 10% of the budget so a
small budget is never dominated) and capped by
`HISTORY_TRIM_HEADROOM_MAX_TOKENS` (32,768). The byte ceiling mirrors the
token mark via the same ratio (`HISTORY_BYTES_PER_TOKEN`).

Trade-off: a trim now sheds a bounded amount of extra old context
(≤ headroom, overshoot ≤ one unit) in exchange for a cut that survives the
following turns — one cold prefix per trim epoch instead of a cold prefix on
nearly every turn. With ~2–3K tokens of growth per turn, an 8K–32K headroom
holds the cut for roughly 3–10 turns.

## Files Changed

| File                          | Change                                                         |
| ----------------------------- | -------------------------------------------------------------- |
| `src/config.ts`               | `HISTORY_TRIM_HEADROOM_RATIO` / `_MIN_TOKENS` / `_MAX_TOKENS`  |
| `src/provider/historyTrim.ts` | Low-water pass after the minimal-fit drop; documented contract |
| `src/test/messages.test.ts`   | Low-water landing test + no-re-trim stability test             |

## Verification

- `npm run lint` clean (editorconfig, ESLint, markdown, prettier, shell,
  TypeScript, tests).
- 466/466 unit tests pass, including the two new cases:
  - a trim lands at or below the low-water mark (`finalTokens ≤ budget − 8,192`
    at a 100K budget) while anchor + current prompt are preserved;
  - five follow-up turns after the trim do not move the cut (`removed == 0`)
    — with zero headroom (control), the same turns re-trim and move it.
- Runtime smoke test against the compiled `out/provider/historyTrim.js` of a
  patched local 0.7.5 build: trim 38 units, land at 91,179 tokens against a
  91,808 low-water mark, cut stable across five turns.
- Real-world expectation (manual follow-up): after this ships, sessions at
  the ceiling should show ~99% hits for several turns after each trim and a
  single ~11% miss per trim epoch, instead of the alternating
  99.8%/11.4% pattern.

## Lessons Learned

A trim that "just fits" is not free: at the context ceiling it turns every
following turn into a cache miss, because the provider prefix cache breaks at
the first changed message. Trimming is only cheap when its cut point is
**stable** — so the trimmer must optimize for stillness, not for keeping the
maximum number of tokens.
