---
name: operator-session-permissions
description: Design and audit session-scoped permissions for tool-using agents. Use when creating, reviewing, or tightening what an agent session may do: named access policies resolved into an immutable snapshot at session creation, fail-closed defaults for missing or ambiguous configuration, a discover-then-load discipline for meta-tools, append-only audit of every refusal, and an emergency block for a whole fleet of sessions.
license: MIT
---

# Operator Session Permissions

A session is where an agent actually acts: which tools it may call, which
accounts it may touch, and in whose name. This skill makes that surface an
explicit, named, auditable fact, frozen at the moment the session is
created, instead of an understanding scattered across prompts, environment
variables, and defaults.

## The four principles

1. **Fail closed.** Missing, ambiguous, or corrupt configuration resolves
   toward fewer permissions or no execution, never toward unrestricted.
2. **Freeze at creation.** A session's permissions and bindings are
   resolved from a named policy into an immutable snapshot when the
   session starts. Editing the policy later does not reach back into
   live sessions; changing a live session goes through an explicit,
   versioned update path.
3. **Separate deciding from doing.** The component that resolves policy
   and the component that executes tools share exactly one contract: the
   snapshot. The executor can never widen it.
4. **Refusal leaves a trace and stops the line.** Every refusal (session
   creation rejected, update conflict, out-of-scope execution attempt)
   appends an audit event, and the executor stops and reports instead of
   shopping for another route around the refusal.

## How to use this skill

<!-- BODY: the full procedure lands with the sealed release.
     Sections to be completed:
     1. Writing a named access policy (allowlists, tiers, bindings)
     2. Creation-time resolution and the snapshot contract
     3. The fail-closed ruleset (unknown keys, empty allowlists,
        missing configuration, optimistic-concurrency updates)
     4. Session lifetime: expiry, rebuild, and the emergency block
     5. Meta-tool discipline (discover, then load, then call)
     6. Audit events: what to record and how to review them
     7. Acceptance checklist mapped to the reference material
-->

See `references/` for the design notes this skill is packaged from and
`README.md` for the repository's current status.

## Boundaries

- This skill governs permission design for agent sessions. It does not
  replace sandboxing, credential management, or human approval for
  irreversible actions; it composes with them.
- A policy you cannot enumerate is a policy you do not have. If a tool or
  binding cannot be named in the policy, the fail-closed answer is no.
