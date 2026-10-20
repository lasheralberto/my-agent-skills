---
name: factory-feature-builder
description: Build a feature end-to-end with an agent factory - spec-writer (Opus) writes the spec, builder (Opus) implements it, then security (Opus), UX, UI design and code (Sonnet) reviewers run in parallel; if any requests changes it loops back to the builder; finally the approver (Opus) either pushes a new branch and opens a PR for human review, or escalates as need-human. Use when the user runs /factory-feature-builder, asks for the "feature factory", or wants a feature built through the spec -> build -> review -> approve pipeline.
argument-hint: <feature description>
---

# Factory Feature Builder

You (the main session) are the **orchestrator**. Subagents cannot spawn subagents, so the loop lives here. You do not write the spec, the code or the reviews yourself — you dispatch, pass artifacts between stages, and decide when to loop.

```
feature
  └─> spec-writer (opus)
        └─> builder (opus) <───────────────────────────┐
              └─> in parallel:                         │
                    security        (opus)             │ changes needed
                    ux              (sonnet)           │ (iteration < MAX)
                    ui-design       (sonnet)           │
                    code            (sonnet)           │
              └─> wait for all ── changes needed? ─────┘
                                   no / MAX reached
              └─> approver (opus) ── APPROVE  → new branch + push + PR
                                  └─ ESCALATE → need-human report, nothing pushed
```

Agents (plugin-namespaced `subagent_type`):

| Stage | subagent_type | Model |
|---|---|---|
| Spec | `factory-feature-builder:spec-writer` | opus |
| Build | `factory-feature-builder:builder` | opus |
| Review | `factory-feature-builder:security-reviewer` | opus |
| Review | `factory-feature-builder:ux-reviewer` | sonnet |
| Review | `factory-feature-builder:ui-design-reviewer` | sonnet |
| Review | `factory-feature-builder:code-reviewer` | sonnet |
| Gate | `factory-feature-builder:approver` | opus |

If a namespaced agent type is not available (plugin agents not loaded), fall back to `general-purpose` with the same `model` override and paste the corresponding file from this plugin's `agents/` folder as the first part of the prompt.

## Settings

- `MAX_ITERATIONS = 3` build→review rounds. The user may override ("max 5 iterations").
- Workspace: `.factory/<slug>/` at the repo root (`<slug>` = kebab-case of the feature, ≤ 40 chars). Add `.factory/` to `.git/info/exclude` so it never gets committed.

## Step 0 — Preflight

1. The feature text is the skill argument. If empty, ask the user for it.
2. Confirm you are in a git repo. Run `git status --porcelain`. If the tree is dirty, stop and ask the user whether to continue (their uncommitted changes would end up in the PR) — do not stash or discard anything yourself.
3. Record the base branch (`git rev-parse --abbrev-ref HEAD`) and create the work branch: `git switch -c factory/<slug>` (append `-2`, `-3`… if it exists).
4. Create `.factory/<slug>/` and write the raw request to `.factory/<slug>/feature.md`.
5. Tell the user in one line: branch, workspace, max iterations.

## Step 1 — Spec

Dispatch `spec-writer` (foreground). Prompt: the feature text, the repo root, and the output path `.factory/<slug>/spec.md`.

It returns either `SPEC_READY` or `QUESTIONS` (blocking ambiguities). On `QUESTIONS`, ask the user (AskUserQuestion when they fit as options), append the answers to `feature.md`, and re-dispatch the spec-writer. Do not guess answers to blocking questions.

## Step 2 — Build (iteration N, starting at 1)

Dispatch `builder` (foreground). Prompt includes:
- path to `spec.md`
- iteration number N
- for N > 1: paths to every `reviews/iter-<N-1>/*.md` with `CHANGES_REQUESTED`, and the instruction to address every blocker/major finding (or justify in the build report why not)
- output report path `.factory/<slug>/build-<N>.md`

The builder edits the working tree but **never commits, pushes or switches branches**.

## Step 3 — Parallel review

In **one message**, dispatch the four reviewers in parallel (foreground, so you wait for all four). Each gets:
- path to `spec.md` and `build-<N>.md`
- base branch name, so it can run `git diff <base>` plus `git status` for untracked files
- its output path `.factory/<slug>/reviews/iter-<N>/<security|ux|ui-design|code>.md`

Always spawn all four. A reviewer whose dimension does not apply (e.g. UX for a pure backend change) returns `NOT_APPLICABLE` — that counts as a pass.

## Step 4 — Do we need changes?

Read the four verdicts (each review file starts with `VERDICT: PASS | CHANGES_REQUESTED | NOT_APPLICABLE`).

- **Changes needed** = any reviewer returned `CHANGES_REQUESTED` (i.e. at least one `blocker` or `major` finding).
- Changes needed and `N < MAX_ITERATIONS` → N = N + 1, back to **Step 2**. Tell the user one line: which reviewers asked for what.
- No changes needed, **or** `N == MAX_ITERATIONS` → go to **Step 5**.

Detect a stuck loop: if the same blocker finding reappears in two consecutive iterations, skip remaining iterations and go to Step 5 (the approver will likely escalate).

## Step 5 — Approver

Dispatch `approver` (foreground) with: `feature.md`, `spec.md`, all `build-*.md`, all `reviews/`, base branch, work branch, iteration count, and whether the loop ended clean or by hitting MAX/stuck.

The approver returns exactly one of:

- `APPROVED` — it has committed on `factory/<slug>`, pushed with `-u origin`, and opened a PR against the base branch (`gh pr create`), or, if `gh` is missing/unauthenticated, pushed and returned the compare URL. Report the PR URL to the user.
- `NEED_HUMAN` — nothing pushed. It wrote `.factory/<slug>/need-human.md` with the reasons and the concrete decisions the user must make. Show the user that summary; leave the branch and working tree as they are.

## Final report to the user

Short: feature, iterations used, per-reviewer final verdict, outcome (PR link or need-human reasons), and the `.factory/<slug>/` path for the full trail.

## Rules

- Never merge, never push to the base branch, never force-push.
- Never skip the reviewers, even on the last iteration.
- Do not edit code yourself between stages; route every fix through the builder.
- Keep your own messages to the user to one line per stage transition.
