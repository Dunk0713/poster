# 小红书笔记生成器 · CLAUDE.md

## 项目概述

单文件 Web 应用（`index.html`，约 5400 行），用于生成小红书风格的基金知识分享卡片图。用户是**公募基金从业者**，在小红书发布基金投资科普内容。设计偏好：**极简高级感**（苹果/Anthropic 官网风格）。

**技术栈**：纯 HTML/CSS/JS，零框架，无构建步骤。外部依赖仅 CDN：
- `html-to-image`（导出 PNG）
- `qrcode-generator`（二维码块）
- `lxgw-wenkai-webfont` / `JetBrains Mono` / `Playfair Display`（字体）

## 文件结构（单文件分区）

| 区段 | 行号范围 | 说明 |
|------|---------|------|
| `<head>` + CDN | 1-14 | 外部脚本和字体引入 |
| `<style>` CSS | 15-1906 | 全部样式 |
| `<body>` HTML | 1908-2061 | 编辑器面板 + 预览区骨架 |
| `<script>` JS | 2063-5398 | 全部逻辑 |

### CSS 细分

| 子区段 | 行号 | 内容 |
|--------|------|------|
| 编辑器 UI | 16-290 | header、layout、form、palette、seg-btn、block editor |
| 页码条/预览/导出/toast | 291-408 | page-strip、preview-panel、export-bar |
| 卡片 design tokens | 412-429 | `--fs-*`、`--space-*`、`--radius-*` |
| 气质 mood 覆盖 | 440-518 | `.mood-editorial/minimal/rounded/mono/handwriting` |
| 主题色 theme | 520-650 | `.theme-terracotta` 到 `.theme-navy`（每个含 `--bg1/--bg2/--accent` 等） |
| 卡片结构 | 685-740 | `.card`、`.card-title`、`.card-body`、`.card-footer` |
| 块样式 `.cb-*` | 742-1906 | 28 种块各自的渲染样式 |

### JS 关键符号定位

| 符号 | 行号 | 用途 |
|------|------|------|
| `BLOCK_TYPES` | ~2065 | 块类型注册表（中文名映射） |
| `THEMES` | ~2098 | 主题色列表 |
| `BLOCK_COLORS` | ~2114 | 块可选颜色 |
| `makeBlock(type)` | ~2129 | 工厂函数：创建块默认数据 |
| `DISCLAIMER_PRESETS` | ~2193 | 免责声明 5 类预设文案 |
| `migrateBlock(blk)` | ~2203 | 旧笔记兼容：字段兜底 + 格式迁移 |
| `createPage(tpl)` | ~2235 | 页面工厂（含 dataCard 表格默认值） |
| `state` 初始化 | ~2262 | 全局状态对象 |
| `inl(text)` | ~2400 | 行内 markdown（`**加粗**` `==高亮==` `~~删除~~`） |
| `parseBody(text)` | ~2409 | 段落级 markdown（按 `\n\n` 分段） |
| `renderEditorFields_raw()` | ~3242 | 左侧编辑面板 HTML 生成（按模板分支） |
| `renderEditorFields()` | ~3467 | 调度 + 事件绑定入口 |
| `_history` / `pushHistory()` | ~3482 | 撤销系统（深拷贝整个 state，最多 40 层） |
| 键盘快捷键 | ~3633 | Ctrl+Z/Shift+Z、Alt+↑↓、Ctrl+D、Ctrl+F |
| `App` 对象 | ~3680 | 暴露给 onclick 的方法集合 |
| `renderBlock(blk, sz)` | ~4554 | 单个块→卡片 HTML（大 switch-case） |
| `renderCard()` | ~4866 | 整张卡片渲染（按模板分支拼装） |
| `renderPageStrip()` | ~5100 | 底部页码缩略图 |
| `render()` | ~5134 | 全量重渲染入口 |
| 事件监听绑定 | ~5140+ | 模板切换、主题选择、导出按钮等 |
| 导出逻辑 | ~5293 | 单页/全部/长图三种导出 |

## 架构模式

### 数据流
```
用户操作 → 修改 state/page/block 数据 → pushHistory() → renderEditorFields() + renderCard()
```

### 关键约定

1. **块（Block）生命周期**：
   - 注册：`BLOCK_TYPES` 加中文名 → `makeBlock()` 加 case → `BLOCK_GROUPS` 分组
   - 编辑：`renderEditorFields_raw()` 里 `switch(blk.type)` 加表单 HTML
   - 绑定：`renderEditorFields()` 里加 `addEventListener` 事件
   - 渲染：`renderBlock()` 里 `switch(blk.type)` 加卡片输出 HTML
   - 兼容：`migrateBlock()` 加字段兜底

2. **模板（Template）** 共 6 种：`text` / `imageText` / `dataCard` / `dashboard` / `cover` / `imageOnly`
   - 编辑面板分支在 `renderEditorFields_raw()` 
   - 卡片输出分支在 `renderCard()`
   - `dataCard` 有独立的表格系统（`page.tableHeaders/tableRows`，字符串数组），与 `table` 块类型（对象数组 `{text,rs,cs}`）数据结构不同

3. **事件绑定**：混合使用 inline `onclick="App.xxx()"` 和 `addEventListener`。编辑器表单的 input 事件在 `renderEditorFields()` 后半段通过 `bnd()` 辅助函数批量绑定。

4. **撤销系统**：`pushHistory()` 深拷贝整个 `state`（含所有页面和块）。`pushHistoryDebounced`（700ms）用于打字输入。栈深 40。

5. **内联样式函数**：`escAttr()` 防 XSS；`esc()` HTML 转义；`inl()` 行内 markdown；`parseBody()` 段落 markdown。

## 28 种块类型

`text` `bullets` `numbered` `checklist` `highlight` `quote` `infocard` `keystat` `herostat` `pullquote` `metric` `ranking` `progress` `kpigrid` `sparkcard` `compare` `callout` `tldr` `source` `panel` `tags` `grid` `divider` `image` `sectionHeader` `gallery` `textWrap` `disclaimer` `qrcode` `table`

## State 结构

```js
state = {
  pages: [Page],           // 页面数组
  currentPageIndex: 0,
  theme: 'terracotta',     // 主题色
  fontSize: '',            // 全局字号
  fontMood: '',            // 气质
  density: '',             // 紧凑/标准/宽松
  authorName/authorTitle/avatar/footerCta/footerStyle,  // 页脚
  typoPreset: '',          // 排版预设
  dataDate: '',            // 数据日期
  seriesName/seriesIndex/seriesTotal,  // 系列信息
  cardW/cardH,             // 卡片尺寸
  _uiFolded: Set,          // 编辑器折叠状态
}

Page = {
  id, template, category, title, subtitle, blocks: [Block],
  image, imageFit, imageH, imageSplit, splitW, cropX, cropY,
  titleAlign, bgStyle,
  // cover 专用
  coverTag, coverDeco, coverVariant, coverIndex,
  // dashboard 专用
  greeting, dashDate, dashRange,
  // imageOnly 专用
  fullImgCaption, fullImgFit,
  // dataCard 专用
  heroValue, heroLabel, tableHeaders, tableRows, tableNote,
  tableTitle, tableStyle, tableLayout, numberColor, highlightRows,
  colAligns, colWidths,
}
```

## 改动规范

### 新增块类型 checklist
1. `BLOCK_TYPES` 加键值对
2. `makeBlock()` 加 case + 默认数据结构
3. `migrateBlock()` 加字段兜底
4. `BLOCK_GROUPS` 把新块归入合适的分组
5. `renderEditorFields_raw()` 的 block switch 加编辑表单
6. `renderEditorFields()` 的事件绑定段加 addEventListener
7. `renderBlock()` 加渲染 case
8. CSS 加 `.cb-xxx` 样式

### 修改现有块
- 找到块的 4 个触点：makeBlock → migrateBlock → editor form → renderBlock
- 修改字段时务必在 `migrateBlock()` 加兜底，否则旧存档打开会报错

### 新增主题色
- `THEMES` 数组加主题名
- CSS 加 `.card.theme-xxx` 变量块（需定义 `--bg1/--bg2/--accent/--title/--body/--sub/--line/--mod/--shad/--accent-m/--accent-l`）

### 新增模板
- `createPage()` 的默认字段已包含所有模板共用属性
- `renderEditorFields_raw()` 加编辑面板分支
- `renderCard()` 加输出分支
- 模板 tab 在 HTML `<div class="template-tabs">` 中

## 验证方法

```bash
# JS 语法校验（提取 script 块后 new Function 检查）
node -e "const fs=require('fs'); const html=fs.readFileSync('index.html','utf8'); const m=html.match(/<script(?:\s[^>]*)?>[\s\S]*?<\/script>/gi); let ok=true; m.forEach((s,i)=>{if(s.includes('src='))return; const code=s.replace(/<\/?script[^>]*>/gi,''); try{new Function(code)}catch(e){console.error('Script block',i,e.message);ok=false}}); if(ok)console.log('JS syntax OK')"
```

浏览器测试重点：
- 切换 6 种模板均无报错
- 添加所有 28 种块类型
- 旧存档 JSON 打开无字段缺失（migrateBlock 兜底）
- 导出 PNG 正常
- Ctrl+Z 撤销恢复

## 设计原则

- **A 股配色**：正数红（`#d63031`）、负数绿（`#1e8a3e`），符合国内投资者习惯
- **基金合规**：免责声明块、数据来源块是行业刚需
- **单文件**：不拆分，不引入构建工具，保持可直接双击打开
- **向后兼容**：任何字段变更必须在 `migrateBlock()` 兜底
