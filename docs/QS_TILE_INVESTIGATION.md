# QS tile glass investigation — full log

This supersedes FINDINGS.md's original QsTileGlass row, which was written before any
of this. Read this file in full before touching QsTileGlass.kt again — several
plausible-looking approaches below have already been tried and ruled out with real
evidence; repeating them wastes a build+install+reboot cycle each time.

## Status as of 2026-11-23: root cause very likely identified, not yet confirmed
**Leading hypothesis:** `Build.VERSION.SEM_PLATFORM_INT` on this device is `<= 170000`,
which at runtime disables both `QpRune.QUICK_PANEL_COMPOSE` and
`QpRune.QUICK_PANEL_BLUR_REFACTORING` (set together, same condition, see below).  This
would mean neither the Compose tile system nor `updateColoredBackground()`'s real body
ever executes — explaining every negative result in this log. **Not yet confirmed**:
nobody has read the actual `SEM_PLATFORM_INT` value off the device. That's the single
next action, and it costs a `getprop` check, not a build cycle.

## Timeline, in order, with what each step actually proved

### 1. Static signature check — TileExpandable param types (confirmed, not the bug)
Real signature: `TileExpandable(Function0, Shape, Function0, TileHapticsViewModel,
Modifier, Function3, Composer, Int)`. Has both a `Modifier` (index 4) and `Shape`
(index 1) param — exactly what `applyTileGlass()`'s `indexOfFirst` lookup needs. Ruled
out: a parameter-shape mismatch is not why this fails.

### 2. Static signature check — drawBehind (confirmed, not the bug)
Real signature: `drawBehind(Modifier, Function1): Modifier` — exactly 2 params, first
one `Modifier` — exactly what `ComposeToolkit.modifierFunction()`'s resolver requires.
Ruled out: this reflective lookup was not the bug either.

### 3. On-device test — magenta fill inside `drawTileGlass` (commit `d134381`)
Unconditional canvas fill, before any gating logic, plus 3 log checkpoints. Built,
installed, **confirmed fresh reboot**. Result: no magenta, no logs at any of the 3
checkpoints (`TileExpandable` hook, `rememberExpandableController` hook,
`getColorForState` hook — 3 independent call sites across 2 separate classes).
Conclusion at the time: the Compose `TileKt`/`TileDefaults` hooks are not firing at all
on this device, despite installing successfully (hook report showed nonzero counts for
all of them) and despite the classes/methods being physically present in the APK.

### 4. Static survey — what else exists under `qs/` (found the real lead)
Surveyed every class under `com.android.systemui.qs.` outside the known Compose path.
Found `com.android.systemui.qs.tileimpl.{LargeTileView, NoLabelTileView, LabelTileView}`
— AOSP's older, pre-Compose View-based tile renderer, compiled in alongside the newer
Compose one (`QSFragmentCompose` also present in the APK). Two systems coexisting,
picked at runtime, is a known Android SystemUI migration pattern.

### 5. Static call-graph check (should have been step 1, not step 5 — see note below)
Direct bytecode scan of every `invoke`/`new-instance` across all 5 dex files:
- `TileExpandable`: exactly 2 references, both from `TileKt.Tile` (same file). Not
  orphaned dead code — `TileKt.Tile` itself has 3 callers (`QuickQuickSettingsKt`,
  `InfiniteGridLayout`) — but also not proof it's the *active* path on this device.
- `QSPanelHost.createTileView` **directly constructs** `LargeTileView`,
  `NoLabelTileView`, `LabelTileView` by name. This is the real, concrete factory method.
  Strongest single piece of evidence in this whole investigation for which system is
  actually live.

**Process note:** this kind of check — "does anything in the compiled APK call this at
all" — is answerable from the APK alone, in seconds, with zero device involvement. It
should be the *first* thing checked for any "is this hook even reachable" question, not
something reached for after 3 build-install-reboot cycles. Lead with this every time.

### 6. On-device test — onDraw hook on LargeTileView/NoLabelTileView/LabelTileView (commit `9e86c6a`)
Mirrored `NotificationGlass`'s proven technique: `hookAllBefore("onDraw")` +
unconditional magenta canvas fill. Built, installed, confirmed fresh reboot. Result:
no magenta, no logs. **Before concluding this disproved the legacy-View hypothesis**,
checked statically (see #7) — turned out the hook itself was malformed, not the
hypothesis.

### 7. Static hierarchy check — why #6 was silent (found a hook bug, not a dead hypothesis)
`LargeTileView` / `NoLabelTileView` / `LabelTileView` all extend
`com.android.systemui.plugins.qs.QSTileView`, which extends `android.widget.LinearLayout`.
**None of them declare `onDraw`** — zero draw-related methods of their own. `onDraw`
hooking was never going to find anything to attach to; the previous round's silence was
a hook-target bug (asked for a method that doesn't exist on these classes), not evidence
against the legacy-View hypothesis. The hypothesis (createTileView evidence) still stands.

### 8. Static method listing — found a real candidate (`updateColoredBackground`)
Declared on the shared `QSTileView` base *and* overridden directly on `LargeTileView`.
Plausible candidate for where the tile's actual background gets set.

### 9. On-device test — hook `updateColoredBackground`, call `setBackgroundColor(MAGENTA)` (commit `8931ad2`)
Plain `View.setBackgroundColor()`, no Compose reflection. Built, installed, reboot.
**Result: still nothing.**

### 10. Static disassembly of `updateColoredBackground` itself (found the real answer, pending confirmation)
Full bytecode of `LargeTileView.updateColoredBackground()`:
```
sget-boolean  QpRune.QUICK_PANEL_BLUR_REFACTORING
if-eqz        -> jump straight to return-void
[... only runs if the flag is true: getBackgroundDrawable() + setBackground() ...]
return-void
```
The method's entire meaningful body is gated behind `QpRune.QUICK_PANEL_BLUR_REFACTORING`.
If that flag is false, the method is a no-op — it still gets *called* (so our
`hookAllAfter` still fires and still should have painted magenta afterward,
independently of what the original body did) — which means step 9's silence needs a
different explanation than "the method never runs": most likely, the *actual* visible
tile background on this flag-state is set by some other method entirely (the "legacy"
pre-refactoring code path), not by `updateColoredBackground` at all, which is why
intercepting it — even successfully — doesn't touch the thing users actually see.

Checked how `QpRune.QUICK_PANEL_COMPOSE` and `QpRune.QUICK_PANEL_BLUR_REFACTORING` get
set (`QpRune.<clinit>`, same source value for both, set together):
```
v0 = Build.VERSION.SEM_PLATFORM_INT
v0 = (v0 > 170000) ? true : false
QUICK_PANEL_COMPOSE = v0
QUICK_PANEL_COMPOSE_DEBUG_LOG = v0
QUICK_PANEL_BLUR_REFACTORING = v0
```
**This directly ties the Compose-tile-system flag and the blur-refactoring flag to the
exact same condition.** If `SEM_PLATFORM_INT <= 170000` on this device, both the Compose
path (step 3's dead end) and the refactored-background path (step 9's dead end) are
explained by the *same single runtime value*, not two unrelated bugs.

`SEM_PLATFORM_INT` is a Samsung-specific `Build.VERSION` field, read at runtime — not
something visible in static APK analysis. **This is the one open question left, and it
doesn't need a build cycle to answer** — just reading a value off the running device.

### 11. `getprop ro.build.version.sem` (wrong property, caught before acting on it)
Returned `3602` — same small magnitude as a documented *different* Samsung property
(`ro.build.version.sem`, a minor firmware counter) seen on a real device dump, not
`SEM_PLATFORM_INT`. Did not act on this value.

### 12. `getprop ro.build.version.sep` (right magnitude, still indirect evidence)
Returned `170500`. Formula derived from known real values (`ro.build.version.sep=130100`
on a documented One UI 4.1 device, pattern `90000 + major*10000 + minor*100`) predicts
exactly `170500` for One UI 8.5 — strong match, **but this is still inferring that `.sep`
backs `SEM_PLATFORM_INT`, not confirming it.** `170500 > 170000`, which would mean
`QUICK_PANEL_COMPOSE` and `QUICK_PANEL_BLUR_REFACTORING` are **true** — directly
contradicting every negative on-device result so far (step 3 and step 9). Hypothesis
from steps 1-10 does not survive this check as stated.

### 13. On-device test — `updateColoredBackground` + `setBackgroundColor(MAGENTA)` (commit `8931ad2`)
Result: **still nothing.** Same as every prior attempt on this surface.

### 14. Static manifest check — ruled out separate-process theory
Parsed the real `AndroidManifest.xml` from `systemui_oneui85.apk`. Every QS/panel/tile
related component (`QSOnboardingActivity`, `PanelSettingsLaunchActivity`,
`VolumePanelDialogReceiver`, `QSBackupRestoreManager$QSBnRReceiver`, etc.) declares no
`android:process` override — all run in the default `com.android.systemui` process,
same as everything GlassLayer hooks. The only separate processes in this APK are
unrelated (`:edgelighting`, `:tuner`, `:screenshot`, `:appclips.screenshot`,
`:coverlauncher`, `:multiwindowui`, `:fgservices`, `:sweetsweetdesserts`). **Ruled out:**
GlassLayer hooking the wrong process is not why these hooks are silent.

## Current state: leading theory from steps 1-10 is contradicted by step 12, not confirmed
Nothing so far definitively explains why 4 independently-evidenced hook targets across
2 completely different rendering systems (Compose `TileKt`/`TileDefaults`, legacy View
`LargeTileView.updateColoredBackground`) are all silent, despite:
- hooks confirmed installing (report.txt, nonzero counts)
- target classes confirmed present with matching signatures (static bytecode)
- target classes confirmed as the real factory output (`QSPanelHost.createTileView`
  directly constructs them)
- the process confirmed correct (manifest check, step 14)

The one thing not yet eliminated: **whether `ro.build.version.sep` is actually what
`Build.VERSION.SEM_PLATFORM_INT` reads from.** This was always an inference from a
different device's property dump, never confirmed for this exact field on this exact
device.

## Next action — commit `<pending>`, not yet tested
Added a direct reflection read, right at the top of `QsTileGlass.install()`, of:
- `android.os.Build.VERSION.SEM_PLATFORM_INT` (the actual field, not an inferred
  property name)
- `QpRune.QUICK_PANEL_COMPOSE` and `QpRune.QUICK_PANEL_BLUR_REFACTORING` (the actual
  resolved flag values in the running process, not inferred from `<clinit>` bytecode)

This removes all ambiguity in one log line. Build, install, reboot, open Quick Settings
once (not required for this specific log line, but needed for the other checkpoints
already in place), check Vector's log for the `ONEUI_DEBUG ... SEM_PLATFORM_INT=...`
line emitted at module-install time.

**If `SEM_PLATFORM_INT` is actually `<= 170000`** (meaning `.sep` was a bad proxy after
all): the original theory (steps 1-10) holds — next step is finding what method sets
the tile background when `QUICK_PANEL_BLUR_REFACTORING` is false, via static search for
another branch on that same flag.

**If `SEM_PLATFORM_INT` is confirmed `> 170000`** (matching step 12's inference): the
flags are genuinely on, and something else entirely is wrong — possibly something about
*how* GlassLayer's reflection-based hooking interacts with Vector specifically (recall:
this project uses Vector, not LSPosed — never separately verified whether Vector's
`hookAll`/`hookAllAfter` behave identically to LSPosed's for these call sites). Worth
checking the Vector/libxposed API docs for any documented difference before writing
more on-device tests.

### 15. Targeted cross-reference sweep (not a full decompile) — found a parallel class hierarchy, then traced it back to the method already hooked
Considered doing a full jadx decompile of the whole APK for easier searching. Decided
against it: not necessary (a scoped bytecode scan gets the same answer), heavy on
context/resources for a sandbox, and "rebuilding" an OEM SystemUI from decompiled
source is a different, much bigger, impractical undertaking not needed for this project
(the actual deliverable is a small patch to GlassLayer's own source, not a
reconstruction of Samsung's app).

Instead, scanned every method in the APK referencing `updateColoredBackground`,
`QUICK_PANEL_BLUR_REFACTORING`, or `QUICK_PANEL_COMPOSE` (same technique as step 5's
`TileExpandable` caller search). Found an entire parallel hierarchy not seen before:
`com.android.systemui.qs.bar.{LargeTileBar, BrightnessBar, BrightnessVolumeBar,
QuickControlBar, QSMediaPlayerBar, MultiSIMPreferredSlotBar, TileChunkLayoutBar}`,
each extending `qs.bar.BarItemImpl` — a completely separate class tree from
`qs.tileimpl.*`, with names ("Quick Control Bar", "Brightness Volume Bar") that sound
like Samsung's actual distinctive One UI panel layout. Also found `QSFragmentLegacy`
repeatedly checking `QUICK_PANEL_COMPOSE` — likely the fragment used specifically when
Compose is off, versus `QSFragmentCompose` seen earlier in the manifest survey.

Disassembled `LargeTileBar.updateColoredBackground()`: same `QUICK_PANEL_BLUR_REFACTORING`
gate, then (if true) `mTiles.forEach(lambda)` plus separately walks a "cloned bar view"'s
children. Disassembled the lambda (`LargeTileBar$$ExternalSyntheticLambda0.accept`):
```
tileView = tileRecord.tileView   // plugins.qs.QSTileView
tileView.updateColoredBackground()
```
**This converges on the exact method already hooked in step 9/13** —
`QSTileView.updateColoredBackground()`, which `LargeTileView` overrides. So the hook
target in step 9 was structurally correct *within this call chain*, assuming the chain
ever executes. This doesn't resolve the open question from step 12/14 — it raises the
stakes on it: if `QUICK_PANEL_BLUR_REFACTORING` really is true, this entire 4-level
chain should run and the step 13 test should have shown magenta. Since it didn't,
either the flag is genuinely false (the `.sep`-based inference was wrong) or something
upstream of `LargeTileBar.updateColoredBackground()` itself never gets called in
practice (would need to find its own caller next, same technique).

### 16. Confirmed via direct reflection (commit `53880fa`, tested): `SEM_PLATFORM_INT=170500`, `QUICK_PANEL_COMPOSE=true`, `QUICK_PANEL_BLUR_REFACTORING=true`
The `.sep`-based inference was correct. Both flags are genuinely on. This fully
resolves step 12's open question — but does not explain the silence, since both
code paths should now be active.

**Bigger finding from this same report:** `QsTileGlass: 7` hooks installed (up from 4,
confirming the 3 new `updateColoredBackground` hooks attached correctly) — but **not
one** of the 4 instrumented hook-firing checkpoints logged anything, across the entire
~4-minute SystemUI session. Only the unconditional install-time `SEM_PLATFORM_INT` log
appears. Zero firing evidence across `TileExpandable`, `rememberExpandableController`,
`getColorForState`, and all 3 `updateColoredBackground` hooks, despite flags confirming
this code should run. Asked the user to confirm Quick Settings was actually opened
during the test session — not yet confirmed either way.

### 17. Traced `LargeTileBar.updateColoredBackground`'s own caller — found it's a rare path, not the main render path
Only 2 callers, both `getClonedBarView()` on subclasses (`BottomLargeTileBar`,
`SmartViewLargeTileBar`) — a special-purpose "cloned bar view" construction, not normal
rendering. This explains step 9/13's silence by itself, no exotic hooking-mechanism
theory needed: the hook was aimed at code that doesn't run during ordinary use.

### 18. Traced the interface-level `QSTileView.updateColoredBackground` — found the real "else" branch
Only 3 call sites total in the whole APK: `LargeTileBar.addTile`, and two branches of a
single merged/deduplicated lambda (`LargeTileBar$$ExternalSyntheticLambda1`, R8-merged
with a `$r8$classId` switch dispatching between several behaviors). Disassembled the
relevant branch:
```
addView(tileView)
if (QUICK_PANEL_BLUR_REFACTORING)
    tileView.updateColoredBackground()
else
    mBGColorHelper.addBarBackground(tileView, false)
```
This **confirms** the flag genuinely branches between two real, different
implementations — not a dead/no-op guard. Since the flag is confirmed true (step 16),
`updateColoredBackground()` should be the one running.

This lambda gets constructed in 4 places: `LargeTileBar.inflateViews`,
`removeAllTiles` (×2 overloads), `setListening`. `inflateViews` is the most likely
candidate for the real, normal view-construction path.

### 19. Checked for overrides before hooking (learned from round 2's onDraw mistake)
`inflateViews` is declared on `LargeTileBar` itself, but **also independently overridden**
by `BottomLargeTileBar` and `SmartViewLargeTileBar` (not calling through a shared base -
each declares its own). Only `TopLargeTileBar` relies on the base implementation.
Hooking just `LargeTileBar` would have silently missed the other two, same mistake as
round 2's `onDraw` attempt. Hooked all 4 classes' own `inflateViews` declarations this
time (harmless no-op on `TopLargeTileBar`, which has none of its own — covered via the
base class hook instead, same inheritance-covers-it logic that's safe here).

## Next action — commit `<pending>`, not yet tested
Hooks `inflateViews` (entry + exit, with `System.currentTimeMillis()`) on all 4 bar
classes. This is now a well-traced, high-confidence target: real method (not a lambda),
confirmed not overridden-and-missed, confirmed to be the thing that constructs the
lambda containing the actual `updateColoredBackground()` call.

**If this fires:** confirms view construction runs as expected; the mystery narrows to
why `updateColoredBackground()` itself still doesn't log even when called from here -
worth hooking the lambda itself directly at that point, or checking if `QUICK_PANEL_BLUR_REFACTORING`'s
value could differ between class-load time (when our `SEM_PLATFORM_INT` log ran) and
whenever `inflateViews` actually executes (unlikely for a value tied to a static final-ish
flag, but not yet ruled out).

**If this does NOT fire either:** the view construction method itself never runs during
this session, which would suggest either (a) it ran once, before boot, in a way that
doesn't apply per-session (unlikely for a view-construction method, these usually run
per-instance not once-globally), or (b) something more fundamental about hook timing -
worth checking whether GlassLayer's hooks install early enough relative to SystemUI's
own early boot-time initialization, since this is exactly the kind of one-time
construction code that could plausibly run before a module has finished attaching all
its hooks, depending on exactly when in SystemUI's boot sequence Vector injects.

## Commits in this investigation, in order
`d134381` → `9e86c6a` → `8931ad2` (all in `app/src/main/java/com/drdisagree/glasslayer/xposed/systemui/QsTileGlass.kt`,
all marked `TEMP DEBUG`, none are fixes, all should be reverted once the real
implementation is written).
