# Technical findings: why some surfaces will port easily and others won't

> **Revision (2026-11-20, after real device data came in):** The prediction below —
> that Samsung never adopted AOSP's Compose-based SystemUI rewrite, so Group 3 modules
> would fail outright — turned out to be **wrong for this specific build**. Two pieces
> of hard evidence overturned it:
> 1. The exported Status report (`device/report/report.txt`) shows **every one of the
>    24 hook modules installed a nonzero number of hooks** on this exact device, including
>    QsTileGlass (4), QsFooterGlass (7), BrightnessGlass (5), VolumeGlass (7),
>    VolumePanelGlass (11), BouncerGlass (9) — the modules predicted to fail outright.
>    App diagnosis: "No problems found."
> 2. Independently, grepping the real `systemui_oneui85.apk`'s dex files for the exact
>    class name strings confirmed they're physically present in Samsung's build:
>    `TileDefaults` (39 hits), `TileKt` (24), `FooterActionsKt` (44), `PinBouncerKt` (35),
>    `BrightnessSliderKt` (34), `VolumeDialogViewBinder` (16), `VolumeNavigator` (15). The
>    one checked class that's genuinely absent: `OngoingActivityChipKt` (0 hits) — the
>    status-bar "live chip" feature doesn't exist in this build, so StatusBarChipGlass's
>    3 installed hooks must be landing on its other, non-Compose targets instead.
>
> **Working hypothesis, not confirmed from source:** One UI 8.5's "fresh new look"
> redesign likely adopted a large share of Google's Compose-based SystemUI internals
> (tiles, shade scrim, volume dialog, bouncer, brightness slider) rather than
> maintaining Samsung's older, fully separate View-based implementations, reskinning
> them via Material3 theme tokens instead of rewriting the UI layer. This is plausible
> given the timing (Android 16 cycle, same "immersive visuals / personalization"
> language Samsung used announcing 8.5) but hasn't been verified by reading Samsung's
> own source — nobody has that. Treat it as the best explanation for the evidence, open
> to revision on any OEM/device/One UI version with different results.
>
> **What this changes:** "Hooks installed" means the class and method were found and
> successfully attached at runtime — it does NOT guarantee the visual result looks
> correct (private field names a hook reads via reflection, e.g. corner-radius or
> background-state fields, could still hold different values under Samsung's theming
> even on a shared class). The remaining open question for the prioritized surfaces
> (Notifications, Quick Settings) is no longer "will the hook attach" — it's "does the
> glass actually render correctly on screen" — see PROJECT.md's current action item.
>
> The table below is kept as originally written (predictions made before this data,
> with the reasoning that produced them) because the per-module *reasoning* about
> Compose vs. legacy classes is still accurate — it's the real-world conclusion drawn
> from it (that Samsung wouldn't have those classes at all) that this device disproved.
> Read the table as "what we'd have guessed from class names alone," not as current status.

## Confirmed device
- Samsung Galaxy A17 5G, model SM-A175F
- One UI 8.5, Android 16 (API 36), build BP4A.251205.006.A175FXXS7CZI4, security patch 2026-09-05
- Knox 3.13, SE for Android: Enforcing
- Xposed framework in use: Vector 2.2 (3080), API 102
- Fork: git@github.com:frizzionics/GlassLayerOneUI.git (cloned locally by user, not yet modified)

Source: github.com/Mahmud0808/GlassLayer, cloned at commit matching the `main` branch
at research time. All file paths below are relative to `upstream/app/src/main/java/com/drdisagree/glasslayer/`.
Line numbers may drift as upstream updates — re-grep if a cited line looks wrong.

## The core finding
GlassLayer's 24 hook modules are registered in one place:
`xposed/systemui/SystemUiModule.kt` (the `installers` list). Each module's `install()`
hooks specific classes/methods via reflection (see `xposed/hook/HookEnv.kt` and
extension helpers like `.field()`, `.call()`). Reading each module's target class
constants shows a clean split that has nothing to do with naming conventions:

- **Group 1 — Android framework classes** (`android.*`, `com.android.internal.*`).
  OEM-independent by definition; same on every Android device regardless of skin.
- **Group 2 — Older, non-Compose SystemUI-core classes** (`com.android.systemui.statusbar.*`,
  `.toast.*`, `.globalactions.*`, etc., without "compose" in the package path). These
  predate Google's Compose rewrite of SystemUI and are more likely to be classes Samsung
  extended/reused rather than replaced outright — but Samsung has still rewritten some
  of these (see per-module notes).
- **Group 3 — Google's Compose-based SystemUI rewrite** (package paths containing
  `.ui.compose.`, `.ui.composable.`, or classes like `FooterActionsKt`, `TileDefaults`,
  `ShadePanelScrimKt`, `PinBouncerKt`). This is Google's in-progress migration of QS,
  shade, volume panel, bouncer, media controls and status chips to Jetpack Compose.
  **Samsung has not adopted this rewrite.** Samsung's visual equivalents for these
  surfaces are separate, long-standing proprietary View-based implementations. These
  classes will not exist in Samsung's SystemUI.apk at all — this isn't a renaming
  problem, it's a different underlying implementation, so porting means writing a new
  Samsung-specific hook from scratch (reusing GlassLayer's rendering layer, not its
  hook-location code).
- **Group 4 — `androidx.compose.material3.*` generic widgets** (Switch/Slider/Button).
  These are AndroidX library classes, not AOSP/OEM-specific — but only useful if
  Samsung's SystemUI actually instantiates these particular Compose widgets somewhere.
  OneUI's own toggle/slider/button styling in QS and dialogs is historically View-based,
  so this is unconfirmed either way — needs an on-device check, not an assumption.

## Full module table

| Module | Targets (abridged) | Group | Expectation on Samsung | Confidence |
|---|---|---|---|---|
| ViewGlassUnderlay | generic View/Canvas rendering helper, no hardcoded AOSP class | infra | works (not ROM-specific) | high |
| DisplayRootTracker | no hardcoded AOSP class found | infra | works (not ROM-specific) | medium |
| WindowDimTracker | `android.view.ViewRootImpl` | 1 | works | high |
| DozeTracker | no hardcoded AOSP class found | infra | works (not ROM-specific) | medium |
| ShadeGlass | `statusbar.phone.ScrimController`, `.statusbar.BlurUtils` (Group 2) **+** `shade.ui.composable.ShadePanelScrimKt`, `notifications.ui.composable.NotificationsKt` (Group 3) | mixed | partial at best — scrim/blur utils may resolve, the actual shade-panel Compose hooks won't | medium |
| QsTileGlass | `qs.panels.ui.compose.infinitegrid.TileDefaults`, `TileKt` | 3 | will not resolve — needs full rewrite against Samsung's own QS tile view classes | high |
| QsFooterGlass | `qs.footer.ui.compose.FooterActionsKt`, `FooterActionsDefaults`, `FooterActionsButtonViewModel` | 3 | will not resolve | high |
| BrightnessGlass | `brightness.ui.compose.BrightnessSliderKt` (Group 3) + `androidx.compose.material3.SliderDefaults/SliderColors` (Group 4) | 3/4 | will not resolve — OneUI's brightness slider is a custom View, not this Compose slider | high |
| MediaGlass | `media.remedia.ui.compose.MediaKt` (Group 3) + `media.controls.ui.controller.MediaControlPanel`, `util.animation.TransitionLayout` (Group 2, legacy) | mixed | Compose part won't resolve; the legacy controller/transition classes might, needs device check | low-medium |
| QsOpenMotion | `shade.ShadeExpansionStateManager/Listener`, `statusbar.StatusBarStateControllerImpl`, `statusbar.notification.stack.NotificationStackScrollLayout` — all Group 2, no Compose refs found | 2 | better odds than most QS-adjacent modules — these are state-management classes, not UI-rendering classes, more likely retained | medium |
| NotificationGlass | `android.view.NotificationHeaderView`, `com.android.internal.widget.NotificationExpandButton` (Group 1) + `statusbar.notification.row.NotificationBackgroundView`, `FooterView` (Group 2) | 1/2 | **confirmed working on this device** | confirmed |
| LockscreenShortcutGlass | `keyguard.ui.binder.KeyguardQuickAffordanceViewBinder`, `keyguard.ui.composable.elements.ShortcutElementProvider`, `com.android.compose.animation.*` | 3 | will not resolve — OneUI lockscreen shortcuts are a separate custom implementation | high |
| BouncerGlass | `bouncer.ui.composable.PinBouncerKt`, `BouncerContentKt` | 3 | will not resolve — OneUI's PIN pad is fully custom | high |
| VolumeGlass | `volume.dialog.ui.binder.VolumeDialogViewBinder`, `...RingerViewBinder`, `...SettingsButtonViewBinder`, `volume.dialog.sliders.ui.compose.VolumeDialogSliderTrackKt` | 3 | will not resolve — OneUI's volume dialog is visibly, famously different from stock Android's | high |
| VolumePanelGlass | `volume.panel.component`, `volume.ui.navigation.VolumeNavigator`, same slider Compose class as above | 3 | will not resolve | high |
| PowerMenuGlass | `globalactions.GlobalActionsLayoutLite`, `GlobalActionsDialogLite$MyAdapter` (Group 2, not Compose) | 2 | uncertain — classes aren't Compose-named, but OneUI's power menu is visually a full custom redesign, so Samsung may have replaced this wholesale rather than subclassed it | low |
| DialogGlass | `statusbar.phone.SystemUIDialog`, `globalactions.*` | 2 | plausible — SystemUIDialog is a base class many internal dialogs extend; worth testing before assuming it fails | medium |
| ToastGlass | `toast.SystemUIToast` | 2 | plausible — toasts are simple, base class may be retained | medium |
| BackGestureGlass | `navigationbar.gestural.BackPanel` | 2 | uncertain — not Compose-named but OneUI's back-gesture edge visual may be a separate proprietary overlay | low |
| StatusBarChipGlass | `statusbar.chips.ui.compose.OngoingActivityChipKt`, `com.android.compose.animation.*` | 3 | will not resolve — this is a newer Android 15/16 "live status chip" feature; unclear if OneUI 8.5 even has an equivalent UI yet | high |
| TransitionGlass | `animation.GhostedViewTransitionAnimatorController`, `animation.TransitionAnimator` (Group 2, legacy) + `com.android.compose.animation.*` (Group 3) | mixed | legacy animator classes might resolve, compose overlay node won't | low-medium |
| SwitchGlass | `androidx.compose.material3.SwitchKt/SwitchColors` | 4 | unconfirmed — depends entirely on whether OneUI SystemUI instantiates this exact Compose widget anywhere; don't assume either way | needs device check |
| SliderGlass | `androidx.compose.material3.SliderDefaults/SliderState` | 4 | same as above | needs device check |
| ButtonGlass | `androidx.compose.material3.ButtonKt/IconButtonKt/SurfaceKt` | 4 | same as above | needs device check |

## What this means for scope
Of 24 modules: 1 confirmed working (Notifications), ~4 likely working or low-risk
(WindowDimTracker, ViewGlassUnderlay, DozeTracker, DisplayRootTracker — all infra, not
visual surfaces users directly toggle), ~11 almost certainly need a full Samsung-specific
rewrite (everything hitting Group 3 Compose classes: QS tiles, QS footer, brightness
slider, lockscreen shortcuts, bouncer, volume dialog + panel, status chips), and the rest
are genuine "test it and see" cases (DialogGlass, ToastGlass, PowerMenuGlass,
BackGestureGlass, ShadeGlass, MediaGlass, QsOpenMotion, TransitionGlass, and the three
generic Compose-widget modules).

**The Quick Settings panel the user's screenshots actually show — background, tiles,
footer, brightness slider — is almost entirely Group 3.** That's the hardest part of
GlassLayer to port, not the easiest, despite being the most visually obvious target.
