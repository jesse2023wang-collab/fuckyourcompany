# FuckYourCompany · 社保公积金欠缴计算器

> 社保公积金欠缴计算器 · 守护你的每一分钱  
> 当前版本：**V 0.0.1**

单文件 HTML 应用，双击即可运行。支持工资记录、社保基数、公积金基数的逐月录入与欠缴计算，支持导出 PDF / 图片 / 保存网页副本。

---

## 一、功能更新

### 基本设置
- 选择城市（上海示例内置）
- 入职日期 / 计算结束日期
- 查看范围：全部 / 仅社保 / 仅公积金
- 自定义社保 / 公积金费率面板
- 入职日期填写后自动生成一条工资、社保基数、公积金基数记录

### 记录管理
- 工资记录、社保基数记录、公积金基数记录均按日期从旧到新排序
- 单条基数记录时，计算始终按该基数计算（不再把最后一个月按工资基数走）
- 增删记录后自动同步状态

### 计算与结果
- 点击「计算欠缴金额」后：
  - 汇总卡片：个人欠缴 / 公司欠缴 / 合计
  - 分类汇总对比表
  - 逐月明细表（全部展开，无滚动限制）
  - 逐月明细中连续相同月份合并为一行，用 `; ` 分隔
- 查看范围切换时，欠缴金额仅显示对应范围（仅社保→只社保，仅公积金→只公积金）
- 总缴对比图、月度差额趋势图位于「计算欠缴金额」按钮下方

### 导出功能
- 导出 PDF：A4 分页，页间 15px 重叠避免数字跨页被切
- 导出图片（JPG）：html2canvas 截图，自动裁掉底部多余空白
- 保存网页：以 `FuckYourCompany_YYYY-MM-DD.html` 命名，内嵌完整状态 + 图表快照 + 结果 HTML，打开即可查看完整内容，不依赖 localStorage
- 导出时页面无闪烁、无拉宽变形

---

## 二、版本日志

### V 0.0.1
- 初始版本
- 基本设置 / 工资记录 / 社保基数 / 公积金基数录入与排序
- 欠缴金额计算（个人 / 公司 / 合计）
- 逐月明细全部展开，连续相同月份合并显示
- 总缴对比图 + 月度差额趋势图
- 导出 PDF / 导出图片 / 保存网页
- 导出时数字完整显示（无底部裁切）、无页面闪烁、无多余空白
- 保存网页不依赖缓存，内嵌完整状态与图表快照

---

## 三、经验教训

### 1. html2canvas 导出数字底部被裁切
**现象**：导出的图片中，输入框内数字的底部约 1/4 被切掉。  
**根因**：Tailwind 的 `overflow-hidden`（用于卡片圆角）+ html2canvas 渲染 `<input>` 比 Chrome 原生高 2-3px，导致最后一行内容的下行像素被裁。  
**解决**：
- 不在真实 DOM 上改样式（避免页面闪烁），只在 html2canvas 的 `onclone` 回调里对克隆 iframe 打 inline style patch
- `<section>`：`overflow: visible` + `padding-bottom +10px`
- `<input>`：`box-sizing: content-box` + 上下对称各 `+4px` padding + `vertical-align: top` + `line-height: 1.6`
- 文本容器：`line-height: 1.66` + `padding-bottom: 2px`

### 2. 导出前页面「拉宽 / 拉长」闪烁
**现象**：点击导出时页面瞬间变形。  
**根因**：在真实 DOM 上切 `.exporting` class（main 宽度从 ~1152px 拉到 1440px）触发整页重排，patchSubtree 的测量在重排前完成，对不上重排后的布局。  
**解决**：真实 DOM 完全不动，所有 patch 只在离线克隆 iframe 内执行。iframe 宽度用真实 DOM 实测的 `preflightW`（非硬编码 1440），保证 1:1 布局。

### 3. 导出图片底部多出一大段空白
**现象**：导出的 JPG / PDF 底部有数百像素纯背景色空白。  
**根因**：传了 `width` 但没传 `height` 给 html2canvas，画布高度被强行填到 `windowHeight`。  
**解决**：
- 不传 `width` / `height`，让 html2canvas 按克隆节点实际渲染尺寸出图
- 像素级兜底：`_trimCanvasBottomBlank` 从最后一行往上扫描，找到最后一个有非背景色像素的行，裁到 `lastNonBgRow + 2`

### 4. 保存网页后打开内容为空
**现象**：保存的 HTML 打开后基本设置参数、图表、结果区全部为空。  
**根因**：
- `outerHTML` 只序列化 HTML attribute，不序列化用户在 input/select 中实时编辑的 `.value`
- Chart.js 画的内容在 canvas 显存里，`<canvas>` 标签本身是空壳
- 保存的 HTML 依赖 localStorage 读数据  
**解决**：
- 保存前 `_serializeAllFormValuesToAttrs()` 把所有 `.value` 写回 attribute
- `_materializeChartsWithImg()` 用 `canvas.toDataURL()` 把图表转为 `<img>`
- 把 state + chartsHtml + resultsHtml 打包成 JSON 注入 `<script type="application/json">`
- 打开时检测到内嵌 JSON 就直接用，跳过 localStorage

### 5. 计算逻辑：单条基数记录
**问题**：只有一条社保 / 公积金基数记录时，计算把最后一个月按工资基数走。  
**解决**：单条记录时始终按该基数计算，不再回退到工资基数。

### 6. 查看范围切换
**问题**：切换查看范围后欠缴金额未更新。  
**解决**：切换时重新计算并仅显示对应范围的欠缴金额。
