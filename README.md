# 元素透镜 / Element Lens

[![CI](https://github.com/mmyddd/dsh-element-lens/actions/workflows/ci.yml/badge.svg)](https://github.com/mmyddd/dsh-element-lens/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/mmyddd/dsh-element-lens)](https://github.com/mmyddd/dsh-element-lens/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

在 DSH 右侧栏浏览器里拾取网页元素，把它送进对话草稿。

## 安装

从 GitHub 装（推荐）：在 DSH 里让助手执行一次 bundle 安装，目标写

```
github:mmyddd/dsh-element-lens
```

装完刷新窗口即可看到地址栏右侧的拾取按钮。

也可以手动从 GitHub 克隆后按本地目录安装：

```bash
git clone https://github.com/mmyddd/dsh-element-lens
```

然后以本地路径安装该目录（等价的落地形态是 profile 的 `package.json` 里有
`"dsh-element-lens": "link:<克隆出来的路径>"`，且 `dsh.profile.bundles` 含 `dsh-element-lens`）。

本地改动不会热更新——link 安装的包在页面加载时才取代码，改完要刷新窗口。

## 它做什么

点侧栏地址栏右侧的拾取按钮后，页面进入选取模式：鼠标划过元素时下方出现高亮框与信息卡（标签名、尺寸、颜色、背景、字体），点击一次即提取该元素并写入对话草稿；Esc 取消。

提取的字段：页面地址与标题、标签名、`role`、可访问名、CSS 选择器、XPath、文本、邻近文本、去脚本化的 HTML 片段、白名单属性、包围盒、以及可读的样式（颜色、背景、显示、字族、字号、字重）。多段序列化成 `## Element N` 提示块。

## 三个通道

侧栏浏览器在不同宿主里挂不同的框架，插件按顺序选最合适的一个：

| 宿主 | 通道 |
| --- | --- |
| 桌面端 Electron `<webview>` | `executeJavaScript` 把选择器注入页面内部执行（选择器是自包含函数，可被序列化） |
| Web 端同源 `<iframe>` | 本进程直接对子文档安装同一个选择器 |
| 跨域 `<iframe>` | 宿主侧 `/element-lens/snapshot` 取回 HTML，叠加同源快照帧供选取 |

快照帧会剥掉脚本、内联事件、`<base>` 与 `srcdoc`，并注入 `<base href>` 让相对资源仍能解析。

## 胶囊的两个方向都与输入框一致

胶囊**不自己保存列表**：它每次都从草稿的 `occurrences` 投影出来，草稿是唯一权威。

- 在胶囊面板里点「删除」：删除草稿里对应的 chip，胶囊随之消失。
- 在输入框里删掉 chip：草稿变更触发订阅，胶囊立即跟着减。
- 因此「胶囊显示 N 个」与「发送时会附带 N 个」永远一致。

删除要落在编辑器的 detect 投影上：一个 chip 在草稿文本里占 `clipboardText.length` 个字符，在 detect 文本里只占 1 个，所以换算由 `chipSpan()` 完成。跨度每次删除时按最新快照重算（`spanOfChip()`），不复用绘制面板时记下的旧偏移——旧偏移配上新版本号会通过版本校验，然后剪掉错位的那一个字符。

## 显示形式与批注胶囊一致

- 胶囊本体：同一套 token 与度量（`z-index` 低一层，见下），`N` 用强调色加粗，后接单位文案。
- 悬浮面板：与批注胶囊同构——`300px` 宽、`10px 12px` 内距、`12px` 圆角、同款边框/底色/阴影；顶部一行标题「网页元素（N 个）」，每行是圆形序号徽章 + 选择器摘要（超 100 字截断）+ 右侧红色「删除」。
- 两者都占输入框右上角时：先量 `[data-annotation-chip]` 的位置，本胶囊让到它左侧留 8px 间隙；那一行放不下就叠到它上方。批注胶囊优先级更高——本胶囊 `z-index: 1149`，它 `1150`，即使某一帧撞在一起也是它压在上面。

## 与参考实现的对应关系

行为对照，逐项；代码全部重写，未复制其源码。

| 行为 | 参考实现 | 本插件 |
| --- | --- | --- |
| 触发按钮 | 地址栏所在工具栏内的图标按钮，`active` 反映拾取状态，未就绪时禁用 | 注入到 `addressBox` 之后的 28×28 图标按钮，`aria-pressed` 反映状态 |
| 注入方式 | 把选择器函数 `toString()` 后拼成表达式在页面内执行，返回 Promise | 同一策略：`installPicker` 自包含，`pickerSource()` 序列化后交 `executeJavaScript` |
| 取消 | 执行页内 `…Picker.cancel()` | `CANCEL_SOURCE` 调用页内 `__elementLensPicker.cancel()` |
| 高亮框 | 固定定位描边框 + 超大外阴影遮罩 | 同（`2px #2563eb`、`rgba(37,99,235,.12)` 填充、`0 0 0 9999px rgba(15,23,42,.10)` 阴影） |
| 信息卡 | 深色毛玻璃卡：标签名 + 宽×高，下列颜色/背景/字体 | 同（同样的字号、字重、圆角、模糊与网格布局） |
| 卡片定位 | 依次尝试下/上/右/左，带视口钳制与重叠回退 | 同策略，另加「与元素矩形重叠则跳过」 |
| 光标 | 进入选取时十字光标，结束恢复 | 同 |
| 取消手势 | Esc | Esc（文档与窗口各绑一份，覆盖焦点在帧内的情况） |
| 提取字段 | 页面地址与标题、标签名、role、可访问名、选择器、XPath、文本、邻近文本、HTML 片段、属性、矩形、样式、抓取时间 | 字段集合一致 |
| role 推断 | 按标签与 input 类型分派 | 同规则 |
| 可访问名 | aria-labelledby → aria-label/alt/title/placeholder → 文本 | 同顺序 |
| 选择器 | 有 id 用 id，否则逐层拼标签+类名，必要时 `:nth-of-type` | 同（8 层上限） |
| XPath | 逐层 `tag[i]` | 同（12 层上限） |
| 属性白名单 | id/class/href/src/alt/title/name/type/placeholder/aria-*，排除 value | 同，每值截 500 |
| HTML 片段 | 克隆后去脚本/样式/模板，输入框去值，截断 | 同，截 6000 并标 `[truncated]` |
| 文本 | 密码框打码；输入类元素按 标签 = 值 呈现 | 同 |
| 邻近文本 | 就近容器（article/section/main/form/li/tr/dialog）的文本 | 同，截 4000 |
| 颜色归一 | `rgb()/rgba()` → 十六进制，全透明归 `transparent` | 同 |
| 提示词形态 | 多个元素合并为 `## Element N` 块，逐行字段 + 代码围栏 | 同格式 |
| 结果消费 | 挂到聊天输入区作为附件 | 写入草稿的 reference chip（`source: 'reference'`） |

差异（本插件多出来或不同的部分）：

- 参考实现直接把元素挂成聊天附件；本插件写入对话草稿的 chip，因此可以在输入框里逐条删除，并由胶囊同步反映。
- 跨域 iframe 在 Web 端没有可用的页面内执行通道，故走宿主侧快照帧；桌面端 `<webview>` 与参考实现走同一条页面内执行路径。


## 文件

- `index.js`（Host）：`/element-lens/snapshot` 路由，抓取并净化目标页面。
- `client.js`（Client）：按钮注入、选择器、胶囊、草稿写入。
- `locale/{zh,en}.json`：本地化元数据。
- `cordis.patch.yml`：bundle 插入。
- `icon.svg`：插件图标。
- `test/`：无头回归测试（`smoke.cjs` 布局与交互、`sync.cjs` 草稿投射、`webview.cjs` guest 通道）。

## 按钮位置

插在侧栏工具栏 `div.addressBox` 之后、`aria-label` 为「在系统里浏览器打开」的按钮之前。宿主工具栏是 `height:38px` 的 flex 行，地址框 `flex:auto`，插入一个 28×28 的按钮只会让地址框自动收窄，不换行也不溢出。按钮复刻宿主 `.tool` 的度量与 hover/禁用配色，并通过复制相邻工具栏按钮的 class 跟随宿主构建的哈希类名。

## 验证

```bash
npm test                  # 五个套件一起跑
node test/manifest.cjs    # 清单、导出、路由、命名一致性
node test/smoke.cjs       # 布局、点击、Esc、释放
node test/sync.cjs        # 胶囊 ↔ 草稿双向一致
node test/chip.cjs        # 胶囊让位批注胶囊的布局
node test/webview.cjs     # guest 通道端到端
```

CI 在 Ubuntu 与 Windows 上、Node 22 与 24 下各跑一遍 `npm test`；推 `v*` 标签时同样的检查通过后才会发布 GitHub Release（附带 `.tgz` 与 `SHA256SUMS.txt`）。

## 许可

MIT，见 [LICENSE](LICENSE)。
