# Smoke Evaluations

Three self-contained drills. Each can be run with only this package as material: give the executing agent the prompt and the listed references, collect the expected output, and grade against the fail criteria. All identifiers in the scenarios are fictional.

---

## Case 1 - Fail-closed configuration audit

**Prompt:**
"You are auditing a proposed session-create implementation against the operator-session-permissions design. The create handler behaves as follows: (a) it validates only top-level keys and silently ignores unknown keys inside nested objects such as the preload list; (b) when the `enable` field is absent from the payload, it treats the session as unrestricted, on the theory that 'nothing was restricted'; (c) when policy lookup fails with a storage timeout, it retries once and, if the retry also fails, creates the session with a built-in default policy that enables the full mail toolkit so the user is not blocked. Identify every defect, cite the registered error code or design rule each one violates, state the correct behavior, and give a pass/fail verdict for the handler."

**Materials:** `SKILL.md` (S2, S3 preconditions), `references/error-codes.md`, `references/design-v0.2.md` sections 3.1-3.3, `references/acceptance-tests.md` tests 1, 2, 4.

**Expected output:**
- (a) is a defect: strict schema validation recurses into every nesting level; any unknown key rejects the whole request with 400 `session_config_unknown_key`, lists all unknown keys with paths, and writes zero configuration (acceptance test 1).
- (b) is a defect and the most dangerous one: absent and empty `enable` both normalize to **zero grants**; an unrestricted reading of an absent field inverts the fail-closed rule (acceptance test 2, which is the release gate).
- (c) is a defect twice over: configuration-missing cases must fail, never degrade - and "on error, continue with default full permissions" is the design's named P0 defect shape. Correct behavior is refusal at creation (or `blocked` + 503 `session_config_unavailable` at runtime load), with an operations alert (acceptance test 4).
- Verdict: FAIL, with all three defects cited.

**Fail criteria (the run fails if any holds):**
- Any of the three defects is missed or excused (e.g. calling (b) "acceptable UX" or (c) "reasonable resilience").
- An error code is invented, misnamed, or the release-gate status of test 2 is not mentioned for defect (b).
- The correct behavior for any defect permits partial writes or a fallback grant.

---

## Case 2 - Snapshot and version-pinning review

**Prompt:**
"Review this sequence against the design and rule on each step. A policy 'ops-mail' is at version 3. A security contraction ships version 4 (one write tool removed); the administrator marks version 3 as security_contracted. Afterwards: (1) an ordinary caller creates a session explicitly pinning version 3, arguing the contraction broke their workflow; (2) an audit-role caller pins version 3 to reproduce an incident; (3) the same audit-role caller pins version 2, a formerly-active version with no security_contracted mark, for a different reproduction; (4) the policy owner edits the active policy to widen a toolkit, and an already-running long-lived session immediately gains the new tool without restart; (5) a fleet (scheduled-task) session is patched in place via the versioned-update endpoint to pick up the contraction faster. For each step: allowed or rejected, under which rule, with which error code where applicable, and what the audit trail must show."

**Materials:** `SKILL.md` (S1, S2, S5; mental model), `references/error-codes.md`, `references/design-v0.2.md` sections 1.3, 1.4, 3.4, 3.5, `references/acceptance-tests.md` tests 5, 6, 12.

**Expected output:**
- (1) Rejected: ordinary callers may pin only the current active version; 400 `policy_version_not_pinnable`, zero writes.
- (2) Rejected: a `security_contracted` version cannot be pinned by anyone, including the audit role; the auditor reads the version's content read-only instead. Same error code.
- (3) Allowed: audit reproduction is the sole pinning exception - formerly active, unmarked version, audit role, session flagged `audit_repro`, audit event records caller, pinned version, and reproduction rationale.
- (4) Defect: policy edits never reach back into existing sessions; a session executes its frozen snapshot for its whole life. The widening also grades as an expansion change requiring second-person review judged by the server-evaluated effective tool-set diff before it can take effect for *new* sessions.
- (5) Rejected: fleet sessions are `rebuild_on_expiry`; versioned update against them returns 400 `session_update_mode_forbidden`, zero writes. The contraction reaches fleet sessions by expiry-and-rebuild, bounded by the 24-hour snapshot cap, or immediately via the emergency block.

**Fail criteria:**
- Any of steps (1), (2), (5) is ruled allowed, or step (3) is ruled rejected.
- Step (4) is not identified as a snapshot-freeze violation.
- The emergency block is not named as the immediate-effect path for step (5)'s underlying need, or the 24-hour fleet cap is misstated (e.g. replaced with a blanket 10-minute fleet TTL).

---

## Case 3 - OAuth callback verification review

**Prompt:**
"Walk this connection flow and rule on the outcome at each numbered point. Initiator A starts a connection for provider 'example-mail' without pre-declaring an account. The authorization link is forwarded, and person B completes the consent using B's own provider account. (1) At callback, the implementation exchanges the code and seals the credential; B's provider account identifier happens to match the account identifier of a different known user of the platform. (2) In a parallel run where the account matches nobody known, A opens the verifier page; the page shows only a generic 'Confirm connection' button with no account identifier, and A clicks it. (3) Before A redeems, a second callback arrives carrying a different, valid code with the same state; the implementation exchanges it and overwrites the sealed credential 'to keep the freshest one'. (4) A then redeems the original (now-overwritten) flow and the connection activates under A's ownership - with the credential from B's account from step (3). State what the design requires at each point, which steps already failed the flow, and what the final state of the connection and the credential must be."

**Materials:** `SKILL.md` (S4), `references/error-codes.md`, `references/design-v0.2.md` sections 4.2-4.4, `references/acceptance-tests.md` tests 7, 8, 15, 16.

**Expected output:**
- (1) The ownership pre-check runs **before** sealing: a consent account matching a different known user fails the flow at that point - the exchanged token is destroyed, no usable credential is sealed, a high-priority security event is written, and the browser sees only a generic failure page. Sealing first and checking later violates the design.
- (2) Defect: with no pre-declaration, redemption runs on the explicit-confirmation track - the page must state the provider account identifier being bound ("you are binding account <id> (provider)") and the initiator must actively confirm; a generic button with no identifier is a silent-activation path and is prohibited. The confirmation is audit-logged.
- (3) Defect: the connect_session state machine is single-transition. A second callback while PENDING_REDEMPTION is ignored and audited, and answered with 410 semantics; the sealed credential is immutable and is never overwritten.
- (4) The described activation is invalid end-to-end: given (1), the flow should already be FAILED with credentials destroyed; given (3), step (4)'s credential could never have been replaced. Correct final state: connection FAILED (or, in the parallel no-match run, ACTIVE only after identifier-showing explicit confirmation by A), any improperly sealed credential destroyed, and security/audit events covering the pre-check hit, the second-callback violation, and the redemption attempt. Replay of the consumed token returns 410 `redemption_consumed` with no state change.

**Fail criteria:**
- The pre-sealing position of the ownership check in (1) is missed, or sealing-then-checking is accepted.
- The missing account identifier in (2) is treated as a cosmetic issue instead of a prohibited silent-activation path.
- The overwrite in (3) is accepted under any rationale ("freshest wins", "same state means same flow").
- The final ruling lets the connection stand as ACTIVE with B's credential, or omits credential destruction / security events from the required end state.
