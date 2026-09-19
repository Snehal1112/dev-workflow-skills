# dev-workflow-skills

A small Claude Code plugin bundling three workflow skills:

- **`/1-git-commit`** — analyzes staged changes (via subagents), checks whether the change
  belongs folded into an existing commit, and drafts a commit message following the Seven
  Rules of great commit messages.
- **`/decompose-and-dispatch`** — splits a non-trivial ask into subtasks and dispatches each
  to a subagent on the cheapest model that can do it.
- **`/decomposing-investigations`** — splits an open-ended investigative question into
  independent sub-questions, answers each with a model-tiered subagent, and synthesizes a
  single evidence-backed verdict.

## Install

```
/plugin marketplace add <owner>/dev-workflow-skills
/plugin install dev-workflow-skills@dev-workflow-skills-marketplace
```

(Replace `<owner>/dev-workflow-skills` with this repo's actual GitHub path once published.)

## Notes

`decompose-and-dispatch` references two Superpowers skills
(`superpowers:subagent-driven-development`, `superpowers:dispatching-parallel-agents`) as
routing alternatives for specific task shapes (executing a written plan, independent
parallel failures). Those rows only apply if the Superpowers plugin is also installed —
this plugin works standalone without it for every other case.

## License

MIT
