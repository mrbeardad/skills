# Skills

Reusable agent skills.

## Explain Architecture

[explain-architecture](skills/explain-architecture/SKILL.md) explains software
architecture and execution flows with Mermaid diagrams grounded in source code.
It supports high, medium, and low abstraction levels, plus an optional detail
mode for channel capacities and boundary-data appendix tables.

Install for Codex across projects:

```sh
npx skills add mrbeardad/skills --skill explain-architecture -g -a codex
```

List available skills without installing:

```sh
npx skills add mrbeardad/skills --list
```

Example prompts:

- "Use explain-architecture to show the host-side architecture at low level."
- "Use detail mode to include channel capacities and boundary data."
