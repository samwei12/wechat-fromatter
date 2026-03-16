# wechat-fromatter

## 仓库定位

这是一个纯前端静态工具，用来把 Markdown 渲染成更适合粘贴到微信公众号编辑器的 HTML。

- 无构建系统、无打包器、无 Node 依赖管理。
- 页面入口是 `index.html`，运行时依赖主要靠本地静态资源和少量 CDN 脚本。
- 核心不是通用 Markdown 预览，而是通过自定义 `marked.Renderer` 输出适配微信公众号的内联样式和代码块结构。

## 技术栈

- 页面框架：Vue 2
- UI：Element UI
- 编辑器：CodeMirror 5
- Markdown 解析：`marked`
- HTTP 请求：`axios`
- 自定义扩展：`FuriganaMD`
- 自定义渲染器：`assets/scripts/renderers/wx-renderer.js`

这些库大多通过全局变量方式接入，脚本加载顺序就是依赖关系，不要随意改成乱序。

## 运行方式

这是静态站点，建议在仓库根目录起一个本地 HTTP 服务后访问，而不是直接双击 `index.html` 用 `file://` 打开。

原因：

- `assets/scripts/editor.js` 会通过 `axios` 请求 `./assets/default-content.md` 作为默认内容；`file://` 下这类请求通常会失败。
- 页面还依赖外部 CDN 样式和脚本，离线环境下会退化或直接不可用。

可用的最小启动方式示例：

```bash
python3 -m http.server 8000
```

然后访问 `http://localhost:8000`。

## 实际代码结构

### 1. 页面装配

`index.html` 负责：

- 搭建双栏布局：左侧编辑器，右侧预览区。
- 引入 CSS、CodeMirror、Vue、Element UI、主题脚本、渲染器和主逻辑。
- 暴露几个用户可调项：编辑器主题、字体、字号、渲染主题。

这里没有模块化系统，所有脚本都挂在全局作用域。

### 2. 应用状态与交互

`assets/scripts/editor.js` 是主控制器，职责包括：

- 创建 Vue 实例。
- 把 `<textarea>` 升级成 CodeMirror。
- 维护当前字体、字号、编辑器主题、渲染主题。
- 在编辑器内容变化时调用 `refresh()`。
- 用 `marked(..., { renderer })` 触发 Markdown -> HTML 渲染。
- 通过 `copy()` 选中预览区内容并调用 `document.execCommand('copy')` 复制。

数据流很直接：

1. 用户编辑 Markdown。
2. CodeMirror `change` 事件触发。
3. `renderWeChat()` 调用 `WxRenderer.getRenderer()`。
4. `marked` 生成 HTML。
5. 结果写入 `v-html="output"` 的预览区域。

### 3. 微信渲染内核

`assets/scripts/renderers/wx-renderer.js` 是仓库最重要的文件。

它做了几件关键事情：

- 把 theme 模板和当前字体/字号合成为最终 style 映射。
- 重写 `heading`、`paragraph`、`blockquote`、`code`、`list`、`image`、`link`、`table` 等渲染行为。
- 为绝大多数元素输出内联样式，保证复制到微信编辑器后尽量保留样式。
- 对普通外链默认转成脚注引用，对 `https://mp.weixin.qq.com` 链接则保留为可点击链接。
- 为代码块生成更接近公众号样式的 HTML 结构和行号列表。

修改渲染语义时，优先改这里，不要先去改 CSS，因为复制到微信后真正起作用的是这里输出的内联样式和结构。

### 4. 主题系统

主题定义在：

- `assets/scripts/themes/default.js`
- `assets/scripts/themes/lupeng.js`

主题本质上是两个对象：

- `block`
- `inline`

`WxRenderer` 会按 token 名取样式，因此主题 key 必须和渲染器里使用的 token 对齐，例如：

- `h2`
- `h3`
- `p`
- `blockquote`
- `code`
- `image`
- `ul`
- `ol`
- `codespan`
- `link`
- `wx_link`

如果新增渲染 token，通常也要同步给所有主题补默认样式，否则可能出现 `undefined` 样式访问。

### 5. 页面壳样式

`assets/css/app.css` 只负责本地编辑器界面和预览容器外观，例如：

- 顶部控制栏布局
- CodeMirror 区域样式
- 手机宽度预览框
- 代码块在本地预览中的辅助样式

它不决定最终粘贴到微信后的大部分视觉效果。最终效果主要由 `wx-renderer.js` 输出的内联样式决定。

## 修改入口建议

按目标选文件，不要大范围乱改：

- 改界面结构或新增控件：`index.html`
- 改状态流、交互、复制逻辑、默认选项：`assets/scripts/editor.js`
- 改 Markdown 到微信公众号 HTML 的语义映射：`assets/scripts/renderers/wx-renderer.js`
- 改主题视觉：`assets/scripts/themes/*.js`
- 改本地编辑器壳层样式：`assets/css/app.css`

## 当前已确认的前提与坑

### 1. `assets/default-content.md` 当前不存在

`assets/scripts/editor.js` 会请求这个文件，但仓库里没有它。

这意味着：

- 页面首次加载时默认示例内容请求会失败。
- 只要用户自己输入内容，主流程仍然可以工作。
- 如果想恢复开箱即用体验，应补回该文件，或者在请求失败时回退到内置字符串。

### 2. 复制能力依赖旧式 API

`copy()` 使用的是 `document.execCommand('copy')`。这是老 API，兼容性还行，但不是现代推荐方案。若后续要改，优先考虑在不破坏微信公众号粘贴格式的前提下再评估 `navigator.clipboard`。

### 3. 运行时是全局脚本拼装，不要随意模块化半改

当前代码默认依赖这些全局符号：

- `Vue`
- `axios`
- `marked`
- `CodeMirror`
- `FuriganaMD`
- `defaultTheme`
- `lupengTheme`
- `WxRenderer`

所以：

- 改脚本加载顺序会直接破页面。
- 只把某一个文件局部改成模块语法，通常会引入新的运行时问题。

### 4. 微信适配优先于浏览器端“语义优雅”

这个仓库的目标是“复制到公众号后看起来对”，不是生成最标准最干净的语义化 HTML。对代码块、列表、脚注、链接做的很多处理，本质上都是为了兼容微信编辑器。

做改动时应优先问：

- 粘贴到微信公众号后台后是否还能保留样式？
- 是否会被微信清洗掉？
- 是否会影响复制体验？

而不是先追求浏览器内预览的纯粹性。

### 5. 外部依赖不是完全本地化

`index.html` 里仍然引用了外部资源：

- Element UI theme CSS
- Vue CDN
- Google Analytics

如果要做内网可用、离线可用或隐私更严格的版本，需要把这些资源本地化并去掉统计脚本。

## 推荐工作方式

在这个仓库里做修改时，默认遵循下面的策略：

- 保持静态站点模型，不要无故引入打包链路。
- 优先做最小改动，先保住“编辑 -> 预览 -> 复制到微信”主链路。
- 任何涉及渲染语义的改动，都先检查 `wx-renderer.js` 与主题 key 是否一致。
- 任何涉及默认加载体验的改动，都检查 `assets/default-content.md` 缺失问题是否被处理。
- 任何涉及外观的改动，都区分“本地壳层样式”与“复制后内联样式”这两个层面。

## 与本仓库无关但存在的目录

根目录下的 `.claude/` 当前是用户本地工具配置/agent 资料，不属于这个格式化工具的运行主链路。分析或改造应用本身时，默认不要把 `.claude/` 里的内容当成产品代码的一部分。
