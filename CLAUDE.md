# operator-session-permissions

A published skill package (MIT, Copyright (c) Jiahui Miao): a permission
model for operator sessions — policy templates, session-create examples,
and an evaluation smoke test. Sealed at v1.0.0 on 2026-11-01.

## How work happens here

- Direction arrives in-repo: GitHub Issues labeled `jarvis-task`, PR
  review comments, and `CLAUDE.md` / `LOOP.md`. Read both at session start.
- External Claude works on `claude/*` branches and opens PRs; it never
  merges its own PRs. Jarvis reviews and merges.
- Review verdicts exist only as PR comments.

## Working rules

- This is a sanitized public edition: it contains no internal coordinates
  (no internal file paths, case numbers, chat identifiers, or internal
  decision numbering). Contributions must keep it that way — mechanism and
  semantics only.
- Releases are versioned: changes land on `main`, and published versions
  are tagged (v1.0.0 is the seal of the first public edition).
- The internal implementation-preconditions record is deliberately not
  part of this package.
