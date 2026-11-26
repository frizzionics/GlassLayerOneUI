# GlassLayer → Samsung One UI 8.5 port — project hub

Read this file first in any new session. It links to everything else and says what's
actually known vs. assumed. `FINDINGS.md` has the technical evidence; this file has
status, constraints, and next actions.

## Goal
Port github.com/Mahmud0808/GlassLayer (an LSPosed/Vector module that renders live
liquid-glass refraction over SystemUI surfaces, built for PixelOS/AOSP) so it works on
Samsung One UI 8.5 (Android 16).

## Environment this was researched in (read before assuming anything persists)
- This is a throwaway container. Nothing in `/home/claude` survives between chat
  sessions. The only things that persist are: (a) files the user downloads from chat,
  (b) commits pushed to an actual git remote.
- **There is currently no git remote configured.** Until the user forks the upstream
  repo on GitHub and this fork is cloned from *that* URL, there is nowhere durable to
  commit to. Treat this as step 0.
- This container's outbound network allowlist does not include Google's Maven
  (`dl.google.com`/`maven.google.com`) or the Android SDK manager endpoints. **This
  environment cannot run `./gradlew build`** — AGP, androidx, and the SDK itself aren't
  reachable here. Source can be read/written/edited here; compiling, installing, and
  on-device testing has to happen on the user's machine or in an environment with that
  network access (e.g. Claude Code running locally, or the user's own Android Studio).
- Upstream source is cloned read-only at `upstream/` alongside this docs folder, for
  reference line numbers cited in FINDINGS.md. It is **not** the fork — don't commit to it.

## Device / confirmed facts
- Samsung Galaxy A17 4G, SM-A175F. Rooted via KernelSU (jailbreak mode). Xposed
  framework in use: **Vector 2.2** (3080, API 102) — not LSPosed. LSPosed is listed as
  supported upstream but Vector is what's actually hooking on this device; don't assume
  LSPosed-only APIs are available.
- One UI 8.5, Android 16 (API 36), build `BP4A.251205.006.A175FXXS7CZI4`, security
  patch 2026-09-05, Knox 3.13, SE for Android: Enforcing.
- Fork exists: `git@github.com:frizzionics/GlassLayerOneUI.git`, cloned locally by the
  user (their own machine, own terminal — not this sandbox). No commits yet. This
  sandbox cannot push to it; any code/doc changes produced here need to be copied into
  that local clone and committed by the user.
- Priority, per user: **Notifications** (already confirmed working) and **Quick
  Settings** (background, tiles, footer, brightness slider) — "that's what people open
  every day." Everything else in FINDINGS.md's module table is lower priority for now.
- **Big update:** the hook-diagnostics report and a direct dex-string search of the
  real SystemUI.apk both show all 24 hook modules — including every QS-related one —
  successfully finding their target classes and attaching hooks on this device. See the
  revision note at the top of FINDINGS.md. This looks far more promising than the
  original architecture-based prediction. The open question is no longer "does the
  class exist" — it's "does the glass actually render correctly," which only a live
  on-screen check can answer.

## Current action item (blocking next real step)
**See `docs/QS_TILE_INVESTIGATION.md`, specifically the "STEPPING BACK" section near the
end — read that before touching `QsTileGlass.kt` again.** Short version: 4 full rounds
of on-device testing across 4 completely different rendering systems (AOSP Compose,
AOSP legacy View, Samsung's `qs.bar`, Samsung's own from-scratch `samsung.quicksetting`
reimplementation - the best-evidenced target found) have all produced the identical
result: hook installs correctly, zero firing, zero visual change. This is very unlikely
to still be a "wrong class" problem on its fourth distinct hierarchy.

Leading open question now: `HookReporter` is per-process state, and GlassLayer loads
into at least 3 processes (main `com.android.systemui`, `:edgelighting`, one more under
a different UID). The report we keep exporting may not be describing the process that
actually renders the screen. See the investigation log for what this does and doesn't
explain, and the recommended next action (determine which process does real UI work,
rather than trying a sixth hook target blind).

Notifications remains confirmed working and untouched — don't re-investigate it.

## Process lesson from this investigation (apply going forward)
Several rounds of "add a debug hook, build via CI, wait, install, reboot, test" were
spent on questions actually answerable by direct static analysis of the APK's bytecode
(does anything call this method at all; is this method even declared on this class) in
seconds, with no device involvement. Lead with static analysis — call-graph/reference
scans, method/field listings, disassembly of specific methods — before writing another
on-device diagnostic. Reserve actual device round-trips for things that genuinely can't
be determined from the APK alone (a runtime property value, whether a shader renders
correctly, real user-visible behavior).

## What's still needed from the user
1. ✅ ~~Fork~~ — done (`frizzionics/GlassLayerOneUI`).
2. ✅ ~~Device/build info~~ — done, recorded above.
3. ✅ ~~Status report~~ — done, see `device/report/report.txt` and the FINDINGS.md revision.
4. ✅ ~~Real SystemUI.apk~~ — done, at `device/systemui_oneui85.apk` (90 MB, 5-way multidex).
5. **Outstanding:** the live visual check above, and — once there's a concrete bug to
   fix — this sandbox will need either (a) the user pastes back specific error output,
   or (b) the user applies patches this sandbox writes out, into their local clone, and
   commits/pushes, since there's no push access from here into `frizzionics/GlassLayerOneUI`.

## Recommended approach, updated for the real data
- Don't assume any module needs a from-scratch Samsung-specific rewrite until the
  visual check above actually shows it rendering wrong. The classes exist and the hooks
  attach — the remaining work may be small, targeted fixes (a wrong field read, a shape
  mismatch) rather than new hook implementations.
- If/when a surface does turn out visually broken: keep GlassLayer's rendering layer
  (`ComposeLiquidGlass`, `LiquidGlassRenderer`, `ViewGlassUnderlay`) untouched — it's
  generic View/Canvas drawing, not AOSP-specific. Look first at whether the *installed*
  hook is reading a field/value that differs under Samsung's theming (e.g. corner
  radius, state holder shape) before assuming a structural rewrite is needed.
- Keep any Samsung-specific changes isolated (separate file, or a clear `if` branch
  checking `Build.MANUFACTURER`/similar) rather than editing shared hook files in place,
  so the fork stays mergeable with upstream and doesn't silently break PixelOS behavior
  for that file if someone else uses this fork on a Pixel.
- GPL-3.0 license: forking and modifying is fine; keep the license file and note
  changes made, per GPL's requirements.

## Open questions (update as answered)
- [ ] Do QS background/tiles/footer/brightness slider actually render correctly once
      switched on, or is something visually off despite the hook attaching?
- [ ] If something's off: which specific visual symptom (invisible, wrong shape,
      flicker, crash) — needed before guessing at a fix.
- [ ] Whether the user wants to build/flash themselves, or wants code handed over for
      them to build (this sandbox still can't compile Android projects — see above).
