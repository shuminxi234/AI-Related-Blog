---
title: "Jelly UI 源码解析：当原生表单控件拥有软体物理"
date: 2026-07-21
tags: ["前端", "Web Components", "Canvas", "可访问性"]
author: "shuminxi234"
description: "从接入方式、软体物理模拟、原生表单语义和无障碍设计四个角度，解析 Jelly UI 如何让 Web 控件获得可交付的果冻触感。"
---

按钮按下时向指尖凹陷，松手后像果冻一样回弹；滑块被拖动时，边缘会产生局部形变。这类效果通常只出现在品牌展示页，因为一旦进入真实表单，就要同时处理键盘、焦点、提交、校验和无障碍语义，视觉实验很容易变成维护负担。

[Jelly UI](https://jelly-ui.com/) 尝试把两者放在同一个组件里：前景仍是真实的表单控件，背景则由 Canvas 绘制软体表面。项目目前以 Web Components 形式提供，运行时没有第三方依赖，使用一个 ESM 入口即可注册组件。需要先纠正一个容易产生的误解：它并非只靠 CSS `transition` 做弹性缩放，而是用 TypeScript 实现物理模拟，构建后由 JavaScript 驱动 Canvas 2D 绘制。

## 五分钟接入：不是替换 CSS 类名

最直接的试用方式是加载官方托管入口，然后使用 `jelly-*` 自定义元素：

```html
<script type="module" src="https://jelly-ui.com/package.js"></script>

<jelly-theme mode="auto" accent="#7c3aed">
  <form id="profile-form">
    <jelly-input
      name="email"
      type="email"
      label="邮箱"
      placeholder="you@example.com">
    </jelly-input>
    <jelly-switch name="notice" checked>接收通知</jelly-switch>
    <jelly-button type="submit" variant="mint">保存</jelly-button>
  </form>
</jelly-theme>
```

```js
const form = document.querySelector('#profile-form');

form.addEventListener('submit', (event) => {
  event.preventDefault();
  console.log(Object.fromEntries(new FormData(form)));
});
```

这里不是给已有 `<button>` 添加一个 class，而是使用 Custom Elements。官方入口会重新导出构建产物 `dist/jelly.js`，后者注册全部组件。生产环境若不希望依赖第三方域名，可以按照仓库说明执行 `npm run build`，再把 `package.js` 与 `dist/` 一并部署到自己的静态资源域名。

截至 2026 年 7 月 21 日，仓库 `package.json` 标注的版本为 1.1.0。组件范围也不只按钮：表单类包含 input、textarea、select、checkbox、radio、switch、slider、range 和 OTP；此外还有 tabs、dialog、drawer、menu、tooltip，以及由 `jellyToast()` 与 `jelly-toaster` 提供的通知能力。它更接近一套设计系统，而不是单个动画插件。代价是当前入口会注册整套组件，选型时应实际测量打包体积、解析成本与页面需求是否匹配。

## 果冻效果究竟怎样工作

源码中的核心抽象是 `JellyBody`。每个需要软体效果的组件，都拥有一圈围绕圆角矩形分布的膜节点。相邻节点由类似弹簧的约束耦合，模拟材料的连续性；压力与体积修正用于避免形状在受力后无限塌陷；指针位置则成为局部的按压目标。于是用户按在按钮左侧时，形变会从左侧产生，而不是整个元素做一次均匀的 `scale(0.95)`。

一次交互大致经历四步：

1. Pointer 事件被映射到组件内部坐标，确定按压点和方向；
2. 模拟器向附近节点施加凹陷、隆起或深度冲量；
3. 弹簧、阻尼和节点耦合在固定大小的子步中更新位置与速度；
4. Canvas 根据新的边界重绘填充、阴影、透视和焦点环。

固定子步很关键。如果简单地把每帧间隔直接带入弹簧公式，页面从 120 FPS 降到 30 FPS 时，结果可能突然变软甚至数值发散。固定步长让模拟在不同刷新率下更稳定。项目还让所有活动的软体共享一个 `requestAnimationFrame` 循环；当物体恢复静止后，循环会暂停，而不是让每个组件永久占用一个动画任务。源码也对非有限数值做了重置保护，避免单个异常状态拖垮后续绘制。

这套方案的边界同样明确。Canvas 负责的是视觉表面，不负责文本和交互语义；大量同时运动的控件仍会产生 CPU 与绘制开销。所谓“静止时无模拟成本”不等于页面加载、组件注册和 Canvas 内存没有成本，因此不应只凭演示页的流畅度推导生产性能。

## 最值得借鉴的是“视觉层与语义层分离”

很多 Canvas UI 的问题不是不好看，而是把按钮画成了按钮，却没有真正的按钮。Jelly UI 的做法相反：例如 `jelly-button` 的 Shadow DOM 内保留原生 `<button>`，Canvas 位于其后方；输入类组件也保留原生可聚焦控件，并通过标准事件向组件外通信。

这带来几个工程收益：

- Enter、Space、Tab 等键盘行为可以建立在浏览器原生能力之上；
- `click`、`change` 等事件能够穿过 Shadow DOM 边界；
- 表单组件借助 `ElementInternals` 参与 `FormData`，不必另外收集一份组件状态；
- `type="submit"` 和 `type="reset"` 可以驱动最近的 light-DOM 表单；
- 组件可保留焦点环、ARIA 模式和强制颜色模式下的降级样式。

这并不代表可访问性天然完成。开发者仍须给纯图标按钮提供 `label`，为输入项提供可访问名称，检查错误信息是否能被辅助技术关联，并用键盘和屏幕阅读器测试完整业务流程。Shadow DOM 只是封装机制，不是质量保证书。

动画本身也可能让部分用户不适。Jelly UI 会检测 `prefers-reduced-motion: reduce`，在用户要求减少动态效果时绕过物理冲量，改为平静、即时的反馈。项目 1.1.0 还增加了显式的 motion 覆盖能力。应用自己的滚动、Lottie 或过场动画也应遵循同一偏好，否则单独让按钮停止晃动并不能形成一致体验。

## 主题、RTL 与框架集成

颜色通过 `--jelly-color-*` 等 CSS 自定义属性传递，`jelly-theme` 可以为局部子树指定 light、dark、auto 和 accent。单个控件还可覆盖 `--jelly-fill`、`--jelly-label`、`--jelly-ring` 等变量。由于 Canvas 颜色不是普通 CSS 背景，主题变化时组件需要重新读取继承后的 token 并重绘；这是实现自定义绘制组件时常被遗漏的一步。

项目也使用逻辑属性和方向感知的键盘逻辑支持 RTL。例如水平滑块在 RTL 环境下会翻转左右方向，overlay 的位置使用 start/end 表达。对于需要同时支持多语言的产品，这比最后阶段用 `transform: scaleX(-1)` 补救可靠得多。

React、Vue 或 Svelte 都能渲染 Custom Elements，但集成不能只看“标签能否显示”。应验证属性与 property 的传递差异、布尔值绑定、自定义事件监听、服务端渲染时未注册元素的表现，以及框架类型系统能否识别标签。Jelly UI 使用 Shadow DOM，外部样式也不能像普通 DOM 那样深入覆盖；定制应优先走公开的 CSS 变量和 shadow parts，而不是依赖内部结构。

## 上线前做一次小型验收

不要直接把展示页效果等同于业务可用。可以先搭建一个包含必填输入、单选、多选、提交和重置的最小表单，然后完成四组检查：第一组只用键盘操作，观察焦点顺序、弹窗焦点回收与错误提示；第二组开启系统的“减少动态效果”和强制颜色模式，确认状态不再只靠动画或颜色表达；第三组在触屏设备上测试拖动、移出后松手、快速连点与滚动冲突；第四组记录 1、20、50 个组件同时存在和同时运动时的长任务、帧率与内存。

还要把 JavaScript 加载失败纳入测试。自定义元素未注册时，标签内容可能仍会显示，但表单关联、提交按钮和交互逻辑不会自动成立。关键流程应选择服务端可用的基础表单作为渐进增强底座，或者明确提供加载失败提示。对注册、支付等高风险页面，漂亮的回弹不能成为单点故障。

## 什么时候值得使用

Jelly UI 适合强调触感和品牌个性的注册流程、消费级工具、作品集或产品落地页。它的真正价值不是“把按钮变成果冻”，而是展示了一条可复用的组件架构：**让原生控件承担语义与交互契约，让 Canvas 承担高自由度视觉表达，再通过 Web Components 封装两者。**

对于高密度后台、低端设备占比较高的产品，或者需要严格控制脚本预算的页面，则应先做真实基准测试。至少记录首次加载与交互延迟、多个组件同时运动时的主线程占用、减少动态效果模式、键盘路径和表单校验结果，并准备未加载脚本时的回退方案。

项目创建时间尚短，API、浏览器兼容性和边界行为仍可能快速变化。官方声明支持当前版本的 Chrome、Edge、Safari 与 Firefox，但其技术基础包括 Custom Elements、Shadow DOM、`ElementInternals`、`ResizeObserver`、Canvas 2D、`color-mix()` 和 CSS 逻辑属性；如果产品需要旧版浏览器或嵌入式 WebView，应按自己的支持矩阵验证，而不是把“当前浏览器”理解为任意历史版本。

Jelly UI 最值得前端工程师关注的，不是某个弹簧参数，而是它没有为了视觉新奇牺牲 Web 的基本契约。炫技可以停留在 Canvas 中，表单、事件、焦点与可访问性仍然应该属于平台。

## 参考资料

- [Jelly UI 官方展示与文档](https://jelly-ui.com/)
- [Jelly UI API Reference](https://jelly-ui.com/api/)
- [jelly-org/ui GitHub 仓库](https://github.com/jelly-org/ui)
- [Hacker News 原始讨论](https://news.ycombinator.com/item?id=48981620)
- [MDN：使用自定义元素](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements)
- [MDN：ElementInternals](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals)
- [MDN：prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)
