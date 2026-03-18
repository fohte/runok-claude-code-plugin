# runok Claude Code Plugin

This repository is a Claude Code plugin for [runok](https://github.com/fohte/runok), a command execution permission manager.

## Skill Description Eval Workflow

When modifying `skills/runok/SKILL.md`'s `description` field, verify trigger accuracy using the eval set before merging.

### Eval Set

- Location: `test/eval_set.json`
- Format: array of `{ "query": string, "should_trigger": boolean }`
- Current baseline: **19/20** (recall 90%, precision 100%)

### Full Eval Run

```bash
python scripts/run_eval.py \
  --eval-set test/eval_set.json --skill-path skills/runok \
  --model claude-opus-4-6 --runs-per-query 1 --num-workers 3 --verbose
```

### Manual Single-Query Test

Run the following command to test whether a specific query triggers the runok skill:

```bash
cd /tmp && CLAUDECODE= claude -p '<test query>' \
  --plugin-dir <path to this repo root> \
  --model claude-opus-4-6 --max-turns 1 --allowedTools 'Skill,Read' \
  --output-format stream-json --verbose
```

If the output contains a Skill tool call with `{"name": "Skill", "input": {"skill": "runok"}}`, the skill was triggered successfully.
