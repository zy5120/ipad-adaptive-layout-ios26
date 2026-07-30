---
name: 小鱼平板适配forOS26
version: "1.0.0"
description: "iPad adaptive dual-mode layout — Sheet or NavigationStack push in portrait, sidebar in landscape. Switch modes per-page via a single config flag. Two rendering paths isolated; no page code changes needed when switching."
---

## 版本

v1.0.0 — 双模式方案（Sheet / FullScreen）完整版。两套底层实现完全隔离，页面仅改标识即可切换。

---

## 核心架构

```
AdaptiveRootView (HStack)
  |-- ContentView (TabView)
  |     ├── Tab 0: DivinationPage  ──┐
  |     ├── Tab 1: HistoryPage     ──┼── 各 tab 内部维护 push 状态 + NavigationStack
  |     └── Tab 2: MorePage        ──┘
  |     [Sheet 路径] .sheet(isPresented:) → SplitDetailPane（仅 fish-sheet 页面触发）
  |-- [if landscape] Divider
  |-- [if landscape] SplitDetailPane（fish-sheet 和 fish-fullscreen 均在此渲染）
```

### 数据流

```
detailSelection（唯一数据源，@State in AdaptiveRootView）
  ├── 竖屏 fish-sheet   → ContentView.sheet → SplitDetailPane
  ├── 竖屏 fish-fullscreen → Tab 内 push state → navigationDestination push
  ├── 横屏（两种模式）  → HStack 右侧 SplitDetailPane
  └── 旋转切换          → pop push state / 关 sheet，副屏接管
```

---

## DisplayMode 枚举

```swift
/// 页面展示模式。仅需修改此标识即可切换方案，无需改动页面业务代码。
enum DisplayMode {
    case inline      // 不展示（tooltip 等）
    case sheet       // fish-sheet：竖屏底部 sheet，横屏副屏
    case fullScreen  // fish-fullscreen：竖屏 NavigationStack push，横屏副屏
}
```

### 页面注册

```swift
enum DetailSelection: Equatable {
    case none
    case methodsPicker              // → .sheet（快速选择器）
    case divinationResult(...)      // → .fullScreen（主内容）
    case historyEntry(HistoryEntry) // → .fullScreen
    case methodsDetail              // → .fullScreen（参考文档）

    /// 仅改此处即可切换 fish-sheet ↔ fish-fullscreen
    var displayMode: DisplayMode {
        switch self {
        case .none:                     return .inline
        case .methodsPicker, .reminder,
             .numberInput, .characterInput,
             .oddEven, .manualSelect,
             .aiModelConfig:            return .sheet
        case .divinationResult, .historyEntry,
             .methodsDetail, .godsDetail,
             .privacyDetail, .disclaimerDetail,
             .onboarding:               return .fullScreen
        }
    }
}
```

---

## 两条路径完全隔离

### fish-sheet 路径（旧方案，不动）

```
点击 → detailSelection = .xxx
     → ContentView.onChange → displayMode == .sheet → showDetailSheet = true
     → .sheet(isPresented:) { SplitDetailPane }
     → SplitDetailPane 内链式切换（selection 驱动）
```

ContentView 仅管理 sheet 路径。`displayMode == .fullScreen` 的页面完全不触发此路径。

```swift
// ContentView.swift — 仅处理 fish-sheet
private func handleSelectionChange(_ sel: DetailSelection) {
    if sel == .none { showDetailSheet = false; return }
    guard !isSidebar else { return }
    if sel.displayMode == .sheet { showDetailSheet = true }
    // fish-fullscreen: 不经过此处，由各 tab 自行处理
}
```

### fish-fullscreen 路径（新方案）

以历史页为参考实现：

```swift
// HistoryPage.swift
struct HistoryPage: View {
    @Binding var detailSelection: DetailSelection
    let isSidebar: Bool
    @State private var pushedEntry: HistoryEntry?  // ← Tab 内 push 状态

    var body: some View {
        NavigationStack {
            List(entries) { entry in
                Button {
                    detailSelection = .historyEntry(entry)  // 全局同步
                    if !isSidebar { pushedEntry = entry }   // 竖屏 push
                } label: { historyRow(entry) }
            }
            .navigationDestination(item: $pushedEntry) { entry in
                ResultSheet(...)  // 与 SplitDetailPane 渲染同一视图
            }
        }
        .onChange(of: isSidebar) { _, sidebar in
            if sidebar {
                pushedEntry = nil              // 横屏：pop，副屏接管
            } else if case .historyEntry(let e) = detailSelection {
                pushedEntry = e                // 竖屏：从 detailSelection 恢复 push
            }
        }
        .onChange(of: detailSelection) { _, sel in
            if !isSidebar, case .historyEntry(let e) = sel {
                pushedEntry = e                // 外部触发（如横屏副屏点 xmark 后旋转）
            }
        }
        .onChange(of: pushedEntry) { _, new in
            if new == nil, !isSidebar { detailSelection = .none }  // 返回时清全局状态
        }
    }
}
```

**关键原则**：
- `detailSelection` 唯一数据源，push 状态是派生量
- 旋转只切换可见性：竖屏 push 可见/副屏隐藏，横屏副屏可见/push pop
- 三个 tab 模式完全一致，区别仅在于 push 状态的类型

---

## 新增全屏页面（3 步）

1. `DetailSelection` 加 case，`displayMode` 返回 `.fullScreen`
2. `SplitDetailPane.makeView()` 加渲染分支（横屏用，与旧方案相同）
3. 对应 Tab 内加 `@State pushState` + `navigationDestination` + 3 个 `onChange`

不用动 ContentView、AdaptiveRootView、任何业务页面代码。

---

## 从 fish-sheet 切到 fish-fullscreen

仅改 `DetailSelection.displayMode` 返回值。同时：
1. 确认目标 Tab 页已按上述 3 步注册 push 状态
2. 确认 `SplitDetailPane` 已有渲染分支（横屏副屏需要）

业务页面零改动。

---

## 横竖屏判断

```swift
private var isLandscape: Bool {
    let override = landscapeOverride
        ?? (UIScreen.main.bounds.width >= UIScreen.main.bounds.height)
    return override
}

// onGeometryChange 更新 landscapeOverride，过滤键盘：
// 宽≈屏幕宽 且 高骤降 >200pt → 维持原值（键盘干扰）
```

---

## 旋转同步速查

| 场景 | fish-sheet | fish-fullscreen |
|------|-----------|----------------|
| 竖屏→横屏 | `showDetailSheet = false`，副屏渲染 | `pushedXxx = nil`，副屏渲染 |
| 横屏→竖屏 | `detailSelection` 不变，用户需重新点击触发 sheet | 从 `detailSelection` 恢复 `pushedXxx`，自动 push |
| 用户返回 | sheet dismiss gate 清 `detailSelection` | `pushedXxx → nil` → 清 `detailSelection` |

---

## 相关

- [Apple HIG: Layout for iPad](https://developer.apple.com/design/human-interface-guidelines/layout)
- 鱼律项目 NavigationStack push 参考: `/Users/yuu/Documents/yulawyer/鱼律/鱼律/Cases/CasesListView.swift`
- 开源仓库: https://github.com/zy5120/ipad-adaptive-layout-ios26
