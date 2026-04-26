# 藤校巡游页面 · 构建日志

记录本项目从零到完成的完整构建过程，每次重大更新同步记录。

## 项目信息

- **线上地址**: https://wesleymiao.github.io/ivy-league-tour/
- **GitHub 仓库**: https://github.com/wesleymiao/ivy-league-tour
- **部署方式**: GitHub Pages + GitHub Actions (自动部署)
- **技术栈**: 纯前端单页 HTML，Leaflet.js 地图，OSRM 路线规划 API

---

## 构建过程

### v1 — 静态信息图 (2026-04-26)

**需求**: Wesley 想要一张藤校巡游7天行程的直观图示。

**实现**:
- 用纯 HTML/CSS 画了一个深色主题信息图
- 左侧用 SVG 手绘了一个美国东北部简略地图，标注8所藤校位置
- 右侧是7天行程时间线 + 底部统计数据（8校/7天/~1500km/¥3万）
- 用 Playwright 截图生成 PNG

**部署**:
- `gh repo create ivy-league-tour --public` 创建仓库
- 添加 `.github/workflows/pages.yml` 配置 GitHub Actions 自动部署到 GitHub Pages
- 工作流使用 `actions/upload-pages-artifact` + `actions/deploy-pages`

**验证**: Playwright 截图线上 URL 确认部署成功。

> commit: `0b7c239` Ivy League 7-day tour infographic
> commit: `e25cd4e` Add GitHub Pages workflow

---

### v2 — 真实地图 (2026-04-26)

**需求**: Wesley 要求用真实地图替代手绘 SVG。

**实现**:
- 引入 Leaflet.js + OpenStreetMap 瓦片
- 8所藤校用精确经纬度定位，自定义 DivIcon 标记
- 虚线连接路线，地图可交互（缩放/拖拽）
- 响应式布局适配手机

**踩坑**:
- 首次用 CARTO 暗色底图 (`dark_all`)，但瓦片在 Playwright 截图时加载不出来（灰色背景）
- 换成 OSM 标准瓦片 (`tile.openstreetmap.org`) 解决

> commit: `8138921` Use real map with Leaflet + OpenStreetMap
> commit: `3fa8f51` Light map, Boston start, km labels on each route edge
> commit: `ee47db8` Switch to OSM tiles for better reliability

---

### v3 — 多方案切换 (2026-04-26)

**需求**: Wesley 希望不强求所有藤校，提供多种优化方案可切换比较。

**实现**:
- 设计4个方案：全部8校(7天) / 去掉康奈尔(6天) / 精简6校(5天⭐推荐) / 精华4校(4天)
- 顶部 Tab 切换，点击后地图路线、学校标记、行程、统计全部联动更新
- 未选中的学校灰显（`opacity: 0.25`），保留位置参考
- 每个方案显示相比全量方案省了多少公里

> commit: `4b5e59c` Add 4 plan tabs: full 8, no Cornell, compact 6, elite 4

---

### v4 — 飞入飞出机场 (2026-04-26)

**需求**: 每个方案需要标注飞入和飞出的城市/机场。

**实现**:
- 定义4个机场数据（BOS/PHL/JFK/SYR）含经纬度
- 每个方案配置 `flyIn` / `flyOut` 机场代码
- 侧边栏显示飞入/飞出机场中英文信息
- 地图上用 🛬/🛫 emoji 标记机场位置

> commit: `9af829d` Add fly-in/fly-out airports to each plan

---

### v5 — 真实公路路线 (2026-04-26)

**需求**: 地图上的直线距离和标注的驾车公里数不匹配，需要沿实际公路画线。

**实现**:
- 接入 OSRM (Open Source Routing Machine) 免费 API
- 每段路线调用 `router.project-osrm.org/route/v1/driving/` 获取真实驾车路径
- 返回 GeoJSON 格式的路线坐标，用 Leaflet polyline 绑定
- 同时获取真实驾车距离(km)和时间(h)，显示在路线中点
- 保留直线 fallback，OSRM 请求失败时退回直线+估算距离

**技术细节**:
- OSRM API 格式: `https://router.project-osrm.org/route/v1/driving/{lng1},{lat1};{lng2},{lat2}?overview=full&geometries=geojson`
- 距离标签放在路线坐标数组中点位置，视觉上贴合实际路线

> commit: `59269c0` Use OSRM real driving routes with actual distances and durations

---

### v6 — 费用明细 (2026-04-26)

**需求**: 每个方案需要详细的费用拆分，不仅仅是总数。

**实现**:
- 每个方案包含8项费用明细：机票、租车、油费、住宿、餐饮、停车过路费、门票、签证保险
- 每项标注金额(¥) + 计算备注（如"5天×$80/天"）
- 底部显示合计总费用
- 加注"价格为暑假旺季估算"说明

> commit: `ee1bf2f` Add detailed cost breakdown for each plan

---

### v7 — 2026年7月具体化 (2026-04-26)

**需求**: 按2026年7月第一周具体化所有日期和开销。

**实现**:
- 每天标注具体日期和星期（如 "7/1 周三"）
- 7月4日是美国独立日 🇺🇸，方案中特别标注可以看烟花 🎆
- 费用根据暑假旺季调整：住宿均价 $140/晚，租车 $80/天+异地还车费
- 机票按开口程（BOS进/PHL或JFK出）分别估价
- 不同方案日期范围：全量6/29-7/5，方案B 7/1-7/5，方案C 7/1-7/4

> commit: `3b00a48` Specify July 2026 dates, detailed itemized costs, Independence Day highlights

---

## 技术架构

```
index.html (单文件应用)
├── Leaflet.js — 交互式地图渲染
│   └── OpenStreetMap 瓦片 — 底图
├── OSRM API — 真实驾车路线和距离
├── 自定义 DivIcon — 学校标记、机场标记、距离标签
├── 方案数据 (plans[]) — 路线/行程/费用/机场
└── renderPlan() — 切换方案时更新地图+侧边栏
```

### 部署流程

```
本地编辑 index.html
  → git push
  → GitHub Actions 自动触发
  → actions/upload-pages-artifact 打包
  → actions/deploy-pages 部署
  → https://wesleymiao.github.io/ivy-league-tour/ 更新
```

### 验证方式

每次部署后用 Playwright 截图线上 URL 验证：
```bash
cd /workspace/group/stock-compare
npx playwright screenshot --viewport-size="1400,900" --wait-for-timeout 8000 \
  "https://wesleymiao.github.io/ivy-league-tour/" /tmp/verify.png
```

---

## 更新日志模板

后续更新请在上方添加新章节，格式：

```markdown
### vN — 简要描述 (日期)

**需求**: 用户原始需求

**实现**: 技术方案和关键决策

**踩坑**（如有）: 遇到的问题和解决方案

> commit: `hash` 提交信息
```
