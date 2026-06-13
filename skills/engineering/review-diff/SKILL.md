---
name: review-diff
description: Disciplined self-review of the current change before it ships. Establish scope → read the diff against the domain model → judge correctness, then design → rank findings → decide what blocks the commit. Use when user says "review this" / "review my changes" / "review the diff", asks for feedback before committing or opening a PR, or asks "is this ready to ship / merge?".
---

# Review Diff

A review is only useful if it catches what _you_ would miss. Your bias is that you just wrote this code and it looks fine. The discipline below exists to break that bias.

Review your **own** change here. To review someone else's PR, the same phases apply — just point Phase 1 at their branch instead of the working tree.

## Phase 1 — Establish scope

Get the exact diff under review. Don't review from memory of what you _intended_ to change.

- Uncommitted work: `git diff` (unstaged) and `git diff --staged`. Review both.
- A branch about to become a PR: `git diff <base>...HEAD` (three dots = changes on your branch since it diverged).
- A single commit: `git show <sha>`.

Then state, in one sentence, what this change is _supposed_ to do. If you can't, the review can't judge whether it succeeds — go get that from the user or the issue/PRD first.

Note the blast radius: which modules does the diff touch, and who calls them? A two-line change to a widely-called function is a bigger review than a 200-line change to a leaf.

## Phase 2 — Read against the domain model

Before judging anything, load the project's shared language. Read `CONTEXT.md` for the domain glossary and check `docs/adr/` for any decisions that govern the area you're touching.

This is what separates a real review from a linter. Two questions a linter cannot ask:

- **Does this change use the established terms correctly?** New names that collide with or contradict the glossary are a defect — they erode the shared language session after session.
- **Does it contradict a recorded decision?** A diff that quietly reverses an ADR is either a bug or a new decision that needs recording. Flag which.

If the project has no `CONTEXT.md`, review against the naming and patterns already present in the surrounding files instead. Consistency with what exists is the bar.

## Phase 3 — Correctness pass

Find ways the change is **wrong**, in rough priority order. Read the actual code paths — do not assume the diff does what its description claims.

- **Logic & edge cases.** Off-by-one, boundary values, empty/null/zero inputs, the unhappy path. What happens when the thing it depends on fails?
- **Error handling.** Are errors swallowed, logged-and-continued, or surfaced? Is that the right choice _here_? A caught-and-ignored error is guilty until proven innocent.
- **Concurrency & ordering.** Shared state, await points, race windows, non-atomic read-modify-write.
- **Contracts.** Does a changed function still honour what its callers expect — return shape, nullability, thrown types, side effects? The diff shows the callee; the bug is usually in a caller it didn't touch.
- **Tests.** Is the new behaviour actually exercised, or does the test assert "didn't crash"? Does a test change quietly weaken an assertion to make a failure go away? A green suite that doesn't test the change is worse than no test — it's false confidence.
- **Security & data.** Untrusted input reaching a sink (SQL, shell, HTML, path), secrets in code or logs, authz checks that moved or vanished.

For anything you suspect but can't confirm by reading, say so explicitly rather than asserting it. If a suspected defect needs a runnable check to confirm, that's a `/diagnose` job — note it and move on; don't start debugging mid-review.

## Phase 4 — Design pass

Correctness keeps today's build green; design decides whether the next change is cheap or a nightmare. Agents accelerate entropy, so this pass is not optional.

- **Module depth.** Did this change add a shallow module — a lot of interface for a little functionality? Could the same capability hide behind a simpler surface? (Ousterhout: the best modules are deep.)
- **Complexity leak.** Does a caller now have to know something it shouldn't — an ordering rule, a flag to pass, an internal state to manage? Pushed-out complexity multiplies across every call site.
- **Duplication vs. coupling.** Is this copy-pasted logic that should be shared, _or_ a shared abstraction being stretched to cover a case it doesn't fit? Both are defects; they have opposite fixes.
- **Naming & consistency.** Do new names match the glossary and the surrounding code? A well-named thing needs no comment to explain what it is.
- **Step size.** Is this one coherent change, or several unrelated ones smuggled into one diff? Small deliberate steps are easier to review, revert, and bisect. An unrelated drive-by refactor riding along is its own finding.

If the change reveals a structural problem larger than itself — no seam to test against, tangled callers, an abstraction buckling under load — that is a `/improve-codebase-architecture` job. Record it as a finding; don't expand the review into a refactor.

## Phase 5 — Rank and report

Group findings by severity. Within each group, most-important first.

- **🔴 Must fix before shipping** — correctness bugs, security holes, contract violations, tests that give false confidence.
- **🟡 Should fix** — design smells that will bite soon, missing edge-case handling that's unlikely but real, glossary drift.
- **🟢 Consider** — genuine nits and judgement calls. Keep this list short; a review that's all nits trains the reader to ignore it.

For each finding give the **location** (`file:line`), **what's wrong**, and **the smallest fix that resolves it** — not a vague "consider improving this."

End with one honest line: **ship it**, **ship after the 🔴s**, or **needs another pass**. If there are zero 🔴s and the 🟡s are deferrable, say "ship it" plainly — a review that can't ever conclude _good enough_ is just friction.

## Phase 6 — Close the loop

- If the user asked you to apply fixes, apply only the agreed findings, then re-run Phase 1 on the new diff to confirm you didn't introduce anything.
- Findings the user chose **not** to fix and that point at a real decision (a deliberate deviation from an ADR, a known-but-accepted edge case) belong in `CONTEXT.md` or a new ADR — not lost in chat. Offer to record them.
- Hand off anything you parked: real bugs to `/diagnose`, structural problems to `/improve-codebase-architecture`.
