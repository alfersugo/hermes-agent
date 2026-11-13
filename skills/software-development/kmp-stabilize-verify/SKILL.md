---
name: kmp-stabilize-verify
description: Stabilize multi-agent KMP worktrees and verify on Android.
version: 0.1.0
author: Hermes
metadata:
  hermes:
    tags: [KMP, Android, E2E, Gradle, Debug]
---

# KMP Stabilize-Verify

Turn a dirty multi-agent KMP worktree into a green build plus on-device proof. Covers triage of concurrent uncommitted edits, background Gradle gates, surgical micro-fixes, and adb E2E with dumps and screenshots. Does NOT replace feature development skills; it is the stabilize-and-prove loop after agents wrote code.

## When to Use

- Several subagents edited the same KMP repo and nothing compiles.
- `assembleDebug` fails with `Unresolved reference` or `overrides nothing` after agent edits.
- Need on-emulator proof (login, navigate, screenshot) for seller/admin/buyer flows.
- Backend RPC exists but the app shows HTTP 400 or empty lists.

## Prerequisites

- KMP repo remote on mac-alex (`~/AndroidStudioProjects/Kambalache_KMP`), SSH host `mac-alex`.
- `adb` at `~/Library/Android/sdk/platform-tools` on mac-alex (NOT in SSH PATH; always export it).
- Emulator online (`adb devices` shows `emulator-5554`); app id `zomitech.kambalache_kmp`.
- Backend reachable at the canonical ngrok URL; test accounts with known passwords.

## How to Run

Work through `terminal` (SSH to mac-alex), `read_file`/`write_file` for local fix scripts, `vision_analyze` for screenshots, `delegate_task` only for small scoped lots. Never edit local KMP clones; all code changes go via `scp` of a fix script plus remote `python3`.

## Quick Reference

- `git status --short`, `git stash list`, `git log --oneline -3` — triage dirt.
- `./gradlew :app:androidApp:assembleDebug --console=plain` in `terminal` background + notify.
- `:app:shared:testAndroidHostTest` — unit gate.
- `adb install -r <apk>`, `monkey -p <pkg> -c android.intent.category.LAUNCHER 1` — launch.
- `adb shell uiautomator dump /data/local/tmp/ui.xml` + `adb pull` + parse `text`/`bounds`.
- `adb shell input tap x y`, `adb shell input text`, `adb shell input swipe x1 y1 x2 y2 ms`.
- `adb shell screencap -p /data/local/tmp/x.png` + `scp` + `vision_analyze`.
- `git checkout -b feat/<x>` + `commit --no-verify` (hook runs full CI); never push unasked.

## Procedure

1. Triage: `git status --short` + untracked list + `git log -1`. Confirm HEAD was clean; all dirt belongs to this session.
2. Duplicates: same basename tracked + untracked (e.g. rewritten screens) usually means a duplicated file. Diff and delete the copy.
3. Wiring checklist for new screens: route in feature `*Routes.kt` (package `...navigation`), `NavDisplay` branch in `App.kt` + imports, Koin `factory` + `viewModel` params + imports, design-system tokens (see Pitfalls).
4. Build in background; fix errors in micro-rounds (one script per round, assert anchors before write).
5. Verify backend first when the app shows HTTP 400: check the RPC signature in PG (`pg_proc`) and the exact arg names before touching the app.
6. E2E per role: fresh `pm clear` before UI login (avoids duplicated text), tap by dump bounds (never estimate), dump after every tap, screenshot key screens, transcribe with `vision_analyze`.
7. New route unreachable? Add a navigation affordance (button + `testTag`) instead of declaring E2E blocked.
8. Commit to a feature branch when green; record test IDs for cleanup.

## Pitfalls

- Orphan code after a premature `}`: agents append methods outside the class/interface. Symptom: `overrides nothing`, `must have a body`. Fix by moving the closing brace, not by deleting the methods.
- Duplicate DTO blocks: same `data class` twice. Keep the first, delete the copy.
- `KambalacheColors` has `Gray500`/`Error`, NOT `textSecondary`/`error`.
- `KText` has no `style` param; use `fontSize`/`fontWeight`.
- Filter chips must send backend enum values (`ACTIVO`, not `APROBADO`; `EN_REVISION`, `RECHAZADO`): a 400 with `estado inválido` means app/backend drift.
- DTO/backend key drift (`web` vs `web_url`, `meses_plazo_maximo` vs `meses_plazos`, bool vs text): align with `@SerialName` and derive visibility flags in the mapper when the backend sends no flags.
- Session injection may be ignored; UI login via taps is the reliable fallback.
- `adb install -r` drops the app to the launcher; relaunch with `monkey` and re-navigate.
- Never paste tool output into source files; a corrupted file is restored from `git show HEAD:<path>` then re-patched cleanly.
- RTK may truncate remote output; `scp` the file locally and use `read_file` for ground truth.

## Verification

`assembleDebug` BUILD SUCCESSFUL plus one `vision_analyze` transcription per role showing the target screen with real data.
