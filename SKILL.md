---
name: operator-session-permissions
description: "Deliverable: a reviewable operator-session permission pack - graded policy review, frozen session snapshots, fail-closed verdicts with registered error codes, and mapped acceptance tests - produced when designing, implementing, or auditing tool-execution permissions, OAuth connection binding, or fail-closed configuration handling in an agent runtime."
---

# operator-session-permissions - Operator Session Permissions for Tool-Executing Agents

## Positioning

This skill turns one question - "what is this agent session allowed to do?" - into an explicit, named, auditable fact that is frozen at session-creation time. It packages a complete, dual-gate-reviewed design (named access policies, creation-time snapshot freeze, a meta-tool three-stage discipline, a fail-closed rule set, and redemption-style OAuth callback verification) into an executable design-and-audit workflow.

Canonical source: `references/design-v0.2.md` (design v0.2, finalized 2026-10-16; in Chinese). This SKILL.md is the operating layer: where any summary here and the design differ, the design prevails.

## When to use

- Designing the permission layer of an agent runtime whose sessions execute tools against external accounts.
- Implementing session creation, policy management, tool execution, or OAuth connection binding against this design.
- Auditing an existing implementation or design for fail-open defects (defaults that grant, silent fallbacks, unpinned versions resurrecting contracted scope, callbacks that bind whoever shows up).
- Reviewing a proposed change to policies, session configuration handling, or callback flows against the finalized parameters below.

## When NOT to use

- Multi-tenant billing, cross-organization policy inheritance, or tool-marketplace/registry design (explicit non-goals of the design).
- The OAuth provider's hosted implementation itself; this package defines only the operator-side session-permission and callback-verification contract.
- Any work that would connect real credentials, production data, or real user accounts before the implementation preconditions in `references/implementation-preconditions.md` are closed (see "Scope limit" below).

## Mental model

Four principles govern everything; every rule in the design is an expansion of one of them:

1. **Fail closed.** Missing, ambiguous, damaged, or out-of-scope configuration always converges to *less* permission or *no execution* - never to "unrestricted". An absent `enable` field and an explicitly empty `enable` list are the same thing: zero grants. A full grant can only be built item by item from an explicit non-empty list.
2. **Freeze at creation.** A session's permissions and connection bindings are resolved from a named policy into an immutable snapshot at creation time. Later policy edits never reach back into existing sessions; permission changes take effect only through new sessions, the explicit versioned-update channel (interactive long-lived sessions only), or fleet expiry-and-rebuild.
3. **Separation of the permission plane and the execution plane.** The component that decides permissions (policy resolver) and the component that uses them (tool executor) contract only through the snapshot. The execution plane can never widen the snapshot.
4. **Rejection means record and stop.** Every rejection (session creation, update conflict, out-of-scope execution, callback redemption mismatch) appends an audit event, and the execution plane stops and reports - it never routes around a rejection.

Key mechanisms, in one line each:

- **A policy is a permission.** Named policies (`sc_...` ids, versioned, owned) declare grants and cut-outs across three dimensions - toolkits, tools, tags - each with an `enable` and a `disable` list. Evaluation is deterministic: any disable match denies; otherwise any enable match allows; otherwise denied.
- **Pinning is not a resurrection channel.** Ordinary callers may pin only the current active policy version. Versions marked `security_contracted` can never be pinned by anyone; the sole exception path is audit reproduction by an audit role pinning a formerly-active, unmarked version, fully logged and flagged `audit_repro`.
- **Meta-tool three-stage discipline.** Broad sessions preload no business-tool definitions. The model sees six meta-tools only (search, fetch schema, execute, list connections, request connection, report blocked), searches *inside the snapshot*, fetches a tool's schema before executing it, and executes only after three checks pass: in-snapshot, arguments valid against the fetched schema, and the connection is in the frozen bindings, owned by the session owner, and active. Tools that were never searched for do not exist for the model; tools outside the snapshot do not even appear in search results.
- **Redemption-style callback verification.** Completing an OAuth consent produces only a sealed, one-time, redeemable credential. The binding takes effect at redemption, when the redeemer must be the initiator, any pre-declared provider account must match, and the redeemer must explicitly confirm the provider account being bound. Consent-account ownership is checked *before* sealing: an account belonging to another known user fails the flow and destroys the token.
- **Registry content is trusted data; execution results are never instructions.** Tool descriptions enter the model context, so registry changes are change-controlled and audited. External content returned by tools is data only; the real defense against indirect prompt injection is the blast radius fenced by snapshot + schema validation.

## Finalized parameters (do not re-litigate)

These values were settled by the design's dual-gate review, including the closure of its two conflict notes by a governance confirmation memo dated 2026-10-16. Where older drafts or notes disagree, this table prevails:

| Parameter | Finalized value |
|---|---|
| `session_uri` TTL | 10 minutes, single segment, counted from connect_session creation across both the INITIATED and PENDING_REDEMPTION states. No widening switch may be pre-built; the remedy for expiry is low-cost re-initiation. |
| Fleet (scheduled/long-lived) session snapshot lifetime | Hard cap <= 24h from creation, enforced expiry-and-rebuild. The concrete TTL is configured per task-cadence category and may be shorter (short-cycle tasks may use 10 minutes). Fleet sessions never use versioned in-place updates. |
| Versioned config update | Interactive long-lived sessions only (`lifecycle_mode: versioned_update`), optimistic concurrency, zero writes on conflict, no automatic retry by server or SDK. |
| Policy review grading | Two axes, stricter wins. Content axis grades *activation* (broad grants - write operations or `destructive` tags - require dual review across groups; read-only grants take a single operator plus audit). Direction axis grades *changes* to active policies (expansions require a second reviewer judged by the server-evaluated effective tool-set diff; contractions take a single-operator fast track). A read-only **expansion** change is an expansion: dual review. |
| Zero-grant policy | Cannot be activated (`policy_empty_scope`); a zero-tool *session* (empty allowlist) is a legal, strict runtime state. |
| Direct preset (narrow preloaded sessions) | Explicit per-session configuration, never an automatic fallback; preload list capped at 50 tools (`preset_limit_exceeded`). |
| Credentials | Connection credentials live in the operator's own credential store (encryption at rest, rotation on every refresh, reuse of an old refresh token treated as a leak signal, verifiable destruction on revoke) - never in an opaque reference-only vault, because connections must refresh tokens programmatically. |

## Inputs

Before starting, assemble: (1) the artifact under work - a policy draft, a session-create/update payload, an implementation, or a callback flow; (2) the operation intended (design, implement, audit, change-review); (3) the applicable registries for reference resolution (tool / toolkit / tag names). No credentials of any kind are an input to this skill.

## Workflow

### S1 - Policy definition and graded review gate

- **Input:** a policy draft (fields per `templates/policy.example.yaml`: `policy_id`, `name`, `version`, `owner`, `state`, `rules` across toolkits/tools/tags), plus the registry for reference resolution.
- **Preconditions:** the draft has a named owner; at least one non-empty `enable` dimension if activation is intended.
- **Steps:** (a) Resolve every rule reference against the registry. A `disable` reference that cannot be resolved blocks activation - 400 `policy_rule_unresolved`, policy state unchanged, error body lists every unresolved disable reference. An unresolvable `enable` reference may activate only with a warning audit event and explicit disclosure in the activation receipt. (b) Grade the activation on the content axis: broad grant (any write operation, or any `destructive` tag hit) requires dual review - owner-group lead submits, a reviewer from a *different* group approves, intermediate state `pending_review` (not referenceable by new sessions); read-only takes the single-operator path. (c) Compute the effective tool set server-side and show the **diff against the previous version** in the receipt; reviewers review the evaluated result, not rule text. (d) For changes to an active policy, grade on the direction axis as well (expansion vs. contraction by server-evaluated effective tool-set diff); when both axes trigger, the stricter grading wins. Audit events for activation and change must carry the reviewer identity and which axis decided the review level.
- **Output:** an `active` policy version, or a refusal; an activation/change receipt echoing the effective tool set and the diff.
- **Failure handling:** zero-grant activation -> 400 `policy_empty_scope`, state unchanged. Unresolved disable reference -> 400 `policy_rule_unresolved`, state unchanged. Caller lacking the management role -> 403 `admin_role_required`, zero writes, audit written.

### S2 - Session creation and snapshot verification

- **Input:** a session-create payload (template: `templates/session-create.example.json`) referencing a `policy_id`, an optional `policy_version`, connection bindings or an explicit `connection_mode: none`, and a `lifecycle_mode`.
- **Preconditions:** the referenced policy is `active`; every declared connection exists, is `active`, and belongs to the session owner.
- **Steps:** (a) Validate the payload against a strict schema that recurses into **every** nesting level - any undeclared key anywhere rejects the whole request: 400 `session_config_unknown_key`, with all unknown keys and their paths listed, and zero configuration writes (only an append-only security-rejection log entry is allowed). (b) Apply absent = empty normalization to grant fields. (c) Enforce the pinning rules from the mental model. (d) Resolve the policy into the snapshot (tool set + rule digest + connection bindings), and write session row + snapshot in a **single transaction** - any failure rolls back completely; no session row without a snapshot, no orphan snapshot. (e) Return a receipt echoing the resolved effective tool set, the connection binding list, and the snapshot hash.
- **Output:** a created session with an immutable snapshot, or a rejection with a registered error code.
- **Failure handling:** unknown policy -> 404 `policy_not_found`; cross-scope policy -> 403 `policy_forbidden` (no policy content leaked); archived or `pending_review` policy -> 400 `policy_archived`; unresolvable-or-wrong-owner connection -> 404 `connection_not_found` / 403 `connection_forbidden`; pinnability violation -> 400 `policy_version_not_pinnable`. All creation failures write zero session/snapshot rows.

### S3 - Meta-tool three-stage execution discipline

- **Input:** a live session snapshot; an agent intent.
- **Preconditions:** the session is not `blocked`; its snapshot loaded and hash-verified at runtime (load failure puts the session into `blocked`, returns 503 `session_config_unavailable`, and raises an operations alert - it never degrades into an unrestricted session).
- **Steps:** (a) **Search** only inside the snapshot's allowed set; non-granted tools are absent from results entirely - no names, no "exists but forbidden" hints. Zero results means zero results: report blocked or change intent, never brute-force by rewording queries. (b) **Fetch schema** for the chosen tool; the server verifies the tool is in the snapshot (else 403 `tool_not_in_snapshot`) and records the fetch in-session. A tool whose schema was never fetched in this session cannot be executed. (c) **Execute** only after the three checks pass: in-snapshot; arguments fully valid against the fetched schema (else 422 `tool_args_schema_invalid`, zero side effects); connection check - the explicitly named connection is in the frozen bindings, `connection.owner == session.owner`, and `active` (else 403 `connection_not_in_snapshot`). There is no server-side "pick any currently available connection" semantics and no automatic re-binding when a bound connection is revoked. Credentials are injected by the connection layer and never enter the model context or session records.
- **Output:** tool results treated strictly as data, or a structured blocked report.
- **Failure handling:** any rejection -> the agent's only legal move is `report_blocked` and stopping this round. Prohibited workarounds: substituting a similar but unauthorized tool, calling an external API with raw HTTP, asking the user to paste credentials into the chat, splitting the task into a more privileged session. Narrow sessions may instead be explicitly configured as direct preset (see parameters); the mode is declared in configuration and never switches at runtime.

### S4 - OAuth callback verification chain

- **Input:** a `request_connection` from a logged-in initiator, optionally carrying a pre-declared provider account identifier (`declared_account_id`).
- **Preconditions:** the initiator is authenticated; the provider flow uses authorization code + PKCE S256 with a confidential client, no exemptions.
- **Steps:** (a) Create a connect_session bound to the initiator, holding a high-entropy one-time `session_uri` (only its hash is stored server-side; the URI carries no user/account identifiers), TTL per the finalized parameters. (b) On callback, verify `state` maps to an unexpired connect_session still in INITIATED; exchange the code, reading the token-exchange target from the connect_session record - never from callback route parameters; an unknown state is rejected without calling any provider. (c) **Pre-sealing ownership pre-check:** read the consenting `provider_account_id`; if it matches a *different* known user's account identifier, fail the flow immediately - destroy any exchanged token, write a high-priority security event, show only a generic failure page. (d) Seal the credential as immutable pending-redemption. The state machine is single-transition: a second callback under the same `state` while PENDING_REDEMPTION is ignored, audited, and answered with 410 semantics - it never overwrites the sealed credential. (e) **Redemption:** the verifier endpoint requires the operator login; judge in order - (i) redeemer == initiator, else destroy the credential, FAIL, security event; (ii) if a pre-declaration exists and the provider supports identifier comparison, provider account == declared account, else treat as identity mismatch (destroy, FAIL, security event, generic failure page that never reveals the expected identifier); (iii) explicit confirmation - the page states "you are binding account <provider_account_id> (provider)" and the redeemer must actively confirm; there is no silent activation, and leaving without confirming leaves the credential sealed until TTL expiry destroys it. All three pass -> credential activates, connection belongs to the initiator, the confirmation is audit-logged, and the `session_uri` is voided immediately. (f) Delegated binding (binding an account that is not the initiator's own) is a separate flow, never this one: if the account belongs to another known user, that user confirms inside their own login; if it belongs to an external party, a policy administrator approves, fully logged; delegated connections are distinctly marked.
- **Output:** a connection in ACTIVE state owned by the initiator, or FAILED / EXPIRED with credentials destroyed.
- **Failure handling:** replay of a consumed token -> 410 `redemption_consumed`, no state change; expired -> 410 `redemption_expired`. Transient provider errors (network failure, provider 5xx during exchange) are **not** terminal - state is preserved and retry within TTL is allowed. Only identity mismatch and token expiry/replay are terminal failures. This separation is deliberate: it neither kills legitimate users for provider flakiness nor gives attackers a "transient error" story to argue for manual override.

### S5 - Audit, versioned update, and emergency block

- **Input:** the audit requirements of the system under review; the session/policy population in scope.
- **Preconditions:** audit storage is append-only; audit records carry identifiers and result codes only - never tool arguments, response bodies, credentials, or plaintext external account identifiers (account identifiers appear as digests).
- **Steps:** (a) Verify audit coverage: policy create/change (with direction grading and reviewer identity)/activation/archive/reference-integrity warnings; session create (with snapshot hash, pinning and audit-reproduction traces)/update/conflict rejections/blocked (including emergency-block actor, dimension, affected-session count)/close; execution rejections (including connection-check rejections); connection initiation/ownership pre-check hits/redemption success (with explicit-confirmation trace)/redemption mismatch/delegated confirmation/revocation; single-transition violations. (b) **Versioned update** (interactive sessions only): the request carries `expected_config_version`; on match, full re-validation, re-resolution, version increment, history append, and a receipt echoing the new effective set, bindings, and hash happen in one transaction. On mismatch -> 409 `session_config_version_conflict`, **zero fields written**, no history entry; neither server nor SDK retries automatically - the caller re-reads and explicitly decides. Rollback is expressed as a new version built from an old version's content; history is never rewritten. Updating a `rebuild_on_expiry` session this way -> 400 `session_update_mode_forbidden`. (c) **Fleet expiry-and-rebuild:** fleet sessions expire and are rebuilt against the then-current active policy; the 24h snapshot cap guarantees contractions reach the whole fleet in bounded time. (d) **Emergency block:** a policy administrator or security role may block all open sessions by policy or by tool dimension in one action - a contraction-direction, single-operator fast track that must not be slowed by review flow. Blocking changes session *state* only, never snapshot content; affected sessions refuse all execution with 503 `session_config_unavailable` and alert. Unblocking is allowed only after the snapshot hash re-validates, and is logged; manually flipping state to bypass validation is prohibited.
- **Output:** an audit-coverage verdict; update/block operations either fully applied with receipts or rejected with registered codes and zero partial writes.
- **Failure handling:** non-owner, non-admin attempts to modify or close another's session -> 403 `session_not_owner`, zero writes, audit written. Any code path shaped like "on error, continue with default full permissions" is a P0 defect - flag it, do not ship it.

## Dynamic execution contract

- **Steps:** S1 -> S5 above, executed in order for a full design/audit pass; a scoped pass must name the steps it covers and record the rest as not-run.
- **stop_conditions:** (normal termination) every in-scope step has produced its output or a registered-code rejection; every applicable acceptance test in `references/acceptance-tests.md` is mapped to pass / fail / not-run with an evidence pointer; the verdict pack is written.
- **abort_sentinels:** (abnormal termination) any step proposes connecting real credentials or production data before the implementation preconditions close; a design-semantic deviation is discovered (stop - register it as an open item, never patch semantics silently inside an implementation); two consecutive steps produce no verifiable evidence; a required registry or reference cannot be resolved and the fail-closed reading is contested.
- **Retry budget (two layers, both required):** `debug_iters` <= 3 per step (fix the step's own defect and re-run that step); `task_runs` <= 2 for a full re-run of the whole pass. Exceeding either layer stops the work and escalates with the evidence collected - no silent retries, mirroring the design's own no-auto-retry rule for conflicted updates.

## Outputs (the deliverable pack)

1. The artifact under review, annotated step by step (S1-S5) with verdicts.
2. Every rejection or defect cited by registered error code (`references/error-codes.md`) - never by invented codes.
3. An acceptance-test mapping against all 22 tests (`references/acceptance-tests.md`), with test 2 explicitly marked as the release gate.
4. An open-items list: unresolved references, unclosed implementation preconditions, and any registered design questions - each with an owner and a landing point.

## Scope limit (efficacy boundary - read before implementing)

This package guides **design and audit only**. The design it packages passed a dual-gate (compliance + security) review, but two things remain open until implementation review closes them: the 17 implementation-precondition items in `references/implementation-preconditions.md`, and the graded-activation-review condition (compliance Condition 1). Until all of them are closed **and** both gates have issued written closure records:

- any implementation of this design runs only in development environments with no real credentials, no production data, and no real user accounts;
- this package's publication does **not** constitute a production-access license for any hosted authorization layer, which requires its own separate approval.

## References

- `references/design-v0.2.md` - the canonical design, verbatim (Chinese).
- `references/error-codes.md` - the error-code registry (design section 6.3, verbatim extract).
- `references/acceptance-tests.md` - the 22 acceptance tests (design section 7, verbatim extract).
- `references/implementation-preconditions.md` - the 17 open implementation-review items plus the Condition 1 closure standard (summary register).
- `templates/policy.example.yaml` / `templates/session-create.example.json` - fictional example payloads.
- `evals/smoke.md` - three self-contained smoke evaluations.
