# CLAUDE.md

<!-- PRAWDUCT:ANCHOR — governance pointer managed by the prawduct plugin; keep it small and version-free. -->

## Governance (Prawduct)

This repo is governed by **Prawduct**, a Claude Code plugin; its methodology and
protocols are read on demand via `/prawduct:methodology`.

**Check first: is the plugin loaded?** If `/prawduct:*` commands are unavailable it
is not, and **governance is OFF** — no Stop gate, no Critic, nothing below enforced.
A clone registers the marketplace but installs nothing. Tell the user to
run `claude plugin install prawduct@prawduct`, then restart — don't proceed as if governed.

**With the plugin loaded — before writing any code, STOP and read the build cycle:
`/prawduct:methodology building`.** Skipping it is the #1 governance failure.

Hardest rules:

- **Tests are contracts** — fix the code, never weaken a test.
- **No "pre-existing" exception** — fix what you find, or flag why you can't.
  The fix half is bounded to BLOCKING; below it, the flag is the whole answer.
- **Never silently drop a requirement** — say so explicitly.
- **Run `/prawduct:critic` after medium+ work** — never write findings
  yourself; the independence is the value. Rigor is stage-keyed: a mid-build
  review blocks only on what would ship broken, the review at the merge
  boundary runs everything and is never skipped, and unsure defaults to the
  cheaper mid-build review.

**Enforcement is structural — while the plugin is loaded:** its Stop hook runs at
session end and **blocks** if code changed against an active build plan with no
Critic findings.
