# my-agent-skills

Personal marketplace **me**. Each plugin lives in `plugins/<name>/skills/`.

```
/plugin marketplace add lasheralberto/my-agent-skills   # or a local path
/plugin install daydream@me
```

Bundles: `gsap` (8 skills), `supabase` (2 skills). playwright-skill needs `npm install` in its folder.

`factory-feature-builder` ships a skill plus 7 agents (`plugins/<name>/agents/`): `/factory-feature-builder <feature>` runs spec-writer → builder → parallel security/UX/UI/code review → loop → approver (PR or need-human).
