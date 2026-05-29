# rn-scaffold

A Claude Code plugin that scaffolds production-ready React Native CLI projects.

**Current version:** v0.1.4
**Detailed docs:** [`docs/guide.md`](docs/guide.md) — what each skill creates, conventions, troubleshooting, changelog.

## What's inside

Three skills, all auto-discovered after install:

| Skill | What it does |
|---|---|
| `/rn-setup` | Interactive scaffold — asks your preferences (package manager, state, API, navigation, env, CI) and sets up the full project. Now also writes a hardened `metro.config.js`, an ESLint `no-console` guard, secrets-hygiene `.gitignore` patterns, and optional GitHub Actions workflows. |
| `/rn-core` | Run after `rn-setup`. Installs production dependencies, creates the theme system, core hooks, and base components. Also writes the state stores — including an optional Keychain-encrypted MMKV storage layer with auto-wired Zustand or `redux-persist` integration. |
| `/rn-security` | Run after `rn-core`. Adds platform-level hardening — Android `network_security_config.xml` (TLS-only release + Metro debug exception) wired into `AndroidManifest.xml`. Idempotent. |

## Install (one-time per machine)

```
/plugin marketplace add JasoliyaShruti/rn-scaffold-plugin
/plugin install rn-scaffold@rn-scaffold-marketplace
```

Then **fully quit and reopen Claude Code**. Skills load at session start.

## Use

In a fresh React Native CLI project, in order:

```
/rn-setup       # 6 preference questions; scaffolds project + lint + metro + CI
/rn-core        # 3 preference questions; installs prod deps + theme + stores
/rn-security    # Android NSC + manifest wiring (no questions)
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
