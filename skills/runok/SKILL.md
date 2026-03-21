---
name: runok
description: >-
  runok: Manage runok configuration files (runok.yml) for command execution permissions.
  Use this skill whenever the user mentions "runok", asks about runok rules, runok.yml,
  runok.local.yml, or runok CLI commands (runok check, runok exec, runok init, runok test,
  runok audit, runok update-presets). Also trigger when the user wants to: add/edit/delete
  allow/deny/ask rules for commands, set up sandbox presets for filesystem or network
  isolation, configure command patterns with wildcards or alternation, write CEL when-clauses
  for conditional rules, use extends to inherit shared configs, use official presets
  (runok-presets), write inline tests for rules (runok test), view audit logs, initialize
  a new runok.yml, investigate why a command was allowed or denied, or understand how runok
  pattern matching works. This skill MUST be used for any question about runok — including
  "what is runok", "how does runok work", debugging unexpected rule matches, and editing
  any runok configuration file. If you see runok.yml or runok.local.yml in the project,
  use this skill to understand and modify them.
---

# runok Configuration Management

## Documentation Reference

When you need detailed runok documentation:

1. **First**, use WebFetch to retrieve the abridged docs from
   `https://runok.fohte.net/llms-small.txt` and extract relevant guidance
2. **If more detail is needed**, fetch the full docs from
   `https://runok.fohte.net/llms-full.txt`

Always refer to these docs for the latest syntax, schema, and examples.

## Configuration File Discovery

runok uses a 4-layer configuration system. Search for config files in this order:

| Priority    | Scope            | Path                              | Purpose                    |
| ----------- | ---------------- | --------------------------------- | -------------------------- |
| 1 (lowest)  | Global           | `~/.config/runok/runok.yml`       | User-wide defaults         |
| 2           | Global override  | `~/.config/runok/runok.local.yml` | Personal adjustments       |
| 3           | Project          | `./runok.yml`                     | Project-specific rules     |
| 4 (highest) | Project override | `./runok.local.yml`               | Personal project overrides |

**Extension fallback**: If `.yml` is not found, check for `.yaml` extension at the same path.

**Note**: Discovery checks all layers. When multiple configs exist, higher priority values override lower ones during merging. The global path defaults to `~/.config/runok/` when `$XDG_CONFIG_HOME` is not set.

**Discovery procedure**:

Use `ls` (via Bash tool) to check for files. Do not use Glob for discovery.

1. Check for project-level files first: `./runok.yml` (or `./runok.yaml`), then `./runok.local.yml` (or `./runok.local.yaml`)
2. Check for global files: `~/.config/runok/runok.yml` (or `.yaml`), then `~/.config/runok/runok.local.yml` (or `.yaml`)
3. If `$XDG_CONFIG_HOME` is set, use `$XDG_CONFIG_HOME/runok/` instead of `~/.config/runok/`
4. If **no config file exists**, report this to the user and propose initialization (see "Configuration Initialization" section)

## Configuration Schema Overview

```yaml
extends: [] # list[str] - inherited config references
defaults:
  action: ask # "allow" | "ask" | "deny" - when no rule matches
  sandbox: null # str - default sandbox preset name
definitions:
  paths: {} # map[str, list[str]] - named path lists
  vars: {} # map[str, VarDefinition] - typed variable definitions
  sandbox: {} # map[str, SandboxPreset] - sandbox presets
  wrappers: [] # list[str] - wrapper patterns (e.g., 'sudo <cmd>')
rules: [] # list[RuleEntry] - rules (evaluated via Explicit Deny Wins)
audit: # audit log settings (global config only)
  enabled: true # bool
  path: null # str - log directory
  rotation:
    retention_days: 7 # int
tests: # test cases for `runok test`
  extends: [] # list[str] - test-only config
  cases: [] # list[TestEntry]
```

**Rule entry fields**:

- One of `allow`, `deny`, or `ask` (required, mutually exclusive) - the command pattern
- `when` (optional) - CEL expression for conditional rules
- `message` (optional) - message shown when the rule matches
- `fix_suggestion` (optional) - alternative command suggestion (for `deny` rules)
- `sandbox` (optional) - sandbox preset name (cannot be used with `deny`)
- `tests` (optional) - inline test cases for this rule (used by `runok test`)

## Rule Priority Model: Explicit Deny Wins

When multiple rules match a command, the **most restrictive** action wins:

| Priority    | Action  | Meaning                          |
| ----------- | ------- | -------------------------------- |
| 2 (highest) | `deny`  | Block the command                |
| 1           | `ask`   | Prompt the user for confirmation |
| 0           | `allow` | Permit the command               |

Rule order in the config file does **not** affect priority. A `deny` rule always wins, even if an `allow` rule appears later. This is inspired by AWS IAM's Explicit Deny model.

## Rule Management

### Adding Rules

When the user requests a new rule in natural language:

1. **Determine the action**: `allow`, `deny`, or `ask` based on intent
2. **Construct the pattern** using runok's pattern syntax (see below)
3. **For `deny` rules**: Always propose a `message` explaining why it's denied, and a `fix_suggestion` with a safer alternative when applicable
4. **For conditional rules**: Generate a `when` clause using CEL syntax
5. **Confirm the write target**: Ask whether to write to global (`~/.config/runok/runok.yml`) or project (`./runok.yml`) config
6. **Validate before writing**: Ensure the rule has exactly one action field, patterns are syntactically valid, and `sandbox` is not used with `deny`

### Pattern Syntax Reference

| Syntax                 | Example                    | Description                                                     |
| ---------------------- | -------------------------- | --------------------------------------------------------------- |
| Literal                | `git status`               | Exact match                                                     |
| Wildcard `*`           | `git *`                    | Matches 0+ tokens                                               |
| Glob                   | `*.txt`, `list-*`          | `*` inside a literal matches 0+ characters                      |
| Alternation            | `push\|pull\|fetch`        | Pipe-separated alternatives                                     |
| Negation               | `!--force`                 | Matches anything except the specified value                     |
| Optional group         | `[-f]`, `[-X POST]`        | Matches with or without the group                               |
| Placeholder `<cmd>`    | `sudo <cmd>`               | Captures wrapped command for recursive evaluation               |
| Placeholder `<opts>`   | `env <opts> <cmd>`         | Absorbs 0+ flag-like tokens                                     |
| Placeholder `<vars>`   | `env <vars> <cmd>`         | Absorbs 0+ KEY=VALUE tokens                                     |
| Path ref `<path:name>` | `cat <path:sensitive>`     | Matches against named path list in definitions                  |
| Var ref `<var:name>`   | `bash <var:test-script>`   | Matches against typed variable definition (arg or cmd position) |
| Multi-word alternation | `"npx prettier"\|prettier` | Alternatives containing spaces                                  |

**Note**: The `\|` in the table above is a Markdown table escape. In actual runok patterns, use an unescaped `|` for alternation (e.g., `git push|pull|fetch`).

**Quotes and glob**: Quotes (`"..."` or `'...'`) act as grouping only -- they allow spaces inside a single token but do **not** disable glob expansion. `*` inside quotes is treated as a glob. Use `\*` to match a literal `*` character.

**Matching behavior notes**:

- **Order-independent flag matching**: Flags match regardless of their position in the command
- **`=`-joined flag values**: `-X=POST` and `--request=POST` are supported
- **Fused short flags**: `-n3` matches patterns like `-n *` (flag and value joined without space)
- **Combined short flags**: `-am` is not split into individual flags -- matched as a single token
- **Recursion limit**: 10,000 steps

### Editing Rules

When editing existing rules:

1. Use the **Edit** tool to modify YAML content
2. **Preserve YAML formatting**: Maintain existing indentation, blank lines, and comments
3. **Preserve comments**: Never remove or modify YAML comments unless explicitly asked
4. When changing a rule's action (e.g., `allow` to `deny`), update the key name and add `message`/`fix_suggestion` if switching to `deny`

### Deleting Rules

When removing rules:

1. Use the **Edit** tool to remove the rule entry
2. Remove the entire rule block including all its fields (`when`, `message`, `fix_suggestion`, `sandbox`, `tests`)
3. Preserve surrounding formatting and comments

### Conditional Rules (when clauses)

The `when` field uses CEL (Common Expression Language) expressions:

**Available context variables**:

- `env` (map) - Environment variables. Example: `env.CI == 'true'`
- `flags` (map) - Parsed flags (leading dashes removed). Example: `flags.request == 'POST'`
- `args` (list) - Positional arguments. Example: `args[0] == 'production'`
- `paths` (map) - Named path lists from definitions. Example: `size(paths.sensitive) > 0`
- `redirects` (list) - Redirect operators. Each element has `type` (`"input"`, `"output"`, `"dup"`), `operator` (e.g., `">"`, `">>"`), `target`, and `descriptor`. Example: `redirects.exists(r, r.type == "output")`
- `pipe` (object) - Pipeline position with `stdin` (bool) and `stdout` (bool) fields. Example: `pipe.stdin` (true if receiving piped input)
- `vars` (map) - Values captured by `<var:name>` placeholders. Example: `vars['instance-ids'] == 'i-prod-001'`

**Supported operators**: `==`, `!=`, `<`, `>`, `<=`, `>=`, `&&`, `||`, `!`, `in`, `size()`, `.startsWith()`, `.endsWith()`, `.contains()`, `.exists()`, `.exists_one()`, `has()`

Example:

```yaml
- deny: 'curl -X|--request * *'
  when: "flags.request == 'POST' && args[0].startsWith('https://prod.')"
  message: 'Direct POST to production API is not allowed.'
  fix_suggestion: 'Use the staging endpoint instead.'

# Block piped execution of sh/bash (e.g., curl | sh)
- deny: 'sh'
  when: 'pipe.stdin'
```

## Definitions Management

### paths

Named path lists referenced by `<path:name>` in rule patterns and sandbox deny lists:

```yaml
definitions:
  paths:
    secrets:
      - ~/.ssh
      - ~/.gnupg
    config:
      - /etc
```

- Each key maps to a list of filesystem paths
- Referenced in patterns as `<path:name>` (e.g., `<path:secrets>`)
- **Validate** that any `<path:name>` in rules resolves to an existing `definitions.paths` entry

### vars

Typed variable definitions referenced by `<var:name>` in rule patterns. Can be used in both argument and command position:

```yaml
definitions:
  vars:
    instance-ids:
      values:
        - i-abc123
        - i-def456
    test-script:
      type: path
      values:
        - ./tests/run
    runok:
      values:
        - runok
        - 'cargo run --'
        - type: path
          value: target/debug/runok
```

- `type` (optional, default: `"literal"`) - `"literal"` for exact match, `"path"` for canonicalized path comparison
- `values` (required) - list of allowed values, each can be a plain string or `{ type, value }` for per-value type override
- Multi-word values (e.g., `"cargo run --"`) consume multiple leading tokens
- In command position: `<var:runok> check` matches `runok check`, `cargo run -- check`, etc.

### wrappers

Patterns for commands that wrap other commands, enabling recursive rule evaluation:

```yaml
definitions:
  wrappers:
    - 'sudo <cmd>'
    - 'bash -c <cmd>'
```

- Each wrapper pattern **must** contain a `<cmd>` placeholder
- When a command matches a wrapper, the inner command is extracted and evaluated recursively
- `<cmd>` only matches token sequences whose first token is not a flag

### sandbox

Named sandbox presets for filesystem and network isolation:

```yaml
definitions:
  sandbox:
    standard:
      fs:
        read:
          deny: [<path:secrets>]
        write:
          allow: [.]
          deny: [.git, .runok, <path:secrets>]
      network:
        allow: true
    strict:
      fs:
        write:
          allow: [./src]
          deny: [<path:secrets>]
      network:
        allow: false
```

- `fs.read.deny` - paths the sandboxed process cannot read (completely inaccessible)
- `fs.write.allow` - directories where writing is permitted
- `fs.write.deny` - paths that cannot be written to, even within writable directories (supports glob patterns and `<path:name>` references)
- `network.allow` - whether network access is permitted (default: `true`). When `false`, TCP/UDP sockets are blocked (Unix domain sockets always permitted)
- Sandbox presets are referenced by name in rule entries or `defaults.sandbox`
- **Cannot** be used with `deny` rules
- **Compound command merging**: Strictest Wins strategy (`write.allow` = intersection, `write.deny`/`read.deny` = union, `network.allow` = AND)

**Legacy format (deprecated)**: The previous `fs.writable`/`fs.deny` format is still accepted but deprecated and emits a warning. Migrate to the new `fs.read`/`fs.write` format:

```yaml
# Deprecated
fs:
  writable: ['.']
  deny: ['.git']

# New format
fs:
  write:
    allow: ['.']
    deny: ['.git']
```

## Extends Management

The `extends` field inherits rules from external configurations:

### Reference Formats

| Format           | Syntax                        | Example                                         |
| ---------------- | ----------------------------- | ----------------------------------------------- |
| Local path       | Relative or absolute          | `./shared/base.yml`, `~/company/runok-base.yml` |
| GitHub shorthand | `github:<owner>/<repo>@<ref>` | `github:example-org/presets@v1.0.0`             |
| Git URL          | HTTPS/SSH + optional `@<ref>` | `https://github.com/org/config.git@v2.0.0`      |

- `@<ref>` accepts tags, branch names, or 40-character commit SHAs
- **When `@<ref>` is omitted** in GitHub shorthand, recommend pinning to a specific version for reproducibility
- Maximum extends depth: 10 levels
- Circular references are detected and rejected

### Official Presets (runok-presets)

The official preset collection at `github:fohte/runok-presets` provides curated rules:

```yaml
extends:
  - 'github:fohte/runok-presets/base@v1'
```

| Preset          | Description                                                         |
| --------------- | ------------------------------------------------------------------- |
| `base`          | Bundles all presets and adds `* --help` / `* --version` rules       |
| `definitions`   | Wrapper definitions (`bash -c`, `sudo`, `xargs`, `find -exec`)      |
| `readonly-unix` | Allow rules for read-only Unix commands (`cat`, `grep`, `ls`, etc.) |
| `readonly-git`  | Allow rules for read-only Git subcommands                           |
| `readonly-gh`   | Allow rules for read-only GitHub CLI subcommands                    |

### Version Pinning

When the user adds a GitHub shorthand without a version (e.g., `github:org/presets`), recommend pinning:

```yaml
# Not recommended - uses default branch, may break unexpectedly
extends:
  - github:example-org/presets

# Recommended - pinned to a specific version
extends:
  - github:example-org/presets@v1.0.0
```

## CLI Commands

### `runok init`

Interactive setup wizard. Creates `runok.yml` and optionally migrates Claude Code Bash permissions to runok rules.

- `--scope user|project` - configuration scope
- `-y`, `--yes` - accept all defaults

### `runok check`

Evaluate a command against rules without executing it.

```sh
runok check [--verbose] [--output-format text|json] [--input-format claude-code-hook] -- <command>
```

- Reads from stdin if no command given (auto-detects JSON or plaintext)
- Exit code `0` on success (regardless of decision), `2` on error

### `runok exec`

Evaluate and execute a command, optionally within a sandbox.

```sh
runok exec [--verbose] [--sandbox <preset>] -- <command>
```

- Exit code: command's own exit code on success, `1` on error, `3` on deny/ask

### `runok test`

Run test cases defined in configuration.

```sh
runok test [-c <path>]
```

- Supports inline per-rule `tests` and top-level `tests.cases`
- Global config is excluded for reproducibility
- Exit code `0` all passed, `1` failures, `2` error

### `runok audit`

View and filter audit log entries.

```sh
runok audit [--action allow|deny|ask] [--since <timespec>] [--until <timespec>] [--command <pattern>] [--dir <path>] [--limit <n>] [--json]
```

- Audit settings are configured only in global `runok.yml`

### `runok update-presets`

Force-update remote presets, bypassing TTL cache. Automatically upgrades version tags based on tag precision (`@v1` → major, `@v1.0` → minor, `@v1.0.0` → patch within major).

## Configuration Initialization

When no runok configuration file exists:

1. **Report** to the user that no config file was found
2. **Recommend** using `runok init` for interactive setup with Claude Code migration support
3. **If manual creation is preferred**, ask where to create it:
   - Global: `~/.config/runok/runok.yml` (applies to all projects)
   - Project: `./runok.yml` (project-specific)
4. **If the target file already exists**, warn the user and ask for confirmation before overwriting
5. **Generate initial configuration** with the following template:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/fohte/runok/main/schema/runok.schema.json

rules: []
```

**Important**: The first line must always be the `# yaml-language-server: $schema=...` comment. This enables schema validation in editors that support yaml-language-server. When editing an existing runok.yml or runok.yaml that lacks this comment, add it as the first line.

6. **Propose additional rules** based on the user's specific requirements - ask what commands they want to allow, deny, or require confirmation for, rather than generating a fixed template

## Investigating Unexpected Rule Matches

When a command is unexpectedly allowed, denied, or asked, follow this procedure **strictly**. Do not speculate about the cause based on config file reading alone.

### Principles

- **Never guess the cause.** Always verify with `runok check --verbose` before stating why a command was allowed/denied.
- **Evidence first.** Every claim about runok's behavior must be backed by actual `runok check` output.
- **Minimal reproduction.** Narrow down the problem to the smallest command that reproduces the unexpected behavior.

### Investigation Workflow

1. **Reproduce with `runok check --verbose`**

   Run the exact command that exhibited unexpected behavior:

   ```bash
   runok check --verbose -- <the-exact-command>
   ```

   Alternatively, pipe the command via stdin to avoid shell escaping issues:

   ```bash
   echo '<the-exact-command>' | runok check --verbose
   ```

   Choose whichever form makes escaping simpler for the command at hand.

   Read the output carefully. It shows which rules were evaluated, which matched, and the final action.

2. **Compare with a known-good case**

   Test a similar command that you expect to produce the correct result:

   ```bash
   runok check --verbose -- <similar-command-expected-to-differ>
   ```

   Comparing the two outputs reveals which rule or pattern causes the divergence.

3. **Binary search for the triggering element**

   If the command is complex (many arguments, flags, subshells, etc.), systematically remove parts to find the minimal command that still triggers the unexpected behavior:
   - Start with the full command
   - Remove roughly half of the arguments/flags
   - Check with `runok check --verbose` after each change
   - Repeat until you find the single element that changes the result

   Example: if `gh pr edit 10 --body "$(echo test)"` is unexpectedly allowed:

   ```bash
   # Full command
   runok check --verbose -- 'gh pr edit 10 --body "$(echo test)"'
   # Remove --body argument
   runok check --verbose -- 'gh pr edit 10'
   # Try different subcommand
   runok check --verbose -- 'gh pr edit'
   ```

4. **State findings with evidence**

   Only after completing the above steps, report:
   - The exact `runok check --verbose` output that demonstrates the issue
   - The minimal command that reproduces the problem
   - Which rule matched (or didn't match) and why

### Common Pitfalls

- **Do not** read `runok.yml` and guess which rule matches. Pattern matching has subtleties (wildcards, wrappers, alternation) that are not obvious from reading config alone.
- **Do not** skip the `--verbose` flag. Without it, you only see the final action, not the rule evaluation trace.
- **Do not** report a root cause without first finding a minimal reproduction.

## Error Handling

### Config File Errors

- **File not found**: Report which paths were checked. Propose initialization
- **YAML syntax error**: Show the error location (line/column). Suggest a fix using the Edit tool
- **Invalid rule structure**: A rule must have exactly one of `allow`, `deny`, or `ask`. Flag rules with zero or multiple action fields

### Pattern Syntax Errors

- **Unclosed quote**: Pattern has `"` or `'` without a matching close
- **Empty alternation segment**: Pattern contains `|` with nothing on one side (e.g., `push|`)
- **Invalid placeholder**: `<...>` content is not a recognized placeholder (`cmd`, `opts`, `vars`, `path:name`, `var:name`)

### Definitions Errors

- **Unresolved path reference**: A rule uses `<path:name>` but `definitions.paths` has no entry for `name`
- **Unresolved variable reference**: A rule uses `<var:name>` but `definitions.vars` has no entry for `name`
- **Missing `<cmd>` in wrapper**: A wrapper pattern in `definitions.wrappers` does not contain `<cmd>`
- **Undefined sandbox preset**: A rule or `defaults.sandbox` references a preset not defined in `definitions.sandbox`
- **Sandbox on deny rule**: A `deny` rule cannot have a `sandbox` field. Remove the `sandbox` field or change the action

### Extends Errors

- **Unresolvable reference**: The file path, GitHub repo, or Git URL cannot be accessed. Check the path/URL and network connectivity
- **Circular dependency**: An extends chain references a config that has already been loaded. Remove the circular reference
- **Depth limit exceeded**: The extends chain is deeper than 10 levels. Flatten the hierarchy

### Security Note

When reporting errors for files referenced via `extends`, **never include the raw file content** in error messages. Only report the file path and the nature of the error (e.g., "YAML parse error at line 5"). An `extends` entry could point to arbitrary local files, and displaying their content would risk exposing sensitive data.

### Resolution Strategy

For all errors:

1. **Identify** the specific error and its location in the YAML file
2. **Explain** what is wrong in plain language
3. **Propose** a concrete fix
4. **Apply** the fix using the Edit tool (with user confirmation for destructive changes)
