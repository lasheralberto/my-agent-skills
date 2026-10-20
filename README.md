# my-agent-skills

Personal marketplace **me**. Each plugin lives in `plugins/<name>/skills/`.

```
/plugin marketplace add lasheralberto/my-agent-skills   # or a local path
/plugin install daydream@me
```

Bundles: `gsap` (8 skills), `supabase` (2 skills). playwright-skill needs `npm install` in its folder.

`factory-feature-builder` ships a skill plus 7 agents (`plugins/<name>/agents/`): `/factory-feature-builder <feature>` runs spec-writer → builder → parallel security/UX/UI/code review → loop → approver (PR or need-human). Config (`/plugin configure factory-feature-builder@me`): `github_repo_url` (empty = never push, local commit only) and `max_iterations` (default 3).

`replica-skill` (11 skills, by Jake Schincariol, MIT; source: https://github.com/Jakeschincariol/replica-skill): clone any app, from `/replica-skill:replica-recon` to `/replica-skill:replica-deploy`. Python 3.8+ tools, no pip install.
