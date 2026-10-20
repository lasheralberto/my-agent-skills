# factory-feature-builder

Build a feature through an agent pipeline: **spec → build → parallel review → fix loop → approve**. The result is a PR for you to review, or a `need-human` report when the agents can't settle something on their own.

```
feature
  └─> spec-writer (opus)
        └─> builder (opus) <────────────────────────┐
              └─> in parallel:                       │
                    security   (opus)                │ changes needed
                    ux         (sonnet)              │ (and rounds left)
                    ui-design  (sonnet)              │
                    code       (sonnet)              │
              └─> wait for all ── changes needed? ───┘
                                   no / max rounds
              └─> approver (opus) ── APPROVE  → commit (+ push + PR if configured)
                                  └─ ESCALATE → need-human.md, nothing pushed
```

## Install

```
/plugin marketplace add <path-or-repo-of-this-marketplace>
/plugin install factory-feature-builder@me
```

On install Claude Code asks for the config values below. From a shell you can pass them instead:

```
claude plugin install factory-feature-builder@me --config github_repo_url=https://github.com/owner/repo
```

## Configure

Change values at any time with `/plugin configure factory-feature-builder@me` (non-sensitive options also appear in `/config`).

| Option | Default | What it does |
|---|---|---|
| `github_repo_url` | *(empty)* | Repo the approver pushes the branch to and opens the PR against, e.g. `https://github.com/owner/repo`. **Empty = nothing is ever pushed**; approved work is only committed on the local branch. |
| `max_iterations` | `3` | Max builder → reviewers rounds (1–10) before the approver decides. |

You can override either for a single run in the request itself: *"…, max 5 iterations"*.

## Use

From inside the git repo where the feature should be built:

```
/factory-feature-builder add CSV export to the invoices page
```

Or just ask for "the feature factory" with your feature description.

### What happens

1. **Preflight** — requires a git repo with a clean working tree (if dirty, it stops and asks; it never stashes or discards your changes). Creates branch `factory/<slug>` and the workspace `.factory/<slug>/` (excluded from git).
2. **Spec** — `spec-writer` reads the codebase and writes `spec.md`. If something is ambiguous enough that a wrong guess would build the wrong thing, it asks you first; otherwise it records a default under *Assumptions*.
3. **Build** — `builder` implements the spec in the working tree and runs the tests/lint/build. It never commits or pushes.
4. **Review** — `security`, `ux`, `ui-design` and `code` reviewers run in parallel. Each returns `PASS`, `CHANGES_REQUESTED` (at least one `blocker` or `major` finding) or `NOT_APPLICABLE` (e.g. UX on a backend-only change, which counts as a pass). Reviewers never edit code.
5. **Loop** — if any reviewer asked for changes and rounds remain, back to the builder with their findings. If the same blocker comes back two rounds in a row, the loop stops early.
6. **Approve** — `approver` re-checks the diff and re-runs the tests, then:
   - **APPROVED** — commits on `factory/<slug>`. With `github_repo_url` set: pushes the branch to that repo and opens a PR (via `gh`, or gives you the compare URL). With it empty: stops at the local commit.
   - **NEED_HUMAN** — nothing is committed or pushed. It writes `.factory/<slug>/need-human.md` with the open findings and the specific decisions it needs from you.

### Output you get

```
.factory/<slug>/
  feature.md             your request (+ answers to spec questions)
  spec.md                the spec
  build-<N>.md           builder report per round
  reviews/iter-<N>/      security.md  ux.md  ui-design.md  code.md
  need-human.md          only when escalated
```

## Guarantees

- Never merges, never pushes to the base branch, never force-pushes, never uses `--no-verify`.
- Pushes only to `github_repo_url`; with it empty it does not push anywhere.
- All fixes go through the builder; the orchestrator never edits code between stages.

## Requirements

- `git`; a git repo to run in.
- To push and open PRs: `github_repo_url` set, push access to that repo, and ideally `gh` authenticated (`gh auth status`). Without `gh` you still get the branch pushed plus a compare URL.

## Agents

| Agent | Model | Role |
|---|---|---|
| `spec-writer` | opus | Request → implementable spec |
| `builder` | opus | Implements spec / fixes findings |
| `security-reviewer` | opus | Injection, authn/z, secrets, data exposure, deps |
| `ux-reviewer` | sonnet | Flows, states, copy, errors, accessibility |
| `ui-design-reviewer` | sonnet | Design-system consistency, layout, responsiveness |
| `code-reviewer` | sonnet | Spec coverage, correctness, tests, maintainability |
| `approver` | opus | Final gate: ship or escalate |

Models are pinned in each file under [agents/](agents/). Edit the `model:` line to change one.

## Troubleshooting

- **"working tree is dirty"** — commit or stash your changes first, then rerun.
- **Nothing was pushed** — `github_repo_url` is empty, or the run ended in `NEED_HUMAN`. Check `.factory/<slug>/need-human.md`.
- **Push failed** — the run escalates with the error (auth, repo missing, base branch not on that repo); the commit stays local on `factory/<slug>`.
- **Agents not found** — run `/reload-plugins`.
