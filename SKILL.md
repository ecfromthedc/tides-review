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
- checks a real guard while a **sibling path** skips it;
- asserts its **own local definition** instead of the shipped code; reads its **own text** (an `include_str!` with the needle spelled beside it); slices a haystack on a marker that **appears twice**; or drains to a **clamp floor** (`GREATEST(x,0)` cannot tell a refund of 1 from a refund of 2 — seed counters ABOVE the amount so magnitude and idempotency are observable);
- or — the deepest lie — belongs to code that was **never compiled at all**: a "feat" commit can land whole files with no `mod`/import declaration anywhere, green by nonexistence, its in-file tests never once run, while a migration it shipped runs live. (Real case: a complete 1,900-line OIDC login flow, landed with a confident commit message, never declared, never built — found only when an angle grepped for the declaration.)

So the review runs adversarial, verifies at the source, and red-proves. "It passes" is the start of the question, never the end.

**The agent era raises the bar on both sides.** Findings arrive from agents (~half dissolve on a close read — Step 3), and increasingly the FIXES arrive from agents too. Both get the same treatment: a fix lane's output is re-verified and re-red-proven by an independent checker, ideally a *different model* — a broken dismissal, a false red-proof, or a fix that guards one door of three all read as "clean" to the lane that produced them, and only fresh adversarial eyes catch their own class of error. Assume the world will run agents against this code looking for holes; run yours first, and check the checkers.

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
3. **Walk the product cold — and walk it as the SECOND user.** Run the primary user journey the goal names, as a user, end to end. A closer *uses* the thing; a journey that dead-ends is a finding of the highest class regardless of how clean the code is. Then walk it again as the second user, the second tenant, the second payee. The first principal's journey is the one every fixture, demo and founder rehearses: it is the most likely to work and the least likely to be representative. A product that cannot hold a second person is not a product with a bug in it.

MISSING and PARTIAL scope items enter triage alongside defects — a lie of omission outranks most bugs.

4. **Check the code is even there.** For every feature the promise list names, verify it is *compiled and reachable*: the module is declared (`grep "mod <name>"` / the import graph reaches it), the route is mounted, the binary carries it. A file with zero inbound references is uncompiled words wearing a commit message — its guarantees don't exist and its tests have never run.

## The prose-drift result — read this before trusting any written claim

One class in this method has never been closed by care, and the record is worth stating exactly, because it is the strongest empirical result the method has produced.

A test file excluded ~22 database constraints from a drift census, each with a written reason. The reasons were validated by nothing but `reason.len() > 20`. What followed, over one project:

1. Five reasons were found false.
2. They were rewritten specifically to fix that. Two of the rewrites were false.
3. A full audit found **six** of the 22 false — four that six adversarial passes across five model families had never noticed.
4. The lane fixing them found its **own first rewrite** was false, and caught it only after building a resolver.
5. Its **committed** rewrite was still false: it claimed two tokens "exist solely as SQL literals in test fixtures" when those tokens appear in zero source files anywhere.

Seven iterations. Careful humans, five model families, and a dedicated fix lane all failed to close it; a mechanical resolver closed most of it in one run. **The lesson is not "review harder."** It is that a written claim about code is unreliable in proportion to how little of it a machine checks, and that the honest response is to shrink the unchecked surface rather than to re-read it.

Two practical rules fall out:

- **Mechanise the claim classes that are cheap, even if you cannot mechanise all of them.** That resolver checked *presence* — every symbol or path a reason names must resolve — and deliberately skipped *absence* claims ("nothing in this workspace reads this") as too large a surface. That judgement was one notch too conservative: the token-level form ("this string appears nowhere in source") is a single grep, and it is exactly the form that survived to iteration seven. Give reason syntax `[present:]` / `[absent:]` predicates so the checkable half is checked and general prose stays free.
- **A red-proof is a claim too, so store it and replay it.** The durable countermeasure to the whole false-proof family is to keep each red-proof as a mutant diff that CI replays, requiring the named test to actually *fail* — and to fail behaviourally, not fail to compile. Every other countermeasure decays: reviewers rotate, angles drift, and a proof that was watched once is thereafter only a sentence in a commit message. Its blind spot is honest and worth naming: a wrong mutant replays green forever, so a stored proof is only as good as the quadrant it visits.

A closing report is subject to all of this. Cite test and function **names**, which survive edits, rather than line numbers, which rot silently — and run the same resolver over the report itself.

## Step 1 — Pick the angles, then cover the whole corpus

Fan out **independent** angles, one background agent each, in parallel. Each angle is a DISTINCT adversary hunting a DISTINCT failure shape — do not collapse them. Choose the set by target.

**Whole repo / service — dimension angles:**
1. **Money** — charge / refund / metering / event ordering / provider-error classification (a non-JSON 5xx billed as success).
2. **Auth & cross-tenant** — session/token confusion, scope guards, roles/seats, deleted-user filtering.
3. **Data-integrity & deletion** — purge / residue / PII, reparent, orphans, FK-less identifying columns.
4. **Core output pipeline** (render / publish / generate) — preflight-before-charge, wrong-target, secret logging, malformed-response-as-success.
5. **Consumer UI** (see lens angles below).
6. **Infra** — migrations (expand-only / rolling-deploy-safe), the deploy gate, idempotency, un-locked read-modify-write.
7. **Day zero & degenerate states** — the states a mature test fixture never visits: fresh install, empty database, zero rows in every list, no provider configured, the first user, the first request after boot, the documented quickstart run *verbatim* on a machine that has never seen the project. Day-zero bugs ship green because every fixture pre-seeds past them (real case: the exact `cargo run` command in three docs broken for weeks by an added second binary — caught only by the verbatim cold-start walk). Expiry/lapse states belong here too: the *designed* outcome of a timeout must leave the system usable (an expired invite that still holds its uniqueness slot bricks the address with no recovery path in the product).
8. **Existence, authority, and correction** — *can a second person use this, on what authority, and what happens when it is wrong?* This angle asks whether the PRODUCT can be used, not whether the CODE is correct, and it is the one most likely to be missing from a review that is otherwise rigorous. Real case that earned it a seat: six adversarial passes across five model families raised ~50 findings on one codebase, every one asking "is this code right" — and none of them found that in production the platform could hold **exactly one human being**. Public signup was an unconditional 503 (a deliberate, defensible decision), the only org-creating function had zero production callers, no invite or add-member route existed anywhere in the mounted surface, and the bootstrap command refused once any org existed. Every downstream rail was then correct *for a population of one*. A single pass that asked a different question found it in one read. Run these against the PRODUCTION path, never the fixtures:
   - **Can a SECOND one exist?** For every principal the product has — user, org, tenant, member, payee, staff, admin — name what creates the second one and confirm that path is reachable by someone. Fixtures seed N of everything and hide this completely; the first principal is usually solved and celebrated, and the second is often not built at all.
   - **Which modelled states does nothing ever write?** Enumerate the state vocabulary from the schema and the enums, then find the ones no code path produces. A terminal state nothing writes means the lifecycle silently ends earlier than the model claims — and every report, metric and test that reads it is describing a state the system cannot reach.
   - **Is the field used for X actually X?** A value borrowed from a neighbouring purpose reads as correct at every single line. (Real case: payouts were sent to the *login* email because no payout-address column existed anywhere in 71 migrations. Change your login, change where your money goes.)
   - **What compiles, passes its tests, and has no production caller?** Grep the call graph, not the test graph. Capability that exists and is unreachable is indistinguishable from capability that works, right up until someone needs it.
   - **On what authority does the system act?** What is captured — consent, terms, rights, ownership, an attestation — that entitles it to do the thing it does on a user's behalf? An absence here is invisible to every correctness angle, because there is no wrong code to find.
   - **Can the correction be EXPRESSED?** For each way the system can be wrong, is there a path that fixes it as new facts, reachable by someone other than an engineer at a database prompt? An append-only audit trail makes this a design question, not an afterthought — immutability is correct *and* it means every repair must have been designed in.
   - **Does the verifier verify the thing it names?** (Real case: a restore verifier probed invariants rather than data, so an emptied ledger table with its triggers armed passed as healthy.)
   - **Does the data survive?** Backup, restore, and rotation as a *scheduled capability*, not a script in the tree that CI exercises once. Ask for the RPO and RTO in hours; if nobody can answer, that is the finding.

**A single consumer-facing UI — lens angles (every button is law):**
1. **Dead / mis-wired controls** — a button that no-ops or fires the wrong action; disabled-forever; a label that lies.
2. **Async state truth** — no success-shown-on-failure, no failed-read-masked-as-empty, no stuck spinner, no un-rolled-back optimistic update.
3. **Wrong-target / stale selection** — a selection surviving an entity switch, a press-time read that's stale, cross-entity bleed.
4. **Race / double-submit / stale-closure** — double-click double-charge, out-of-order responses clobbering fresh state, effects over stale signals.
5. **Navigation / routing / deep-link / persistence** — dead-ends, lost work on nav, a gated screen reachable unauthed, storage that throws in private mode.
6. **Input edges / malformed data / rendering** — empty/huge/paste/malformed input or API row; crash, silent data loss, a bad active-state from a degenerate response, unescaped user text.

**Coverage is the multiplier, not the angle count.** Six agents skimming a large repo is a rubber stamp with extra steps. Chunk the corpus so **every part of the code is seen by every angle** — split into coherent slices (by crate/module/screen) and run each angle over each slice; on the reference sweep this was ~40 slices × 6 angles ≈ 240 passes. If a slice is too big for one agent to read closely, it re-delegates. Anchor the whole run to one **baseline commit** so findings are reproducible and re-runnable.

**On a corpus already swept clean, weight by delta — but review the delta's blast radius, not its lines.** The newest code is the least-baked; give the diff since the last clean baseline the densest attention, and extend each angle to the *callers and callees of what changed* (a changed helper is judged at every call site, a changed contract at every consumer). The rest of the corpus still gets the standing angles and the claim classes earlier sweeps under-ran — that pairing is where a re-sweep of a "clean" repo finds its real yield.

Run **the devil-in-the-words lens inside every angle**, and as its own pass if the surface is claim-dense.

For a **consumer UI**, static code review cannot catch every runtime/visual/interaction defect (a control that renders off-screen, a layout that breaks on overflow, a focus trap). Pair the code sweep with an actual interaction pass — a browser-driven or manual click-through of every screen — before calling it flawless.

## Step 2 — The per-angle agent prompt

Each agent: **read-only**, pinned to the baseline commit; hunt REAL, prod-reachable defects, not style; for each candidate open the actual code and confirm the mechanism exists and is reachable (not test-only, not dry-run, not already guarded by a sibling it didn't read); **discard anything that dissolves on a close read**; report `file:line` + a concrete failure (inputs → wrong outcome) ranked by severity; **no padding — an honest short list beats a padded one**; end with "no defect found in scope" when genuinely clean.

## Step 3 — Verify every finding at the source (never relay raw)

**~1 in 2 raised findings do not survive.** For each, open the cited `file:line` yourself and confirm the claim. A finding dies when it is: test-only or dry-run (not a prod path); already closed by a **sibling** the finding didn't read; citing a route/symbol that doesn't exist; a **stale doc** no caller obeys; or a theoretical race that can't actually interleave. Kill the phantoms. **Second-checker on dismissals:** have an independent pass re-check what you dismissed — a broken dismissal is a live bug wearing a "safe" label, and an independent checker is what makes the final "clean" credible. Use a *different model* for the checker when one is available: same-model review inherits same-model blind spots, and the reference close's cross-model checker is the one that caught a false proof every same-model pass had waved through. **Carry a verdict on every survivor** — CONFIRMED (mechanism verified at source, red-provable) vs PLAUSIBLE (real-looking, not yet proven) — and never let a PLAUSIBLE drive a fix or a dismissal on its own. Relaying unverified findings is how a review loses trust; this step is non-negotiable.

## Step 4 — Red-prove each survivor

A real finding earns a **compiling** mutation that reverts the fix and makes a **behavioral** test fail — the line `test failed`, NOT `could not compile` / `error[E####]` (invalid proofs). Watch for confounds (an FK lock can make a naive concurrency test pass with or without the fix — isolate the actual guard, e.g. `FOR NO KEY UPDATE` vs `FOR UPDATE`). Confirm the test actually RAN (a bare fn name with `--exact` runs zero tests and exits 0). Red-proving also exposes a **partial fix** — a broadened scan that never seeded the new case, a guard added on one path only. If you cannot make a test fail, the finding — or the fix — is not real yet.

**A claimed red-proof is itself a claim — verify it by quadrant.** A guard's plausible mutants each differ from correct code in *specific input quadrants*, and a killing test must visit the quadrant where its mutant differs — not a quadrant the mutant also passes. (Real case: a two-flag guard `a || b`; the arm meant to kill the swap-to-`(b, b)` mutant tested `(false, false)`, which that mutant passes identically — all four pins green, the mutant alive, and the commit message's "it dies here" false. The cross-model checker caught it by *evaluating the mutant against each pin by hand*.) So: for every red-proof, name the mutant, name the quadrant it differs in, and confirm the failing arm visits that quadrant. An independent checker re-runs this evaluation on every claimed proof — a false red-proof is worse than none, because it certifies. Label **timing-assisted** proofs as such (a sleep-then-assert can false-pass on a slow box); prefer deterministic rendezvous (hold-the-lock-and-observe-blocking, barriers) over races against the clock.

**At scale, automate the mutants.** Per-finding red-proofs only test the mutations you thought of. On money/auth/data crates, run a mutation-testing tool over the changed surface (`cargo-mutants` or the ecosystem's equivalent) and treat every *surviving* mutant in guard/refund/scope logic as a suite blind spot to triage. Same spirit for inputs: any parser, spec, or user-supplied structure the diff touched gets a property-based or fuzz pass (proptest/quickcheck/fuzzer) — hand-picked edge fixtures only cover the edges someone already imagined.

**Red-prove the suite, not just the findings.** Findings only test what an angle happened to see; the suite's blind spots stay dark. So enumerate the system's load-bearing invariants explicitly — money conserved to the cent, tenants isolated, the state machine closed, idempotency under replay, fail-closed on unknown input — and for each one, break it deliberately on a scratch branch and watch which test goes red. **An invariant no test defends is a FIX-NOW finding in its own right** (the deliverable is the missing test), because every future regression against it will ship green. Watch completeness-gate *granularity* while you're here: a per-TABLE gate lets a new column on an already-covered table ride in unseeded; the gate must force coverage at the granularity the invariant actually lives at.

## Step 5 — Triage, fix, and re-run until clean twice

Rank by **blast radius**, not tidiness: money loss, cross-tenant leak, data-loss / PII residue, publish-to-wrong-audience, silent corruption come first. Triage every survivor into one of three, each recorded with its evidence:
- **FIX-NOW** — real and reachable. Fix through the normal gate, matching CI *exactly*: **pinned-toolchain** fmt + clippy (a default rustfmt version-skews and reddens CI), the library/migration invariants (e.g. an expand-only migration test), and the **FULL** test suite — clippy-clean is not enough (a change can pass clippy and break a downstream test). Re-run the angle that found it. **Fix-commit discipline:** each fix lands with the regression test that would have caught it and *nothing else* — no drive-by refactors riding along in a closing commit; a fix without its pinning test is half a fix.
- **BANK** — real but latent / edge / self-healing (a drift a reconcile pass corrects, a 2-source edge). Document it precisely with the settling facts; don't silently drop it and don't ship a half-fix.
- **DISMISS** — dissolved on the source read. One line with the fact that settles it, for the second-checker. (The Step-3 second-checker pass re-reads every one of these.)

**When a fix adds a rule, enumerate every door.** A predicate added where the finding pointed is half a fix if the same state is reachable through another door — the highest-yield question of the whole verification pass is "what OTHER path reaches the state this rule now guards?" (Real case: an ambiguity marker correctly gated the wedged-cancel refund, while a wedged row *released back to pending* kept its marker and refunded through the untouched fast-path cancel — release-then-cancel was the refund the direct cancel refused. Cancel doors, retry doors, admin doors, expiry doors: the rule holds at all of them or it doesn't hold.)

**Parallel fix lanes merge into semantic wounds git can't see.** Two branches, each correct alone, can merge textually clean and lie together — the canonical shape is a docs lane truthfully softening a claim against the baseline while a code lane makes the original claim true, leaving a merged header that claims LESS than the code enforces (or two guards whose contracts contradict). After every lane merge, re-read the claims and contracts at the merge points, and gate the MERGED tree, never just the branches.

**Each lane gates itself before it lands.** The merged-tree gate stays authoritative — but a lane that never ran its own suite can ship code *contradicting its own pins*, which is worse than an untested fix because the pins exist and are red and nobody looked. (Real case: a rate-limit keying fix landed with the polarity backwards while the lane's own three tests asserted the correct semantics — failing in the lane's worktree, which had never run them; the same batch carried 25 files of formatter drift because no lane ran fmt.) So: before a lane offers itself for merge, it runs its own scoped gate — its crate's tests (asserting the passed-count, not just the exit), fmt, clippy — and the integration lead treats "the lane's own tests were red at merge time" as a process finding in its own right, recorded in the close. Per-lane gating is cheap; it is what makes the integrated-tree gate a *check* instead of a *net*.

**Your own harness is a test surface.** The review lies to itself the same ways the suite does: a pipeline's exit status is its LAST command's (`cmd | tail` buries the failure — capture `$?` directly); a test filter that matches nothing exits 0 (assert the passed-count, every time); two positional filters silently run neither; backticks inside a quoted commit message execute as command substitution and eat words; zsh doesn't word-split unquoted variables; and near shared CI runners you kill by PID, never by pattern. Every proof you run passes through this layer — treat a harness artifact exactly like a suspect test.

Report each finding as: `id · severity · file:line · one-line mechanism · concrete failure scenario · red-prove line · verdict (CONFIRMED / BANKED / DISMISSED)`, plus the angles that came back clean. **A single clean pass is not done** — re-run after fixing, and make the re-run a *hunt on the fixes themselves*, not a re-scan: fresh adversarial agents (different eyes, ideally a different model) tasked to break each fix, find second sites of the round's own new rules, and catch regressions the fixes introduced. On the reference close that pass found a MEDIUM in the round's own plumbing, two second-sites of its own rules, and the false red-proof — none visible to the lanes that produced them. The bar is **two consecutive passes that surface nothing new**. That is what turns "green" into an *earned* clean rather than a rubber stamp.

## Standing angles — never optional, whatever else you pick

Four angles have earned a permanent seat because they catch what a green gate is *structurally* blind to:

- **Live-substrate verification.** Runtime-bound SQL, env-driven config, and vendor adapters can be compile-green and wrong. Run the gated live suites against a real scratch substrate (database, queue, filesystem); boot the actual binary and hit its critical endpoints. A test suite that never touches the real substrate cannot vouch for it — the canonical case is a foreign-key violation that no amount of clippy will ever see. The deploy is substrate too: watch the actual flip (deployed revision == the commit you shipped, health green after), and know the rollback path before you need it — a migration that strands the previous revision mid-roll is a defect the expand-only rule exists to prevent, and the close verifies the rule held, not that it was written down.

- **Concurrency & idempotency.** Hunt TOCTOU: every cap, uniqueness, or balance rule must be enforced *inside* the transaction that writes, with the row locked — a check outside the lock is a race with a polite error message. Every externally-triggered effect (webhook, payout, queue claim, delivery) must be provably once-only under replay AND under crash-between-two-writes: ask "if the process dies on this exact line, what does the retry do?"

- **Degraded-mode honesty & leak sweep.** Walk every unconfigured/missing-credential path: it must refuse loudly (503, typed error), never simulate success — a stub that fakes the happy path is a defect even with no bug behind it. Then sweep what crosses boundaries: error `Display` impls that quote provider bodies onto the wire, logs that capture bearer capabilities (presigned URLs, session tokens, checkout links), diagnostics that leak schema.

- **Supply chain & secrets.** `cargo audit` (or ecosystem equivalent) clean or consciously waived; RC-grade dependencies pinned exact; the dependency tree diffed against intent (a review that never looks can't catch the accidental 300-crate import); repo and history grepped for credential-shaped strings.

- **The attacker's story, per money/auth change.** For every change touching authentication, authorization, or money, write the one-paragraph attack before reviewing the defense: who is the adversary, what do they hold (a leaked email? a forwarded link? a stolen DB dump? a replayed webhook?), and what does this change let them do that yesterday's code didn't. Then hunt the code for the step that makes the story work. This is threat modeling at review altitude — it caught an SSO-only account re-armable by a forgot-password round trip (email access alone re-opened the exact path SSO shut) that every functional test called correct.

- **Observability of the new failure paths.** Every failure arm the round added or touched must be *visible to an operator*: logged with enough detail to act on (while leaking nothing to the client — the two are different sinks with different rules), counted where a bill or quota depends on it, and paging someone where silence would cost money. A deployment where "nothing pages anyone" is a finding even when every guard is correct — the guards' first real failure will be discovered by a customer.

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

  The branch-sweep mechanics matter, because misclassifying one branch
  destroys real work: **patch-id comparison lies under squash and rebase**
  (a landed fix reads as "unmerged"), so classify by whether the *mechanism*
  the branch carries exists in main — open the file main-side and look —
  never by commit identity alone. A branch quietly holding an unlanded
  money/auth fix is a lie of omission; find those FIRST (the reference sweep
  found three charge-integrity fixes written after the last round and never
  routed anywhere). And before any deletion, **bundle every doomed tip**
  (`git bundle create` over the delete list) into the close's archive —
  zero-loss insurance that makes the sweep boring instead of brave.

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

## Step 8 — The last check is the documentation itself

Before anything is turned in or called done, the close returns to the words one final time: **make sure the documentation is really clean.** Not "written" — *true*.

1. **Re-read every doc the build ships, cold, after the fixes.** Every README, AGENTS/CLAUDE instruction file, closing report, comment header, and UI string written or touched this round is a claim (see the signature move). The final sweep reads them *as a user would*, hunting for prose that references code that no longer exists, counts that have drifted (an angle default, a table count, a "six" that became eight), steps that no longer work, or promises the merged tree quietly under-delivers.
2. **Walk the docs verbatim, one last time.** Clone-and-run the quickstart exactly as written, deviating for nothing. Every place you had to know something the docs did not say is a defect — fix the doc or fix the code, never leave the gap.
3. **Check the docs agree with each other.** Two files each telling the truth about one commit can still disagree after a merge (a README lane and a SKILL lane landing the same change at different depths). Cross-read sibling docs for contradicting claims — numbers, defaults, flags, install paths — and reconcile them to the code, not to each other.
4. **Zero stale references.** Grep the docs for every symbol, flag, filename, URL, and count they name, and confirm each still resolves in the tree. A doc naming a deleted file or a renamed flag is a lie wearing a bookmark. Fix or delete the line — never leave it "to be safe."

A close whose code is clean twice but whose documentation lies is not done. The documentation check is the **last** gate before the report is signed, because every earlier step may have edited the code the docs describe.

## When not to reach for this

A one-line change with an obvious blast radius does not need eight agents over the whole corpus. This is for surfaces that ship to users, touch money/auth/data, or are about to be called "done." When in doubt on something consumer-facing, run it — the cost of the sweep is an hour; the cost of a green lie is a day and a half.
