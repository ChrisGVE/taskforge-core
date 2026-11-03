# taskforge-core

The engine behind taskforge: one task model shared by Taskwarrior, OmniFocus and the
`TODO:` comments in your code, with bottom-up planning and a critical path on top.

Taskwarrior is the upstream store. OmniFocus is downstream and only gets written when a plan
is sealed, as a minimal diff. Code comments carrying a `[task:<uuid>]` marker are tasks too.

Sibling repos:

- [taskforge-mcp](https://github.com/ChrisGVE/taskforge-mcp): MCP server for OmniFocus and
  Taskwarrior
- [taskforge.nvim](https://github.com/ChrisGVE/taskforge.nvim): Neovim front end

## Status

Early design. There's no code here yet.

## License

Apache-2.0, see [LICENSE](LICENSE). Copyright 2026 Christian C. Berclaz.
