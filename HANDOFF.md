# HANDOFF — casehub-claudony

## Last Session

Phase 2 canonical layer complete and landed on main (6 squashed commits). All 4 issues delivered: #231 fluent DSL, #232 annotations, #233 YAML parity, #234 ops plugin integration. Test baseline: 282 → 320 (+38). Blog: "Three Frontends, One Model."

**Cross-repo work:** `ProcessExecutor` SPI built in platform-api (5 classes, 14 tests) — landed on canonical platform main. Fills the Layer 1 gap for local process management. Platform's yaml-step-runtime already wired it into `ProcessInvokeHandler` (#446) and `ProcessPlugin` (@StepPlugin).

**Design investigation (#234):** Brainstormed pool-as-desiredstate-node. Key findings: pool is the node (not individual sessions), pool's internal orchestration stays in Claudony's runtime (not in the provisioner), ProcessExecutor belongs in platform. Filed desiredstate#152 (suspend/resume lifecycle SPI) — implemented and landed by another session.

**YAML plugin feasibility:** Drafted a pure YAML agent pool plugin using platform's process plugin + structural step types (if/else, try/catch, forEach, parallel). The pool provisioner is expressible in ~100 lines of YAML, no Java. Draft at `specs/issue-205-llm-fleet-manager/agent-pool-yaml-plugin-draft.yaml`.

## Immediate Next Step

Epic #235: align the landed canonical layer with platform YAML language improvements. Two issues:
- #236 — migrate `AgentPoolYamlParser` to platform typed parameter model (S/Low)
- #237 — migrate `AgentPoolAnnotationScanner` to `@StepPlugin` pattern (S/Med)

Run `work start #235`.

## Pre-Existing Failures

- Maven in slot 202: use `mvn install:install-file` to re-register cached JARs
- Canonical claudony repo has a stash (`stash-before-slot-202-land`) — inverse of McpDomain migration from another session. Likely safe to drop.

## References

- `specs/issue-205-llm-fleet-manager/2026-09-21-llm-fleet-manager-design.md`
- `specs/issue-205-llm-fleet-manager/decisions.md` (D1–D11)
- `specs/issue-205-llm-fleet-manager/agent-pool-yaml-plugin-draft.yaml`
- `blog/2026-09-25-mdp01-three-frontends-one-model.md`
- casehubio/casehub-desiredstate#152 (suspend/resume SPI — landed)
- Tracking: #229 (EvictionPolicy SPI), #230 (test utilities) — deferred
