---
name: 小鱼平板适配forOS26
description: "iPad adaptive layout using HStack + conditional pane (NOT NavigationSplitView). Use when: building iOS 26+ SwiftUI apps that need portrait-sheet/landscape-sidebar behavior without UIKit size class interference."
---

# iPad Adaptive Layout: HStack + Conditional Pane (iOS 26+)

> Replace NavigationSplitView with a predictable HStack-based adaptive layout where portrait = sheet overlay, landscape = inline sidebar — surviving rotation without state loss.

## Quick Reference

| Problem | Solution |
|---------|----------|
| NavigationSplitView destroys child state on rotation | Use `HStack` with conditional right pane |
| `horizontalSizeClass` triggers wrong layout on iPad | Detect layout via `windowSize.width >= windowSize.height` |
| Sheet-on-top-of-sheet stacking in landscape | Use `.overlay` instead of `.sheet` for sub-prompts |
| ObservableObject dies on rotation | Hoist it to the root `AdaptiveRootView` |
| Split-pane close button needs same code in both modes | `onClose: { dismiss(); selection = .none }` |
| UIKit split-view gestures interfere with custom layout | Force `.environment(\.horizontalSizeClass, .compact)` on sidebar |

## The Problem

`NavigationSplitView` appears to be the canonical iOS 26+ iPad layout API, but it has fatal flaws: it destroys and recreates child view hierarchies on rotation, resets `@State` and `@StateObject`, and fights you on sheet presentation. UIKit's built-in split-view gestures can steal touches from your custom UI. The result is a sidebar that works in one orientation and breaks in the other, with state randomly resetting.

## The Architecture

The entire layout lives inside a single `HStack` container. The left pane (sidebar/tab bar) is always present. The right pane (detail) is added conditionally when `isLandscape` is true. When `isLandscape` is false, detail content is shown via `.sheet` modals instead.

```
AdaptiveRootView (HStack)
  |-- ContentView (TabView, always present)
  |     .frame(maxWidth: sidebar ? 35% : .infinity)
  |     .environment(\.horizontalSizeClass, .compact)
  |-- [if landscape] Divider
  |-- [if landscape] SplitDetailPane
```

### Why NOT NavigationSplitView

| NavigationSplitView | HStack + Conditional Pane |
|---|---|
| Destroys/recreates children on rotation | Children never leave the hierarchy |
| `@StateObject` resets | State survives rotation |
| UIKit manages sidebar show/hide | You control everything |
| Three-column API is confusing | Simple two-pane HStack |

## Solutions

### Option 1: AdaptiveRootView (Root Layout)

The entry point. Hoists all shared state (`ObservableObject`s) here so they survive orientation changes. Detects layout mode from geometry, not size class.

```swift
struct AdaptiveRootView: View {
    @StateObject private var sharedState = SharedObservableState()
    @State private var detailSelection: DetailSelection = .none
    @State private var windowSize: CGSize = .zero

    private var isLandscape: Bool {
        windowSize.height > 0 && windowSize.width >= windowSize.height
    }
    private var showSidebar: Bool { isLandscape }

    var body: some View {
        HStack(spacing: 0) {
            // Left pane — always present
            ContentView(detailSelection: $detailSelection, isSidebar: showSidebar)
                .frame(maxWidth: showSidebar
                    ? max(360, windowSize.width * 0.35)
                    : .infinity)
                .environment(\.horizontalSizeClass, .compact)

            // Right pane — only in landscape
            if showSidebar {
                Divider()
                SplitDetailPane(
                    selection: $detailSelection,
                    showCloseButton: true
                )
                .frame(maxWidth: .infinity)
            }
        }
        .background(.background)
        .environmentObject(sharedState)           // <-- survives rotation
        .onGeometryChange(for: CGSize.self, of: { $0.size }) { newSize in
            windowSize = newSize
        }
    }
}
```

**Critical detail:** `.environment(\.horizontalSizeClass, .compact)` on the sidebar prevents UIKit from applying its own split-view behaviors (swipe-to-show/hide, overlay gestures) that conflict with the custom layout.

### Option 2: DetailSelection Enum (Routing)

An enum that drives all detail/sheet routing. The `needsSheet` computed property distinguishes between lightweight overlays (like card tooltips) and full-screen flows that need a sheet in portrait mode.

```swift
enum DetailSelection: Equatable {
    case none
    case cardDetail(Card)              // lightweight — never a sheet
    case historyEntry(HistoryEntry)    // heavy — sheet in portrait
    case startFlow(FlowType, params...) // heavy — sheet in portrait

    /// True when this case should be presented as a sheet in portrait mode.
    /// Lightweight cases (tooltips, quick info) should be false.
    var needsSheet: Bool {
        switch self {
        case .none, .cardDetail: return false
        case .historyEntry, .startFlow: return true
        }
    }
}
```

The `.none` case is special — selecting it dismisses whatever is currently shown.

### Option 3: ContentView (Sheet Orchestrator)

The left pane (usually a `TabView`). Manages a single `@State private var showDetailSheet = false` that controls sheet presentation. The `onChange` handlers react to both orientation changes and selection changes.

```swift
struct ContentView: View {
    @Binding var detailSelection: DetailSelection
    let isSidebar: Bool
    @State private var showDetailSheet = false

    var body: some View {
        TabView(selection: $selectedTab) {
            // ... tab pages ...
        }
        .onAppear {
            showDetailSheet = !isSidebar && detailSelection.needsSheet
        }
        // Rotate gate: landscape -> dismiss sheet; portrait -> show sheet if needed
        .onChange(of: isSidebar) { _, sidebar in
            if !sidebar && detailSelection.needsSheet { showDetailSheet = true }
            if sidebar { showDetailSheet = false }
        }
        // Selection gate: new selection in portrait triggers sheet
        .onChange(of: detailSelection) { _, sel in
            if sel == .none { showDetailSheet = false }
            if sel.needsSheet && !isSidebar { showDetailSheet = true }
        }
        // Dismiss gate: sheet swipe-dismiss clears selection
        .onChange(of: showDetailSheet) { _, showing in
            if !showing, !isSidebar { detailSelection = .none }
        }
        .sheet(isPresented: $showDetailSheet) {
            SplitDetailPane(selection: $detailSelection)
        }
    }
}
```

**Key insight:** The `.sheet(isPresented: $showDetailSheet)` call only fires in portrait mode. In landscape, `showDetailSheet` is always `false`, so the sheet never presents. Instead, `SplitDetailPane` renders inline in the `HStack` of `AdaptiveRootView`.

### Option 4: SplitDetailPane (Dual-Mode Detail View)

A single view that works in both contexts. The `showCloseButton` flag controls behavior:

- **Landscape (showCloseButton: true):** Renders inline. Shows a toolbar xmark button that resets `selection = .none`.
- **Portrait (showCloseButton: false):** Renders inside a `.sheet`. Close button is hidden (system swipe-dismiss handles it).

**Critical rule — close button style:** Every sidebar view uses the same **toolbar xmark** in `NavigationStack` (placement: `.cancellationAction`). Simple views use the `morePane { }` wrapper; views with existing NavigationStack (like CardDrawView) handle it inline; dark-background views add `.toolbarBackground(.hidden)`.

```swift
struct SplitDetailPane: View {
    @Binding var selection: DetailSelection
    var showCloseButton: Bool = false
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        switch selection {
        case .none:
            EmptyPlaceholderView()
        case .cardDetail(let card):
            CardDetailView(card: card)
        case .startFlow(let type, ...):
            // ✅ WRAP with ZStack, let SplitDetailPane own the close button
            ZStack(alignment: .topLeading) {
                FlowView(
                    onComplete: { result in
                        selection = .nextStep(result)
                    },
                    onClose: {
                        dismiss()               // works in sheet mode
                        selection = .none       // works in both modes
                    }
                )
                if showCloseButton { closeButton }
            }
        }
    }

    // Generic wrapper for simple content (lists, static views)
    private func morePane<Content: View>(
        @ViewBuilder content: () -> Content
    ) -> some View {
        NavigationStack {
            content()
                .toolbar {
                    if showCloseButton {
                        ToolbarItem(placement: .cancellationAction) {
                            Button { dismiss(); selection = .none } label: {
                                Image(systemName: "xmark")
                                    .font(.system(size: 16, weight: .semibold))
                            }
                        }
                    }
                }
        }
    }
}
```

**Wrapper pattern summary — every SplitDetailPane branch that needs a close button:**

| View type | Wrapping method | Example |
|-----------|----------------|---------|
| Simple list/static content | `morePane { }` wrapper (NavigationStack + toolbar xmark) | TarotCatalogView, PrivacyDetailView |
| Complex interactive view | Own NavigationStack + toolbar xmark inline | CardDrawView, reminderView |
| Fullscreen dark background | NavigationStack + `.toolbarBackground(.hidden)` | RitualView with white foreground |

All use the same **toolbar xmark** pattern defined in Option 5.

### Option 5: The Toolbar Close Button Pattern

A native navigation-bar xmark button. Every view that needs a close button wraps itself in `NavigationStack` and adds this toolbar item. Only shown when `showCloseButton` is true (landscape mode).

```swift
NavigationStack {
    ContentView(...)
        .toolbar {
            if showCloseButton {
                ToolbarItem(placement: .cancellationAction) {
                    Button {
                        dismiss()                // portrait: dismiss sheet
                        selection = .none        // landscape: clear sidebar
                    } label: {
                        Image(systemName: "xmark")
                            .font(.system(size: 16, weight: .semibold))
                    }
                }
            }
        }
}
```

**Why toolbar xmark instead of floating overlay?**

| Toolbar xmark | Floating overlay |
|---|---|
| Native nav-bar appearance, matches iOS HIG | Custom Circle+material, non-standard |
| Zero z-index conflicts | Must fight with scroll views and sheets |
| Same position in every view (nav bar top-left) | Position varies with padding tweaks |
| Works with NavigationStack title, search bar, other toolbar items | Blocks content area |

**Why `dismiss()` + `selection = .none`?** `dismiss()` handles the sheet case (system dismiss animation). `selection = .none` handles the inline case (clears the binding, right pane reverts to empty state). Calling both is safe — `dismiss()` is a no-op when not in a sheet context.

**For dark-background fullscreen views** (like a card-drawing ritual): add `.toolbarBackground(.hidden, for: .navigationBar)` to make the nav bar transparent while keeping the xmark visible. Use `.foregroundStyle(.white.opacity(0.8))` for visibility on dark backgrounds.

### Option 6: rightAlignSheet (Sheet vs Overlay for Sub-prompts)

When a view inside `SplitDetailPane` needs to present a sub-prompt (e.g., an AI input sheet), using `.sheet` in landscape causes a sheet-on-top-of-the-sidebar that looks broken. The `rightAlignSheet` flag switches to `.overlay` in landscape.

```swift
struct InnerView: View {
    var rightAlignSheet: Bool = false
    @State private var showPrompt = false

    var body: some View {
        // ... main content ...

        // Only show as .sheet in portrait
        .sheet(isPresented: rightAlignSheet
            ? .constant(false)          // never show sheet in landscape
            : $showPrompt
        ) {
            PromptSheetContent(onDismiss: { showPrompt = false })
        }
        // In landscape, show as .overlay instead
        .overlay {
            if showPrompt, rightAlignSheet {
                ZStack {
                    Color.black.opacity(0.2)
                        .ignoresSafeArea()
                        .onTapGesture { showPrompt = false }
                    VStack {
                        Spacer()
                        PromptSheetContent(onDismiss: { showPrompt = false })
                            .padding(.horizontal, 16)
                            .padding(.bottom, 16)
                    }
                }
                .transition(.opacity)
                .zIndex(100)
            }
        }
        .animation(.easeOut(duration: 0.25), value: showPrompt)
    }
}
```

**The trick:** `.sheet(isPresented: .constant(false))` permanently disables the sheet in landscape. The `.overlay` block then shows the same content inline. In portrait, the overlay block is never rendered (`rightAlignSheet` is false), so the normal `.sheet` fires instead.

## Rotation Survival: Hoist ObservableObject

Any `@StateObject` declared inside a view that is conditionally rendered (like `SplitDetailPane`) will be destroyed and recreated on rotation. The fix: declare it at the root level and pass it down via `.environmentObject()`.

```swift
// WRONG — dies on rotation
struct AdaptiveRootView: View {
    var body: some View {
        HStack {
            ContentView()
            if showSidebar {
                SplitDetailPane()  // Any @StateObject here resets
            }
        }
    }
}

// CORRECT — survives rotation
@MainActor
final class SharedState: ObservableObject {
    @Published var data: String = ""
    var task: Task<Void, Never>?
}

struct AdaptiveRootView: View {
    @StateObject private var sharedState = SharedState()  // lives at root

    var body: some View {
        HStack {
            ContentView()
                .environmentObject(sharedState)  // injected, not owned
            if showSidebar {
                SplitDetailPane()
                    .environmentObject(sharedState)  // same instance
            }
        }
    }
}
```

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| HStack + conditional pane | State survives rotation. Full control over layout. | More boilerplate than NavigationSplitView. Must handle split ratio manually. |
| NavigationSplitView | Built-in. Little code. | Destroys children on rotation. Poor sheet interaction. UIKit gestures interfere. |
| `horizontalSizeClass` detection | Apple-recommended. | Unreliable on iPad — can report `.regular` in portrait on larger devices. |
| `windowSize.width >= windowSize.height` | Always correct. Predictable. | Must manually compute. No built-in SwiftUI hook — uses `onGeometryChange`. |
| Sheet for sub-prompts | Native feel in portrait. | Looks wrong when stacked on a sidebar in landscape. |
| Overlay for sub-prompts | Clean in landscape sidebar. | Must reimplement dim-background, tap-to-dismiss, animation manually. |

## Edge Cases

- **iPad in Stage Manager with narrow window:** `windowSize.width >= windowSize.height` may report `false` even on iPad. This is correct behavior — the app should switch to portrait/sheet mode when the window is narrow.
- **Slide Over on iPad:** The app runs in a compact width. `windowSize` reports the actual slide-over size, so layout correctly switches to portrait mode.
- **External display mirroring:** `onGeometryChange` reports the window size, not the screen size. Layout adapts to the actual window.
- **Deep linking / state restoration:** The `DetailSelection` enum is `Codable`-compatible by design. Serialize at `scenePhase` changes and restore on launch.
- **Multiple levels of sub-sheets:** Each level must implement its own `rightAlignSheet` flag and overlay fallback. Chain the flag down: parent passes `rightAlignSheet: true` to children when itself is in sidebar context.

## Code Template: New Project Bootstrap

Starting a new iPad-adaptive app from scratch:

```swift
// 1. Define your selection enum
enum AppSelection: Equatable {
    case none
    case detail(Item)
    case flow(FlowParams)

    var needsSheet: Bool {
        if case .none = self { return false }
        if case .detail = self { return false }
        return true
    }
}

// 2. Root state object
@MainActor
final class AppState: ObservableObject {
    @Published var items: [Item] = []
}

// 3. Root view
struct AppRoot: View {
    @StateObject private var appState = AppState()
    @State private var selection: AppSelection = .none
    @State private var windowSize: CGSize = .zero

    private var isLandscape: Bool {
        windowSize.height > 0 && windowSize.width >= windowSize.height
    }

    var body: some View {
        HStack(spacing: 0) {
            SidebarView(selection: $selection, isSidebar: isLandscape)
                .frame(maxWidth: isLandscape ? max(360, windowSize.width * 0.35) : .infinity)
                .environment(\.horizontalSizeClass, .compact)

            if isLandscape {
                Divider()
                DetailPane(selection: $selection, showCloseButton: true)
                    .frame(maxWidth: .infinity)
            }
        }
        .environmentObject(appState)
        .onGeometryChange(for: CGSize.self, of: { $0.size }) { windowSize = $0 }
    }
}
```

## Common Pitfalls & Fixes

Issues encountered when applying this pattern to a real app, and how to fix them.

### Pitfall 1: Close button appears in portrait sheets

**Symptom:** Toolbar xmark is visible in portrait mode when it should be hidden (portrait should rely on system swipe-dismiss).

**Cause:** Forgetting the `if showCloseButton` guard on `ToolbarItem`.

**Fix:**
```swift
// ❌ Always shows — wrong
.toolbar {
    ToolbarItem(placement: .cancellationAction) {
        Button { dismiss() } label: { Image(systemName: "xmark") }
    }
}

// ✅ Only shows in landscape — correct
.toolbar {
    if showCloseButton {
        ToolbarItem(placement: .cancellationAction) {
            Button { onClose?() } label: { Image(systemName: "xmark") }
        }
    }
}
```

### Pitfall 2: Double NavigationStack wrapping

**Symptom:** View already has its own `NavigationStack` (like AIModelConfigSheet, CardDrawView). Wrapping it in `morePane { }` creates nested NavigationStacks — broken layout, double nav bars.

**Cause:** `morePane` adds its own `NavigationStack`. If the child view already has one, they stack.

**Fix:** For views with their own NavigationStack, render them directly in the switch and pass `showCloseButton`/`onClose` params:
```swift
// ❌ Double NavigationStack
case .myView:
    morePane { MyViewWithOwnNavStack() }

// ✅ Direct render, pass params
case .myView:
    MyViewWithOwnNavStack(
        showCloseButton: showCloseButton,
        onClose: { dismiss(); selection = .none }
    )
```

Use `morePane { }` only for views WITHOUT their own NavigationStack (simple List views, static content).

### Pitfall 3: Missing `body` closing brace after switch

**Symptom:** Compiler errors like "Attribute 'private' can only be used in a non-local scope" on struct members that should be at struct level.

**Cause:** The switch statement's closing `}` is the last thing in `body`. If you add a closing `}` after the switch, you've actually closed `body`. But if your edit accidentally removes that `}`, all subsequent struct members are treated as local declarations inside `body`.

**Fix:** After the switch's `}`, you need another `}` to close `body`:
```swift
var body: some View {
    switch selection {
    case .none: emptyPane
    // ... cases ...
    }        // ← closes switch
}            // ← closes body ← DON'T FORGET THIS

// Struct-level members below
private var emptyPane: some View { ... }
```

### Pitfall 4: `@Binding var isPresented` incompatible with SplitDetailPane

**Symptom:** View uses `@Binding var isPresented: Bool` for dismissal (common sheet pattern). Can't close from SplitDetailPane because there's no binding to set.

**Cause:** The `isPresented` pattern works when the parent owns a `@State` and passes a `$binding`. SplitDetailPane doesn't own such state — it uses `selection = .none` to dismiss.

**Fix:** Replace `@Binding var isPresented: Bool` with `var onDismiss: () -> Void`:
```swift
// ❌ Binding — can't close from SplitDetailPane
struct OnboardingView: View {
    @Binding var isPresented: Bool
    // ... isPresented = false
}

// ✅ Callback — works from any context
struct OnboardingView: View {
    var onDismiss: () -> Void
    // ... onDismiss()
}
```

Call sites:
```swift
// From AdaptiveRootView (local sheet)
OnboardingView(onDismiss: { showOnboarding = false })

// From SplitDetailPane (sidebar/sheet)
OnboardingView(onDismiss: { dismiss(); selection = .none })
```

### Pitfall 5: NavigationLink bypasses the sidebar

**Symptom:** Content appears in a narrow left sidebar (35% width) instead of the right detail pane in landscape.

**Cause:** `NavigationLink` pushes onto the local `NavigationStack` which lives inside `ContentView` (the left 35% panel). The pushed view never enters `SplitDetailPane`.

**Fix:** Replace every `NavigationLink` with a `Button` that sets `detailSelection`:
```swift
// ❌ Stays in left sidebar
NavigationLink { TarotCatalogView(...) } label: { Label("图鉴", ...) }

// ✅ Routes through SplitDetailPane
Button { detailSelection = .tarotCatalog } label: { Label("图鉴", ...) }
```

### Pitfall 6: Choosing the wrong close button style

**Symptom:** Using a floating Circle+material close button overlay that looks non-native and has z-index conflicts with scroll views.

**User feedback:** Prefer native toolbar xmark — looks like it belongs, zero z-index issues, consistent position across all views.

**Decision:** Use `NavigationStack` + `.toolbar { ToolbarItem(placement: .cancellationAction) { xmark } }` for all close buttons. The floating overlay pattern (`closeButton` with Circle+ultraThinMaterial) is documented as an alternative but the toolbar pattern is the primary recommendation.

### Pitfall 7: Fixed-width content overflows in sidebar

**Symptom:** Content looks fine in portrait (full screen) but overflows or clips in landscape sidebar (35% width). Typically happens with horizontal layouts, number pickers, button rows.

**Cause:** Sidebar is only ~35% of screen width (min 360pt on iPad). Fixed-width elements (`frame(width: 32)`) × N items + fixed spacing easily exceed available space.

**Fix:** Always test content at narrow widths. Use responsive layouts:
```swift
// ❌ Fixed-width HStack — overflows in sidebar
HStack(spacing: 12) {
    ForEach(1...10, id: \.self) { n in
        Text("\(n)").frame(width: 32, height: 44)
    }
}

// ✅ LazyVGrid with flexible columns — adapts to any width
LazyVGrid(
    columns: Array(repeating: GridItem(.flexible(), spacing: 10), count: 5),
    spacing: 10
) {
    ForEach(1...10, id: \.self) { n in
        Text("\(n)")
            .frame(maxWidth: .infinity)
            .frame(height: 44)
    }
}
```

**Rule of thumb:** Sidebar width ≈ `max(360, screenWidth × 0.35)`. On a 1024pt iPad landscape, that's ~360pt. Design for that minimum. Use `.frame(maxWidth: .infinity)`, `LazyVGrid`, `ScrollView(.horizontal)`, or `ViewThatFits` instead of fixed-width horizontal stacks.

### Checklist: Adding a new view to the sidebar

When adding a new screen that should show in the right pane:

1. Add case to `DetailSelection` enum, set `needsSheet: true`
2. Add `case .yourCase:` in `SplitDetailPane` switch
3. Does the child view have its own `NavigationStack`?
   - **Yes** → Render directly, pass `showCloseButton`/`onClose` params
   - **No** → Wrap with `morePane { }`
4. Change trigger in sidebar from `NavigationLink` to `Button { detailSelection = .yourCase }`
5. If the view used `@Binding var isPresented`, change to `var onDismiss: () -> Void`
6. **Always** guard toolbar close buttons with `if showCloseButton { }`
7. **Test at narrow width** — sidebar is ~360pt. Fixed-width rows overflow. Use `LazyVGrid`/`.frame(maxWidth: .infinity)`.
8. Build, test both orientations, verify close button only appears in landscape

## Related

- [Apple HIG: Layout for iPad](https://developer.apple.com/design/human-interface-guidelines/layout)
- `horizontalSizeClass` docs: `EnvironmentValues.horizontalSizeClass`
- `onGeometryChange`: iOS 18+ API for reading view geometry without GeometryReader
