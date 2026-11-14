# gbrain-kb-router

Three-tier query routing for [gbrain](https://github.com/garrytan/gbrain)-connected
agent harnesses (Claude Code / Codex / ZCode / OpenClaw / any MCP client):

```
L1  zero-cost lookup          → search / recall / entity
L2  zero-cost harness synth   → DEFAULT: variants + parallel retrieval + harness-written answer
L3  paid think heavy artillery → gbrain think, only when it's actually worth it
```

**L1 for points, L2 for planes (default), L3 for doctrine and hard problems.**

## Why

gbrain deliberately prices its verbs: `recall` is zero-LLM, `query`/`search`
are free-ish, and `think`/`synthesize` are intentionally expensive slow verbs.
The official docs then leave *when to escalate* to the host agent's judgment —
there is no shipped router.

That gap matters most under one specific — and increasingly common — cost
structure: **your harness LLM is flat-rate** (a coding-plan subscription)
**while the gbrain server pays per LLM call**. In that asymmetry, letting the
harness do the synthesis (retrieve wide for free, write the final answer
itself) is both cheaper *and* often higher quality, because the harness model
is usually stronger than the server's fast-tier synthesis model.

This skill encodes that doctrine as one installable `SKILL.md`, field-tested
on a 1,100+ page personal knowledge vault since October 2026, and merged with
gbrain's official `brain-first` conventions (mandatory lookup chain,
notice-first reading, source precedence).

## What the router does

- **Step 0 (10s)**: classify the question → L1 / L2 / L3 / outside.
- **L2 recipe** (the workhorse): 3–5 self-generated query variants (free
  expansion clone) → parallel retrieval with `expand:false` (`query` with
  `return_unit:"section"`, `token_budget:12000`, plus `search` /
  `takes_search` / `traverse_graph` arms) → read whole sections → harness
  writes the cited final answer → `rate_answer` feedback when `answer_id` is
  available.
- **L3 triggers**: doctrine-critical corpus questions (with a pinned anchor),
  two failed L2 rounds, decision-grade synthesis, or the user naming think.
  `think` output is always treated as an **evidence pack + draft** — the
  harness writes the final. Includes a **health check**: if `think` returns
  `synthesis_status ≠ ok`, degrade to L2 and report the channel failure.
- **Determinism rule**: when L2 and L3 disagree, the file on disk wins.

## Install

Copy `SKILL.md` into your harness skills directory:

```bash
# Claude Code
mkdir -p ~/.claude/skills/kb-router && cp SKILL.md ~/.claude/skills/kb-router/

# Codex
mkdir -p ~/.codex/skills/kb-router && cp SKILL.md ~/.codex/skills/kb-router/

# ZCode
mkdir -p ~/.zcode/skills/kb-router && cp SKILL.md ~/.zcode/skills/kb-router/
```

Requires a gbrain MCP connection (stdio or HTTP). No other dependencies — the
skill is prose; your harness executes it.

Then just ask naturally: *"what does my brain know about X"*, *"deep-dive
across pages on Y"*, or *"gbrain think: Z"* to force the heavy tier.

The original Chinese doctrine (`SKILL.zh-CN.md`) is included as the reference
version; translate or localize triggers freely.

## Relationship to official gbrain

- Official conventions this builds on: `skills/conventions/brain-first.md`
  (lookup chain, notices, source precedence), `docs/protocol/MEMORY_VERBS_v1.md`
  (verb cost tiers), `docs/guides/chat-fallback.md` (degradation philosophy),
  `docs/guides/retrieval-feedback.md` (`rate_answer` loop).
- This project is independent of the gbrain repo and follows its upgrade
  cadence opportunistically — parts get absorbed as the official project ships
  them.

## License

[MIT](LICENSE)
