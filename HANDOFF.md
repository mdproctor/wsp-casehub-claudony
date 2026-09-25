# HANDOFF — casehub-claudony

## Last Session

Phase 1 complete and landed on main (6 squashed commits). Phase 2 set up: canonical Java layer for agent pool — fluent DSL, annotations, YAML parity, ops plugin integration.

**Phase 1 delivered (all on main):** Agent pool (#205, 15 issues), audit fixes (#220–#226, epic #227), consumer surface polish (#228 — consumer guide, javadoc, CDI `@Produces` exposure). Test baseline: 721. Blog: "Hardening the Agent Pool."

**Consumer surface investigation:** Explored desiredstate and ops codebases via IntelliJ. Key finding: desiredstate is domain-agnostic (no pool changes needed). Ops `deployment/AgentProvisionHandler` is the integration point — it already bridges to engine's `ProvisionerConfigRegistry` SPI. Pool integration adds session lifecycle alongside the existing config chain.

**Canonical layer gap identified:** The pool has a raw Java API but no fluent DSL, annotations, or YAML parity. The casehub ecosystem pattern (engine, blocks, work, desiredstate) is: Java API → fluent DSL + annotations → YAML parity → ops plugin. Pool is at step 1.

## Immediate Next Step

Start #231 (fluent DSL). Study how engine (`CaseDefinition.builder()`), desiredstate (`NodeSpec.builder()`), and blocks structure their DSLs. Decide whether abstractions live in `claudony-casehub` (Phase B) or `platform-agent-api` (Phase A extraction). Run `work continue`.

## Pre-Existing Failures

- Maven in slot 202: use `mvn install:install-file` to re-register cached JARs (see garden GE-20260802-44a85e alternative fix). IntelliJ diagnostics work for compilation.
- Engine: pre-existing checkstyle errors and neocortex build failures (not caused by this work)
- Soredium#377: work-end landing step tries to land repos with no branch commits — filed, not yet fixed

## Queue

Position 0/4. #231 (fluent DSL) → #232 (annotations) → #233 (YAML parity) → #234 (ops plugin).

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11)
- `blog/2026-09-22-mdp01-hardening-the-pool.md`
- `docs/guides/consumer-guide.md` — Agent Pool Management section
- Tracking: #229 (EvictionPolicy SPI), #230 (test utilities) — deferred to scaffold spike
