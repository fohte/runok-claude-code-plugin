<<<<<<< before updating
# runok Claude Code Plugin

This repository is a Claude Code plugin for [runok](https://github.com/fohte/runok), a command execution permission manager.

## Skill Description Eval Workflow

When modifying `skills/runok/SKILL.md`'s `description` field, verify trigger accuracy using the eval set before merging.

```bash
python scripts/run_eval.py \
  --eval-set test/eval_set.json --skill-path skills/runok \
  --model claude-opus-4-6 --runs-per-query 1 --num-workers 3 --verbose
```
=======
# AGENTS.md

## Code organization rules

### Split files before they grow past ~500 lines of production code

When a change would push a file's non-test code past ~500 lines, split it along responsibility seams before adding more. Splits must be move-only commits: no logic changes, renames, or reformatting mixed in. Keep external import paths unchanged by keeping the entrypoint file in place and re-exporting the pieces you split out into new files. Tests move together with the code they verify.

Prefer creating a new focused file over appending to the largest existing one.
>>>>>>> after updating
