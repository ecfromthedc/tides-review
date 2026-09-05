# Tides Review

**A green build is not a clean build.**

Tides Review is **the closer**: a rigorous, multi-angle adversarial review-and-finish methodology — packaged as a [Claude Code](https://claude.com/claude-code) skill. It exists because a codebase can have every test passing and CI green while a day and a half of *live* defects sit underneath: a PII residue a name-list test was structurally blind to, money lost-updates, a migration that breaks the old revision mid-deploy, "fixes" whose own tests can't fail. And because "no bugs" is not the same as "finished" — the closer also audits the build against what was actually promised, fixes what it confirms, sweeps the cruft, and turns in a report a stakeholder can sign off from cold.

It is built to be as thorough as a heavyweight cloud review, and it earns its "clean" instead of asserting it.

## The idea in one line

Establish **scope truth** first (the promise list: MET / PARTIAL / MISSING — "no bugs found" in a thing that was never built is the most dangerous report there is). Fan out independent adversarial **angles** over the whole corpus in parallel, **re-verify every finding at the source** (roughly half dissolve), **red-prove** the survivors — and the test suite's own invariant coverage — with compiling mutations that make real tests fail, **fix everything confirmed**, and **re-run until two consecutive passes come back clean.** Then earn "finished": documentation proven by a cold README walk, every piece of build cruft swept out of the repo and off the device, a mainline that carries the whole build with history legible cold, and a closing report that answers every question a stakeholder would ask.

## The signature move: the devil is in the words

Every comment, docstring, test name, error message, and UI label is a *claim* about behavior — the guarantee the author intended. The bug is where the code quietly fails to honor it. Harvest the claims, hunt the dishonouring path for each, and **distrust any guarantee a test supposedly proves** — because the worst offender is "an assertion a comment satisfies": a test named for a behavior that only greps a substring.

This single lens surfaced the most: a "your account alone" export that read an empty team-list as *no filter*; a `refused_in_production` test that checked a string not a return value; a "SQL-folded refund" doc a second caller could still double-fire.

## What's in the box

- **[`SKILL.md`](./SKILL.md)** — the full playbook. Self-contained; readable as a methodology on its own, or invoked as a Claude Code skill.

The method, briefly:

1. **Pick the angles** — eight by default. A *dimension* set for a whole service (money, auth & cross-tenant, data-integrity & deletion, core output pipeline, consumer UI, infra, day zero & degenerate states, and **existence, authority & correction** — the angle that asks whether a *second* person can use the product at all, on what authority it acts, and whether a correction can even be expressed), or a *lens* set for a single consumer UI (dead controls, async-state truth, wrong-target, race/double-submit, navigation/persistence, input-edges, accessibility). Plus the devil-in-the-words pass through all of them.
2. **Cover the whole corpus** — coverage is the multiplier, not the angle count. Chunk the code so every part is seen by every angle.
3. **Verify at the source** — never relay a raw finding; ~1 in 2 dissolve. Independently second-check what you dismiss — a broken dismissal is a live bug wearing a "safe" label.
4. **Red-prove** — a compiling mutation that reverts the fix and makes a *behavioral* test fail (`test failed`, not `could not compile`).
5. **Triage & re-run** — rank by blast radius; fix / bank / dismiss with evidence; gate the way CI actually gates; the bar is two consecutive clean passes. A red waved off as "flake" must be proven flake — an unproven dismissal is itself a finding. Every angle's report ends with a coverage receipt; a pass counts only when the receipts cover the chunk plan.
6. **Turn it in clean** — the last gate is the documentation itself: re-read every doc cold after the fixes, walk the quickstart verbatim, cross-check sibling docs for contradicting claims, and grep out every stale reference. A close whose docs lie is not done.

## Install as a Claude Code skill

Clone into your user skills directory (available in every project):

```bash
git clone https://github.com/ecfromthedc/tides-review ~/.claude/skills/tides-review-repo
cp ~/.claude/skills/tides-review-repo/SKILL.md ~/.claude/skills/tides-review/SKILL.md   # or symlink
```

Or drop `SKILL.md` at `~/.claude/skills/tides-review/SKILL.md` (user-level) or `.claude/skills/tides-review/SKILL.md` (project-level). Then, in Claude Code:

```
/tides-review
```

and point it at the change, branch, PR, or surface you want audited.

## Use as a methodology (no tooling required)

`SKILL.md` is a plain playbook. Read it, and run the passes yourself or with any agent framework — the discipline is the value, not the harness.

## When not to reach for it

A one-line change with an obvious blast radius does not need eight agents over the whole corpus. Tides Review is for surfaces that ship to users, touch money/auth/data, or are about to be called "done." The cost of the sweep is an hour; the cost of a green lie is a day and a half.

## License

MIT — see [`LICENSE`](./LICENSE).
