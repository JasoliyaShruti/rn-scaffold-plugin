# rn-scaffold

A Claude Code plugin that scaffolds production-ready React Native CLI projects.

## What's inside

Two skills, both auto-discovered after install:

| Skill | What it does |
|---|---|
| `/rn-setup` | Interactive scaffold — asks your preferences (package manager, state, API, navigation, env config) and sets up the full project (folder structure, path aliases, ESLint/Prettier/Husky, providers, etc.) |
| `/rn-core` | Run after `rn-setup`. Installs production dependencies (FlashList, reanimated, MMKV, fast-image, gesture-handler) and creates the theme system, core hooks, and base components. |

## Install (one-time per machine)

In any Claude Code session, run:

```
/plugin marketplace add JasoliyaShruti/rn-scaffold-plugin
/plugin install rn-scaffold@rn-scaffold-marketplace
```

After this, the skills are available in every new project — no per-project setup needed.

## Usage

In a fresh React Native CLI project:

```
/rn-setup
```

Answers preferences in one message (e.g. `1, 1, 2, 1, 1`), then scaffolds the whole project.

After that:

```
/rn-core
```

Installs production deps and creates the theme system + core components.

## Update

When a new version is published:

```
/plugin marketplace update rn-scaffold-marketplace
/plugin update rn-scaffold
```

## Uninstall

```
/plugin uninstall rn-scaffold
/plugin marketplace remove rn-scaffold-marketplace
```

## Troubleshooting

**`/rn-setup` doesn't appear after install.**
Quit and restart Claude Code. Skills are loaded at session start.

**Marketplace add fails with "not found".**
Confirm the repo exists at `github.com/JasoliyaShruti/rn-scaffold-plugin` and is public.

**Install fails with "invalid manifest".**
Open an issue with the error message — the manifest format may have changed.

## License

MIT
