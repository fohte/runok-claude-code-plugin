# Disposition checklists and proposal format

Read this before filling out step 4 ("Propose to the user") of
`SKILL.md`. This is the single source of truth for the checklist items --
when delegating to a sub-agent, copy this file's content verbatim into the
prompt rather than paraphrasing it.

Leaving "rationale" free-form invites buzzwords ("read-only," "exists,"
"narrow") that don't actually verify anything. Instead, every disposition
must fill a **fixed set of checklist items**, each with a **concrete,
checkable fact**. If even one item can't be filled with a fact -- reaching
for "probably safe" or "seems fine" instead -- the disposition is wrong for
this candidate.

**"Concrete fact" means**: an enumeration of the pattern's actual argument
structure, a confirmation command you actually ran, an official docs URL,
real examples `*` would expand to -- something verifiable. Opinion,
impression, or self-assessment doesn't count.

## allow: R1-R5 checklist

All five must be filled to propose `allow`. If even one can't be, don't
allow -- narrow to a literal, register a wrapper, or fall back to
explicit-ask instead.

| #   | Item                                       | How to answer (concrete fact)                                                                                                                                                                                         | Not acceptable                                      |
| --- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| R1  | No arbitrary code execution                | Show from the command's spec that no script-interpreting flag (`-e`/`-c`/`--eval`/`--exec`, etc.) is present in the argument surface                                                                                  | "it's not a code-execution command" (unsupported)   |
| R2  | No arbitrary HTTP writes                   | Show that no `-X POST/PUT/PATCH/DELETE`, `--method`, or write subcommand is present. Required whenever an HTTP client (`curl`, `gh api`, `http`, ...) is involved                                                     | "it's read-only" (unsupported)                      |
| R3  | No broad destructive operation             | One line showing deletion / overwrite / `git push --force` / recursive removal, etc. is absent                                                                                                                        | "it's not dangerous" (unsupported)                  |
| R4  | Subcommand exists in the CLI               | State how you verified it: the literal confirmation command you ran (e.g. `gh pr view --help`) or an official docs URL                                                                                                | "I've used it before" / "probably exists"           |
| R5  | Where the pattern's `*` can actually reach | Quote the usage line from `<cmd> --help` (e.g. `gh pr view {<number> \| <url> \| <branch>} [flags]`), list 2-3 concrete strings `*` could expand to, and state why the subcommand position itself can't be hit by `*` | "it's narrow" / "as expected" (no usage-line quote) |

If R1-R3 find something present, fall back to a narrower literal, a
wrapper, or explicit-ask (state which R number failed in Q1 below). If R4
fails (subcommand doesn't exist), fall back to explicit-ask and note the
"auto-deny unknown subcommand" feature gap. If R5 shows `*` reaching further
than intended, narrow the pattern or switch to a literal.

## explicit ask: Q1-Q2 checklist

Both required, each with a concrete reason.

- **Q1: why `allow` doesn't work** -- one of:
  - `R# fails`: name which R item fails and why, in one line (e.g. "R1
    fails: `cargo run` builds and executes arbitrary Rust code"). If it's
    R4 (subcommand doesn't exist), also state the "auto-deny unknown
    subcommand" feature gap.
  - `runok feature gap`: name the missing capability precisely (e.g. "URL
    host matching," "auto-deny unknown subcommand," "git-tracked-path
    helper"), and add a `# awaiting runok feature: <feature name>` comment
    to the rule.
  - `positive intent`: state, in one line, what the user wants to confirm
    each time (e.g. "wants to review the merge method before every `gh pr merge`").
- **Q2: why `deny` doesn't work** -- one line establishing there's at
  least occasional legitimate use (e.g. "running the dev build has a
  legitimate use during development").

Not acceptable: "allow and deny both seem hard, so explicit-ask" /
"explicit-ask just in case" / "too broad, so explicit-ask" -- none of these
fill Q1 or Q2 with an actual fact.

## deny: D1-D2 checklist

- **D1: why `allow` doesn't work** -- which R item fails.
- **D2: why explicit-ask doesn't work** -- establish there's **no**
  legitimate occasional use (always wrong, immediate harm, an equivalent
  safe path exists elsewhere).

If D2 can't be filled, downgrade to explicit-ask (leave room for the user's
per-case judgment).

## wrapper: W1-W2 checklist

- **W1: not already registered** -- confirm no existing
  `definitions.wrappers` entry, in this config or any `extends`-referenced
  config, already matches the command. State how you checked (e.g. "grepped
  `definitions.wrappers` in `runok.yml` and the extended preset; no
  `bash -c <cmd>` entry present").
- **W2: write target** -- state the layer chosen per step 3d and why
  (universal, usable by anyone extending a shared preset -> that preset;
  specific to this stack or workflow -> your own config).

If W1 shows it's already registered, there's nothing to write for this
candidate -- report it and move the inner command to the candidate list for
its own R1-R5/Q1-Q2/D1-D2 evaluation instead.

## Proposal table format

Two-part format: a summary table, then a checklist expansion block per
candidate. **Don't cram rationale into table cells** -- there's no room,
and it collapses into buzzwords.

Summary table (numbered so the user can approve/reject by number):

| #   | disposition | proposal        | write target        | source commands (count)                    |
| --- | ----------- | --------------- | ------------------- | ------------------------------------------ |
| 1   | allow       | `gh pr view *`  | project `runok.yml` | `gh pr view 123` (5), `gh pr view 456` (2) |
| 2   | wrapper     | `bash -c <cmd>` | global `runok.yml`  | `bash -c '...'` (3)                        |
| 3   | ask         | `cargo run *`   | project `runok.yml` | `cargo run --bin foo` (1)                  |

For each row, expand the checklist answers below the table. **Header format
is `**#N** (checklist evaluation)` only** -- disposition/pattern/write-target
live in the summary table alone; don't repeat them in the expansion (avoids
drift between the two).

```
**#1** (R1-R5 evaluation)
- R1 no code execution: confirmed -- `gh pr view`'s arguments are limited to
  <pr-number-or-url> and display flags (`--json`, `--web`, `--comments`,
  ...); no script-interpreting flag (verified via `gh pr view --help`)
- R2 no HTTP writes: confirmed -- `gh pr view` is GET-only
  (https://cli.github.com/manual/gh_pr_view)
- R3 no destructive operation: confirmed -- display-only, no file or
  git-ref mutation
- R4 subcommand exists: confirmed via `gh pr view --help`
- R5 `*` reach: usage line `gh pr view [<number> | <url> | <branch>]
  [flags]` quoted. `*` expands to a PR number (e.g. `123`), a URL (e.g.
  `https://github.com/x/y/pull/1`), or a flag (`--json fields`, `--web`).
  Subcommand position is fixed at `view`; `*` cannot reach it by the
  grammar.

**#2** (W1-W2 evaluation)
- W1 not registered: confirmed -- grepped `definitions.wrappers` in
  `runok.yml` and the extended preset; no `bash -c <cmd>` entry present
- W2 write target: global `runok.yml` -- used across every project on this
  machine, not specific to one stack

**#3** (Q1-Q2 evaluation)
- Q1 allow fails: R1 fails -- `cargo run` builds and executes arbitrary
  Rust source from `Cargo.toml`. Literal-izing the args doesn't help either
  -- `--bin <name>` lets `<name>` select any binary.
- Q2 deny fails: legitimate use running dev builds during development.
- awaiting runok feature: none (positive intent + R1 failure)
```

Approval comes back as a number selection (`1,3 only`, `2 but widen to gh pr
*`, `all of it`, ...).
