# agents-md

LLM instructions and MCP configurations for bootstrapping Convex-powered SvelteKit applications with [Crush](https://charm.land/crush).

## Overview

This repository provides:

- **`AGENTS.md`** — Comprehensive instructions for LLMs to create and manage SvelteKit + Convex projects
- **`crush.json`** — MCP server configurations for Convex, Svelte, and GitHub integrations

## Setup

### 1. Install Crush Configuration (One-Time)

Copy `crush.json` to your Crush config directory:

```bash
# Linux/macOS
cp crush.json ~/.config/crush/crush.json

# Windows (PowerShell)
Copy-Item crush.json $HOME\.config\crush\crush.json
```

> **Note:** Set the `GH_PAT` environment variable with your GitHub Personal Access Token for GitHub MCP integration.

### 2. Add Agent Instructions to Projects

Copy `AGENTS.md` to the root of any new project you want to bootstrap:

```bash
cp AGENTS.md /path/to/your/new-project/
```

## What's Included

### AGENTS.md

Detailed instructions covering:

- Project bootstrapping with SvelteKit, Convex, Tailwind CSS, and Bits UI
- Convex MCP server tools usage (status, tables, functions, logs, etc.)
- Svelte 5 runes and reactive patterns
- convex-svelte integration patterns
- Bits UI headless component documentation access
- Nano Banana Pro image generation integration
- Best practices and troubleshooting

### crush.json

MCP server configurations:

| Server | Type | Description |
|--------|------|-------------|
| `github` | HTTP | GitHub Copilot MCP for repository management |
| `convex-mcp` | stdio | Convex deployment interaction |
| `svelte` | HTTP | Svelte 5 and SvelteKit documentation |

## Usage

1. Create a new empty directory for your project
2. Copy `AGENTS.md` into it
3. Start Crush in that directory
4. Ask the LLM to bootstrap a new SvelteKit + Convex application

The LLM will follow the instructions in `AGENTS.md` to set up a complete project with:

- SvelteKit with TypeScript
- Convex backend with convex-svelte
- Tailwind CSS with typography and forms plugins
- Bits UI for headless components
- ESLint, Prettier, and Vitest
- Paraglide for internationalization
- GitHub repository creation

## License

[MIT](LICENSE)
