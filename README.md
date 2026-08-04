# 小鱼平板适配forOS26（iPad Adaptive Layout）

用可预测的 `HStack` 双栏结构取代 `NavigationSplitView` 的 iOS 26+ 自适应布局方案：竖屏 sheet / NavigationStack push，横屏右侧副屏；旋转不丢状态、不受 UIKit 尺寸类别干扰。

> 本技能来自「小鱼塔罗 / zhu.yu.tarot」iOS App（iOS 26+）实测沉淀（横竖屏数千次旋转验证），并被「鱼律 / yulawyer」等 App 使用。

## 解决的问题

- **旋转存活**：视图不离开层级，`@State` / `@StateObject` 在横竖屏切换后不丢失。
- **方向判断可靠**：用 `windowSize.width >= windowSize.height` 判断横竖屏，不用不可靠的 `horizontalSizeClass`。
- **干净的 sheet 处理**：仅竖屏弹出 sheet；横屏同一视图以内嵌方式呈现并带关闭按钮。
- **无 UIKit 干扰**：副屏强制 `.horizontalSizeClass = .compact`，避免 UIKit 分屏手势冲突。

## 架构

```
AdaptiveRootView (HStack)
  |-- ContentView (常驻，TabView)
  |-- [仅横屏] Divider + SplitDetailPane
```

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

---

# 小鱼平板适配forOS26 (iPad Adaptive Layout) — English

A reusable SwiftUI pattern that replaces `NavigationSplitView` with a predictable `HStack`-based adaptive layout. Designed for iOS 26+ apps that need portrait-sheet / landscape-sidebar behavior without UIKit size class interference or rotation-induced state loss.

> Extracted from the "小鱼塔罗 / zhu.yu.tarot" iOS app (iOS 26+), proven across thousands of rotation cycles on physical iPad hardware; also used by "鱼律 / yulawyer" and other apps.

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

## Companion Skill

This skill works **together with [xiaoyu-tabview (小鱼 TabView)](https://github.com/zy5120/xiaoyu-tabview)** (iOS 26 beautiful tab bar: bottom long action button + bottom search):

1. First adapt each page's dual-mode layout with this skill (portrait sheet/push, landscape sidebar, rotation-safe state);
2. Then adapt the beautiful TabView's bottom action button and search with `xiaoyu-tabview` (button follows focus, search moves to the sidebar, non-searchable pages do nothing on search tap).

Each skill owns one layer. Follow this order when adapting any page.

## License

This project is licensed under the **Apache License 2.0**.

**Attribution required**: any use, invocation, copy, or adaptation of this skill (including using it as development guidelines, generating code/content, or integrating it into other skills/projects) must retain this copyright notice and credit the source:

- Repository: <https://github.com/zy5120/ipad-adaptive-layout-ios26>
- Author: zy5120 (小鱼塔罗 / 鱼律 iOS apps)

Please include the attribution and the license text in derived works (docs, code, skill files).

Copyright © 2026 zy5120
