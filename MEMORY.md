# MEMORY.md - 项目长期记忆

## 项目：Daily Booking Dashboard
- **仓库**: https://github.com/Elaine-Teh/daily-booking-dashboard
- **GitHub Pages**: https://elaine-teh.github.io/daily-booking-dashboard/
- **生成脚本**: `generate_daily_booking_dashboard.py`
- **架构**: 双文件架构 — `index.html`（UI 壳 ~31KB）+ `db_data.json`（数据异步加载 ~27MB）
- **Chart.js**: CDN 引入（不内联）
- **数据源**: `daily booking.xlsx` 从 SFTP (10.5.4.2:6622) `Master Data-Bob/` 自动下载；POR Region 映射硬编码在脚本中（69 个，不再依赖 Income Data Base-Marketing.xlsx）
- **输出**: `generate_daily_booking_dashboard.py` 生成 `db_data.json`（忽略其生成的 HTML 模板）；部署由 `deploy_data.py` 完成
- **部署流程**: (1) 运行 `generate_*.py` → (2) 运行 `deploy_data.py`（仅推送 db_data.json + 更新 version.txt，**绝对不碰 index.html**）
- **⚠️ 禁止直接推送  generate_*.py 生成的 index.html**：它不含手动修复（SUL_YN filter、collapsible POL TOTAL、ETB 筛选逻辑等），会覆盖线上功能
- **⚠️ deploy_data.py 已于 2026-06-05 重写为 DATA ONLY + Git Data API 模式**：只推 db_data.json + version.txt，永远不修改 index.html；使用 Git Data API（blob→tree→commit→ref）避开 Contents API 对 30MB+ 大文件的 403 限制
- **数据规模**: ~57,000 条 booking 记录，44 Lanes，66 POLs，87 DELs，859 CULs

### 2026-05-22 修复记录
- **问题**: 原 `index.html` 有 16.7 MB（内联了完整 Chart.js + 数据），浏览器无法加载
- **根因**: GitHub 上的 `index.html` 不是由生成脚本产生的，且有人把 Chart.js 内联嵌入了 HTML
- **修复**: 重构为 fetch 异步加载 `db_data.json` 的双文件架构，index.html 降至 ~31KB
- **验证**: 页面已在 preview 中打开，数据正常加载

### 2026-05-22 二次修复（缓存问题）
- **问题**: 页面仍显示 "Loading dashboard data..."，数据无法加载
- **根因**: GitHub Pages CDN 缓存 + 企业网络可能限制大文件下载（27MB JSON）
- **修复**: 
  - fetch URL 添加 `?v=2` 缓存清除参数
  - 添加 `AbortController` 30 秒超时机制
  - 增强错误提示（区分超时 vs 网络错误）
- **状态**: 已推送更新，待 Elaine 刷新验证

### 2026-05-22 功能增强
- **POL 排序**: Summary table POL 列按字母顺序排列
- **ETD 日期范围筛选**: 新增 From/To 日期选择器，自动设置 min/max
- **级联筛选（Lane → CUL/POL/DEL）**: 选中 Lane 后，其他筛选项只显示该 Lane 下的选项
- **级联筛选（ETD → CUL/POL/DEL）**: 选择 ETD 范围后，CUL/POL/DEL 只显示该日期范围内有数据的选项
- **统一级联函数** `cascadeFilters()`: Lane 选择 + ETD 范围共同决定 CUL/POL/DEL 可用选项
- **ETD 范围自适应**: 选中 Lane 后，ETD 日期选择器自动调整为该 Lane 的实际数据日期范围，避免选到无数据的区间
- **Bug 修复 (2026-05-22)**: `cascadeFilters()` 中 DEL 字段名错误 `r.del` → `r.del_port`，导致 ETD/Lane 筛选后数据全部消失
- **Bug 修复 (2026-05-22)**: Reset All 按钮只重置 Lane 和 ETD，CUL/POL/DEL 仍保持之前选择。根因是用了 `updateItems()` 而非 `setAll()`，修复后改为 `setAll()`
- **Bug 修复 (2026-05-23)**: `LANE_ETD_MAP` 未从 payload 加载（漏写 `LANE_ETD_MAP = payload.lane_etd_map || {}`），导致 `onLaneChange` 访问时抛出 TypeError，Lane 筛选选择后 cascadeFilters/refreshAll 从未执行，数据始终不更新
