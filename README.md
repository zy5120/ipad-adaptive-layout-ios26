# iPad Adaptive Layout: HStack + Conditional Pane

> ⭐ **If this saved you time, please [star](https://github.com/zy5120/ipad-adaptive-layout-ios26) and [fork](https://github.com/zy5120/ipad-adaptive-layout-ios26) the repo!**

A reusable SwiftUI pattern that replaces `NavigationSplitView` with a predictable `HStack`-based adaptive layout. Designed for iOS 26+ apps that need portrait-sheet / landscape-sidebar behavior without UIKit size class interference or rotation-induced state loss.

## What It Solves

- **Rotation survival:** `@StateObject` and `@State` survive orientation changes because views never leave the hierarchy.
- **Predictable layout:** Detects landscape via `windowSize.width >= windowSize.height` instead of unreliable `horizontalSizeClass`.
- **Clean sheet handling:** Sheets only fire in portrait. In landscape, the same view renders inline with a close button.
- **No UIKit interference:** Forces `.horizontalSizeClass = .compact` on the sidebar to prevent UIKit split-view gestures.

## Architecture

```
AdaptiveRootView (HStack)
  |-- ContentView (always present, TabView)
  |-- [landscape only] Divider + SplitDetailPane
```

## Key Patterns

| Pattern | File |
|---------|------|
| Root layout with HStack + conditional pane | `AdaptiveRootView` in SKILL.md |
| Selection enum with `needsSheet` computed property | `DetailSelection` in SKILL.md |
| Sheet orchestrator with rotation guards | `ContentView` in SKILL.md |
| Dual-mode detail pane (sheet vs inline) | `SplitDetailPane` in SKILL.md |
| Floating close button (xmark + Circle + ultraThinMaterial) | `closeButton` in SKILL.md |
| Sheet vs overlay switching for sub-prompts | `rightAlignSheet` in SKILL.md |
| ObservableObject hoisting for rotation survival | Rotation Survival section in SKILL.md |

## When to Use

- Building an iPad-adaptive iOS 26+ app with a sidebar layout
- `NavigationSplitView` is causing state resets or sheet conflicts
- You need sub-prompts (sheets on sheets) that work correctly in both orientations
- You have long-lived state (AI streaming, form progress) that must survive rotation

## Source

Extracted from the 小鱼塔罗 iOS app (zhu.yu.tarot, iOS 26+), where this pattern was proven across thousands of rotation cycles on physical iPad hardware.
