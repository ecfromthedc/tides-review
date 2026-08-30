---
name: tides-review
description: The closer — a rigorous multi-angle adversarial review that treats "green" as unproven, then finishes the job. Use to close out a build, PR, branch, or whole surface before it's called done. Establishes scope truth first (was the promised thing actually built?), fans out independent adversarial angles over the whole corpus in parallel, re-verifies every finding at the source (roughly half dissolve), red-proves the survivors AND the test suite's own invariant coverage, fixes everything confirmed, and re-runs until two consecutive passes come back clean. Then it earns "finished": documentation that tells the truth, cruft swept from repo and device, a mainline carrying the whole build, and a closing report a stakeholder can sign off from cold. Its signature move is auditing every CLAIM in the words — comments, docstrings, test names, error/UI copy — against what the code actually enforces. Especially for money, auth, data-integrity, and consumer-facing UI, where a passing test can still hide a live bug.
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

## Step 0 — Scope truth: audit the promise before the code

Defect-hunting an unfinished product produces the most dangerous report there is: "no bugs found" in a thing that was never built. Before any angle runs:

1. **Rebuild the promise list.** Read the goal prompt, brief, acceptance criteria, ticket — whatever defined "done" — and extract every concrete promise ("an artist can submit in under 15 minutes", "statements reconcile to the cent", "viewers cannot mutate").
2. **Verdict each promise: MET / PARTIAL / MISSING**, with evidence — the route/function exists, a behavioral test exercises it, and you walked it. A promise with code but no path a user can actually reach is PARTIAL, not MET.
3. **Walk the product cold.** Run the primary user journey the goal names, as a user, end to end. A closer *uses* the thing; a journey that dead-ends is a finding of the highest class regardless of how clean the code is.

MISSING and PARTIAL scope items enter triage alongside defects — a lie of omission outranks most bugs.

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

**Red-prove the suite, not just the findings.** Findings only test what an angle happened to see; the suite's blind spots stay dark. So enumerate the system's load-bearing invariants explicitly — money conserved to the cent, tenants isolated, the state machine closed, idempotency under replay, fail-closed on unknown input — and for each one, break it deliberately on a scratch branch and watch which test goes red. **An invariant no test defends is a FIX-NOW finding in its own right** (the deliverable is the missing test), because every future regression against it will ship green.

## Step 5 — Triage, fix, and re-run until clean twice

Rank by **blast radius**, not tidiness: money loss, cross-tenant leak, data-loss / PII residue, publish-to-wrong-audience, silent corruption come first. Triage every survivor into one of three, each recorded with its evidence:
- **FIX-NOW** — real and reachable. Fix through the normal gate, matching CI *exactly*: **pinned-toolchain** fmt + clippy (a default rustfmt version-skews and reddens CI), the library/migration invariants (e.g. an expand-only migration test), and the **FULL** test suite — clippy-clean is not enough (a change can pass clippy and break a downstream test). Re-run the angle that found it. **Fix-commit discipline:** each fix lands with the regression test that would have caught it and *nothing else* — no drive-by refactors riding along in a closing commit; a fix without its pinning test is half a fix.
- **BANK** — real but latent / edge / self-healing (a drift a reconcile pass corrects, a 2-source edge). Document it precisely with the settling facts; don't silently drop it and don't ship a half-fix.
- **DISMISS** — dissolved on the source read. One line with the fact that settles it, for the second-checker. (The Step-3 second-checker pass re-reads every one of these.)

Report each finding as: `id · severity · file:line · one-line mechanism · concrete failure scenario · red-prove line · verdict (CONFIRMED / BANKED / DISMISSED)`, plus the angles that came back clean. **A single clean pass is not done** — re-run the sweep after fixing; the bar is **two consecutive passes that surface nothing new**. That is what turns "green" into an *earned* clean rather than a rubber stamp.

## Standing angles — never optional, whatever else you pick

Four angles have earned a permanent seat because they catch what a green gate is *structurally* blind to:

- **Live-substrate verification.** Runtime-bound SQL, env-driven config, and vendor adapters can be compile-green and wrong. Run the gated live suites against a real scratch substrate (database, queue, filesystem); boot the actual binary and hit its critical endpoints. A test suite that never touches the real substrate cannot vouch for it — the canonical case is a foreign-key violation that no amount of clippy will ever see.

- **Concurrency & idempotency.** Hunt TOCTOU: every cap, uniqueness, or balance rule must be enforced *inside* the transaction that writes, with the row locked — a check outside the lock is a race with a polite error message. Every externally-triggered effect (webhook, payout, queue claim, delivery) must be provably once-only under replay AND under crash-between-two-writes: ask "if the process dies on this exact line, what does the retry do?"

- **Degraded-mode honesty & leak sweep.** Walk every unconfigured/missing-credential path: it must refuse loudly (503, typed error), never simulate success — a stub that fakes the happy path is a defect even with no bug behind it. Then sweep what crosses boundaries: error `Display` impls that quote provider bodies onto the wire, logs that capture bearer capabilities (presigned URLs, session tokens, checkout links), diagnostics that leak schema.

- **Supply chain & secrets.** `cargo audit` (or ecosystem equivalent) clean or consciously waived; RC-grade dependencies pinned exact; the dependency tree diffed against intent (a review that never looks can't catch the accidental 300-crate import); repo and history grepped for credential-shaped strings.

And two mechanical sweeps cheap enough to always run:

- **The bounds table.** Every outbound call has a timeout; every list endpoint has a limit; every ingest has a size ceiling; every retry has a cap and backoff; every queue claim has a lease. Grep the call sites and fill the table — an unbounded anything is a finding, because production will find it for you.
- **The env inventory.** Grep every environment read the code performs. Each variable is documented, classified (required / optional / secret), and its absence has a *stated* degrade behavior that matches what the code actually does. An env var the docs don't know about is drift; a documented one the code no longer reads is a lie.

## Step 6 — Done means shippable-clean, not just defect-free

Two consecutive clean passes prove the code; they do not prove the *project*.
"Finished" carries two more obligations, checked as deliberately as any angle:

- **Clean documentation, proven by the README walk.** The docs a newcomer
  actually needs exist and are TRUE: what the system is, how to run it, how
  to deploy it, the invariants it enforces, and the seams it exposes. Docs
  are claims (see the signature move) — so prove them the only way that
  counts: **cold-start the project following the docs verbatim**, from
  clone to running system, deviating for nothing. Every place you had to
  know something the docs did not say is a finding. A README that describes
  a flag that no longer exists, or a setup step that no longer works, gets
  fixed or deleted, never left "to be safe."

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

- **A mainline that tells the whole story.** Everything "finished" claims
  to include is ON main — no work stranded on unmerged branches, no open
  PRs quietly holding pieces the release notes assume, no force-push scars
  or WIP-noise commits a repo auditor would trip over. Every feature/seat
  branch is either merged or deliberately closed with its reason recorded;
  the commit history reads as a legible account of what was built and why.
  Someone opening the repo cold should see a build that looks finished —
  because on main, it is.

The review's final report states all four: defects (clean twice), docs
(accurate and sufficient), cruft (repo and device swept), mainline
(complete and legible) — and it is not "done" until every column says so.

## Step 7 — Turn it in

A+ work is *turned in*, not just done. The close produces one document of record — `CLOSING-REPORT.md` in the repo (or the PR body for a scoped close) — written for a cold reader:

1. **The promise list with verdicts** (Step 0): every acceptance item MET / PARTIAL / MISSING, with evidence links. Nothing hand-waved.
2. **The four columns** (Steps 5–6): defects (clean twice — say which passes), docs (walked cold), cruft (repo and device, what was swept), mainline (branches/PRs accounted for).
3. **The BANK register**: every banked finding with its settling facts — the next engineer inherits knowledge, not surprises.
4. **The go-live checklist**: exactly what stands between this state and production — credentials to set, env vars to fill, accounts to provision — each with where it plugs in.
5. **Deliberate non-goals**: what was consciously not built, so absence reads as a decision rather than an oversight.

The grade for this document is a single question: **could a stakeholder who watched none of the work sign off from this report alone?** If they would have to ask a question first, the report is not finished — and neither is the close.

## When not to reach for this

A one-line change with an obvious blast radius does not need six agents over the whole corpus. This is for surfaces that ship to users, touch money/auth/data, or are about to be called "done." When in doubt on something consumer-facing, run it — the cost of the sweep is an hour; the cost of a green lie is a day and a half.
