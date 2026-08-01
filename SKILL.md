---
name: xiaoyu-tabview
description: "将传统 TabView（3 个标签 + 右上角加号/搜索）改造为 iOS 26 新版精美标签栏（独立搜索标签 + 底部长条操作按钮 + 底部搜索）的决策原则与实现要点。适用于：用户要求“把 TabView 改成新版/精美样式/底部操作按钮/底部搜索”、设计长条按钮或搜索行为、处理 iOS 26/27 搜索框首次激活跑到顶部的位置 bug、决定 iPad 上标签栏与搜索的表现、以及新旧版开关共存。Migrate a legacy TabView to the iOS 26 beautiful tab bar (independent search tab + bottom action button + bottom search): decision principles, per-page behavior mapping, and known iOS 26/27 search-placement pitfalls."
---

# iOS 26 精美 TabView 改造

## 总览

把一个传统 TabView（每个标签自带右上角 + / 搜索）改造成 iOS 26 新形态：4 个标签（3 个主标签 + 最右侧独立搜索标签）、底部一整条操作按钮（`tabViewBottomAccessory`）、搜索框在底部展开。核心是两条决策原则加一组已知系统坑。

## 目标结构（改造后）

```swift
TabView(selection: $selectedTab) {
    Tab("列表", systemImage: "folder", value: 0) { MainListPage() }
    Tab("待办", systemImage: "checklist", value: 1) { TodosPage() }
    Tab("更多", systemImage: "ellipsis.circle", value: 2) { MorePage() }
    Tab(value: 3, role: .search) { searchContent }   // 独立搜索标签，固定在最右侧
}
.tabViewSearchActivation(.searchTabSelection)   // 搜索标签独立：点按后标签栏收起、搜索框从右侧向左展开
.tabBarMinimizeBehavior(.onScrollDown)          // 下滚时标签栏收成单行
.tabViewBottomAccessory { bottomAction }        // 底部“长条按钮”
```

要点：
- 搜索标签用**一个稳定容器 + 唯一一个 `.searchable`**；切换页面只换容器内部内容，绝不重建搜索框（`id()` 重建会反复触发系统位置 bug）。
- 用开关（如 `@AppStorage("showFab")`）在“传统版 / 新版”之间切换：关闭时完全还原传统 3 标签 + 右上角按钮。
- iPhone 与 iPad 分开对待：iPhone 用精美版；iPad 保持系统原生（见“已知系统坑”）。

## 原则一：长条按钮 = 原页面右上角的主操作

- 把每个页面右上角的按钮操作（+ 新建、编辑、保存、分享、查询）原样搬到底部一整条按钮或菜单，不写新功能。
- 新版开启时，右上角**不再重复显示**这些按钮；非主操作图标（如删除）可保留在右上角。
- 列表页 → “新建 X”（多个新建用 `Menu` 弹出，选项顺序与旧版一致）；详情页 → “编辑 / 关联 / 分享”；表单页 → “保存 / 执行查询”；没有主操作的页 → 装饰或高频入口。
- 搜索态时，长条按钮保留用户进入搜索前那个页面的按钮（不消失）。

## 原则二：搜索 = 搜当前页面正在展示的数据

- 每个页面出现时登记自己的“搜索上下文”；点搜索标签时，稳定容器按上下文切换内容，提示文字（prompt）跟着变。
- 列表 / 详情页有内容可搜就搜自己；**纯表单页（新建 / 编辑 / 设置操作表单）隐藏搜索按钮**，不要硬搜无关内容（如“表单页搜案件列表”属于错误回退）。
- 所有页面共享同一个搜索文本绑定；过滤逻辑留在各页面内。
- 点搜索结果后自动退出搜索，返回原主标签。

## 核心代码（已实测可用的最小骨架）

### 1. 单一稳定搜索容器 + 上下文状态

```swift
// 搜索上下文：每页出现时登记“搜什么”，搜索文本全局共享
@Observable final class SearchContextState {
    var current: PageContext = .list      // 当前页面上下文（决定搜索内容与提示）
    var searchText = ""                   // 唯一搜索框的共享文本
    var focusToken = 0                    // 进入搜索标签时 +1，触发底部框聚焦
    var closeToken = 0                    // 搜索结果点击后返回原主标签
}

// 搜索标签内容：搜索框只注册一次；切换页面只换内部内容，绝不重建外层
Tab(value: 3, role: .search) {
    NavigationStack {
        pageContentForCurrentContext      // 按 searchContext.current 切换的内容
    }
    .searchable(text: $searchContext.searchText, prompt: promptForCurrentContext)
}
.tabViewSearchActivation(.searchTabSelection)
```

页面登记示例（`onAppear` 时调用）：列表页 `.list`、详情页 `.detail`、子列表 `.subList`、更多入口 `.settings`；纯表单页不登记搜索（隐藏搜索按钮）。

### 2. 启动预热（规避 iOS 26/27 首次激活搜索框跑顶部）

```swift
// 启动时隐藏整个界面，把搜索标签“选中→切走→再选中”，等效系统第二次点击，再显示默认页
private func warmupSearchTab(returnTo target: Int) {
    isWarmingUp = true
    revealContent = false
    selectedTab = 3
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
        selectedTab = target
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
            selectedTab = 3
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
                selectedTab = target
                isWarmingUp = false
                revealContent = true
            }
        }
    }
}
```

要点：预热期间 `onChange(of: selectedTab)` 里禁止 `requestFocus()`（否则弹键盘）；预热仅 iPhone 需要，iPad 走原生可跳过。

## 与 iPad 双模式副屏联动（可选增强）

当 App 在横屏（iPhone / iPad 一致）采用“主屏 TabView + 右侧副屏详情”双模式时，精美 TabView 的控件要与副屏联动，否则控件（搜索、长条按钮）和内容会“分家”。副屏开关不要限制设备（`showSidebar = isLandscape`，不加 `userInterfaceIdiom == .pad`，否则 iPhone 横屏只拉伸不出副屏）。要点（均为通用做法）：

- **焦点跟随**：共享状态记录当前焦点在主屏还是副屏；长条按钮显示焦点所在侧的内容（副屏有详情 → 显示详情操作；否则显示主屏按钮）。打开详情时把焦点切到副屏；触摸/滚动主屏内容区时切回主屏（手势用 `DragGesture(minimumDistance: 0).onChanged`，`onEnded` 不灵敏；只挂内容区、不挂标签栏，避免点搜索按钮时抢焦点）。
- **搜索进副屏**：横屏点搜索 → 搜索改在右侧副屏呈现（用系统原生 `.searchable`，自带灵动动画），左侧保持当前主标签；焦点在副屏详情 → 搜该详情内容，否则搜主屏当前页。切换主标签时退出搜索。
- **搜索结果联动**：点搜索结果 → 关闭搜索并把详情同步到全局状态（旋转不丢）。
- **紧凑模式分区**：详情页内容加 `.environment(\.horizontalSizeClass, .compact)` 隐藏右上角与长条按钮重复的旧按钮；**搜索界面本身保持宽屏**（否则原生搜索框会消失）；搜索内容区可单独压缩，隐藏残留按钮；搜索时只保留一个关闭 X。
- **副屏内容强制重建**：详情视图把对象 / 子标签存在 `@State` 时，切换对象要用 `.id(...)` 强制重建，否则 SwiftUI 复用同一视图、显示旧内容。
- **导航目标挂栈内**：`navigationDestination` 必须挂在 `NavigationStack` 内部，否则“状态已更新但推不进去”。
- **旋转恢复**：每个主标签都要注册恢复联动（记录来源标签），否则从该标签进入的详情旋转后丢失。

## 已知系统坑（iOS 26/27，务必规避）

1. **首次激活搜索框被放到页面顶部**（iPhone）：根因是搜索框注册时机与标签栏搜索容器初始化竞态；每次“第一次进搜索”都可能复发，第二次点才正常。
   - 规避 A：单一稳定 `.searchable`（挂在固定容器上，内容在内部切换）。
   - 规避 B：启动预热——先隐藏整个界面，`selectedTab = 3`，等 ~0.2s 切回默认页，再等 ~0.2s 再次选中搜索标签，最后切回默认页并显示。等效“第二次点击”，让注册落点正确；预热期间禁止请求焦点（否则弹键盘）。
   - 预热只对 iPhone 需要；iPad 走原生，跳过预热以保持秒开。
2. **iPad 上搜索框只能在右上角**：搜索标签的底部变形（morph）是 iPhone 运行时特性；iPadOS 26 设计就是顶部/右上角，没有公开 API 强制到底部。
   - `.tabViewStyle(.tabBarOnly)` 在 iPad 上不强制底部（实测无效）。
   - `.environment(\.horizontalSizeClass, .compact)` 能把标签栏压到底部，但窗口拉伸时内容会变形，且破坏 iPad 自由缩放。
   - **无副屏时：iPad 保持系统原生**（自适应标签栏 + 右上角搜索）；除非用户明确要“跑 iPhone 软件”（iPhone 专用 App，窗口固定不可缩放）。
   - **有横屏副屏时**：不硬把底部搜索塞进 iPad，改用“搜索进副屏 + 焦点跟随”联动（见上一节）。

## 工作流

1. 确认是否需要新旧版开关共存（建议默认新版、可回退）。
2. 逐个页面：找到右上角主操作 → 映射为长条按钮；确定搜索上下文（列表页搜自己、表单页隐藏）。
3. 实现稳定搜索容器 + 各页面上下文登记（`onAppear` 时设置）。
4. iPhone 加启动预热；iPad 保持原生。
5. 真机验证：每个页面**第一次**点搜索都必须在底部展开；长条按钮功能与原右上角一致；开关切回传统版完全还原。

遇到无法确认页面形式的页面：先把该页面及其候选行为列成清单交给开发者决定，不要擅自猜测。

## 参考

- 通用页面类型映射与兜底决策流程：见 [references/example-mapping.md](references/example-mapping.md)。
- 相关技能：iPad 自适应双模式布局（竖屏 sheet/推入、横屏侧栏）→ `小鱼平板适配forOS26`（~/.agents/skills/小鱼平板适配forOS26）
