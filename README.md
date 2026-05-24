# rn-scaffold

A Claude Code plugin that scaffolds production-ready React Native CLI projects.

**Current version:** v0.1.3
**Detailed docs:** [`docs/guide.md`](docs/guide.md) — what each skill creates, conventions, troubleshooting, changelog.

## What's inside

Two skills, both auto-discovered after install:

| Skill | What it does |
|---|---|
| `/rn-setup` | Interactive scaffold — asks your preferences (package manager, state, API, navigation, env config) and sets up the full project. |
| `/rn-core` | Run after `rn-setup`. Installs production dependencies and creates the theme system, core hooks, and base components. |

## Install (one-time per machine)

```
/plugin marketplace add JasoliyaShruti/rn-scaffold-plugin
/plugin install rn-scaffold@rn-scaffold-marketplace
```

Then **fully quit and reopen Claude Code**. Skills load at session start.

## Use

In a fresh React Native CLI project:

```
/rn-setup    # answers 5 preference questions, scaffolds the project
/rn-core     # installs production deps + theme + core components
```

## Update

```
/plugin marketplace update rn-scaffold-marketplace
/plugin update rn-scaffold
```

Restart Claude Code.

## Uninstall

```
/plugin uninstall rn-scaffold
/plugin marketplace remove rn-scaffold-marketplace
```

## License

MIT
