---
name: xiaoyu-tabview
description: "将传统 TabView（3 个标签 + 右上角加号/搜索）改造为 iOS 26 新版精美标签栏（独立搜索标签 + 底部长条操作按钮 + 底部搜索）的决策原则与实现要点。适用于：用户要求“把 TabView 改成新版/精美样式/底部操作按钮/底部搜索”、设计长条按钮或搜索行为、处理 iOS 26/27 搜索框首次激活跑到顶部的位置 bug、决定 iPad 上标签栏与搜索的表现。鱼律已决定一律新版、不再保留传统版开关。Migrate a legacy TabView to the iOS 26 beautiful tab bar (independent search tab + bottom action button + bottom search): decision principles, per-page behavior mapping, and known iOS 26/27 search-placement pitfalls."
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
- **不再保留传统版开关**：鱼律已决定“以后一律用新版”，删除 `showFab` 开关与旧 3 标签 TabView；旧版兼容代码（`legacyTabBar`、`horizontalSizeClass == .regular || !showFab` 之类的条件、右上角重复按钮的旧逻辑）一并移除。若未来要加回传统版，需同时恢复：旧 TabView、右上角 +/搜索按钮、旧按钮在详情/表单页的显隐条件，并处理横屏副屏下旧式 push 的钻取路由（旧式在横屏 + 更多页钻取存在路由缺口，见下文）。
- iPhone 与 iPad 一致用精美版：左栏内容套 `.compact` 强制底部标签栏（鱼律当前做法，以代码为准）；若某 App 希望 iPad 保持原生标签栏，取舍见“已知系统坑”。

## 原则一：长条按钮 = 原页面右上角的主操作

- 把每个页面右上角的按钮操作（+ 新建、编辑、保存、分享、查询）原样搬到底部一整条按钮或菜单，不写新功能。
- **右上角是否保留，由产品/用户自由选择**（鱼律 2026-08 起选择“都保留”，两条路线都可用）：
  - 方案 A（只留底部）：右上角隐藏主操作，仅由底部长条按钮承担；非主操作图标（如删除）可留在右上角。
  - 方案 B（都保留，鱼律现行）：右上角与底部长条按钮同时显示、调用同一个动作，互为兜底——竖屏底部单手可及；横屏副屏时表单在右屏、底部按钮在左屏，右上角是必要兜底；键盘弹起时底部可能被遮挡，右上角始终可用。
  - **选择后要全局一致**：要么全部隐藏、要么全部显示，避免“有的页面有、有的没有”的错位。
- 列表页 → “新建 X”（多个新建用 `Menu` 弹出，选项顺序与旧版一致）；详情页 → “编辑 / 关联 / 分享”；表单页 → “保存 / 执行查询”；没有主操作的页 → 装饰或高频入口。
- 搜索态时，长条按钮保留用户进入搜索前那个页面的按钮（不消失）。
- **灵活接管规则**：先看当前页右上角有没有功能——有，则长条按钮 = 该功能；右上角没有功能，再看左上角是什么（一般是返回），长条按钮 = 返回。
- **长条按钮只在“当前页面已注册动作（`pad.saveEditAction`）且可见”时显示**，标题与动作成对出现——不能用 `isEditSelection` 单独判断，否则会出现“标题残留、动作已清空”的死按钮（显示保存/添加委托人却点不动）。
- **动作生命周期由来源页确定性恢复**：子页面 `onAppear` 注册自己的动作（`saveEditAction` + `saveEditTitle`），来源页在子页关闭时用 `onChange` 确定性恢复/清理（不能只靠 onAppear，时序不可靠）；子页 `onDisappear` 只在“来源页不会恢复”的流程里自行清空（如钻取栈兜底“返回”），避免父子互相清空导致按钮残留或失效。
- **推入的选择/子页面也要注册长条按钮（如文书文件选择页、案件选择页）**：凡是推入覆盖父页的页面，都必须 `onAppear` 注册自己的主操作（`pad.saveEditAction`/标题，如“下载”）、`onDisappear` 清空；否则长条按钮会残留父页的按钮（如列表页的“新增文书下载”），出现“右上角是新功能、底部却是父页按钮”的错位。带数量/禁用态的选择按钮（如“下载 (2)”）右上角保留，长条按钮复用同一动作并内部判空。
- **操作完成后的回退**：点长条按钮执行右上角功能（如“保存”）后，页面会关闭并返回上一页；此时长条按钮必须**跟随返回后的页面**——上一页有右上角功能就显示它，否则显示“返回”（左上角）。实现上靠页面 `onDisappear` 清掉已注册的保存动作（`pad.saveEditAction = nil`），否则长条按钮会一直卡在“保存”。校验每个带保存的表单页：保存后必须回退到上一页且按钮状态复原。
- **嵌套表单页的关闭清理（如 新建案件→添加委托人、案件表单→参与人表单）**：被推入的子表单页（新增/编辑委托人、参与人表单等）`onAppear` 注册 `pad.saveEditAction` 后，`onDisappear` 必须成对清理——但**只有“真正关闭”才清**（旋转/容器重建/被子页覆盖时不清，靠来源页的全局打开状态如 `addClientOpen`/`pickerAddClientOpen`/`partyEditContext` 判别：全局仍开着 = 存档草稿；全局已清 = 清 `pad.saveEditAction = nil`）。来源页（选择委托人）也要在 `onChange(showingAddClient==false)` 里**恢复自己的长条动作**（如“添加委托人”），在自身 `onDisappear` 里**清空**（整体关闭回列表时按钮必须还原为列表页按钮）；新建/编辑案件这类以 `detailSelection` 驱动的表单页可依赖根视图兜底：全局 `detailSelection` 变化离开表单/新增类页面（含副屏 X 整体关闭且子表单还在栈顶的情况）时，在根视图的 `onChange(of: detailSelection)` 里兜底清一次 `saveEditAction = nil`，防止任何关闭路径残留“保存”。
- **跨标签跟随**：长条按钮跟随“当前正在看的页面”，不跟标签。例如从待办/更多进入的案件详情，长条按钮应显示该详情的操作（编辑案件/添加时间线/关联文件），而不是待办的“新建待办”或更多的“新增委托人”；详情关闭后恢复标签页自己的按钮。
- 判定用「该页是否注册了主操作」（如 `pad.saveEditAction != nil`）统一驱动，不只局限详情编辑页；更多钻取内打开的编辑页（编辑个人信息等）也要注册保存动作，底部长条显示“保存”而不是“返回”。
- **钻取页不要自带 NavigationStack**（编辑个人信息 / 云同步诊断 / 隐私政策等已去除自身栈）：返回统一由外层提供（竖屏路径导航 / 横屏 MoreDrillPageView 按 `pageHasOwnNavigationStack == false` 自供返回按钮），页面只渲染内容；云同步诊断等页此前因嵌套栈出现“一点击自动返回 + 返回崩溃”。
- **保存关闭的经典竞态（“保存后又弹出来”）**：更多钻取内带保存的编辑页，关闭时必须**先弹出钻取栈（`pad.moreDrillBack()`）再 `dismiss()`**。否则 `dismiss()` 先把外层推入弹掉，而钻取栈要到下一次状态更新才弹出，返回瞬间更多页 `onAppear` 看到栈仍非空，会把编辑页重新推回来。只 `dismiss()` 不弹栈 = 大概率复现。**路径导航下**：弹出钻取栈后路径会自动同步重建，`dismiss()` 冗余但无害，可省略。
- **嵌套 NavigationStack 破坏 dismiss（“保存后返回错乱”）**：竖屏 push 的表单页（时间线节点 / 待办 / 案件节点的新增与编辑）绝不能自带 `NavigationStack`——被推入外层导航栈时再套一层内嵌栈会让 `dismiss()` 行为不可靠（返回错位/不返回）。正确写法：`presentedAsSheet == true` 时才包 `NavigationStack`，推入模式直接渲染 `Form`（导航栏由外层承担，与编辑案件页一致）。同时这些表单必须在 `onDisappear` 里清掉 `pad.saveEditAction = nil`，否则底部按钮卡在“保存”。
- **竖屏钻取页导航的正确架构（路径导航，已验证）**：更多页竖屏导航必须用 `NavigationStack(path:)` + 统一路由（`enum MoreDrillRoute { case page(MoreDrillPage), detail(DetailSelection) }`）。**不要用多个 item 绑定**（多层推入会“替换”层级，左滑直接跳层）、**不要嵌套 NavigationStack**（双返回按钮）。钻取页 = `.page` 元素、子详情 = `.detail` 元素，全部走同一个 path；`Binding<DetailSelection?>` 用桥接 binding 映射到 path 追加/弹出 `.detail`；旋转恢复、嵌套 push（详情内编辑）、外部关闭详情分别用 `onChange(of: pad.isSidebar / pendingPushToken / detailSelection)` 同步。左滑返回逐级正确且每层只有一个返回按钮。

## 原则二：搜索 = 搜当前页面正在展示的数据

- 每个页面出现时登记自己的“搜索上下文”；点搜索标签时，稳定容器按上下文切换内容，提示文字（prompt）跟着变。
- 列表 / 详情页有内容可搜就搜自己；**纯表单页（新建 / 编辑 / 设置操作表单）点击搜索无反应**，不要硬搜无关内容（如“表单页搜案件列表”属于错误回退）。
- 所有页面共享同一个搜索文本绑定；过滤逻辑留在各页面内。
- 点搜索结果后自动退出搜索，返回原主标签。
- **不适合搜索的页面 → 点击搜索无反应**：统一在根视图判断「当前页面是否适合搜索」（`isCurrentPageSearchable`）——编辑/新建表单（`pad.isEditSelection || pad.saveEditAction != nil || pad.drillFormTitle != nil`）、更多钻取内的纯工具/设置子页（云同步诊断、备份恢复、偏好设置、模板编辑、限高查询等）都不适合，选中搜索标签时直接重定向回原标签，不展开搜索框。只有列表页（案件/待办/委托人/文书）、案件详情、更多首页可搜。限高查询等查询页**绝不能**把上下文注册成别的列表（曾犯过注册成 `.cases` 的错）。

## 核心代码（已实测可用的最小骨架）

### 1. 单一稳定搜索容器 + 上下文状态

```swift
// 搜索上下文：每页出现时登记“搜什么”，搜索文本全局共享
@Observable final class SearchContextState {
    var current: PageContext = .list      // 当前页面上下文（决定搜索内容与提示）
    var searchText = ""                   // 唯一搜索框的共享文本
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

页面登记示例（`onAppear` 时调用）：列表页 `.list`、详情页 `.detail`、子列表 `.subList`、更多入口 `.settings`；纯表单页不登记搜索（点击搜索无反应）。

### 2. 点搜索“只展开、不弹键盘”（键盘抑制窗口）

`.searchTabSelection` 的语义是“选中搜索标签即激活搜索框”，系统会在选中瞬间自动聚焦并弹键盘。App 层关不掉（见“已知系统坑”），但可以在**选中后的短暂窗口内把弹出动作压回去**：

```swift
// SearchContextState
var suppressKeyboardUntil = Date.distantPast
func armSearchKeyboardSuppression() { suppressKeyboardUntil = Date().addingTimeInterval(0.8) }
func shouldSuppressKeyboard() -> Bool { Date() < suppressKeyboardUntil }

// ContentView：选中搜索标签（iPhone 竖屏）时启用抑制窗口
.onChange(of: selectedTab) { _, newValue in
    if newValue == 3, UIDevice.current.userInterfaceIdiom == .phone {
        searchContext.armSearchKeyboardSuppression()
    }
}

// 根视图（能收到键盘通知的地方）：窗口内把键盘弹起动作取消
.onReceive(NotificationCenter.default.publisher(for: UIResponder.keyboardWillShowNotification)) { _ in
    if searchContext.shouldSuppressKeyboard() {
        UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder),
                                        to: nil, from: nil, for: nil)
        return
    }
    // 正常键盘处理…
}
```

效果：点搜索 → 搜索框向左展开、不弹键盘；0.8s 窗口过后用户点进输入框，键盘正常弹出。

### 3. 启动预热（旧方案，仅 iOS 26 仍复现时才需要）

实测结论（iOS 26/27 + 单一稳定容器）：**不再需要启动预热**。旧方案是在启动时“选中搜索标签 → 切走 → 再选中”来等效“第二次点击”，但每次选中都会触发搜索框展开动画 + 键盘激活尝试，导致开屏闪烁（启动 Logo / 遮罩都挡不干净，键盘在系统窗口层）。鱼律已在 iOS 27 实测移除预热后首次点搜索位置正常。

若目标设备仍复现“首次激活跑顶部”且必须保留预热：
- 必须用**独立 UIWindow（`windowLevel = .alert + 1`）**做全屏静态遮罩盖住预热动画（普通 `.overlay` 层级不够，动画会透出来）；键盘弹起属于系统窗口，遮罩挡不住，仍需配合上面的键盘抑制窗口。
- 预热期间 `onChange(of: selectedTab)` 里禁止任何请求焦点逻辑（否则弹键盘）。
- 仅 iPhone 需要；iPad 走原生可跳过。

## 与 iPad 双模式副屏联动（可选增强）

当 App 在横屏（iPhone / iPad 一致）采用“主屏 TabView + 右侧副屏详情”双模式时，精美 TabView 的控件要与副屏联动，否则控件（搜索、长条按钮）和内容会“分家”。副屏开关不要限制设备（`showSidebar = isLandscape`，不加 `userInterfaceIdiom == .pad`，否则 iPhone 横屏只拉伸不出副屏）。要点（均为通用做法）：

- **焦点跟随**：共享状态记录当前焦点在主屏还是副屏；长条按钮显示焦点所在侧的内容（副屏有详情 → 显示详情操作；否则显示主屏按钮）。打开详情时把焦点切到副屏；滚动主屏内容区时切回主屏。**手势必须用 `DragGesture(minimumDistance: 4).onChanged` 且只在值变化时赋值**（`if pad.focusedPane != .main { pad.focusedPane = .main }`）——`minimumDistance: 0` 会在按下瞬间触发状态变更，把整页点击（列表行 + 系统返回按钮）都取消掉（iOS 26 / iPadOS 27 / Mac 实测整页点不动的根因）。只挂内容区、不挂标签栏，避免点搜索按钮时抢焦点。
- **搜索进副屏**：横屏点搜索 → 搜索改在右侧副屏呈现（用系统原生 `.searchable`，自带灵动动画），左侧保持当前主标签；焦点在副屏详情 → 搜该详情内容，否则搜主屏当前页。切换主标签时退出搜索。
- **搜索结果联动**：点搜索结果 → 关闭搜索并把详情同步到全局状态（旋转不丢）。
- **紧凑模式分区**：详情页内容加 `.environment(\.horizontalSizeClass, .compact)` 用于 iPad 标签栏压到底部；**若选择“都保留”，不要再用 `horizontalSizeClass == .regular` 当右上角显隐条件**（右上角统一显示，与底部按钮互为兜底）；**搜索界面本身保持宽屏**（否则原生搜索框会消失）；搜索时只保留一个关闭 X。
- **副屏内容强制重建**：详情视图把对象 / 子标签存在 `@State` 时，切换对象要用 `.id(...)` 强制重建，否则 SwiftUI 复用同一视图、显示旧内容。
- **导航目标挂栈内**：`navigationDestination` 必须挂在 `NavigationStack` 内部，否则“状态已更新但推不进去”。
- **旋转恢复**：每个主标签都要注册恢复联动（记录来源标签），否则从该标签进入的详情旋转后丢失。

## 已知系统坑（iOS 26/27，务必规避）

1. **首次激活搜索框被放到页面顶部**（iPhone）：根因是搜索框注册时机与标签栏搜索容器初始化竞态；每次“第一次进搜索”都可能复发，第二次点才正常。
   - **首选规避 A：单一稳定 `.searchable`（挂在固定容器上，内容在内部切换），已实测足以解决，无需预热**。鱼律在 iOS 27 上移除预热后冷启动首次点搜索位置正常。
   - 规避 B（启动预热）仅在 A 不够、bug 仍复现时启用；启用会带来开屏闪烁（搜索标签展开动画 + 键盘激活尝试），需配最高层级 UIWindow 遮罩 + 键盘抑制窗口（见上文）。
2. **选中搜索标签必然触发键盘激活，App 层关不掉**：`.searchTabSelection` 官方语义就是“选中即激活”。UIKit 的 `UISearchTab.automaticallyActivatesSearch = false` 虽能“只展开不聚焦”，但 SwiftUI 会在每次刷新时把它改回 `true`（实测日志铁证），因此该开关对 SwiftUI TabView 无效。想要“展开不弹键盘”，只能靠选中后的短窗口键盘抑制（见上文）。
3. **iPad 上搜索框只能在右上角**：搜索标签的底部变形（morph）是 iPhone 运行时特性；iPadOS 26 设计就是顶部/右上角，没有公开 API 强制到底部。
   - `.tabViewStyle(.tabBarOnly)` 在 iPad 上不强制底部（实测无效）。
   - `.environment(\.horizontalSizeClass, .compact)` 能把标签栏压到底部，但窗口拉伸时内容会变形，且破坏 iPad 自由缩放。
   - **鱼律的实际做法（以代码为准）**：左栏内容一律 `.environment(\.horizontalSizeClass, .compact)`——iPad（含无副屏竖屏）也用 iPhone 式底部标签栏，与 iPhone 完全一致；横屏副屏时标签栏在左栏底部，“搜索进副屏 + 焦点跟随”联动（见上一节）。
   - 若某 App 不想要底部标签栏、希望 iPad 保持系统原生（自适应标签栏 + 右上角搜索），则不要对左栏套 compact；两种方案按产品需求取舍。
4. **List 行按钮点击被“焦点跟随”手势吞掉（iOS 26）**：内容区挂的 `DragGesture(minimumDistance: 0)`（焦点跟随）会让部分设备（iOS 26，iPhone Air 的 iOS 27 正常）上 List 默认样式的行按钮点击失效（更多页、备份恢复页整页点不动）。案件列表能点是因为行按钮用了 `.buttonStyle(.plain)`。
   - **根因与根治**：手势 `minimumDistance: 0` 在按下瞬间触发状态变更（`focusedPane` 赋值 → 视图重绘）→ 取消整页点击（不止列表行，**系统返回按钮也会点不动**；iPadOS 27 / Mac / iOS 26 均实测）。**根治：手势改为 `minimumDistance: 4` 且仅在值变化时赋值**，点按不再触发、滚动仍触发焦点跟随。
   - **TapGesture 焦点跟随是更大的坑（鱼律 2026-08-07 实测定论）**：在 Tab 根视图 / 副屏挂“点击任意处 → 切换 `focusedPane`”的 `TapGesture().onEnded { pad.focusedPane = .main/.sidebar }`，会让**所有 push 进导航栈的页面**里的原生控件（Form/List 的 Picker 行、SwiftUI Button 行：日期/电话/公司搜索/添加参与人等）点击事件被吞——点击有按压反馈但动作不执行，文本框（按下即聚焦）正常，根页面（列表/详情/更多首页）正常。构建 530 无此手势全部正常，588 加入后案件/委托人/参与人表单全部点不动。
     - **根治（已验证）**：不要用 `TapGesture` 做焦点跟随，**直接移除**；需要“点击切换焦点”时只用 `DragGesture(minimumDistance: 4)`（值变化才赋值）。`DispatchQueue.main.async` 延后状态变更实测仍不稳，别依赖。移除后所有表单控件恢复正常。
     - **不要用“Picker 改 Menu”绕**：只绕开选择器，日期/电话/公司搜索仍坏，且偏离原生行为；根因手势移除后原生 Picker 直接可用。
     - 判据：根页面正常、push 页面整页控件（选择器/按钮）点不动、文本框正常 = 根视图 TapGesture 焦点手势吞点击。
   - 兜底：行按钮用**自定义纯 SwiftUI 样式**（`contentShape(Rectangle())` 保证整行可点 + 按压反馈），不要用 List 默认行按钮样式；应用在更多页及其钻取子页（委托人管理 / 委托人案件 / 文书下载 / 备份恢复等）的 List 上。
   - 判据：整页（含返回按钮）点不动 = 手势吞点击；仅文字可点 = 行按钮样式问题。

## 工作流

1. 一律新版，**不做**新旧版开关共存（鱼律已摒弃传统版，`showFab`/`legacyTabBar` 已删除）。
2. 逐个页面：先看右上角主操作 → 映射为长条按钮；没有右上角再看左上角（返回）；右上角显隐按产品选择全局一致（隐藏或都保留）；确定搜索上下文（列表页搜自己、表单页点击无反应）。
3. 实现稳定搜索容器 + 各页面上下文登记（`onAppear` 时设置）。
4. 无需启动预热（单一稳定容器已解决首次激活位置 bug）；iPad 与 iPhone 一致（左栏 compact 底部标签栏）。
5. 真机验证：每个页面**第一次**点搜索都必须在底部展开；长条按钮功能与原右上角一致，保存后按钮状态复原；每个页面左上角只有一个返回按钮。

遇到无法确认页面形式的页面：先把该页面及其候选行为列成清单交给开发者决定，不要擅自猜测。

## 参考

- 通用页面类型映射与兜底决策流程：见 [references/example-mapping.md](references/example-mapping.md)。
- 相关技能：iPad 自适应双模式布局（竖屏 sheet/推入、横屏侧栏）→ `小鱼平板适配forOS26`（~/.agents/skills/小鱼平板适配forOS26）
- **许可证与署名**：本技能采用 Apache License 2.0；使用/调用/改编本技能必须保留版权声明并注明来源（作者 zy5120，仓库 https://github.com/zy5120/xiaoyu-tabview ）。与本技能相互配合的 `小鱼平板适配forOS26` 为同一作者、同一协议。
