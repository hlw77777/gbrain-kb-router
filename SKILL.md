---
name: kb-router
description: >-
  Three-tier knowledge-base query routing for gbrain-connected harnesses.
  L1 zero-cost lookup (search / recall / entity). L2 zero-cost harness synthesis
  (DEFAULT: 3-5 query variants + parallel retrieval with expand:false +
  takes_search/traverse_graph arms + harness-written final answer with
  citations + rate_answer feedback). L3 paid think heavy artillery (doctrine-
  critical content, evidence still thin after two L2 rounds, decision-grade
  synthesis, or the user explicitly names think). Triggers on: query the brain,
  knowledge base question, deep answer, cross-page synthesis, think routing,
  kb-router, "use gbrain think".
---

# kb-router — three-tier KB query routing for gbrain

> Doctrine: **harness synthesis is the workhorse** (flat-cost harness LLM,
> high-quality cited final drafts); **the server-side `think` pipeline is the
> heavy artillery** (paid per call, reserved for what actually needs it).

Routing mantra: **L1 for points, L2 for planes (default), L3 for doctrine and
hard problems.**

Why this exists: gbrain ships retrieval verbs at three price points
(`recall` ≈ zero-LLM, `query`/`search` ≈ free, `think`/`synthesize`
deliberately expensive) but deliberately leaves *when to escalate* to the host
agent. When your harness LLM is flat-rate (a coding-plan subscription) while
the gbrain server pays per call, routing most synthesis through the harness is
both cheaper and often produces better final answers. This skill encodes that
routing as an installable doctrine.

## Step 0 — classify (<10 seconds)

| Signal | Route |
|---|---|
| Exact lookup (where is a file / single fact / entity card) | L1 |
| Cross-page synthesis, comparison, provenance, why/how — not doctrine-critical | L2 (default) |
| Doctrine-critical corpus questions (course teachings, canonical material — anything with an anchor map) | L3 (iron rule) |
| User explicitly says "gbrain think" / "go deep with think" | L3 (user-named) |
| User says "quick answer / one line / keep it short" this turn | L1 or shortest path |
| Pure external info (nothing in the brain) | outside this router → web search |

## L1 — lookup

`search` (exact tokens / names) → `entity` (card) → `recall` (facts).
Answer and move on. No variant fan-out for single facts.

## L2 — harness synthesis (DEFAULT tier)

1. **Free expansion clone**: generate 3–5 query variants yourself (synonym
   rewrites, upstream/downstream concepts, mixed languages). This replaces the
   server-side paid LLM expansion.
2. **Parallel retrieval — always `expand:false`** (skips the server-side paid
   expansion call):
   - `query`: `expand:false, return_unit:"section", token_budget:12000`
     (concept / synthesis variants)
   - `search`: exact-token variants
   - `takes_search`: when the question involves opinions / judgments / bets
   - `traverse_graph`: when a clear entity anchor exists (`slug` + `depth:2`)
3. **Read whole sections**: `get_page` when evidence is thin. Any quoted text
   must be verified at the file level (open the source file, cite by line).
4. **Harness writes the final answer**: citations with page slugs / clickable
   links, gaps stated explicitly, conflicts shown side by side. Never silently
   pick one side of a conflict.
5. **Feed the loop**: when `query`/`search` return an `answer_id`, call
   `rate_answer` (1–5). On remote HTTP MCP channels without `answer_id`, skip
   (feedback flows back through owner-side calls).

## L3 — think heavy artillery (escalate when ANY hits)

1. **Doctrine-critical material**: always `think` with an anchor, e.g.
   `anchor: "sources/<your-course-map>"` (pin your canonical corpus map once).
   Quotes from the corpus must be re-verified in the source files.
2. **Two L2 rounds still not enough**: thin evidence, unresolvable conflicts,
   key pages not retrieved.
3. **Decision-grade synthesis**: long-investment questions worth feeding the
   citation-weight learning loop.
4. **User named think.**

**Always treat `think` output as an evidence pack + draft.** The harness writes
the final answer (translate its `conflicts` / `gaps` sections, verify quotes at
file level). When L2 and L3 disagree, the file on disk wins — determinism beats
any model synthesis.

**L3 health check**: if `think` returns `synthesis_status` ≠ `ok` (no LLM /
insufficient balance / not_json), the server-side LLM channel is broken —
immediately degrade and finish the question with the L2 recipe (the `think`
gather evidence still works as a lead), and report the channel failure
honestly. A working L2 is the only deep-answer path while L3 is down.

## Channel differences

| | stdio / local-CLI MCP | remote HTTP MCP |
|---|---|---|
| `answer_id` / `rate_answer` | returned → **always rate** | not returned → skip |
| owner-perspective takes (`holder=self`) | may be hidden from agent channels | same — fall back to owner CLI: `gbrain takes list --who self` |
| tool names | server-native (`search`, `query`, …) | harness-prefixed (`mcp__gbrain__*`) |

## Official brain-first disciplines (merged in)

Adopted from gbrain's `skills/conventions/brain-first.md`:

- **Mandatory lookup chain**: `search` (exact) → `query` (concept / landscape)
  → `get_page` (once you have a slug) → external APIs only after 1–2 come back
  empty. Ask the brain before grepping the vault.
- **score > 0.5 = use it.** Don't reach for external search when the brain
  answered.
- **Read notices first**: a degraded or empty result is NOT proof the brain
  has nothing; `[STALE]` results must be flagged to the user.
- **The user's direct statements are the highest-authority source**
  (user statements > compiled truth > timeline > external).
- **Bank paid external pulls** into the brain (`capture`) before the
  conversation moves on — a lookup you paid for and didn't bank is a lookup
  you'll pay for again.
- A nonzero `search` count is not a completeness signal; enumerate with
  `list_pages`, re-phrase with `query`.

## Failure boundaries (when this router is overkill)

- Single facts don't need a 3–5 variant retrieval fan-out (L1 is enough).
- Pure external questions never enter this router.
- When the user asks for brevity, skip the router entirely.
