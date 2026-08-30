---
name: tides-review
description: Rigorous multi-angle adversarial code review that treats "green" as unproven. Use to really audit a change, PR, branch, or whole surface — not rubber-stamp it. Fans out independent adversarial angles (six by default) over the whole corpus in parallel, re-verifies every finding at the source (roughly half dissolve), red-proves the survivors, and re-runs until two consecutive passes come back clean. Its signature move is auditing every CLAIM in the words — comments, docstrings, test names, error/UI copy — against what the code actually enforces. Especially for money, auth, data-integrity, and consumer-facing UI, where a passing test can still hide a live bug.
---

# Tides Review

**A green build is not a clean build.** Tests pass, CI is green, and a day and a half of live defects sit underneath: a PII residue a name-list test was structurally blind to, money lost-updates, a migration that breaks the old revision mid-deploy, "fixes" whose own tests can't fail. This skill is the method that surfaces them. The discipline *is* the value — do not shortcut it.

## The core principle: green proves nothing on its own

A test can be green because it:
- asserts a **substring of the source**, not a behavior (it pins the spelling, not the code);
- never **exercises the failing case** (a single-owner fixture makes a cross-owner leak invisible);
- collapses to `could not compile` when you try to break it — which is **not** a red-prove;
- checks a real guard while a **sibling path** skips it.

So the review runs adversarial, verifies at the source, and red-proves. "It passes" is the start of the question, never the end.

## ⚑ The signature move: the devil is in the words

**Every comment, docstring, test name, error message, README line, and UI label is a CLAIM about behavior.** The author is telling you the guarantee they *intended*. The bug is where the code quietly fails to honor it. This is the highest-yield lens in the whole method — run it through every angle, and as an angle of its own.

1. **Harvest the claims.** Read the prose and collect the promises: "scoped to your account alone," "refunds on failure," "never logs the key," "checks the volume before charging," "refused in production," "this cannot happen," "the test proves X."
2. **For each claim, hunt the dishonouring candidate.** Find the code that must honor it, then adversarially look for the path, sibling, edge, or race where the words stay true but the behavior doesn't. A word that means two things ("empty" = no-widening or no-filter?) is a flashing light — find which one the code picked and whether the prose assumed the other.
3. **Distrust any claim a test supposedly proves.** The worst offender is "an assertion a comment satisfies": a test named for a behavior that only greps the source for a string, or seeds a fixture that can't trigger the failure. If the guarantee is real, a compiling mutation that breaks it must make a *behavioral* test fail. If it doesn't, the words are decoration.

Real examples this caught: a "your account alone" export that read an empty team-list as *no filter*; a `self_completing_a_checkout_is_refused_in_production` test that verified a substring, not a return; a "SQL-folded refund" doc a second caller could still double-fire; a purge whose comment said "31 of 33 tables cleared" while a name-blind residue scan never saw the 34th column.

## Step 1 — Pick the angles, then cover the whole corpus

Fan out **independent** angles, one background agent each, in parallel. Each angle is a DISTINCT adversary hunting a DISTINCT failure shape — do not collapse them. Choose the set by target.

**Whole repo / service — dimension angles:**
1. **Money** — charge / refund / metering / event ordering / provider-error classification (a non-JSON 5xx billed as success).
2. **Auth & cross-tenant** — session/token confusion, scope guards, roles/seats, deleted-user filtering.
3. **Data-integrity & deletion** — purge / residue / PII, reparent, orphans, FK-less identifying columns.
4. **Core output pipeline** (render / publish / generate) — preflight-before-charge, wrong-target, secret logging, malformed-response-as-success.
5. **Consumer UI** (see lens angles below).
6. **Infra** — migrations (expand-only / rolling-deploy-safe), the deploy gate, idempotency, un-locked read-modify-write.

**A single consumer-facing UI — lens angles (every button is law):**
1. **Dead / mis-wired controls** — a button that no-ops or fires the wrong action; disabled-forever; a label that lies.
2. **Async state truth** — no success-shown-on-failure, no failed-read-masked-as-empty, no stuck spinner, no un-rolled-back optimistic update.
3. **Wrong-target / stale selection** — a selection surviving an entity switch, a press-time read that's stale, cross-entity bleed.
4. **Race / double-submit / stale-closure** — double-click double-charge, out-of-order responses clobbering fresh state, effects over stale signals.
5. **Navigation / routing / deep-link / persistence** — dead-ends, lost work on nav, a gated screen reachable unauthed, storage that throws in private mode.
6. **Input edges / malformed data / rendering** — empty/huge/paste/malformed input or API row; crash, silent data loss, a bad active-state from a degenerate response, unescaped user text.

**Coverage is the multiplier, not the angle count.** Six agents skimming a large repo is a rubber stamp with extra steps. Chunk the corpus so **every part of the code is seen by every angle** — split into coherent slices (by crate/module/screen) and run each angle over each slice; on the reference sweep this was ~40 slices × 6 angles ≈ 240 passes. If a slice is too big for one agent to read closely, it re-delegates. Anchor the whole run to one **baseline commit** so findings are reproducible and re-runnable.

Run **the devil-in-the-words lens inside every angle**, and as its own pass if the surface is claim-dense.

For a **consumer UI**, static code review cannot catch every runtime/visual/interaction defect (a control that renders off-screen, a layout that breaks on overflow, a focus trap). Pair the code sweep with an actual interaction pass — a browser-driven or manual click-through of every screen — before calling it flawless.

## Step 2 — The per-angle agent prompt

Each agent: **read-only**, pinned to the baseline commit; hunt REAL, prod-reachable defects, not style; for each candidate open the actual code and confirm the mechanism exists and is reachable (not test-only, not dry-run, not already guarded by a sibling it didn't read); **discard anything that dissolves on a close read**; report `file:line` + a concrete failure (inputs → wrong outcome) ranked by severity; **no padding — an honest short list beats a padded one**; end with "no defect found in scope" when genuinely clean.

## Step 3 — Verify every finding at the source (never relay raw)

**~1 in 2 raised findings do not survive.** For each, open the cited `file:line` yourself and confirm the claim. A finding dies when it is: test-only or dry-run (not a prod path); already closed by a **sibling** the finding didn't read; citing a route/symbol that doesn't exist; a **stale doc** no caller obeys; or a theoretical race that can't actually interleave. Kill the phantoms. **Second-checker on dismissals:** have an independent pass re-check what you dismissed — a broken dismissal is a live bug wearing a "safe" label, and an independent checker is what makes the final "clean" credible. Relaying unverified findings is how a review loses trust; this step is non-negotiable.

## Step 4 — Red-prove each survivor

A real finding earns a **compiling** mutation that reverts the fix and makes a **behavioral** test fail — the line `test failed`, NOT `could not compile` / `error[E####]` (invalid proofs). Watch for confounds (an FK lock can make a naive concurrency test pass with or without the fix — isolate the actual guard, e.g. `FOR NO KEY UPDATE` vs `FOR UPDATE`). Confirm the test actually RAN (a bare fn name with `--exact` runs zero tests and exits 0). Red-proving also exposes a **partial fix** — a broadened scan that never seeded the new case, a guard added on one path only. If you cannot make a test fail, the finding — or the fix — is not real yet.

## Step 5 — Triage, fix, and re-run until clean twice

Rank by **blast radius**, not tidiness: money loss, cross-tenant leak, data-loss / PII residue, publish-to-wrong-audience, silent corruption come first. Triage every survivor into one of three, each recorded with its evidence:
- **FIX-NOW** — real and reachable. Fix through the normal gate, matching CI *exactly*: **pinned-toolchain** fmt + clippy (a default rustfmt version-skews and reddens CI), the library/migration invariants (e.g. an expand-only migration test), and the **FULL** test suite — clippy-clean is not enough (a change can pass clippy and break a downstream test). Re-run the angle that found it.
- **BANK** — real but latent / edge / self-healing (a drift a reconcile pass corrects, a 2-source edge). Document it precisely with the settling facts; don't silently drop it and don't ship a half-fix.
- **DISMISS** — dissolved on the source read. One line with the fact that settles it, for the second-checker.

Report each finding as: `id · severity · file:line · one-line mechanism · concrete failure scenario · red-prove line · verdict (CONFIRMED / BANKED / DISMISSED)`, plus the angles that came back clean. **A single clean pass is not done** — re-run the sweep after fixing; the bar is **two consecutive passes that surface nothing new**. That is what turns "green" into an *earned* clean rather than a rubber stamp.

## Step 6 — Done means shippable-clean, not just defect-free

Two consecutive clean passes prove the code; they do not prove the *project*.
"Finished" carries two more obligations, checked as deliberately as any angle:

- **Clean documentation.** The docs a newcomer actually needs exist and are
  TRUE: what the system is, how to run it, how to deploy it, the invariants
  it enforces, and the seams it exposes. Docs are claims (see the signature
  move) — a README that describes a flag that no longer exists, or a setup
  step that no longer works, is a finding, same as a lying comment. Stale
  docs get fixed or deleted, never left "to be safe."

- **Cruft cleaned out — from the repo AND off the device.** The build leaves
  droppings; sweep them before calling it done:
  - *In the repo:* dead scaffolding, commented-out code, orphaned files,
    stale TODOs that got done, prompt/handoff files that only mattered
    mid-build, build artifacts that leaked past .gitignore, branches whose
    work is merged.
  - *On the device:* temporary worktrees, scratch databases, throwaway
    scripts in temp dirs, stale background watchers/sessions the build
    spawned, downloaded fixtures. If it only existed to get the build done,
    it goes. Anything kept on purpose gets named in the docs with why.

The review's final report states all three: defects (clean twice), docs
(accurate and sufficient), cruft (repo and device swept) — and it is not
"done" until every column says so.

## When not to reach for this

A one-line change with an obvious blast radius does not need six agents over the whole corpus. This is for surfaces that ship to users, touch money/auth/data, or are about to be called "done." When in doubt on something consumer-facing, run it — the cost of the sweep is an hour; the cost of a green lie is a day and a half.
