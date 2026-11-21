# Contributing

New skills and fixes are welcome. Each skill is one folder at the repo root.

## Add a skill

1. Create `my-skill/` with a `SKILL.md`. The folder name must equal the `name` in its front matter.
2. Keep `SKILL.md` under 500 lines. Put endpoint detail in `references/` so it loads only when needed.
3. Write the `description` so an agent knows exactly when to use it (under 1,024 characters).
4. Only use public Modulate endpoints from https://docs.modulate.ai. Never put an API key, a real recording of a person, or private data in the repo.
5. Add `evals/trigger_prompts.json` with prompts that should and should not load the skill.
6. Validate it:

```bash
pip install "git+https://github.com/agentskills/agentskills#subdirectory=skills-ref"
skills-ref validate ./my-skill
```

7. Add the skill to the table in `README.md` and a line to `CHANGELOG.md`.

## Rules for results and claims

- Report what the API returned. Do not invent a pass or fail threshold.
- Detection results are probability signals, not proof. Say so in the skill.
- Tell the user that audio is uploaded to Modulate's API.

## Test clips

Only add clips you recorded yourself or whose licence allows redistribution. Record the source and licence in `fixtures/SOURCES.md`.
