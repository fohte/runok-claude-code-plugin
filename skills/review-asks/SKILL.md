---
name: review-asks
description: >-
  Drive the output of `runok audit --action ask --recheck --json`, filtered to
  asks not covered by any rule in the current config, to zero -- by
  converting each pending ask into an allow rule, a deny rule, or an explicit
  `ask` rule in runok config. Use only when the user explicitly asks to
  review pending `ask` entries, clean up `runok audit` history, or convert
  approved ask history into allow rules. Do not auto-trigger from incidental
  mentions of `runok audit`.
---

# review-asks

Convert every uncovered `ask` entry in the runok audit log into an explicit
disposition -- `allow`, `deny`, or an explicit `ask` rule -- until the
extraction query below returns nothing.

This skill assumes familiarity with runok's rule syntax, CEL `when` clauses,
and the four-layer config resolution order (global `runok.yml` -> global
`runok.local.yml` -> project `runok.yml` -> project `runok.local.yml`); see
the `runok` skill for that reference.

## Goal

The only exit condition is an **empty result** from:

```sh
runok audit --action ask --recheck --json \
  | jq 'select(.recheck.command_evaluations | any(.matched_rules == null))'
```

This lists `ask` entries where at least one command branch matches no rule in
the _current_ config -- i.e. it would still fall through to
`defaults.action` today, regardless of what happened when it was originally
recorded. See <https://runok.fohte.net/cli/audit/#--recheck> and
<https://runok.fohte.net/cli/audit-log-schema/> for the full field reference
(`recheck.action`, `recheck.command_evaluations[].matched_rules`, `approved`,
etc.).

Every entry still appearing in this output must be assigned to exactly one
of:

- **allow**: write an `allow` rule (narrowest safe pattern) to the
  appropriate config layer.
- **deny**: write a `deny` rule. Reserved for commands that are unsafe in
  essentially all situations.
- **explicit ask**: write an `ask` rule with a `message` explaining why
  confirmation should always be required. This intentionally keeps the
  command in `ask` state, but because it's now backed by a real rule,
  `matched_rules` is non-null on recheck and the entry drops out of the
  query above.

"Defer", "revisit later", or "skip for now" are not valid outcomes --
allowing them turns deferral into an unstated fourth option and the query
never reaches zero. **An entry you can't safely allow is not automatically
an explicit-ask candidate** -- explicit ask is for cases where the user
genuinely wants to keep confirming every time (positive intent). If the
reason you can't allow it is a gap in runok's pattern language, say so
explicitly (see disposition rules below) rather than quietly parking it
under "explicit ask".

## Core principle: least privilege

An `allow` pattern must be scoped to the narrowest capability the command
actually needs -- never approved by command name alone. Evaluate what the
command _can do_, not what it's usually used for:

- Interpreters (`perl`, `python`, `node`, `ruby`, `awk`, ...) can execute
  arbitrary code via flags like `-e`/`-c`/`--eval`. Only allow specific,
  literal invocations that can't reach that surface.
- Commands that run an inner command (`xargs <cmd>`, `find -exec <cmd>`,
  `bash -c <cmd>`, `eval <cmd>`, ...) have an arbitrary inner payload.
  Register them as `definitions.wrappers` so the inner command is evaluated
  recursively (see step 3c) instead of writing a blanket `allow`.
- Subcommands like `<tool> exec`, `<tool> run`, `<tool> shell` are
  effectively interpreters too -- prefer wrapper registration over allowing
  them outright.

If the user asks for a broader `allow` than the command's actual capability
justifies, push back and propose the narrower pattern instead of writing
what was asked verbatim. Analogies between commands are a trap (e.g. `sed
-i` and `perl -i` look similar, but `sed`'s DSL isn't Turing-complete while
Perl is a full language) -- evaluate each command's own capability, not what
a "similar" command allows.

## Workflow

Repeat until the query in **Goal** returns nothing.

### 0. Self-check (start of every loop)

Confirm each of these against what you've actually read, not from memory:

- [ ] Read every `extends`-referenced config's `definitions.wrappers` and
      rules (avoid proposing duplicates)
- [ ] Read the existing rules across all config layers in effect (avoid
      proposing duplicates; note any code smell -- hardcoded absolute paths,
      overly broad `allow`, etc. -- as a separate cleanup candidate)
- [ ] Confirmed no wrapper-eligible candidate is being routed to
      explicit-ask instead of `definitions.wrappers`
- [ ] Confirmed default `ask` isn't being escalated to `deny` just because
      you don't want to write an `allow`
- [ ] Confirmed only user-approved dispositions are about to be written --
      nothing is being added silently
- [ ] Confirmed every `allow` pattern's capability was evaluated (least
      privilege), not approved by command-name similarity
- [ ] Confirmed candidates needing risk judgment were split into sub-agent
      batches (not evaluated solo -- see 3a)
- [ ] Confirmed any CEL `when` clause referencing the home directory uses
      `env.HOME`, not a hardcoded path

### 1. Survey existing state (first loop only)

- Read every config layer currently in effect and every `extends`-referenced
  shared config, including their `definitions.wrappers` and rules.
- **Don't assert a runok feature doesn't exist from memory.** Check the
  schema
  (`https://raw.githubusercontent.com/fohte/runok/main/schema/runok.schema.json`)
  for `definitions.{aliases,wrappers,vars,flag_groups,paths,sandbox}` etc.
  before telling the user a capability is missing.

### 2. Fetch candidates

Run the query from **Goal**. Empty output means the goal is already met --
stop here.

If the user specified a scope, translate it into the matching `runok audit`
flag rather than fetching everything and filtering yourself:

- "just this repo" -> `--dir <path>` (filters by recorded cwd, not by
  command content)
- "just `<command>`" -> `--command <substring>`
- "recent / today / this week" -> `--since 1h|7d|<date>`

Flags combine. For a first broad pass, raise `--limit` (default 50) to cover
more history; later passes can reuse the same scope to pick up where the
previous pass left off.

As a secondary signal (not a substitute for the risk evaluation below), each
`ask` entry carries `approved` when the user's Claude Code session recorded
an approval. An entry that's `approved: true` across most/all of its
occurrences is evidence the user actually wants it allowed; one that's never
`approved` might just be noise or something usually rejected -- weigh this
alongside the risk checklist, not instead of it.

### 3. Group and pre-assign disposition

Group raw commands by shared argv prefix (e.g. `gh pr view 1`, `gh pr view
2` -> `gh pr view`). Every group must get a disposition (allow / wrapper /
deny / explicit-ask) before you move to proposing anything to the user.

**Verify the subcommand actually exists** (e.g. via `<cmd> --help`) before
designing a rule for it. A typo or a subcommand that no longer exists will
simply fail at the CLI layer -- writing a `deny` for it accomplishes
nothing. Treat nonexistent subcommands as a gap in runok's own
auto-deny-unknown-subcommand capability (see step 3c) rather than something
to hand-roll a rule for.

#### 3a. Split risk evaluation across sub-agents (required)

**The main agent must never evaluate the full candidate set's risk profile
and least-privilege pattern design alone.** Batching many candidates into
one pass makes later ones get judged more casually -- a classic path to an
arbitrary-code-execution command sliding through as "the same as the
others."

What the main agent does directly:

- Grouping and subcommand-existence checks.
- Only the two **mechanically decidable** fast-path cases:
  - The command already matches an existing preset/config wrapper as-is
    (e.g. `bash -c '<cmd>'`, `time <cmd>`) -> mark as wrapper-covered, and
    queue the _inner_ command as a separate candidate for evaluation.
  - The command matches an **existing** `allow` rule literally -> report
    only (nothing to write).
- Everything else goes to a sub-agent. If a candidate "feels obviously safe"
  or "obviously explicit-ask," that feeling is not a fast-path
  justification -- fast-path is for the two mechanical cases above only.
- **Keep the fast-path share under ~30% of total candidates.** If it's
  higher, that's a sign risk evaluation is being done in bulk by the main
  agent -- go back and split it out.

Everything not fast-pathed goes into batches of 5-10 candidates, evaluated
by parallel `general-purpose` sub-agents (launch them together, in one
message, for true parallelism). Skip batching only when total candidates <=
3 (overhead isn't worth it there).

Each sub-agent prompt must be self-contained:

- Its batch's candidate list (raw commands + frequency)
- The full "Core principle: least privilege" section above
- The full contents of `references/checklists.md` (copy verbatim -- that
  file is the single source of truth; don't hardcode a stale copy into the
  prompt template)
- The full "3c. Disposition rules" section below
- Required output shape: `{disposition, pattern, rationale, risk_profile,
suggested_tests, write_target}` per candidate, with `rationale` fully
  filling every checklist item from `references/checklists.md`
- An explicit instruction: "If you can't fill a checklist item with a
  concrete fact, the disposition is explicit-ask, not allow. Buzzwords like
  'read-only' or 'safe' or 'narrow' with no supporting fact are a
  violation."

The main agent may edit sub-agent results as follows:

- **Narrow further**: fine, e.g. `<tool> *` -> `<tool> <subcommand> *`.
- **Downgrade to explicit-ask**: fine.
- **Reclassify a missed wrapper opportunity**: fine -- if a sub-agent marked
  something explicit-ask that's actually wrapper-eligible, move it.
- **Loosen**: not allowed. Never upgrade a sub-agent's explicit-ask/deny
  call to allow unilaterally -- surface it to the user as borderline
  instead.

#### 3b. Risk categories that push away from `allow`

Detailed evaluation criteria live in `references/checklists.md` (R1-R3 are
the source of truth); this is just the category list:

- **Arbitrary code execution** (R1): `cargo run *`, `npx *`, `python -c *`,
  `node -e *`, `perl *`, `awk *`, ...
- **Arbitrary HTTP writes** (R2): `gh api -X POST|PUT|PATCH|DELETE *`, `curl
-X POST *`, ...
- **Broad destructive operations** (R3): recursive `rm`, `git push --force
*`, ...

Don't allow these globally on the assumption that cwd/target can be trusted.
Prefer a narrow literal, a wrapper, or explicit-ask.

#### 3c. Disposition rules

- Commands that run an inner command (`bash -c <cmd>`, `eval <cmd>`, `time
<cmd>`, `xargs <cmd>`, `find -exec`, `mise exec --`, ...) -> **wrapper**.
  If already registered in a config layer or extended preset, the inner
  command is already evaluated recursively -- nothing to add. Otherwise
  register it in `definitions.wrappers` at the layer decided in 3d.
- Safe to allow broadly (e.g. a family of read-only subcommands) ->
  **allow**, with a pattern scoped to that family.
- Safe to allow only as a narrow literal -> **allow**, literal form.
- Looks risky at first glance but can be made safe via cwd scoping, argv
  constraints, wrapper registration, a CEL `when` clause, `<var:name>`, or
  `definitions.aliases` -> **allow**. Don't stop at "what the pattern syntax
  obviously supports" -- use the full feature set before concluding it
  can't be done.
- Can't currently be written as a safe `allow` given runok's present feature
  set -> this is a **runok feature gap**. Name the missing capability
  precisely (e.g. "URL host matching in CEL," "auto-deny for unknown
  subcommands," "git-tracked-path helper," "nested wrapper support") and
  propose filing an issue at `https://github.com/fohte/runok/issues`
  describing it. In the meantime, write an explicit-ask rule with a `#
awaiting runok feature: <feature name>` comment so the goal query still
  reaches zero.
- The user genuinely wants to keep confirming every time (e.g. reviewing
  the merge method before every `gh pr merge`, confirming every destructive
  op individually) -> **explicit ask**. This is the only legitimate reason
  for this disposition.
- "Not sure, so explicit-ask for now" or "don't want to write allow, so
  explicit-ask" are not on this list.

**What explicit-ask actually means**: keep the default `ask` behavior _and_
remove the entry from the review queue, backed by a real rule instead of
external state. It exists for genuine "I want to decide every time" intent
-- using it as a dumping ground for "couldn't figure out allow" erases the
distinction from genuinely-intentional asks and loses the trail for
revisiting it later.

**Deny threshold**: reserve `deny` for commands that are unsafe with
essentially no exception -- "don't want to allow it" is not sufficient
justification. `deny` forecloses the `ask`-and-decide-per-case flow
entirely, so if there's any legitimate occasional use, prefer explicit-ask
over deny.

**No hardcoded home paths**: a CEL `when` clause that needs to test
something under the user's home directory must use `env.HOME + "/..."`,
never a literal `/Users/<name>/...` or `/home/<name>/...`.

**Consider extending runok itself, not just working around its current
syntax.** Some example triggers:

- A CEL variable you need for safe cwd-scoping doesn't exist yet -> propose
  adding it.
- A dev-build invocation should be treated identically to the installed
  command (e.g. `cargo run -- check` == `mytool check`) ->
  `definitions.aliases`.
- A wrapper doesn't evaluate its inner command -> register it (universal
  wrappers go in a shared preset if you maintain/extend one and it meets
  that preset's contribution bar; stack-specific ones go in your own config
  -- see 3d).
- Failed or not-yet-installed commands keep showing up as `ask` in history
  -> this is the "auto-deny unknown subcommand" feature gap.
- You need to match on a URL's host -- this is the "CEL URL-parsing helper"
  feature gap.
- When a feature gap needs a separate task to close, park it as
  explicit-ask (with the comment above) and move on -- don't let it block
  reaching zero.

#### 3d. Choosing the write target

runok resolves four config layers, from lowest to highest priority: global
(`~/.config/runok/runok.yml`), global override
(`~/.config/runok/runok.local.yml`), project (`./runok.yml`), project
override (`./runok.local.yml`). If any of these `extends` a shared config
repo, treat that as an additional, higher-leverage target. Decide in this
order:

1. **A shared config you maintain via `extends`** (e.g. a preset repo
   referenced by multiple projects or multiple people): only for patterns
   that are useful to essentially everyone who extends it, not just your
   own workflow. Check that config's own contribution policy (if it
   documents one) before proposing a rule there -- "universal for me" isn't
   the same as "universal for every consumer of that config."
2. **Global layer**: rules useful across more than one project on this
   machine.
3. **Project layer**: rules specific to this one project (project-specific
   subcommands, in-repo script paths, project-local service endpoints).

Within each of those, choose the plain file vs. the `.local.yml` override
based on sharing intent: **the plain `runok.yml` is for rules you want
version-controlled and shared** (with your team, or with your own other
machines via a dotfiles-style setup); **`.local.yml` is for personal-only
adjustments you don't want propagated** even if the surrounding config is
shared.

Decision axes, summarized:

- Global vs. project: does this rule make sense outside this one project?
- Shared vs. local: is the surrounding config file committed/shared, and do
  you want this specific rule to travel with it?

Also check `metadata.cwd` and `approved` across an entry's occurrences: if a
command is only ever approved from one specific repo's cwd, that's a signal
to scope it to that project's layer (or a CEL `when` cwd condition) instead
of the global layer, even if the pattern itself looks generic.

### 4. Propose to the user

Split the proposal into three confidence groups. **Don't cram more than
~100 rows into one table.**

- **Clear-cut**: disposition is uniquely determined by this skill's rules
  -- e.g. a high-frequency read-only subcommand, a known wrapper addition,
  an arbitrary-code-execution command routed to explicit-ask. Still show it
  once for confirmation, but the user should just be approving, not
  deciding.
- **Borderline**: needs the user's judgment -- e.g. a broad `allow` pattern
  whose full power is ambiguous, a workflow call only the user can make
  (should PR auto-merge be allowed?).
- **Unclear**: the candidate's intent itself isn't understood yet. Don't
  propose a disposition -- investigate further and bring it back in a later
  round.

Show the clear-cut table first; move to writing only after the user
confirms it. Handle borderline items in a separate pass.

#### 4a. Fill the checklist and proposal table from `references/checklists.md`

Before proposing any disposition, read `references/checklists.md`. It has:

- The **R1-R5 checklist** every `allow` proposal must fill (arbitrary code
  execution, arbitrary HTTP writes, destructive operations, subcommand
  existence, and the reach of `*` in the pattern).
- The **Q1-Q2 checklist** every explicit-ask proposal must fill (why
  `allow` doesn't work, why `deny` doesn't work).
- The **D1-D2 checklist** every `deny` proposal must fill (why `allow`
  doesn't work, why explicit-ask doesn't work).
- The two-part **proposal table format** (summary table + per-candidate
  checklist expansion) and a worked example of both.

Every checklist item must be filled with a **concrete, checkable fact** --
an argument-structure enumeration, a confirmation command you actually ran,
an official docs URL, real `*`-expansion examples. Opinion or
self-assessment ("probably safe," "seems narrow") doesn't count; a
candidate you can't fill in gets downgraded to explicit-ask.

**Never mix into a write**:

- Anything the user hasn't explicitly approved -- no "while I'm at it"
  additions. Anything not in the proposal table goes into a later round
  instead.
- Repeated "is this okay?" confirmations for dispositions this skill's
  rules already determine unambiguously -- that's deferral by another name.
  Ask the user only when they want to deviate from what this skill's rules
  would otherwise produce (e.g. turning a would-be explicit-ask into a
  literal allow).

### 5. Write

Apply every approved disposition.

**allow**: append to the chosen config layer. **Every rule gets an inline
`tests:`** with at least the real command(s) that motivated the proposal --
the broader the pattern (more `*`), the more test cases it should carry, to
catch unintended matches early.

```yaml
- allow: 'gh pr view *'
  tests:
    - allow: 'gh pr view 123'
    - allow: 'gh pr view 456'
```

**deny**: append with `message` explaining why, and `fix_suggestion` when a
safer alternative exists, plus `tests:`.

**wrapper**: add to `definitions.wrappers` at the layer decided in 3d. If it
belongs in a shared config you don't maintain yourself (e.g. an upstream
preset repo), propose it as a separate contribution to that repo rather
than writing it locally.

**explicit ask**: append with `message` stating the confirmation intent
(this is what replaces an external ignore-list -- the reason now lives in
the rule itself), plus `tests:`.

```yaml
- ask: 'gh pr merge *'
  message: 'Confirm merge method and target branch every time.'
  tests:
    - ask: 'gh pr merge 123'
```

### 6. Test

Run `runok test` (or `runok test -c <config-path>` for a specific layer). On
failure, diagnose and fix it yourself:

- Pattern conflicts with another rule (e.g. collides with an existing
  `deny`) -> narrow the pattern.
- A test's expected outcome was wrong -> fix the test.
- A priority conflict with an existing rule can't be resolved cleanly ->
  withdraw the new rule and fall back to explicit-ask (reaching zero takes
  priority).

Don't ask the user "what should I do?" here -- fix it and retry. Only report
back if it still doesn't pass after repeated attempts.

### 7. Confirm convergence

Re-run the query from **Goal**.

- Empty -> done, proceed to step 8.
- Non-empty -> back to step 0. Remaining entries are either something
  missed on the previous pass or a new `ask` recorded in the meantime --
  same procedure applies.

There's no cap on how many loops this takes; an empty result is the only
exit condition.

### 8. Wrap up

Once `runok test` passes and the goal query is empty, commit the changes
with a message describing the disposition work (e.g. which commands were
converted to allow/deny/explicit-ask rules).

## Don'ts

- Don't ask "since when?" / "where should this go?" for every candidate
  when the user gave no scope -- decide it yourself. When the user _did_
  specify a scope, always translate it into the matching `runok audit`
  flag.
- Don't stop the loop with "the rest next time" -- keep going until the
  goal query is empty.
- Don't escalate to `deny` just because you don't want to write `allow` --
  the deny threshold is high.
- Don't route a candidate to explicit-ask just because a safe `allow`
  wasn't obvious. Explicit-ask is for positive intent only; if the blocker
  is runok's expressiveness, name the feature gap, propose filing an issue,
  and add the `# awaiting runok feature` comment.
- Don't add an `allow` the user hasn't explicitly approved, even "while
  you're at it."
- Don't propose `allow`/`deny` without filling every checklist item from
  `references/checklists.md` (R1-R5 / Q1-Q2 / D1-D2) with a concrete fact.
  A candidate you can't fill in gets downgraded to explicit-ask and **stays
  in the table** -- dropping it from the table silently means the user
  never learns it's still sitting in `ask` history.
- Don't route a wrapper-eligible command to explicit-ask -- register the
  wrapper so the inner command gets evaluated.
- Don't write a broad `allow` without evaluating the command's actual
  capability (least-privilege principle).
- Don't let the main agent evaluate risk for the full candidate batch alone
  -- split into sub-agent batches (3a).
- Don't let the main agent unilaterally upgrade a sub-agent's
  explicit-ask/deny call to allow (loosening edits are out of scope for the
  main agent).
- Don't hardcode an absolute home-directory path in a CEL `when` clause --
  use `env.HOME`.
- Don't propose a rule without reading the existing presets and config
  layers first (avoid duplicate proposals).
- Don't assert a runok feature doesn't exist without checking the schema
  first.
- Don't design a `deny` rule for a subcommand that doesn't actually exist
  -- it's a CLI-level failure already; treat it as the "auto-deny unknown
  subcommand" feature gap instead.
- Don't write a project-specific literal rule into a shared/global config
  layer -- keep it in the project layer.
- Don't propose a rule for a shared/upstream preset config without checking
  that config's own contribution policy first.
