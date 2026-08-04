# 小鱼 TabView（xiaoyu-tabview）

将传统 TabView（3 个标签 + 右上角加号/搜索）改造为 iOS 26 新版精美标签栏（独立搜索标签 + 底部长条操作按钮 + 底部搜索）的决策原则与实现要点：长条按钮接管规则、搜索上下文、iOS 26/27 已知系统坑、竖屏钻取路径导航等。

> 本技能来自「鱼律 / yulawyer」iOS App（iOS 26+）实战沉淀。

## 配套使用

本技能与 **[小鱼平板适配forOS26](https://github.com/zy5120/ipad-adaptive-layout-ios26)**（iPad/横屏双模式自适应）**相互配合**：

1. 先按「小鱼平板」技能适配页面的横竖屏双模式（竖屏 sheet/推入、横屏副屏）；
2. 再按本技能适配精美 TabView 的底部长条按钮与搜索（长条按钮跟随焦点、搜索进副屏等）。

两个技能各管一层：一个管布局与旋转，一个管底部控件与搜索。适配任何页面时按此顺序执行，可同时验证。

## 许可证

本项目采用 **Apache License 2.0**。

**署名要求**：任何使用、调用、复制或改编本技能（含将其作为开发规范、用于生成代码/内容、或整合进其他技能/项目）的，必须保留本版权声明，并注明来源：

- 仓库：<https://github.com/zy5120/xiaoyu-tabview>
- 作者：zy5120（鱼律 / yulawyer）

在派生产物（文档、代码、技能文件）中请包含以上署名与许可证文本。

Copyright © 2026 zy5120

---

# 小鱼 TabView (xiaoyu-tabview) — English

Decision principles and implementation notes for migrating a legacy TabView (3 tabs + top-right add/search) to the iOS 26 beautiful tab bar: an independent search tab, a bottom long action button (`tabViewBottomAccessory`), and bottom search. Covers the long-button takeover rules, search context per page, known iOS 26/27 pitfalls, and portrait drill-path navigation.

> Extracted from the "鱼律 / yulawyer" iOS app (iOS 26+).

## Companion Skill

This skill works **together with [小鱼平板适配forOS26 / iPad Adaptive Layout](https://github.com/zy5120/ipad-adaptive-layout-ios26)** (portrait sheet/push + landscape sidebar):

1. First adapt the page's dual-mode layout with the iPad skill (portrait sheet/push, landscape sidebar);
2. Then adapt the beautiful TabView's bottom action button and search with this skill (button follows focus, search moves to the sidebar, etc.).

Each skill owns one layer: one handles layout & rotation, the other handles the bottom controls & search. Follow this order when adapting any page.

## License

This project is licensed under the **Apache License 2.0**.

**Attribution required**: any use, invocation, copy, or adaptation of this skill (including using it as development guidelines, generating code/content, or integrating it into other skills/projects) must retain this copyright notice and credit the source:

- Repository: <https://github.com/zy5120/xiaoyu-tabview>
- Author: zy5120 (鱼律 / yulawyer)

Please include the attribution and the license text in derived works (docs, code, skill files).

Copyright © 2026 zy5120
