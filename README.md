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

## 配套使用（与 小鱼 TabView 相互配合）

本技能与 **[xiaoyu-tabview（小鱼 TabView）](https://github.com/zy5120/xiaoyu-tabview)**（iOS 26 精美 TabView：底部长条操作按钮 + 底部搜索）**相互配合**，各管一层：

1. 先按本技能适配页面的横竖屏双模式（竖屏 sheet/推入、横屏副屏、旋转不丢状态）；
2. 再按 `xiaoyu-tabview` 适配精美 TabView 的底部长条按钮与搜索（长条按钮跟随焦点、搜索进副屏、不适合搜索的页面点击无反应）。

适配任何页面时按此顺序执行，可同时验证两个技能的效果。

## 许可证

本项目采用 **Apache License 2.0**。

**署名要求**：任何使用、调用、复制或改编本技能（含将其作为开发规范、用于生成代码/内容、或整合进其他技能/项目）的，必须保留本版权声明，并注明来源：

- 仓库：<https://github.com/zy5120/ipad-adaptive-layout-ios26>
- 作者：zy5120（小鱼塔罗 / 鱼律 等 iOS App）

在派生产物（文档、代码、技能文件）中请包含以上署名与许可证文本。

Copyright © 2026 zy5120
