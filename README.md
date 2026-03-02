# runok-claude-code-plugin

Claude Code plugin for [runok](https://github.com/fohte/runok) - command execution permission framework for LLM agents.

## Overview

This plugin provides Claude Code with knowledge of runok configuration files (`runok.yml`), enabling it to:

- Read, create, and edit runok configuration files
- Add, modify, and remove command permission rules
- Manage definitions (paths, wrappers, sandbox presets, commands)
- Configure extends for shared presets
- Handle configuration errors with clear guidance

## Installation

1. Add the marketplace:

   ```
   /plugin marketplace add fohte/runok-claude-code-plugin
   ```

2. Install the plugin:

   ```
   /plugin install runok@runok-claude-code-plugin
   ```

## Usage

The plugin activates automatically when you work with runok configuration or mention "runok" in conversation. You can also invoke it directly:

```
/runok
```

## License

[MIT](LICENSE)
