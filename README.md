# Pretext

纯 JavaScript/TypeScript 多行文本测量与排版库。快速、精确，支持你甚至不知道存在的各种语言。可渲染到 DOM、Canvas、SVG，服务端渲染即将支持。

Pretext 绕过了 DOM 测量（如 `getBoundingClientRect`、`offsetHeight`）的需求，这些操作会触发布局回流——浏览器中最昂贵的操作之一。它实现了自己的文本测量逻辑，以浏览器自身的字体引擎作为基准真值（非常适合 AI 友好的迭代方式）。

## 安装

```sh
npm install @chenglou/pretext
```

## 演示

克隆仓库，运行 `bun install`，然后 `bun start`，在浏览器中打开 `/demos`（不要带尾部斜杠，Bun 开发服务器会出 bug）。
或者在线查看：[chenglou.me/pretext](https://chenglou.me/pretext/)。更多演示见 [somnai-dreams.github.io/pretext-demos](https://somnai-dreams.github.io/pretext-demos/)

## API

Pretext 服务于两种使用场景：

### 1. 无需接触 DOM 即可测量段落高度

```ts
import { prepare, layout } from '@chenglou/pretext'

const prepared = prepare('AGI 春天到了. بدأت الرحلة 🚀', '16px Inter')
const { height, lineCount } = layout(prepared, textWidth, 20) // 纯算术运算，无 DOM 布局和回流！
```

`prepare()` 完成一次性工作：规范化空白字符、分段文本、应用粘连规则、使用 canvas 测量分段宽度，并返回一个不透明的句柄。`layout()` 是之后的低开销热路径：基于缓存宽度的纯算术运算。对于相同的文本和配置，不要重复调用 `prepare()`，那会浪费它的预计算优势。例如，在窗口调整大小时，只需重新调用 `layout()`。

如果你需要类似 textarea 的文本效果——普通空格、`\t` 制表符和 `\n` 硬换行保持可见——请向 `prepare()` 传入 `{ whiteSpace: 'pre-wrap' }`：

```ts
const prepared = prepare(textareaValue, '16px Inter', { whiteSpace: 'pre-wrap' })
const { height } = layout(prepared, textareaWidth, 20)
```

在当前已提交的基准测试快照中：
- `prepare()` 对共享的 500 条文本批次耗时约 `19ms`
- `layout()` 对同一批次耗时约 `0.09ms`

我们支持你能想到的所有语言，包括 emoji 和混合双向文本，并针对特定浏览器的怪异行为进行了处理。

返回的高度是解锁 Web UI 的关键拼图：
- 无需估算和缓存的精确虚拟化/遮挡剔除
- 高级自定义布局：瀑布流、JS 驱动的类 flexbox 实现、无需 CSS hack 即可微调布局值（想象一下）等
- _开发阶段_ 验证（尤其在 AI 时代）标签文本（如按钮上的文字）不会溢出到下一行，无需浏览器
- 防止新文本加载时的布局偏移，便于重新锚定滚动位置

### 2. 手动排版段落行

将 `prepare` 替换为 `prepareWithSegments`，然后：

- `layoutWithLines()` 在固定宽度下返回所有行：

```ts
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments('AGI 春天到了. بدأت الرحلة 🚀', '18px "Helvetica Neue"')
const { lines } = layoutWithLines(prepared, 320, 26) // 最大宽度 320px，行高 26px
for (let i = 0; i < lines.length; i++) ctx.fillText(lines[i].text, 0, i * 26)
```

- `walkLineRanges()` 返回行宽和游标，无需构建文本字符串：

```ts
let maxW = 0
walkLineRanges(prepared, 320, line => { if (line.width > maxW) maxW = line.width })
// maxW 现在是最宽行的宽度——能容纳文本的最紧凑容器宽度！这种多行"收缩包裹"一直是 Web 缺失的功能
```

- `layoutNextLine()` 允许你逐行排版，每行可使用不同的宽度：

```ts
let cursor = { segmentIndex: 0, graphemeIndex: 0 }
let y = 0

// 文本环绕浮动图片：图片旁边的行更窄
while (true) {
  const width = y < image.bottom ? columnWidth - image.width : columnWidth
  const line = layoutNextLine(prepared, cursor, width)
  if (line === null) break
  ctx.fillText(line.text, 0, y)
  cursor = line.end
  y += 26
}
```

此用法支持渲染到 canvas、SVG、WebGL 以及（最终）服务端。

### API 词汇表

场景 1 的 API：
```ts
prepare(text: string, font: string, options?: { whiteSpace?: 'normal' | 'pre-wrap' }): PreparedText // 一次性文本分析 + 测量过程，返回一个不透明值传递给 `layout()`。确保 `font` 与你 CSS 中的 `font` 简写声明同步（如大小、粗细、样式、字族）。`font` 格式与 `myCanvasContext.font = ...` 相同，例如 `16px Inter`。
layout(prepared: PreparedText, maxWidth: number, lineHeight: number): { height: number, lineCount: number } // 根据最大宽度和行高计算文本高度。确保 `lineHeight` 与你 CSS 中的 `line-height` 声明同步。
```

场景 2 的 API：
```ts
prepareWithSegments(text: string, font: string, options?: { whiteSpace?: 'normal' | 'pre-wrap' }): PreparedTextWithSegments // 与 `prepare()` 相同，但返回更丰富的结构以满足手动行排版需求
layoutWithLines(prepared: PreparedTextWithSegments, maxWidth: number, lineHeight: number): { height: number, lineCount: number, lines: LayoutLine[] } // 手动排版的高级 API。接受所有行的固定最大宽度。类似 `layout()` 的返回值，但额外返回行信息
walkLineRanges(prepared: PreparedTextWithSegments, maxWidth: number, onLine: (line: LayoutLineRange) => void): number // 手动排版的底层 API。接受所有行的固定最大宽度。每行调用一次 `onLine`，传入实际计算的行宽和起止游标，无需构建行文本字符串。适用于需要试探性测试多个宽高边界的场景（如通过反复调用 walkLineRanges 并检查行数及高度是否"合适"来二分搜索理想宽度。可实现文本消息的收缩包裹和平衡文本排版）。调用 walkLineRanges 后，使用满意的最大宽度调用一次 layoutWithLines 即可获取实际行信息。
layoutNextLine(prepared: PreparedTextWithSegments, start: LayoutCursor, maxWidth: number): LayoutLine | null // 类迭代器 API，每行可使用不同宽度！返回从 `start` 开始的 LayoutLine，段落用尽时返回 `null`。将前一行的 `end` 游标作为下一行的 `start` 传入。
type LayoutLine = {
  text: string // 此行的完整文本内容，例如 'hello world'
  width: number // 此行的测量宽度，例如 87.5
  start: LayoutCursor // 在预处理段/字素中的包含起始游标
  end: LayoutCursor // 在预处理段/字素中的排除结束游标
}
type LayoutLineRange = {
  width: number // 此行的测量宽度，例如 87.5
  start: LayoutCursor // 在预处理段/字素中的包含起始游标
  end: LayoutCursor // 在预处理段/字素中的排除结束游标
}
type LayoutCursor = {
  segmentIndex: number // prepareWithSegments 预处理的富分段流中的段索引
  graphemeIndex: number // 该段内的字素索引；在段边界处为 `0`
}
```

其他辅助函数：
```ts
clearCache(): void // 清除 Pretext 内部由 prepare() 和 prepareWithSegments() 使用的共享缓存。当你的应用循环使用大量不同字体或文本变体时，可用于释放累积的缓存
setLocale(locale?: string): void // 可选（默认使用当前语言环境）。为后续的 prepare() 和 prepareWithSegments() 设置语言环境。内部也会调用 clearCache()。设置新语言环境不会影响已有的 prepare() 和 prepareWithSegments() 状态（不会对其产生修改）
```

## 注意事项

Pretext 并不试图成为完整的字体渲染引擎（暂时？）。目前针对常见的文本设置：
- `white-space: normal`
- `word-break: normal`
- `overflow-wrap: break-word`
- `line-break: auto`
- 如果传入 `{ whiteSpace: 'pre-wrap' }`，普通空格、`\t` 制表符和 `\n` 硬换行将被保留而非折叠。制表符遵循浏览器默认的 `tab-size: 8`。其他换行默认值不变：`word-break: normal`、`overflow-wrap: break-word` 和 `line-break: auto`。
- 在 macOS 上 `system-ui` 对 `layout()` 精度不安全，请使用具名字体。
- 由于默认目标包含 `overflow-wrap: break-word`，非常窄的宽度仍可能在单词内部断行，但仅在字素边界处。

## 开发

请参阅 [DEVELOPMENT.md](https://github.com/chenglou/pretext/blob/main/DEVELOPMENT.md) 了解开发环境搭建和命令。

## 致谢

Sebastian Markbage 十年前通过 [text-layout](https://github.com/chenglou/text-layout) 最早播下了种子。他的设计——使用 canvas `measureText` 进行字形塑形、从 pdf.js 引入双向文本支持、流式断行——奠定了我们在此持续推进的架构基础。
