# Vmotif skills

[Agent skills](https://skills.sh) for [Vmotif](https://vmotif.com), the node-based design canvas. A skill is a `SKILL.md` an agent loads when a task matches it; these teach Claude Code, Codex, Cursor and other agents how to design on a Vmotif canvas through the hosted MCP server.

The skills assume the Vmotif MCP server is connected. Install it from https://vmotif.com/docs/mcp or the [`vmotif/mcp`](https://github.com/vmotif/mcp) repository.

## Install

```sh
npx skills add vmotif/skills
```

Add `-a claude-code`, `-a codex` or `-a cursor` to install for one agent only, or `--skill vmotif` to pick a skill by name. Any agent that reads `SKILL.md` files can also take the folder as is.

## Skills

| Skill | What it teaches |
|---|---|
| `vmotif` | The working loop for the MCP server: choose the billing scope, pick or create the canvas, generate with a `client_request_id`, wait by event or by polling, hand back the canvas link. Inputs and uploads, brands and DESIGN.md, graph edits, saved workflows, browser nodes, MCP Events subscriptions, and what spends the wallet. |

## Support

https://vmotif.com/docs/support · Privacy: https://vmotif.com/privacy · Terms: https://vmotif.com/terms

## Contributing

This repository is published from Vmotif's private monorepo, so pull requests here can't be merged. To report a problem or suggest a change, open an issue or contact [support](https://vmotif.com/docs/support).
