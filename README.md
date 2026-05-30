# rn-scaffold

A Claude Code plugin that scaffolds production-ready React Native CLI projects.

**Current version:** v0.1.5
**Detailed docs:** [`docs/guide.md`](docs/guide.md) — what each skill creates, conventions, troubleshooting, changelog.

## What's inside

Three skills, all auto-discovered after install:

| Skill | What it does |
|---|---|
| `/rn-setup` | Interactive scaffold — asks your preferences (package manager, state, API, navigation, env, CI) and sets up the full project. Writes a hardened `metro.config.js`, ESLint `no-console` guard, secrets-hygiene `.gitignore` patterns, optional GitHub Actions workflows, and (when env config is on) Android `productFlavors` for development/stage/production, a guided iOS scheme setup with a verifier script, and 18 env-aware npm scripts (`yarn android:development`, `yarn apk:production`, `yarn aab:stage`, etc.). |
| `/rn-core` | Run after `rn-setup`. Installs production dependencies, creates the theme system, core hooks, and base components. Writes the state stores — including an optional Keychain-encrypted MMKV storage layer with auto-wired Zustand or `redux-persist` integration. Offers a multi-select Q4 for optional utilities: deep-linking base skeleton, error handler, toast manager, location permission helper, network status hook + offline banner. |
| `/rn-security` | Run after `rn-core`. Adds platform-level hardening — Android `network_security_config.xml` (TLS-only release + Metro debug exception) wired into `AndroidManifest.xml`. Idempotent. |

## Roadmap — deferred skills

Coming separately, in approximate priority order:

| Skill | Scope |
|---|---|
| `/rn-firebase` | Firebase Analytics + Crashlytics + Performance + Remote Config + Sentry, bundled as one adoption. |
| `/rn-update-ota` | Force-update gate (semver-driven) + OTA delivery + `compareVersions` util + `AppUpdateBottomSheet`. |
| `/rn-push` | FCM lifecycle + permission + push-payload deep-link routing + adapter to WebEngage/MoEngage/CleverTap (user picks vendor). |
| `/rn-ci-cd` | Fastlane lanes + GitHub Actions release workflow + keystore-from-base64 + AAB upload + Sentry source-map upload. |

## Install (one-time per machine)

```
/plugin marketplace add JasoliyaShruti/rn-scaffold-plugin
/plugin install rn-scaffold@rn-scaffold-marketplace
```

Then **fully quit and reopen Claude Code**. Skills load at session start.

## Use

In a fresh React Native CLI project, in order:

```
/rn-setup       # 6 preference questions; scaffolds project + lint + metro + CI + (opt) flavours
/rn-core        # 4 preference questions; installs prod deps + theme + stores + (opt) utilities
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
