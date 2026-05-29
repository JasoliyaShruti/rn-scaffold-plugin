---
name: rn-security
description: Adds the foundational platform-level security hardening that /rn-setup and /rn-core don't cover — Android network security config (TLS-only, with Metro debug exception) wired into AndroidManifest.xml. Idempotent.
allowed-tools: Bash Read Write Edit Glob Grep
---

# React Native Security — Platform Hardening

This skill adds the Android-side network security config that's missing from the default RN template and wires it into `AndroidManifest.xml`. iOS App Transport Security (ATS) is already on by default in the RN template — no action needed there for the baseline.

**Prerequisite:** run `/rn-setup` and `/rn-core` first. The scaffolded `AndroidManifest.xml` must exist at `android/app/src/main/AndroidManifest.xml`.

**Recommended order:** `/rn-setup` → `/rn-core` → `/rn-security`.

---

## What this skill covers (and what it deliberately doesn't)

| Control | Status | Notes |
|---|---|---|
| Android `network_security_config.xml` — TLS-only release + Metro localhost debug exception | ✅ This skill | Universal baseline |
| Wires `android:networkSecurityConfig` into `AndroidManifest.xml` | ✅ This skill | Idempotent |
| iOS App Transport Security | ✅ Already on | Default in the RN template `Info.plist`; no action |
| Certificate pinning | ⏳ Deferred — feature-time | Requires real production API host; pin rotation needs operational ownership |
| Android ProGuard / R8 release minification | ⏳ Deferred — feature-time | Needs a release-build smoke test in the consumer project |
| Screenshot prevention + app-switcher privacy overlay (Android `FLAG_SECURE`, iOS overlay) | ⏳ Deferred — feature-time | Selective per-screen control (OTP, payment) — added when those flows exist |
| Sentry / crash-reporting `beforeSend` PII scrubbing | ⏳ Deferred — feature-time | Per-project rules; install Sentry first |
| Root / jailbreak detection | ⏳ Deferred — feature-time | High false-positive rate on Indian Android; use risk scoring, not hard block |

> The deferred items aren't oversights — they all depend on consumer-project specifics (real API host, real release-build flow, real flows that need overlays). Adding them here would either ship inert config or wrong defaults. Surface them to the user at the end of the skill so they know what's coming.

---

## Step 1 — Android network security config (release)

Create `android/app/src/main/res/xml/network_security_config.xml`. **If the file already exists, skip — do not overwrite.** A teammate may have customised it.

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!--
        Production network policy: TLS-only, system CAs only.
        - Blocks all plaintext HTTP traffic at the OS level (Android 7+).
        - Disallows user-installed CAs, which blocks most casual MITM
          (corporate proxies, dev tools like Charles/Proxyman in non-debug builds).
        - System CAs are still trusted, so normal HTTPS to public APIs works.
    -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

> **What this changes vs. the RN template default:** the template sets `android:usesCleartextTraffic="true"` in `AndroidManifest.xml` so Metro can talk to localhost in debug. That permission applies to *all* network traffic, including production. This file overrides it: cleartext is blocked everywhere except where the debug variant (Step 2) re-allows it for Metro.

---

## Step 2 — Android network security config (debug — Metro localhost exception)

Create `android/app/src/debug/res/xml/network_security_config.xml`. **If the file already exists, skip.**

The `src/debug/` resource directory overrides `src/main/` only for debug builds — release builds use the strict config from Step 1.

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!--
        Debug-only override: allow cleartext traffic to Metro's localhost endpoints.
        This file ONLY applies to debug builds. Release builds use src/main/res/xml/
        which forbids all cleartext.

        Hosts allowed:
        - localhost / 127.0.0.1: local development on a physical device via adb reverse
        - 10.0.2.2: Android emulator's loopback to the host machine
        - 10.0.3.2: Genymotion emulator's loopback
    -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">localhost</domain>
        <domain includeSubdomains="false">127.0.0.1</domain>
        <domain includeSubdomains="false">10.0.2.2</domain>
        <domain includeSubdomains="false">10.0.3.2</domain>
    </domain-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

> **Tradeoff to flag to the user:** this means Charles/Proxyman won't see traffic from the app even in debug, unless they explicitly add the proxy host to `<domain-config>` here or install a debug-only user-CA trust block. That's by design — proxy interception is a deliberate, named action, not a side effect of running a debug build. If the team is mid-debug-session, share that they should add their proxy host temporarily and remember to remove it before merging.

---

## Step 3 — Wire it into `AndroidManifest.xml` (idempotent)

Open `android/app/src/main/AndroidManifest.xml`. Look for the `<application>` opening tag.

**Check first:** if the attribute `android:networkSecurityConfig="@xml/network_security_config"` is already present anywhere on the `<application>` tag, do NOT modify the file. Print: `AndroidManifest.xml already references network_security_config — skipping.` and continue.

**Otherwise:** add the attribute to the existing `<application>` tag. The default RN-template `<application>` looks like:

```xml
<application
    android:name=".MainApplication"
    android:label="@string/app_name"
    android:icon="@mipmap/ic_launcher"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:allowBackup="false"
    android:theme="@style/AppTheme"
    android:supportsRtl="true">
```

Add the `android:networkSecurityConfig` line (placement among attributes doesn't matter to Android, but conventionally goes near `android:theme`):

```xml
<application
    android:name=".MainApplication"
    android:label="@string/app_name"
    android:icon="@mipmap/ic_launcher"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:allowBackup="false"
    android:theme="@style/AppTheme"
    android:networkSecurityConfig="@xml/network_security_config"
    android:supportsRtl="true">
```

> **Why idempotent matters:** running this skill twice on the same project shouldn't add the attribute twice (which would be invalid XML), and shouldn't overwrite a user-customised manifest. The file-exists / attribute-exists checks make re-runs safe. If the user has done deeper customisation (e.g. added their own `<network-security-config>` block), the skill bails out and tells them — it doesn't try to reconcile.

> **About the existing `android:usesCleartextTraffic` attribute:** if the manifest has `android:usesCleartextTraffic="true"` (RN template default), leave it alone. The `<network-security-config>` reference takes precedence on Android 7+ (API 24+), so the `usesCleartextTraffic` flag becomes a fallback for older devices. RN's minSdk has been >= 24 for several majors now, so the older-device path is rare — but if minSdk is bumped below 24 in the future, the cleartext flag does the right thing on those builds too.

---

## Step 4 — Verify

Run a release-flavour build to confirm the manifest still parses and the strict config is wired:

```bash
cd android && ./gradlew :app:assembleRelease --dry-run && cd ..
```

(Use `--dry-run` so you don't actually need a release keystore to verify the manifest.)

If the user wants to actually exercise the runtime block: in a debug build, attempt to fetch an `http://` URL outside the allowlist (e.g. `http://example.com`) — Android should refuse with a `CLEARTEXT communication ... not permitted by network security policy` error. That confirms the config is live.

---

## Step 5 — Summarise what's done, and what's still on the user

Print (to the user, not the file):

```
✅ Network security config (Android): TLS-only release + Metro debug exception
✅ AndroidManifest.xml wired
✅ iOS ATS: on by default in the RN template — no action needed

⏳ Deferred (add when feature-time arrives):
   - Certificate pinning (needs real production API host + pin rotation owner)
   - ProGuard / R8 (needs release-build smoke test)
   - Screenshot prevention (FLAG_SECURE on Android, app-switcher overlay on iOS)
   - Sentry beforeSend PII scrubbing (after Sentry is installed)
   - Root/jailbreak risk scoring (after auth flow exists)

Recommended companion baseline (already covered by /rn-setup + /rn-core if you ran them):
   - ESLint no-console + Metro pure_funcs (no console.log/info/debug in prod bundles)
   - .gitignore patterns for keystores and env files
   - Encrypted MMKV with Keychain-stored key (if /rn-core Q3 = Yes)
   - GitHub Actions CI with lint + tsc + test + weekly yarn audit (if /rn-setup Q6 = Yes)
```

---

## Rules

- Do NOT overwrite `network_security_config.xml` if it already exists in either `src/main/` or `src/debug/`.
- Do NOT add `android:networkSecurityConfig` to `AndroidManifest.xml` if it's already present.
- Do NOT remove `android:usesCleartextTraffic` from the manifest — leave it for the < API 24 fallback path.
- Do NOT touch iOS `Info.plist` — ATS is already on by default and the template is correct.
- If any of the idempotency checks find prior installation, print what was skipped and continue cleanly. Never fail the skill on already-configured state.
