# Task: reverse-engineer Samsung's real SystemUI against GlassLayer's hook targets

## Context
GlassLayerOneUI (fork of github.com/Mahmud0808/GlassLayer) is being ported to Samsung
One UI 8.5 (Android 16). GlassLayer is an LSPosed/Vector module that hooks 24 SystemUI
surfaces to render live glass refraction. It was written against PixelOS/AOSP. We're
adapting it for:

- Device: Samsung Galaxy A17 4G, SM-A175F
- Build: BP4A.251205.006.A175FXXS7CZI4, One UI 8.5, Android 16 (API 36), patch 2026-09-05
- Fingerprint: samsung/a17xx/a17:16/BP4A.251205.006/A175FXXS7CZI4_OXM7CZI4:user/release-keys

Priority surfaces: **Notifications** (confirmed working, no change needed — don't spend
time on it) and **Quick Settings** (background, tiles, footer, brightness slider —
currently renders as stock One UI despite GlassLayer's hooks successfully attaching;
root cause unknown).

## Your job
Produce ground-truth facts about what Samsung's real, compiled SystemUI.apk actually
contains and does, for each of GlassLayer's 24 hook targets. Nothing else. Do not
propose fixes, do not redesign anything, do not decide what to build. A separate pass
will do that, using your findings as input. Your only output is verified fact.

## Why the evidence-category rule exists (read this before starting)
Earlier in this project, a prediction was made from reading GlassLayer's hook target
class names alone: since the targets were Jetpack Compose classes from Google's newer
SystemUI rewrite, and Samsung was assumed not to have adopted that rewrite, those hooks
were predicted to fail outright on Samsung. That assumption was then stated in project
docs as a near-conclusion. It was wrong. Direct evidence — a live hook-diagnostics
report from the device, and a grep of the actual SystemUI.apk's dex files — showed
those exact classes physically present and successfully hooked. The architecture-based
guess and the real device data disagreed, and the guess had been allowed to read like
fact until someone checked.

Do not repeat this. Every claim you make must be tagged with where it came from, and a
hypothesis must never be phrased the way a confirmed fact is phrased.

## Required format — one block per hook target, these four tags only
```
### <ModuleName> — <what it targets, in plain words>

**[STATIC-BYTECODE]** What you directly observed by decompiling the real APK: exact
class path, exact method signature (parameter types in order, return type, any Kotlin
name-mangling suffix present), exact field names/types for anything accessed by name.
Cite how you obtained it (decompiler used, class/method path) so it can be re-checked.
If you didn't check this for a given target, write "not checked" — do not leave it
implying a conclusion by omission.

**[RUNTIME-HOOK-REPORT]** What `docs/device/report-20261120-160333.txt` shows for this
module: hook count, broken flag, any error lines. Quote the exact line(s). If the
report says nothing relevant, write "nothing in report" — do not infer success or
failure from silence.

**[VISUAL-OBSERVATION]** What a human has actually reported seeing on screen for this
surface, if anything, with when they reported it. If nothing has been observed, write
"not observed" — do not guess what it probably looks like.

**[HYPOTHESIS]** Anything you want to propose as an explanation for a gap between the
tags above. Must start with "Hypothesis:" and state what would prove it wrong. Never
state a hypothesis without that label, even if you're confident.
```

### Worked example (apply this exact pattern)
```
### QsTileGlass — TileExpandable (tile background glass)

**[STATIC-BYTECODE]** Decompiled classes2.dex via androguard, class
`com/android/systemui/qs/panels/ui/compose/infinitegrid/TileKt`, method
`TileExpandable(Function0, Shape, Function0, TileHapticsViewModel, Modifier, Function3,
Composer, Int)`. Has both a Shape param (index 1) and a Modifier param (index 4), which
is what GlassLayer's `applyTileGlass()` looks for via
`types.indexOfFirst { it.name == ComposeToolkit.MODIFIER_CLASS }`. One extra parameter
vs. the Pixel build GlassLayer targets: `TileHapticsViewModel` (haptics, MSDL).

**[RUNTIME-HOOK-REPORT]** report line 29: "QsTileGlass: 4" — 4 hooks installed, no
broken flag, Diagnosis section (line 2) says "No problems found."

**[VISUAL-OBSERVATION]** User reported 2026-11-20, after confirming the tiles/
background/footer/brightness toggles were on in-app: "it looks stock."

**[HYPOTHESIS]** Hypothesis: the index lookup succeeds (ruled out by STATIC-BYTECODE
above) so the failure, if any, is downstream — inside `ComposeModifiers.proxy(...)` or
`ComposeModifiers.drawBehind(...)` returning null, or inside `drawTileGlass` itself
throwing in a way not yet confirmed to reach Guard/HookReporter. This would be proven
wrong by finding a STATIC-BYTECODE or RUNTIME-HOOK-REPORT entry showing those calls
succeed and the draw actually executes.
```

## Inputs available to you
- `systemui_oneui85.apk` — the real SystemUI from the device above (90 MB, 5-way
  multidex). Decompile with jadx if available (gives Kotlin-readable signatures,
  CompositionLocals, etc.) rather than raw string/dex grepping — you likely have a more
  complete toolchain for this than the sandbox that produced the worked example above,
  which only had androguard (pure Python dex parsing) and no JVM/jadx.
- GlassLayerOneUI fork source (`app/src/main/java/com/drdisagree/glasslayer/`) — this is
  the thing whose hook targets you're checking. Read each module's actual hook-install
  code (not just the class-name constants) before deciding what "the target" even is —
  some modules hook multiple classes/methods per surface.
- `docs/FINDINGS.md` and `docs/PROJECT.md` — prior findings and project status. Treat
  anything in FINDINGS.md's original per-module table as **unverified prediction**, per
  the revision note at its top — re-derive it from the real APK, don't cite it as fact.
- `docs/device/report-20261120-160333.txt` — the only runtime evidence available so far.

## Checklist — all 24 modules
For every one of these, read the actual source file first (don't assume from the name
alone), then fill in the four-tag block above. Prioritize QS-related ones first per
project priority, but cover all 24 — partial coverage of the list is fine if you run
out of budget, as long as what's covered is rigorous, but say explicitly what's left.

**Priority (Quick Settings):** QsTileGlass, QsFooterGlass, BrightnessGlass, ShadeGlass,
QsOpenMotion, MediaGlass

**Also needed:** ViewGlassUnderlay, DisplayRootTracker, WindowDimTracker, DozeTracker,
NotificationGlass (sanity-check only — this one's confirmed working, don't deep-dive),
LockscreenShortcutGlass, BouncerGlass, VolumeGlass, VolumePanelGlass, PowerMenuGlass,
DialogGlass, ToastGlass, BackGestureGlass, StatusBarChipGlass, TransitionGlass,
SwitchGlass, SliderGlass, ButtonGlass

## Specific open question to resolve if you get to QsTileGlass
Does `Guard.run` (in `xposed/hook/Guard.kt`) actually wrap the call path from the
`TileExpandable` hook through to `drawTileGlass`'s actual Canvas/DrawScope drawing code,
or does an exception somewhere in that chain have a path to fail silently without
reaching `Guard.fail()`/`HookReporter`? Trace the actual call chain in source and state
which `[STATIC-BYTECODE]`-equivalent (in this case, static *source*-code) evidence
supports your answer — i.e. read the real call chain rather than assuming Guard wraps
everything.

## Output
One file: `docs/APK_REVERSE_ENGINEERING.md`, one block per module in the format above.
Don't fix anything. Don't skip the tags for a module just because you're confident —
confidence isn't evidence, cite it or mark it unchecked.
