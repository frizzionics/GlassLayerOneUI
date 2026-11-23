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

## Next action (cheap — not a build cycle)
Get the actual `SEM_PLATFORM_INT` value from the device. Try, in order:
```
getprop | grep -i "sem_platform\|platform_int\|version.sem"
```
If nothing matches, a short Termux/root script reading
`android.os.Build.VERSION.SEM_PLATFORM_INT` directly (via `app_process`/a tiny helper)
would be the fallback — ask before building this if getprop comes up empty.

If confirmed `<= 170000`: next step is finding what method *does* set the tile
background when `QUICK_PANEL_BLUR_REFACTORING` is false — likely the `else`-equivalent
of whatever called `updateColoredBackground`, or a different, older method entirely.
Check this statically (search for other code paths branching on the same flag, or
search `LargeTileView`/`ColoredBGHelper` for a second background-setting method) before
writing another on-device test.

If `SEM_PLATFORM_INT` actually is `> 170000` (hypothesis wrong): back to genuinely not
knowing why 3 different, well-evidenced hook targets have all been silent — would need
to reconsider from scratch, likely worth getting a fresh hook-diagnostics report
(`docs/device/`) to compare against the first one, since the device's state has changed
a lot since that export.

## Commits in this investigation, in order
`d134381` → `9e86c6a` → `8931ad2` (all in `app/src/main/java/com/drdisagree/glasslayer/xposed/systemui/QsTileGlass.kt`,
all marked `TEMP DEBUG`, none are fixes, all should be reverted once the real
implementation is written).
