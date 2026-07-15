# ADR-0001: JewelryAdvisor ⊣ Jewellery Workshop Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-3211` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-3211` publishes an OSS blueprint for jewellery-
workshop **plant operations coordination** (production-batch metal-
type/purity/weight/defect-rate data logging, casting/setting/
polishing-equipment maintenance scheduling, safety-concern flagging,
and outbound product shipment coordination). Like every actor in this
fleet, the blueprint alone is not an implementation: this ADR records
the governed-actor architecture that promotes it to real, tested code,
following the same langgraph StateGraph + independent Governor + Phase
0->3 rollout pattern established across the cloud-itonami fleet.

Identity ({:id "3211" :name "Manufacture of jewellery and related
articles"}) was independently verified against a fresh clone of
`kotoba-lang/industry`'s `resources/kotoba/industry/registry.edn`
before any work began, per this fleet's ID/name-mismatch caution
(prior agents in this fleet have mislabeled their assigned ISIC
class). The entry's own `:repo` field pointed at a stale, never-
created `gftdcojp/cloud-itonami-C3211` placeholder; the real
`cloud-itonami` org target name was independently confirmed 404 via
`gh api repos/cloud-itonami/cloud-itonami-isic-3211` before
scaffolding began.

The closest domain analog is `cloud-itonami-isic-3250` (Manufacture of
medical and dental instruments and supplies): both are back-office
coordination actors for a fixed processing PLANT/WORKSHOP with
precision equipment and a real safety/consumer-protection dimension,
and both share the same four-op shape
(`:log-production-batch`/`:schedule-maintenance`/
`:flag-safety-concern`/`:coordinate-shipment`) and the same two-entity
verified/registered gate structure (equipment for maintenance
scheduling, batch for shipment coordination). This build mirrors
`cloud-itonami-isic-3250`'s architecture closely but adapts the hazard
profile and equipment/product vocabulary to the jewellery workshop:
this vertical's central physical hazard is casting/setting/polishing-
equipment operation and materials-safety (solvent/acid pickling and
plating chemistry) exposure, plus a genuinely distinct theft/security
and authenticity-fraud dimension inherent to precious-metal/gemstone
inventory (not present in 3250's own hazard profile) -- its permanent
equipment-actuation block guards casting/setting/polishing EQUIPMENT
(`:actuate-equipment?`) rather than machining/molding/sterilization
EQUIPMENT; its production-batch record declares a `:metal-type`
(closed set spanning gold/silver/platinum/palladium) and a
`:purity-permille` (the ISO 9202 hallmarking fineness convention,
plausibility-checked 1-999) and a `:weight-grams` reading (this
vertical's own genuinely new physical check, plausibility-checked
above 0 up to a 5000g per-batch ceiling -- tracking the batch's
precious-metal weight, a domain-specific data point 3250 has no
analog for) in addition to a `:defect-rate-percent`, rather than
3250's `:device-class`/`:sterility-assurance-level`/
`:nonconformance-rate-percent`; and its shipment quantity is tracked
in finished-jewellery-piece UNITS (`:units`/`:quantity-units`/
`:shipped-units`), the same counted-not-weighed shape 3250 uses for
finished instruments.

This vertical additionally has a DOMAIN-SPECIFIC permanent block
mirroring 3250's FDA-510(k)/CE-mark block but for a different
regulatory/attestation regime: manufacture of jewellery is subject to
hallmarking / purity-assay certification (the fineness stamp an
accredited assay office applies to certify a piece's actual precious-
metal content, e.g. UK Assay Offices under the Hallmarking Act 1973,
or equivalent national/regional assay authorities). This actor is
never the hallmarking/purity-assay authority -- any proposal
(regardless of op) that declares `:issue-hallmark-certification?
true` is a HARD, PERMANENT, unconditional block
(`jewellerymfg.governor/hallmark-authority-blocked-violations`), the
same "no phase, no human override" posture as the equipment-actuation
block.

This vertical has NO pre-existing `kotoba-lang/jewellerymfg`-style
capability library to wrap (verified: no such repo exists, and no
`jewelry`/`jewellery`-named repo exists in `kotoba-lang` either, via
GitHub code/repo search -- the one incidental hit, `kotoba-lang/
com-baker-hughes-jewel`, is an unrelated oilfield-services JewelSuite
integration, not a jewellery-manufacturing capability library). This
build therefore uses self-contained domain logic -- pure functions in
`jewellerymfg.registry` (equipment/batch verification, shipment-
quantity recompute, metal-type validation, purity-permille
plausibility validation, weight-grams plausibility validation,
defect-rate plausibility validation) are re-verified independently by
the governor, the same "ground truth, not self-report" discipline
established across prior actors (most directly `cloud-itonami-isic-
3250`'s `medinstrmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:jewellery-workshop-plant-operations-governor`, is grep-verified
UNIQUE fleet-wide (`gh search code
"jewellery-workshop-plant-operations-governor" --owner
cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external jewellery-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
jewellery-workshop vertical has NO pre-existing capability library to
wrap. The equipment/batch-verification / shipment-quantity /
metal-type / purity-permille / weight-grams / defect-rate validation
functions live as pure functions in `jewellerymfg.registry` and are
re-verified independently by `jewellerymfg.governor` -- the same
"ground truth, not self-report" discipline established across prior
actors (most directly `cloud-itonami-isic-3250`'s
`medinstrmfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of jewellery-
workshop plant operations. It does NOT:
- Control casting, setting, or polishing equipment directly
- Make workshop-safety, security, or hallmarking/purity-certification decisions (exclusive to the human workshop supervisor / accredited assay office)
- Actuate casting/setting/polishing equipment
- Self-issue a hallmark or purity-assay certification

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human workshop-
supervisor approval. This is not a replacement for the supervisor's
authority or the assay office's authority — it is a
proposal-screening and documentation layer.

**CRITICAL SAFETY/SECURITY BOUNDARY**: jewellery manufacturing is a
safety- and security-relevant domain (solvent/acid materials-safety
hazard, theft/security exposure inherent to precious-metal/gemstone
inventory, authenticity-fraud risk, hallmarking/purity-assay
certification, consumer-protection consequence downstream).
Safety-concern flagging NEVER auto-commits. All safety concerns
escalate immediately to human review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (materials-safety solvent/acid concern,
theft/security concern, authenticity concern) ALWAYS escalates, never
auto-commits. This is not a "low-stakes proposal" — it is a
circuit-breaker that must reach human authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-3250`, this vertical has TWO entity kinds
each gating a different op: `:schedule-maintenance` independently
verifies the referenced **equipment** unit's own `:verified?`/
`:registered?` fields; `:coordinate-shipment` independently verifies
the referenced **batch**'s own `:verified?`/`:registered?` fields.
Both are the same "workshop/batch record must be independently
verified/registered before any action" HARD invariant applied to the
two distinct record kinds this domain actually has.
`:coordinate-shipment` additionally independently recomputes whether
a batch's own recorded shipped-to-date unit quantity plus the
proposal's own claimed unit quantity would exceed the batch's own
recorded production quantity — never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into thirteen concrete
checks in `jewellerymfg.governor`, mirroring `cloud-itonami-isic-
3250`'s own elaboration of its HARD invariants into concrete checks,
plus one additional check unique to this vertical -- weight-grams
plausibility) block proposals and cannot be overridden by human
approval:
1. Workshop/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct casting/setting/polishing-equipment control, equipment actuation, or self-issued hallmark/purity-assay certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Jewellery-workshop plant operations back-office now has a
documented, governed, auditable coordination layer that funnels all
decisions through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human workshop-
supervisor sign-off, and no hallmark/purity-assay certification can
ever be self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into thirteen concrete governor checks) protect against scope creep
into unauthorized equipment operation, equipment actuation, or
hallmarking/purity-certification self-issuance. Safety concerns are a
circuit-breaker, not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real workshop-operations
control system. Equipment actuation, line operation, and hallmarking/
purity-assay certification issuance remain human-/institution-
controlled via external channels.

(-) No integration with real workshop-management databases (equipment
telemetry, batch tracking, freight dispatch, assay-office APIs) —
this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-3211`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-quantity-exceeded, equipment-actuate-
  blocked, hallmark-authority-blocked, already-scheduled,
  invalid-metal-type, invalid-purity, invalid-weight,
  invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
