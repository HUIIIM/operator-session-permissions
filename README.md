<h1 align="center">operator-session-permissions</h1>

<p align="center"><strong>What a session can do should be a fact, not a vibe.</strong><br/>
A skill pack for designing agent session permissions: named policies, frozen at creation, fail-closed by default, audited at every refusal.</p>

<p align="center">
<img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-green"/>
<img alt="Format: SKILL.md" src="https://img.shields.io/badge/format-SKILL.md-blue"/>
<img alt="Status: packaging" src="https://img.shields.io/badge/status-packaging-yellow"/>
</p>

---

An agent session is the layer that actually acts: the tools it can call, the accounts it can touch, the identity it acts under. In most setups that surface is implicit, spread across prompts, environment variables, and library defaults, and it drifts. This pack turns it into an explicit design:

- **Named access policies.** What a class of sessions may do is written down once, under a name, and reviewed like any other contract.
- **Frozen at creation.** When a session starts, its policy resolves into an immutable snapshot. Later policy edits do not silently widen live sessions.
- **Fail-closed rules.** Unknown configuration keys are rejected, an explicitly empty allowlist means zero tools, and missing configuration never falls back to an unrestricted session.
- **Refusals are events.** Every refusal is appended to an audit trail, and the executor stops instead of routing around it.

## Repository status

This repository is the packaging shell for the skill. The structure is final; the skill body is being packaged from its design document and lands with the sealed release.

```
SKILL.md        The skill itself (frontmatter + body; body in packaging)
references/     Design notes the skill is packaged from
scripts/        Helper scripts shipped with the skill
CHANGELOG.md    Release history
```

## The model in one screen

| Question | This pack's answer |
|---|---|
| Who decides what a session may do? | A named policy, resolved before the session exists |
| When is it decided? | At creation, into an immutable snapshot |
| What if configuration is missing or malformed? | The session is not created, or runs with fewer permissions |
| What if the agent tries something out of scope? | Refusal, an audit event, and a full stop |
| How do permissions change mid-flight? | Only through a versioned update with conflict detection, never silently |
| How do you stop everything at once? | A fleet-level emergency block that sessions check before acting |

## Limits

- This pack designs the permission layer. It does not ship a hosted service, and it does not hold credentials for you.
- Freezing permissions at creation trades flexibility for auditability. Short-lived sessions should expire and be rebuilt rather than endlessly updated.
- No permission design survives an executor that ignores it. The pack's acceptance checks exist to catch exactly that.

## License

MIT. Copyright (c) 2026 Jiahui Miao. See [LICENSE](LICENSE).
