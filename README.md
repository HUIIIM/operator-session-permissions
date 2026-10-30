# operator-session-permissions

A publishable skill package that makes "what is this agent session allowed to do?" an explicit, named, auditable fact frozen at creation time - named access policies, creation-time snapshot freeze, a meta-tool three-stage discipline, a fail-closed rule set with a registered error-code table, and redemption-style OAuth callback verification for tool-executing agents.

## Package structure

```
operator-session-permissions/
|-- SKILL.md            # the skill itself: mental model, workflow S1-S5, execution contract
|-- README.md           # this file
|-- references/
|   |-- design-v0.2.md                  # canonical design, verbatim (Chinese)
|   |-- error-codes.md                  # error-code registry (design section 6.3 extract)
|   |-- acceptance-tests.md             # 22 acceptance tests (design section 7 extract)
|   `-- implementation-preconditions.md # 17 open implementation-review items (summary)
|-- templates/
|   |-- policy.example.yaml             # fictional named-policy example
|   `-- session-create.example.json     # fictional session-create payload example
`-- evals/
    `-- smoke.md                        # three self-contained smoke evaluations
```

## Usage

Load `SKILL.md` as the skill entry point. It runs a five-step workflow - S1 policy definition and graded review gate, S2 session creation and snapshot verification, S3 meta-tool three-stage execution discipline, S4 OAuth callback verification chain, S5 audit, versioned update, and emergency block - and cites `references/error-codes.md` for every rejection and `references/acceptance-tests.md` for verification. The templates are fictional examples of the two payload shapes; the evals are ready-to-run audit drills.

## Design source and version

The canonical design is design v0.2, finalized on 2026-10-16 after passing a dual-gate (compliance + security) review; it is reproduced verbatim as `references/design-v0.2.md`. Two conflict notes left open in earlier drafts (fleet-session TTL values, and how the two review-grading axes combine) were closed by a governance confirmation memo dated 2026-10-16; where any older wording disagrees, that memo prevails - the settled values are summarized in the "Finalized parameters" table of `SKILL.md` (session URI TTL of 10 minutes in a single segment; fleet snapshot lifetime capped at 24 hours with per-cadence TTLs; dual-axis review grading with the stricter axis winning, so read-only expansion changes take dual review).

## Scope limit

This package guides design and audit only. Until the implementation preconditions in `references/implementation-preconditions.md` are closed in implementation review and both review gates have issued written closure records, any implementation of this design runs only in development environments with no real credentials, no production data, and no real user accounts; publication of this package does not constitute a production-access license for any hosted authorization layer, which requires separate approval.

## License

License: see the LICENSE file in this repository (to be finalized by the publishing line at repository ingest).
