# iPad 自适应布局：HStack + 条件侧栏（iOS 26+）

**中文** | [English](#english)

> ⭐ **如果这个 skill 帮你省了时间，请给仓库点个 [Star](https://github.com/zy5120/ipad-adaptive-layout-ios26) 和 [Fork](https://github.com/zy5120/ipad-adaptive-layout-ios26)！**

一套可复用的 SwiftUI 布局模式：用可预测的 `HStack` 自适应布局替代 `NavigationSplitView`。面向 iOS 26+ 应用，实现「竖屏 = sheet 弹层、横屏 = 内联侧栏」，不受 UIKit size class 干扰，旋转不丢状态。

## 解决什么问题

- **旋转不丢状态**：视图永不离开层级，`@StateObject` / `@State` 在横竖屏切换中完好存活
- **布局可预测**：用 `windowSize.width >= windowSize.height` 判断横屏，而不是不可靠的 `horizontalSizeClass`
- **Sheet 管理干净**：sheet 只在竖屏触发；横屏时同一个视图内联渲染并带关闭按钮
- **无 UIKit 干扰**：对侧栏强制 `.horizontalSizeClass = .compact`，屏蔽 UIKit 分屏手势

## 架构

```
AdaptiveRootView (HStack)
  |-- ContentView（常驻，TabView）
  |-- [仅横屏] Divider + SplitDetailPane
```

## 核心模式

| 模式 | 位置 |
|------|------|
| HStack + 条件侧栏的根布局 | SKILL.md 中 `AdaptiveRootView` |
| 带 `needsSheet` 计算属性的 Selection 枚举 | SKILL.md 中 `DetailSelection` |
| 带旋转守卫的 sheet 编排器 | SKILL.md 中 `ContentView` |
| 双模式详情面板（sheet / 内联） | SKILL.md 中 `SplitDetailPane` |
| 链式 Sheet 铁律（链内禁 `dismiss()` + isChainTransition 门） | SKILL.md 中「链式 Sheet 铁律」 |
| 浮动关闭按钮（xmark + Circle + ultraThinMaterial） | SKILL.md 中 `closeButton` |
| 子弹层的 sheet / overlay 双轨切换 | SKILL.md 中 `rightAlignSheet` |
| ObservableObject 上提实现旋转存活 | SKILL.md 中 Rotation Survival 一节 |

## 适用场景

- 开发带侧栏布局的 iPad 自适应 iOS 26+ 应用
- `NavigationSplitView` 导致状态重置或 sheet 冲突
- 需要横竖屏都能正常工作的子弹层（sheet 套 sheet）
- 有必须在旋转中存活的长生命周期状态（AI 流式输出、表单进度）

## 来源

提炼自 iOS 应用「小鱼塔罗」（zhu.yu.tarot，iOS 26+），并在「小鱼六壬」中二次验证；该模式已在实体 iPad 上经受数千次旋转循环考验。链式 Sheet 铁律来自两个项目各踩一次的真实教训。

---

<a name="english"></a>

# iPad Adaptive Layout: HStack + Conditional Pane (iOS 26+)

[中文](#ipad-自适应布局hstack--条件侧栏ios-26) | **English**

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
| Chained-sheets iron rule (no `dismiss()` inside chain + isChainTransition gate) | "链式 Sheet 铁律" in SKILL.md |
| Floating close button (xmark + Circle + ultraThinMaterial) | `closeButton` in SKILL.md |
| Sheet vs overlay switching for sub-prompts | `rightAlignSheet` in SKILL.md |
| ObservableObject hoisting for rotation survival | Rotation Survival section in SKILL.md |

## When to Use

- Building an iPad-adaptive iOS 26+ app with a sidebar layout
- `NavigationSplitView` is causing state resets or sheet conflicts
- You need sub-prompts (sheets on sheets) that work correctly in both orientations
- You have long-lived state (AI streaming, form progress) that must survive rotation

## Source

Extracted from the 小鱼塔罗 iOS app (zhu.yu.tarot, iOS 26+) and battle-tested again in 小鱼六壬, proven across thousands of rotation cycles on physical iPad hardware. The chained-sheets iron rule comes from hitting the same bug once in each project.
