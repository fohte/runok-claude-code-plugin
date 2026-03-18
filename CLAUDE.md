# runok Claude Code Plugin

This repository is a Claude Code plugin for [runok](https://github.com/fohte/runok), a command execution permission manager.

## Skill Description Eval Workflow

When modifying `skills/runok/SKILL.md`'s `description` field, verify trigger accuracy using the eval set before merging.

```bash
python scripts/run_eval.py \
  --eval-set test/eval_set.json --skill-path skills/runok \
  --model claude-opus-4-6 --runs-per-query 1 --num-workers 3 --verbose
```
