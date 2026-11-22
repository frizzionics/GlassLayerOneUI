# SystemUI APK reverse-engineering findings

APK examined: `/mnt/data/systemui_oneui85.apk`  
Repository: `frizzionics/GlassLayerOneUI`, branch `main`, HEAD `74c21040424a0e842f921e5f4ee853bf142045d9`  
Device/build identity is copied from the repository brief and runtime report: SM-A175F, Android 16/API 36, build `A175FXXS7CZI4`, One UI 8.5.  
APK contains five DEX files. SHA-256:
- `classes.dex`: `30d6d504bf6ab4a2b0417470dbd256fbdcb946ef5fcf6b551317464fb550a7ac`
- `classes2.dex`: `e7de25f6ef62edaa6985a7b4f44293b52297219f4ad7c6f6e9179a4af1bcc188`
- `classes3.dex`: `4a9db848763c00cdc82bce221a4533d981020ead1b6c10b3e00b21e6e437d913`
- `classes4.dex`: `cb919a4468b4a133b65769b0bdb589a29d97f2f6d8256399909caacaea760233`
- `classes5.dex`: `4e2963d4ae30a796fd62e5b285768476e0a963ac567b94226c35a581ffd172c1`

Method signatures below were read from DEX `method_ids`, `type_ids`, `proto_ids`, and class definitions using a local DEX parser. They are JVM/DEX descriptors rendered in readable order, not JADX-decompiled Kotlin. Parameter types and return types are exact. For this pass, non-priority module targets are not yet fully enumerated; they are explicitly marked not checked rather than inferred.

Evidence tags are kept separate. Runtime counts show installed hooks, not successful callback execution or visible drawing.

## Quick Settings priority

### ShadeGlass — shade background and scrim

**[STATIC-BYTECODE]** Verified in `classes.dex` and `classes3.dex`:
- `com.android.systemui.statusbar.phone.ScrimController.updateScrimColor(View, Float, Int): Unit` (`classes3.dex`).
- `com.android.systemui.shade.ui.composable.ShadePanelScrimKt.ShadePanelScrim(ContentScope, Boolean, Modifier, Composer, Int): Unit` (`classes3.dex`).
- `com.android.compose.modifiers.AnimatedBackgroundKt.animatedBackground(Modifier, Function0, Function0, Shape): Modifier` (`classes.dex`).
- `androidx.compose.foundation.BackgroundKt.background(Modifier, Brush, Shape, Float): Modifier` (`classes.dex`).
These are exact target classes/methods visible in the APK. The source also names a blur utility target, but its exact target class/method was not resolved in this pass.

**[RUNTIME-HOOK-REPORT]** `docs/device/report-20261120-160333.txt`: `ShadeGlass: 4`; log: `14:58:52.048 I ShadeGlass: 4 hooks`. Diagnosis: `No problems found`.

**[VISUAL-OBSERVATION]** User's 2026-11-20 report, as recorded in `docs/REVERSE_ENGINEERING_BRIEF.md`, says the QS background still looked stock. No separate visual capture is included in the inspected repository evidence.

**[HYPOTHESIS]** Hypothesis: the four installed hooks do not establish that the Compose background modifier is replaced or that the glass draw callback executes. This would be disproven by a runtime trace or visual capture proving the replacement modifier and draw callback execute.

### QsTileGlass — Quick Settings tile glass

**[STATIC-BYTECODE]** Verified in `classes3.dex`:
`com.android.systemui.qs.panels.ui.compose.infinitegrid.TileKt.TileExpandable(Function0, Shape, Function0, com.android.systemui.haptics.msdl.qs.TileHapticsViewModel, Modifier, Function3, Composer, Int): Unit`.
The APK method has eight parameters. `Shape` is parameter index 1 and `Modifier` is index 4; the intervening `TileHapticsViewModel` parameter is an additional Samsung-build parameter. GlassLayer's `applyTileGlass()` finds the Shape and Modifier indexes by runtime parameter type (`indexOfFirst`), not by hard-coded indexes, so this signature is compatible with that lookup.
Also verified:
- `com.android.systemui.qs.panels.ui.compose.infinitegrid.TileDefaults.getColorForState(TileUiState, Boolean, Composer, Int): TileColors` (`classes3.dex`).
- `com.android.compose.animation.ExpandableControllerKt` contains `rememberExpandableController-ARWlkTg(J, Shape, J, BorderStroke, ComposableControllerFactory, Composer, Int, Int): ExpandableControllerImpl` and `rememberExpandableController-T042LqI(Function0, Shape, J, BorderStroke, ComposableControllerFactory, Composer, Int, Int): ExpandableControllerImpl` (`classes.dex`).
Source files checked: `QsTileGlass.kt`, `ComposeModifiers.kt`, `ComposeToolkit.kt`, `Guard.kt`.
Guard path: `ComposeModifiers.proxy()` -> `ComposeToolkit.function1()` -> `ComposeToolkit.function()`. The proxy's `invoke` branch calls `Guard.run(name, null) { block(values) }`; `drawTileGlass()` separately wraps `ComposeLiquidGlass.draw()` in `Guard.run("QsTileGlass.tile_glass", Unit)`. Therefore exceptions inside the invoked callback and inside `ComposeLiquidGlass.draw()` are guarded. However, `ComposeModifiers.drawBehind()` catches `Throwable` from reflective `drawBehindMethod?.invoke(...)` and returns `null` without calling `Guard.fail()`. `applyTileGlass()` then returns on a null drawer or null modifier result. A failure during modifier lookup/injection can therefore be silent even while the hook-install count is nonzero.

**[RUNTIME-HOOK-REPORT]** `docs/device/report-20261120-160333.txt`: `QsTileGlass: 4`; log: `14:58:52.053 I QsTileGlass: 4 hooks`. Diagnosis: `No problems found`.

**[VISUAL-OBSERVATION]** User's 2026-11-20 report, as recorded in the brief, says the Quick Settings tiles/background still looked stock.

**[HYPOTHESIS]** Hypothesis: the unresolved point is downstream of method discovery: modifier construction/injection or callback execution may fail without appearing in the install count. This would be disproven by a runtime trace showing the modifier returned by `drawBehind()` is installed and `drawTileGlass()` is entered.

### QsFooterGlass — Quick Settings footer actions

**[STATIC-BYTECODE]** Verified in `classes3.dex`, class `com.android.systemui.qs.footer.ui.compose.FooterActionsKt`:
- `IconButton(Int, Composer, Modifier, Function0): Unit`.
- `IconButton(FooterActionsButtonViewModel, Modifier, Composer, Int): Unit`.
- `AnimatedFooterTextButton(FooterTextButtonViewModel, Modifier, Composer, Int): Unit`.
- `NumberButton(Int, String, Boolean, Function1, Modifier, Composer, Int): Unit`.
- `CircleExpandable-vMW4a_I(Long, Modifier, Long, BorderStroke, Function1, MutableInteractionSource, Function3, Composer, Int, Int): Unit`.
These match the names/regexes hooked in `QsFooterGlass.kt`. `ExpandableControllerKt` is also present in `classes.dex` with the two `rememberExpandableController...` methods listed under QsTileGlass.

**[RUNTIME-HOOK-REPORT]** `docs/device/report-20261120-160333.txt`: `QsFooterGlass: 7`; log: `14:58:52.056 I QsFooterGlass: 7 hooks`. Diagnosis: `No problems found`.

**[VISUAL-OBSERVATION]** User's 2026-11-20 report, as recorded in the brief, says the Quick Settings footer still looked stock.

**[HYPOTHESIS]** Hypothesis: hook presence on the footer composables does not prove the footer modifier/drawing callback is installed or reached. This would be disproven by runtime evidence of the injected modifier and its draw callback executing.

### BrightnessGlass — Quick Settings brightness slider

**[STATIC-BYTECODE]** Verified in `classes2.dex`:
`com.android.systemui.brightness.ui.compose.BrightnessSliderKt.BrightnessSliderContainer(BrightnessSliderViewModel, Modifier, ContainerColors, Composer, Int): Unit`.
Also verified in `classes.dex`, `androidx.compose.material3.SliderDefaults`:
- `Track-4EFweAY(SliderState, Modifier, Boolean, SliderColors, Function2, Function3, Float, Float, Composer, Int, Int): Unit`.
- `Track-mnvyFg4(SliderState, Float, Modifier, Boolean, SliderColors, Function2, Function3, Float, Float, Composer, Int, Int): Unit`.
- `TrackImpl-VvwgllI(SliderState, Float, Modifier, Boolean, SliderColors, Function2, Function3, Float, Float, Boolean, Composer, Int, Int): Unit`.
- `drawTrackPath-zXTsYAs(DrawScope, Orientation, Long, Long, Long, Float, Float): Unit`.
The `BrightnessSliderContainer` signature has a Modifier parameter at index 1. The track methods and `drawTrackPath` target family are present.

**[RUNTIME-HOOK-REPORT]** `docs/device/report-20261120-160333.txt`: `BrightnessGlass: 5`; log: `14:58:52.061 I BrightnessGlass: 5 hooks`. Diagnosis: `No problems found`.

**[VISUAL-OBSERVATION]** User's 2026-11-20 report, as recorded in the brief, says the brightness slider still looked stock.

**[HYPOTHESIS]** Hypothesis: the hooks match compiled Compose methods but the replacement track/draw path is not being reached or does not produce a visible difference. This would be disproven by a runtime trace or capture showing the modified track draw executing.

### QsOpenMotion — Quick Settings expansion and opening motion

**[STATIC-BYTECODE]** Partially verified:
- `com.android.systemui.shade.ShadeExpansionStateManager.addExpansionListener(ShadeExpansionListener): ShadeExpansionChangeEvent` exists in `classes3.dex`.
- `com.android.systemui.statusbar.notification.stack.NotificationStackScrollLayout.onLayout(Boolean, Int, Int, Int, Int): Unit` exists in `classes3.dex`.
- `com.android.systemui.qs.panels.ui.compose.infinitegrid.TileKt.TileExpandable(...)` exists in `classes3.dex` with the exact signature recorded under QsTileGlass.
- `BrightnessSliderKt.BrightnessSliderContainer(...)`, `MediaKt.Media(...)`, and `FooterActionsKt.FooterActions(...)` are present in the DEX set.
The remaining touch, status-bar-state, and transition-layout targets in `QsOpenMotion.kt` have not all been resolved to exact method signatures in this pass.

**[RUNTIME-HOOK-REPORT]** `docs/device/report-20261120-160333.txt`: `QsOpenMotion: 13`; log: `14:58:52.082 I QsOpenMotion: 13 hooks`. Diagnosis: `No problems found`.

**[VISUAL-OBSERVATION]** No separate motion-specific visual observation or capture was present in the inspected evidence.

**[HYPOTHESIS]** Hypothesis: 13 installed hooks may cover only a subset of the state/touch/transition path exercised by a particular QS opening gesture. This would be disproven by event-level runtime tracing that shows each relevant listener and transition callback firing for the tested gesture.

### MediaGlass — Quick Settings media player

**[STATIC-BYTECODE]** Verified in `classes2.dex`, `com.android.systemui.media.remedia.ui.compose.MediaKt`:
- `CardBackground(Icon, MediaKt$rememberAnimatedColorScheme$1$1, Modifier, Composer, Int): Unit`.
- `Media(MediaViewModel$Factory, MediaPresentationStyle, MediaUiBehavior, Function0, Modifier, Composer, Int, Int): Unit`.
- `Card(MediaCardViewModel, MediaPresentationStyle, Modifier, Composer, Int): Unit`.
The source `MediaGlass.kt` hooks `MediaKt.CardBackground`, `MediaControlPanel.bindPlayer`, and `TransitionLayout.dispatchDraw`. The Compose class and the two named composables above are present. The exact class/method resolution for `MediaControlPanel.bindPlayer` and `TransitionLayout.dispatchDraw` was not completed in this pass.

**[RUNTIME-HOOK-REPORT]** `docs/device/report-20261120-160333.txt`: `MediaGlass: 3`; log: `14:58:52.065 I MediaGlass: 3 hooks`. Diagnosis: `No problems found`.

**[VISUAL-OBSERVATION]** No separate media-player visual observation or capture was present in the inspected evidence.

**[HYPOTHESIS]** Hypothesis: the Compose `CardBackground` hook may be installed without proving that its modifier/draw callback reaches the liquid-glass renderer. This would be disproven by a runtime trace of `MediaGlass.draw` and its renderer call.

## Remaining modules — runtime evidence captured, bytecode enumeration not completed

For each entry below, the installed-hook count is copied from `docs/device/report-20261120-160333.txt`. The module's exact source target methods have not yet been exhaustively matched against DEX method definitions. "Not checked" is intentional; it is not a claim that a target is absent.

### ViewGlassUnderlay

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `ViewGlassUnderlay: 2`; log: `14:58:52.032 I ViewGlassUnderlay: 2 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### DisplayRootTracker

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `DisplayRootTracker: 1`; log: `14:58:52.036 I DisplayRootTracker: 1 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### WindowDimTracker

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `WindowDimTracker: 2`; log: `14:58:52.041 I WindowDimTracker: 2 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### DozeTracker

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `DozeTracker: 1`; log: `14:58:52.043 I DozeTracker: 1 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### NotificationGlass

**[STATIC-BYTECODE]** Not checked in this pass; the brief says this surface is already confirmed working and requested only a sanity check.
**[RUNTIME-HOOK-REPORT]** `NotificationGlass: 4`; log: `14:58:52.086 I NotificationGlass: 4 hooks`.
**[VISUAL-OBSERVATION]** `docs/REVERSE_ENGINEERING_BRIEF.md` records Notifications as confirmed working; no dated screenshot or more detailed visual description was included in the inspected materials.
**[HYPOTHESIS]** None recorded.

### LockscreenShortcutGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `LockscreenShortcutGlass: 6`; log: `14:58:52.090 I LockscreenShortcutGlass: 6 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### BouncerGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `BouncerGlass: 9`; log: `14:58:52.098 I BouncerGlass: 9 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### VolumeGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `VolumeGlass: 7`; log: `14:58:52.102 I VolumeGlass: 7 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### VolumePanelGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `VolumePanelGlass: 11`; log: `14:58:52.107 I VolumePanelGlass: 11 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### PowerMenuGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `PowerMenuGlass: 4`; log: `14:58:52.113 I PowerMenuGlass: 4 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### DialogGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `DialogGlass: 2`; log: `14:58:52.115 I DialogGlass: 2 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### ToastGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `ToastGlass: 2`; log: `14:58:52.117 I ToastGlass: 2 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### BackGestureGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `BackGestureGlass: 1`; log: `14:58:52.118 I BackGestureGlass: 1 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### StatusBarChipGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `StatusBarChipGlass: 3`; log: `14:58:52.119 I StatusBarChipGlass: 3 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### TransitionGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `TransitionGlass: 3`; log: `14:58:52.122 I TransitionGlass: 3 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### SwitchGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `SwitchGlass: 5`; log: `14:58:52.128 I SwitchGlass: 5 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### SliderGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `SliderGlass: 6`; log: `14:58:52.131 I SliderGlass: 6 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

### ButtonGlass

**[STATIC-BYTECODE]** Not checked for this module.
**[RUNTIME-HOOK-REPORT]** `ButtonGlass: 5`; log: `14:58:52.135 I ButtonGlass: 5 hooks`.
**[VISUAL-OBSERVATION]** Not observed in the inspected evidence.
**[HYPOTHESIS]** None recorded.

## Evidence limits and immediate conclusions

1. The five DEX files contain the exact `TileKt.TileExpandable`, `BrightnessSliderKt.BrightnessSliderContainer`, `MediaKt.CardBackground`, `FooterActionsKt` composables, and `ShadePanelScrimKt.ShadePanelScrim` classes/methods enumerated above. The Compose-based QS targets are not absent from this Samsung APK.
2. `TileExpandable` includes an additional `TileHapticsViewModel` parameter. GlassLayer's source locates the Shape and Modifier parameters by type, so this extra parameter alone does not explain a failed tile appearance.
3. Runtime hook counts are positive for all 24 modules, but they measure installed hooks. They do not prove the target callback ran, that a Compose modifier was successfully created/attached, or that pixels were drawn.
4. The `QsTileGlass` source has a concrete unreported failure path: `ComposeModifiers.drawBehind()` swallows reflective invocation exceptions and returns `null`. That path can prevent the glass drawer from being attached without recording a `Guard` failure.
5. The 18 non-priority modules are deliberately left `not checked` for static bytecode. This document is a first evidence-grounded pass, not a claim that the requested exhaustive 24-module decompilation is complete.
