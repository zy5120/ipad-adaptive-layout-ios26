---
name: 小鱼平板适配forOS26
description: "iPad 自适应双模式布局：竖屏用 Sheet 或 NavigationStack push，横屏用侧栏副屏；每个页面通过单个配置标志切换模式，两套渲染路径隔离、页面业务代码零改动。适用于：要求 iPad 竖屏/横屏自适应布局、旋转时状态不丢失、双模式（sheet/全屏）切换、保持 iPhone 与 iPad 界面一致。iPad adaptive dual-mode layout — Sheet or NavigationStack push in portrait, sidebar in landscape; switch modes per page via a single config flag; two rendering paths isolated; rotation-safe, no page code changes needed."
---

## 版本

v1.1.0 — 双模式方案（Sheet / FullScreen）完整版。两套底层实现完全隔离，页面仅改标识即可切换。
新增「三大反复出现的坑」实战总结：旋转残留重复页面、sheet↔push 不同步、旋转丢输入，以及键盘压缩误判方向的防护规则。

## 铁律：横屏副屏不限制设备

横屏副屏对 iPhone 与 iPad 一视同仁：`showSidebar = isLandscape`，**不要**用 `userInterfaceIdiom == .pad` 把 iPhone 排除——否则 iPhone 横屏只会主屏拉伸、不出副屏（实测踩坑）。iPhone 横屏出副屏是用户认可的效果（与 iPad 一致）。

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

### 如何选择展示模式

| 场景 | 用 sheet | 用 fullScreen |
|---|---|---|
| 轻量选择器 / 快捷输入（Picker、快速弹层） | ✅ | ❌ |
| 主内容 / 长文阅读（结果、文档、详情） | ❌ | ✅ |
| 竖屏时需要返回上一级 | ❌ | ✅（NavigationStack 自动返回箭头） |
| 横屏时希望副屏可关闭 | 两者均可（副屏接管） | 两者均可 |
| 临时弹层、随手关闭 | ✅ | ❌ |
| 需要保留页面状态、旋转不丢 | 均可（状态在根视图） | 均可（状态在根视图） |

无法确定时：把候选模式与页面列给开发者选择，不要擅自决定。

### 适配默认原则（重要）

适配时**原则上尽量用 `.fullScreen`**（稳定、无 bug）：主内容 / 详情 / **需要输入的表单**一律 `.fullScreen`（竖屏 push、横屏副屏）。只有**极轻量的小交互**才保留 `.sheet`：单项选择、快速确认这类“不需要完整页面、不需要键盘输入、随手开随手关”的小弹层。具体取舍交给实现者（AI）：能进副屏就 `.fullScreen`，只有纯轻量小交互才用 `.sheet`。

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

### 新页面接入检查清单

- [ ] `DetailSelection` 加 case，`displayMode` 按决策表返回 `.sheet` 或 `.fullScreen`
- [ ] `SplitDetailPane.makeView()` 已加渲染分支（横屏副屏需要）
- [ ] 对应 Tab 已注册 push 状态 + 3 个 `onChange`（fullScreen）；sheet 路径无需
- [ ] 页面内容宽度自适应（`.frame(maxWidth: .infinity)` / `LazyVGrid` / 横向滚动），副屏 ~35% 宽度不溢出
- [ ] fullScreen 模式无多余导航按钮（`grep -n "cancellationAction\|topBarLeading" <file>.swift` 为空）
- [ ] 横屏副屏内子页面用 `detailSelection` 路由而非 push
- [ ] 副屏 `xmark` 关闭时同时 `selection = .none`

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

**踩坑：方向判断不要用初始为 0 的尺寸**。若 `windowSize` 初始为 `.zero`，`0 >= 0` 会被判成横屏；启动时若首个几何事件恰好是键盘弹出尺寸（被过滤），真实尺寸一直没记录，App 会误以为横屏（竖屏点搜索也会出副屏）。修法：尺寸未知时默认竖屏——`isLandscape = windowSize != .zero && windowSize.width >= windowSize.height`；或直接使用 `UIScreen.main.bounds`（六壬/塔罗做法，无此问题）。

---

## 旋转同步速查

| 场景 | fish-sheet | fish-fullscreen |
|------|-----------|----------------|
| 竖屏→横屏 | `showDetailSheet = false`，副屏渲染 | `pushedXxx = nil`，副屏渲染 |
| 横屏→竖屏 | `detailSelection` 不变，用户需重新点击触发 sheet | 从 `detailSelection` 恢复 `pushedXxx`，自动 push |
| 用户返回 | sheet dismiss gate 清 `detailSelection` | `pushedXxx → nil` → 清 `detailSelection` |

---

## 铁律

### fish-fullscreen 模式下禁止额外导航按钮

NavigationStack push 自带返回箭头。页面内**禁止**添加取消、完成、关闭等 toolbar 按钮。`dismiss()` 仅限取消路径使用。

```swift
// ❌ push 模式下的多余按钮
.toolbar {
    ToolbarItem(placement: .cancellationAction) { Button("取消") { dismiss() } }
}

// ✅ 不设取消/关闭按钮，返回箭头自动出现
```

排查命令：`grep -n "cancellationAction\|topBarLeading" <file>.swift`

### 竖屏全屏时导航栏按钮去重

NavigationStack 自带返回箭头。若页面 toolbar 内出现功能重复的图标（如既有 `<` 又有 `xmark`），只保留一个。原则：

- 左上角 `.cancellationAction`：**不放任何按钮**。返回由 NavigationStack 自动提供
- 右上角 `.confirmationAction` / `.primaryAction`：最多一个按钮。功能相似则合并

横屏副屏（`showCloseButton = true`）不受此限制——需要 `xmark` 关闭副屏。

### 导航按钮用符号不用文字

竖屏返回由 NavigationStack 自动提供。自定义按钮全部用 SF Symbol：

| 操作 | 符号 | 位置 |
|------|------|------|
| 取消/关闭（横屏副屏） | `xmark` | `.cancellationAction`，仅横屏显示 |
| 保存/确认 | `checkmark` | `.confirmationAction` |
| 新增 | `plus` | `.primaryAction` |

禁止使用文字按钮（"取消""完成""保存"）。

```swift
// ✅ 正确
ToolbarItem(placement: .confirmationAction) {
    Button { save() } label: { Image(systemName: "checkmark") }
}

// ❌ 错误——文字按钮
ToolbarItem(placement: .confirmationAction) {
    Button("保存") { save() }
}
```

---

## 踩坑记录

### 副屏 xmark 关闭需同时 reset selection

`onClose` 回调中仅 `dismiss()` 不够，必须同时 `selection = .none`。横屏副屏的 xmark 调用 `onClose`，缺失 `selection = .none` 会导致副屏空白但不关闭。

```swift
// ✅ 正确
onClose: { context.dismiss(); context.selection = .none }

// ❌ 只 dismiss 不清 selection——副屏关不掉
onClose: { context.dismiss() }
```

### context 改为 optional 后的兼容写法

当 `DetailPaneContext?` 改为可选时，`@Binding var selection` 不能通过可选链直接赋值。写法：

```swift
// ✅ 正确
onClose: { context?.dismiss(); if let ctx = context { ctx.selection = .none } }

// ❌ 不能用 wrappedValue
context?.selection.wrappedValue = .none  // 编译失败
```

### 自定义 init 扩展参数

视图有自定义 `init` 时，新增的默认参数不会自动透传。必须在 `init` 签名中显式添加。

```swift
// 旧
init(card: Card, startReversed: Bool) { ... }

// 新（加 showCloseButton + onClose）
init(card: Card, startReversed: Bool, showCloseButton: Bool = false, onClose: (() -> Void)? = nil) {
    self.card = card
    self.showCloseButton = showCloseButton  // ← 必须显式赋值
    self.onClose = onClose
    ...
}
```

### 页面内容需基于宽度自适应

竖屏 push 全宽，横屏副屏仅 ~35% 宽度。页面内固定宽度的元素（如 `frame(width: ...)` 的 Picker、网格等）在副屏会溢出。使用 `.frame(maxWidth: .infinity)`、`LazyVGrid`、`ScrollView(.horizontal)` 等响应式布局。

### 横屏时子页面用 detailSelection 路由而非 push

当页面在横屏副屏内时，其子页面的点击应设 `detailSelection` 让副屏切换，而非 push 到当前 NavigationStack（会挤在窄栏内）。

```swift
// ✅ 横屏用 detailSelection 路由副屏
if showCloseButton {
    detailSelection = .cardDetail(card)
} else {
    pushedCard = card  // 竖屏 push
}
```

## 三大反复出现的坑（鱼律实战总结，务必先读）

以下三个问题在多个页面反复出现，根因都是「局部状态 + 条件呈现器 + 旋转重建」的组合。任何新页面接入双模式前，先对照本清单自查。

### 坑 1：横屏后左屏残留右屏的页面（重复页面）

**现象**：横屏时左屏也出现了右屏的页面（如搜索公司、参与人编辑）。

**根因**：页面 A 被推入左屏栈后，A 内部再用 `navigationDestination(item:)/(isPresented:)` 推入子页 B（B 的打开状态是 A 的局部 @State）。旋转到横屏时外层 push（`pushedDetail`）被 pop，但**内层 item 推入可能残留在旧栈**；随后 B 的呈现状态再次为 true 时，会在旧容器里再推一份 → 左屏出现重复页面（日志特征：一次打开出现两个 `CompanySearchView 出现`，一个 `presentedAsSheet=false` 一个 `=true`）。

**铁律**：
- 旋转时显式清掉旧容器里的嵌套推入：在页面 `onChange(of: pad.isSidebar)` 里把子页 item 置 nil（pop 旧栈），新容器再按全局会话恢复。
- 子页的「打开状态」必须放全局（如 `pad.companySearchOpen`），旋转后新容器 `onAppear` 时按当前方向重建呈现。
- 关闭回调（`onChange(item → nil)`）**不要**顺手清全局会话；由子页自身 `onDisappear` 判断「用户关闭 vs 旋转重建」：
  - 旋转：`onChange(of: pad.isSidebar)` 里置 `wasRotated = true`，`onDisappear` 只存档、不清会话；
  - 用户关闭：`onDisappear` 时清空全局会话（打开状态 + 草稿）。

### 坑 2：横竖屏切换页面不同步（sheet ↔ push 不转换）

**现象**：竖屏 sheet 打开，转横屏后没有变成右屏推入；或反过来，横屏推入转竖屏没有变成 sheet。

**根因**：条件呈现器 `if pad.isSidebar { navigationDestination(...) } else { sheet(...) }` 在同一绑定上切换，SwiftUI 不会自动把旧呈现转成新呈现——旧的被拆掉后，新的不会自动出现（旋转时页面重建，局部 `isPresented` 状态归零）。

**铁律**：
- 条件呈现器的「打开状态」放全局；旋转后重建的页面 `onAppear` 读取全局状态重新呈现（此时 presenter 已按新方向渲染）。
- 旋转瞬间（`onChange(of: pad.isSidebar)`）先置 `shown = false` 拆掉旧呈现，避免新旧并存/残留。

### 坑 3：旋转丢输入（临时书写记录不保存）

**现象**：表单输入后旋转，内容丢失（新建/编辑案件、刑事新建、AI 配置、参与人、搜索公司都中过招）。

**根因**：字段全是视图内 `@State`；旋转导致视图重建（竖屏 push ↔ 横屏副屏切换）→ `@State` 归零。

**铁律（表单草稿全局化）**：
- 打开时把会话/草稿写入全局状态（pad）。
- `onChange(of: pad.isSidebar)` 与 `onDisappear` **双点存档**（旋转瞬间立即存，不依赖消失回调先后）。
- 新实例 `onAppear` 恢复；再加「草稿版本号」兜底：每次存档 version+1，新实例 `onChange(of: version)` 恢复（解决“旧实例消失晚于新实例出现”的竞态）。
- 保存/用户关闭时显式清空草稿（避免下次打开恢复旧内容）。

### 方向判断：键盘压缩误判横屏（宽度变化原则）

**现象**：窄竖屏（iPad mini 744×1133）+ 键盘弹出 → 可视高度被压到 < 宽度（744×720）→ 误判横屏 → 双模式呈现器当场切换 → 弹层被拆掉（“页面崩了”）、或闪现分栏。

**根因**：`isLandscape = width >= height` 只看宽高比；键盘只压缩高度。

**铁律**：
- 键盘期间只接受「宽度明显变化（>40pt）且宽高比翻转」的尺寸（真旋转会改变宽度）。
- **该判断必须无条件生效，不能依赖键盘通知先后**：几何事件可能先于 `keyboardWillShow` 到达（竞态），所以「宽度没变、高度压缩 >60pt」直接拒绝，不看 `keyboardVisible`。
- 真旋转（宽度变化）正常接受；键盘收起恢复高度正常接受（高度变大不拦截）。

```swift
.onGeometryChange(for: CGSize.self, of: { $0.size }) { newSize in
    guard newSize.width > 0, newSize.height > 0 else { return }
    // 无条件：宽度没变、高度明显压缩 → 键盘压缩（即使键盘通知还没到也忽略）
    if windowSize != .zero,
       abs(newSize.width - windowSize.width) <= 40,
       newSize.height < windowSize.height - 60 { return }
    // 键盘期间：仅接受宽度明显变化且宽高比翻转（真旋转）
    if keyboardVisible || Date().timeIntervalSince(keyboardChangeAt) < 0.6 {
        guard windowSize == .zero
            || (abs(newSize.width - windowSize.width) > 40
                && (windowSize.width >= windowSize.height)
                   != (newSize.width >= newSize.height))
        else { return }
    }
    windowSize = newSize
}
```

### 嵌套推入的旋转清理清单（接入新页面时逐条核对）

旋转到横屏时，检查旧容器（左屏栈）里是否有 `navigationDestination(item:)/(isPresented:)` 驱动的嵌套推入：
1. 外层 `pushedDetail` 由 DetailSelectionLink 自动 pop；
2. 内层 item 推入：在页面的 `onChange(of: pad.isSidebar)` 里显式置 nil（pop 旧栈）；
3. 全局会话（打开状态 + 草稿）由新容器 `onAppear` 恢复，按当前方向重建（竖屏 sheet、横屏右屏推入）；
4. 所有「打开状态」和「表单草稿」统一放全局，页面内不留决定性状态。

## 相关

## 带精美 TabView 的 App（横屏副屏联动）

目标 App 已有“精美 TabView”（独立搜索标签 + 底部长条按钮 + 底部搜索）时，直接套用 35% 左栏方案会让控件与副屏“分家”：搜索、长条按钮要**按焦点跟随主屏或副屏**，副屏内容切换要**强制重建**（否则显示旧内容）。完整要点见 [references/tabview-apps.md](references/tabview-apps.md)。

> 当前范围：只适配了 `.fullScreen`（详情类）页面；`.sheet`（轻量弹层类）页面尚未适配。

---

## 相关

- [Apple HIG: Layout for iPad](https://developer.apple.com/design/human-interface-guidelines/layout)
- NavigationStack push 的返回箭头与 toolbar 去重：遵循系统默认行为即可，无需额外文件
- 开源仓库: https://github.com/zy5120/ipad-adaptive-layout-ios26
- 相关技能：iOS 26 精美 TabView 改造（长条按钮 + 底部搜索）→ `xiaoyu-tabview`（~/.codex/skills/xiaoyu-tabview）
