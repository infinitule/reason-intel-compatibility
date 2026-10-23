# Verification ledger

This document records the basis for the README's factual claims. It separates direct artifact inspection, official source references, and historical local observations. The JSON record contains the artifact-check UTC timestamp. This is a local verification record, not an independent audit or a full test suite.

## Evidence categories

| Label | Meaning |
| --- | --- |
| Artifact inspection | Read from the supplied or rebuilt DMG, extracted archive, or app bundle |
| Official reference | Electron, Apple, or GitHub's published material or metadata |
| Local observation | Output captured while launching or checking the app on the test machine |
| User report | The user explicitly reported that the installed app works |
| Not tested | No result is claimed |

## Original artifact

The source was the user's downloaded Reason DMG. It was mounted read-only for inspection. No modification to that source installer was needed.

- `hdiutil verify` completed successfully for the original DMG.
- `lipo -archs` reported `arm64` for the main executable and the native `reason-hang-recovery` helper.
- `Info.plist` declared app version `0.1.109` and minimum macOS `13.0`.
- The original Electron framework's plist declared `44.9.0`.
- The original CLI manifest declared `0.2.529` and included a checksum entry for the Intel CLI archive.

These are properties of the specific supplied artifact. They do not establish that every Reason distribution is ARM-only or that an official Intel release will never exist.

## Rebuilt artifact

The rebuilt DMG was also mounted read-only.

1. Read the app and Electron framework plists.
2. Enumerated regular, non-symlink Mach-O files in the desktop app and ran `lipo -archs` on each. All 14 returned `x86_64`. Files inside the compressed CLI archive are excluded from this count.
3. Parsed both ASAR headers and compared their complete path sets and file bytes: 53 entries in each, 52 unchanged, with only `dist/main.cjs` differing.
4. Applied the two documented substitutions to the original `main.cjs` in memory and required byte-for-byte equality with the rebuilt entry.
5. Recomputed each ASAR entry's integrity hash and block hashes; compared the archive header hash to `ElectronAsarIntegrity` in the rebuilt plist.
6. Compared both main plists: only `ElectronAsarIntegrity` and the added `ReasonLocalIntelRebuild` marker differed.
7. Ran `codesign --verify --deep --strict` successfully. `codesign -dv` reported `Signature=adhoc`.
8. Verified that the DMG contains the app, installation notes, and an `Applications` symlink targeting `/Applications`.
9. Ran `hdiutil verify` successfully on the rebuilt DMG.

Detailed fields, the Mach-O inventory, and signature output are in [verification.json](verification.json).

### What signature verification means here

A successful local `codesign` verification checks the bundle against its current signature. In this package that signature is ad-hoc. It does not identify the Reason publisher, prove notarization, or replace a security review. Apple's [app trust guidance](https://support.apple.com/en-us/102445) explains the distinct Developer ID and notarization checks.

## Input provenance

### Electron

- Source: [official Electron 44.9.0 release](https://github.com/electron/electron/releases/tag/v44.9.0).
- Asset: `electron-v44.9.0-darwin-x64.zip`.
- The local archive SHA-256 matched the [official SHASUMS256.txt](https://github.com/electron/electron/releases/download/v44.9.0/SHASUMS256.txt) entry and the asset digest returned by GitHub's release API.
- The bundle's framework plist also reports `44.9.0`.

### Reason CLI

- Source: [Reason-hosted Intel CLI archive](https://reasonmachines.com/cli/releases/download/cli-v0.2.529/reason-machines-cli-darwin-x64-0.2.529.tgz).
- Asset: `reason-machines-cli-darwin-x64-0.2.529.tgz`.
- Its SHA-256 matched the Intel archive entry in `SHA256SUMS` inside the original supplied DMG.
- Running the extracted CLI with `--version` returned `0.2.529` during the initial test.

All recorded hashes are in [verification.json](verification.json). Matching checksums establish consistency with those sources, not an independent malware audit.

## Release asset integrity

GitHub's [release API](https://api.github.com/repos/infinitule/reason-intel-compatibility/releases/tags/v0.1.109-intel-local) returned all three intended assets with `state: uploaded`. For the installer:

```text
Filename: Reason-0.1.109-Intel.dmg
Size:     186774846 bytes
SHA-256:  2833074b90362aac9763442a70fd74a1cee1ee5cd3655122cef61be60e2e4e83
```

This digest matched the locally generated file. The release was marked as a prerelease. Release status can subsequently change; the linked API provides current metadata.

## Local runtime observations

These observations were captured during the initial build-and-install session, separately from the later artifact inspection. They were not silently reclassified as a comprehensive test suite.

| Observation | Evidence captured | Boundary |
| --- | --- | --- |
| Test environment | `uname -m`: `x86_64`; `sw_vers -productVersion`: `15.7.9` | One machine |
| Isolated Electron startup | Test harness logged a loaded `https://reasonmachines.com/auth/desktop` page with `arch: x64`, `electron: 44.9.0`, then rendered Reason interface text | Harness used an isolated profile and did not install the CLI |
| Normal app startup | App was copied from the finished DMG to Applications and opened through macOS; process inspection showed the packaged Reason executable and renderer | This was the app, not the Electron demo |
| Ready state | `desktop.app.ready` recorded at approximately `2026-10-22T17:41:37Z` | Historical observation; breadcrumbs rotate |
| CLI setup | CLI installation receipt appeared and an Intel CLI background process was observed during initial installation | A receipt is not proof that every CLI operation works |
| User confirmation | User explicitly stated that the installed app works | User report, not an automated test result |

The later inspection still found the CLI installation receipt and retained isolated-test output. The live breadcrumb file had rotated and no longer contained the original ready event. The JSON exposes this distinction rather than asserting that the ready event remains present.

Only these limited observations are published. Account details, raw application logs, authentication data, and the user's résumé are not part of this evidence bundle.

## Not tested

- Complete agent tasks and end-to-end workflows.
- Every application feature, integration, and permission path.
- Other Intel machines, other macOS versions, or Apple Silicon execution of this rebuild.
- Native hang recovery: it is disabled, so no replacement functionality is claimed.
- Desktop update or rollback behavior: automatic desktop updating is disabled.
- Compatibility with future CLI updates or future remote-service changes.
- Comprehensive security properties or publisher release acceptance.

## Read-only checks a reviewer can repeat

For an app installed as `/Applications/Reason.app`:

```sh
# Identify the machine and OS.
uname -m
sw_vers -productVersion

# Inspect the main executable's architecture.
lipo -archs /Applications/Reason.app/Contents/MacOS/Reason

# Read app version and minimum OS metadata.
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' \
  /Applications/Reason.app/Contents/Info.plist
/usr/libexec/PlistBuddy -c 'Print :LSMinimumSystemVersion' \
  /Applications/Reason.app/Contents/Info.plist

# Verify the local signature, then inspect its identity.
codesign --verify --deep --strict /Applications/Reason.app
codesign -dv /Applications/Reason.app
```

From a directory containing the downloaded DMG and checksum file:

```sh
shasum -a 256 -c Reason-0.1.109-Intel.sha256
hdiutil verify Reason-0.1.109-Intel.dmg
```

These commands inspect bytes and metadata; they do not assert full application correctness.
