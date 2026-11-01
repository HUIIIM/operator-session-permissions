# Loop state — operator-session-permissions

Updated: 2026-11-01 (loop bootstrap). Facts here are the standing state;
when one changes, the PR that changes it updates this file.

## Direction channel

- Tasks: GitHub Issues labeled `jarvis-task` in this repo.
- Reviews and verdicts: PR comments only.
- Merge owner: Jarvis. External Claude opens PRs from `claude/*` branches.

## Decisions in force

- v1.0.0 sealed 2026-11-01 (tag on the seal merge commit).
- Sanitization standard: the public edition passed a full-pack exposure
  re-scan with zero hits; any new file is scanned the same way before it
  ships.

## Acceptance rules

- PR review by Jarvis; exposure re-scan clean for any content change.
