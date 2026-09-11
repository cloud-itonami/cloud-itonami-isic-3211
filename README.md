# cloud-itonami-isic-3211: Manufacture of jewellery and related articles

Open Business Blueprint for **ISIC 3211**: manufacture of jewellery and related articles — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **jewellery-workshop plant operations**: production-batch data logging (metal-type/purity/weight/defect-rate), casting/setting/polishing-equipment maintenance scheduling, safety-concern flagging, and outbound product shipment coordination.

This repository designs a forkable OSS business for jewellery-workshop
plant operations: run by a qualified operator so a workshop keeps its
own operating records instead of renting a closed SaaS.

## Scope: plant operations coordination, not workshop-line control

ISIC 3211 covers the **jewellery workshop** that casts (rings/chains/
pendants/bracelets in gold/silver/platinum/palladium), sets stones/
gems, and polishes the resulting finished jewellery and related
articles. This actor coordinates the back-office record keeping around
that workshop — it never touches the casting/setting/polishing
equipment directly, and it is never the hallmarking/purity-assay
authority (e.g. an assay office) that certifies a piece's precious-
metal fineness.

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — casting/setting/polishing batch, precious-metal-weight/purity data logging (administrative, not an operational decision)
- `:schedule-maintenance` — casting/setting/polishing-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface a materials-safety (solvent/acid)/theft-security/authenticity concern (always escalates)
- `:coordinate-shipment` — outbound product shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety- and security-relevant
domain** (casting/setting/polishing-line equipment, solvent/acid
materials hazard, theft/security exposure inherent to precious metals,
hallmarking/purity-assay certification, consumer-protection
consequence downstream):

- Does NOT control casting, setting, or polishing equipment directly
- Does NOT make workshop-safety, security, or hallmarking/purity-certification decisions (that's the workshop supervisor's / accredited assay office's exclusive human/institutional authority)
- Does NOT actuate casting/setting/polishing equipment (human workshop supervisor decides)
- Does NOT self-issue a hallmark or purity-assay certification (the accredited assay office's exclusive authority — a PERMANENT, unconditional block)
- ONLY proposes/coordinates operations back-office; all actuation and hallmarking/purity certification requires explicit human/institutional authority
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`jewellerymfg.operation/build`, a langgraph-clj StateGraph):
1. **`jewellerymfg.advisor`** (sealed intelligence node, `JewelryAdvisor`): proposes decisions only, never commits
2. **`jewellerymfg.governor`** (independent, `Jewellery Workshop Plant Operations Governor`): validates against domain rules, re-derived from `jewellerymfg.registry`'s pure functions and `jewellerymfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Workshop/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct casting/setting/polishing-line-equipment control)
     - Directly actuating casting/setting/polishing equipment (`:actuate-equipment? true`) is a PERMANENT, unconditional block
     - Self-issuing a hallmark / purity-assay certification (`:issue-hallmark-certification? true`, any op) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped quantity past its own logged production quantity (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:metal-type` value on a production-batch patch
     - No physically/regulatorily implausible `:purity-permille` value on a production-batch patch
     - No physically implausible `:weight-grams` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`jewellerymfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`jewellerymfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
kbb -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
kbb -M:dev:test

# Run the demo
kbb -M:dev:run

# Lint
kbb -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
