# CLAUDE.md — DLAG 场景资料图库 · Location Scout Gallery

> 本文件记录了该项目的架构、数据结构、已实现功能及修改规范。每次新对话开始前请先阅读此文件。

---

## 项目概述

**名称**：DLAG 场景资料图库（Scene Reference Gallery）  
**用途**：剧组勘景参考图库，供美术/制片团队浏览、筛选、标注拍摄候选地点  
**技术栈**：纯静态 HTML（单文件），无框架，无构建工具，无后端  
**运行方式**：直接用 Chrome 打开 `index.html`（双击或 `open index.html`）  
**GitHub 仓库**：`https://github.com/lexi-44/DLAG-location_scout.git`，主分支 `main`

---

## 文件结构

```
DLAG-location_scout/
├── index.html                  # 唯一的应用文件（CSS + HTML + JS 全部内嵌）
├── CLAUDE.md                   # 本文件
├── images/                     # 原始高清图片（435 张，按场景分文件夹）
│   ├── small_road/             # 7 张
│   ├── road_branch/            # 15 张
│   ├── urban_highway/          # 1 张
│   ├── highway/                # 11 张
│   ├── tunnel/                 # 8 张
│   ├── highway_service/        # 84 张
│   ├── parking_lot/            # 11 张
│   ├── wasteland/              # 15 张
│   ├── factory/                # 48 张
│   ├── warehouse/              # 38 张
│   ├── yingzi_house/           # 49 张
│   ├── ren_house/              # 20 张
│   ├── wang_villa/             # 53 张
│   ├── yingzi_school/          # 16 张
│   ├── arcade/                 # 20 张
│   ├── bookstore/              # 24 张
│   └── luna_park/              # 14 张
├── thumbs/                     # 缩略图（结构与 images/ 完全镜像，.jpg 格式）
├── data/
│   ├── metadata.json           # 已上传图片的元数据数据库（JSON 数组）
│   ├── uploaded_images/        # 用户从网页上传的图片（{uuid}.ext 命名）
│   ├── processed_images/       # 历史批量提取的图片（UUID 命名，勿删）
│   └── raw_docx/               # 原始 Word 勘景文档（仅存档，不影响网页）
└── new/                        # 待处理的新 Word 文档（仅存档）
```

---

## 数据架构

### 层级结构（三层）

```
大类 Group（5个）
  └── 场景 Category（18个）
        └── 地点 Location
              └── 具体地点 Instance（同名地点用 #1/#2 区分）
                    └── 图片 Image
```

| 大类 key | 中文名 | 包含场景 |
|---|---|---|
| `outdoor_road` | 道路与公路类 🛣️ | small_road, road_branch, urban_highway, highway, tunnel, highway_service |
| `exterior_open` | 户外开放空间 🌾 | parking_lot, wasteland |
| `industrial` | 工业与物流 🏭 | factory, warehouse |
| `residential` | 居住建筑 🏠 | yingzi_house, ren_house, wang_villa |
| `public` | 公共建筑 🏫 | yingzi_school, arcade, bookstore, luna_park |

### 主数据（硬编码）

原有 435 张图片的数据定义在 `index.html` 的 `const DATA` 对象中（约第 502 行，单行 JSON）：

```javascript
const DATA = {
  groups: { /* 5个大类 */ },
  categories: {
    "small_road": {
      key, name_zh, name_en, source, image_count, instance_count,
      locations: [{
        title, title_en, base, location,
        instances: [{
          label,   // "" 或 "#1", "#2"
          seq,
          desc,    // 详细描述文字
          images: [{ src, filename, size, thumb }]
        }]
      }]
    },
    // ... 其余17个场景
  }
}
```

**修改 DATA 的注意事项**：整个 DATA 是一行 JSON，嵌在 `<script>` 中。修改时务必保持 JSON 合法，可用 `JSON.parse(DATA)` 在浏览器控制台验证。

### 上传图片元数据（`data/metadata.json`）

JSON 数组，每条记录格式：

```json
{
  "id": "uuid-v4",
  "path": "data/uploaded_images/{uuid}.ext",
  "levels": ["大类中文名", "场景中文名", "地点名"],
  "name": "用户填写的图片名",
  "notes": "备注",
  "status": "未筛选",
  "comments": [{ "text": "内容", "created_at": "ISO时间戳" }],
  "created_at": "ISO时间戳"
}
```

### 用户状态（localStorage）

键：`dlag_photo_data`，值为对象，key = 图片 ID（src 路径或 uploaded path）：

```json
{
  "images/small_road/small_road_01.jpeg": {
    "status": "未筛选 | 备选 | 确定",
    "comments": [{ "text": "...", "created_at": "..." }],
    "deleted": true   // 可选，为 true 时从图库隐藏
  }
}
```

状态对所有图片生效（含原有 435 张 + 新上传图片），页面刷新后保留。

---

## 已实现功能清单

### 1. 图库浏览
- 三层层级展示：大类 → 场景 → 地点 → 具体地点 → 图片网格
- 实时搜索（搜索地名、场景名、文件名）
- 大类筛选 chip（全部 All / 道路 / 户外 / 工业 / 居住 / 公共）
- 目录索引（TOC，可收起）
- 滚动后 header 自动收缩
- 图片懒加载

### 2. 状态筛选视图
- 第二行 chip：**全部 | 未筛选 | 备选 | 确定**
- 筛选当前 `currentFilter.status`，控制 render() 的 `visibleImages` 过滤
- 图片缩略图右上角显示状态徽章（备 = 橙色，✓ = 绿色）

### 3. 大图灯箱（Lightbox）
- 点击缩略图打开，← → 键或按钮翻页，Esc 关闭
- 图片下方状态操作栏：
  - 当前状态标签（未筛选 / 备选 / 确定）
  - **加入备选** 按钮（再次点击取消）
  - **确定** 按钮（再次点击取消）
  - **重置** 按钮（恢复未筛选）
  - **🗑 删除** 按钮（红色，见下）
- 评论区：显示已有评论列表（带时间）+ 输入框（Enter 发送）
- 状态和评论即时写入 localStorage，无需保存

### 4. 上传图片
- 导览栏右侧 **＋ 上传图片** 按钮
- 上传弹窗含：
  - 图片选择（点击或拖拽）+ 即时预览
  - 三级级联下拉（大类 → 场景 → 地点，每级可选"自定义…"并输入）
  - 图片名称 + 备注文本框
- 保存流程：
  1. 存入 IndexedDB（blob，页面刷新后可重建 blob URL）
  2. 若浏览器支持 File System Access API（Chrome）：请求项目根目录权限，写入 `data/uploaded_images/{uuid}.ext`，更新 `data/metadata.json`
  3. 上传后图片立即出现在对应类别下（"已上传 · Uploaded" 标签区）
  4. Toast 弹出 git 同步指令

### 5. 删除图片
- 灯箱中点击 **🗑 删除** → 弹出确认对话框（显示图片名）
- 确认后：
  - localStorage 标记 `deleted: true`，图片立即从图库消失（刷新后仍然消失）
  - 若为上传图片：从 IndexedDB 和 `data/uploaded_images/` 删除，更新 `metadata.json`
  - Toast 弹出对应 git 命令：
    - 上传图片：`git rm data/uploaded_images/{file} && git add data/metadata.json && git commit -m "..." && git push`
    - 原有图片：`git rm images/{cat}/{file} thumbs/{cat}/{file} && git commit && git push`

### 6. GitHub 同步机制
浏览器无法直接执行 git 命令，同步流程为：
1. 通过 File System Access API 写入/删除本地文件（Chrome 支持，Safari/Firefox 降级到仅 IndexedDB）
2. 页面弹出 toast，显示需要手动运行的 git 命令
3. 用户在终端执行即可同步至 GitHub

---

## 代码结构（`index.html`，约 1421 行）

| 行号范围 | 内容 |
|---|---|
| 1–50 | CSS 变量、基础样式 |
| 51–233 | 组件样式（header、grid、tile、lightbox、TOC）|
| 234–430 | 新增样式（状态筛选、徽章、灯箱面板、上传弹窗、删除确认、Toast）|
| 431–500 | HTML：header（搜索 + 大类 chip + 上传按钮 + 状态 chip + TOC）|
| 501–535 | HTML：main 内容区、灯箱（含状态/评论面板）|
| 536–550 | HTML：上传弹窗、Toast、删除确认弹窗 |
| 551–560 | `<script>` 开始，`const DATA = {...}` |
| 561–600 | 工具函数：`escapeHtml`, `slug`, `locImageCount` |
| 601–660 | localStorage 工具：`getPhotoData`, `setStatus`, `addComment`, `markDeleted`, `isDeleted` |
| 661–660 | UUID 生成器 `genUUID()` |
| 661–730 | IndexedDB 工具：`openIDB`, `idbSave`, `idbGet`, `idbDelete` |
| 731–760 | `loadUploadedImages()` + Toast 工具 `showToast()` |
| 761–880 | `makeTileHtml()` + `render(filter)` |
| 881–940 | `buildHeader()` + 搜索/状态 chip 事件 |
| 941–1060 | 灯箱逻辑：`openLightbox`, `showLightboxImage`, `refreshLightboxStatus`, `refreshLightboxComments`, `closeLightbox` |
| 1061–1100 | 灯箱按钮事件（状态、评论、键盘） |
| 1101–1330 | 上传弹窗逻辑（`openUploadModal`, 级联选择, `handleFileSelected`, FSA 写入）|
| 1331–1420 | 删除逻辑（`deletePhoto`, 确认弹窗），初始化调用 |

---

## 修改指南

### 添加新场景/大类

1. 在 `const DATA` 的 `groups` 中添加新大类（或在已有大类的 `items` 数组中追加场景 key）
2. 在 `categories` 中添加新场景对象，结构参考已有条目
3. 将图片放入 `images/{scene_key}/`，缩略图放入 `thumbs/{scene_key}/`
4. 图片命名规范：`{scene_key}_{02d}.jpeg`（如 `new_scene_01.jpeg`）
5. commit & push

### 修改现有图片描述

直接编辑 `const DATA` 中对应 location 的 `desc` 字段。

### 调整 UI 样式

CSS 全部在 `<style>` 标签内（行 1–430）。核心设计变量在 `:root`（行 8–18）：
- `--accent: #c45a2c`（主橙色）
- `--bg: #f6f5f1`（页面背景米白色）
- `--ink: #1f1d1a`（主文字色）

### 添加/修改上传图片的分类

上传弹窗的大类和场景下拉从 `DATA.groups` 和 `DATA.categories` 动态读取，无需手动维护。

---

## 浏览器兼容性

| 功能 | Chrome | Safari | Firefox |
|---|---|---|---|
| 基础浏览/筛选/灯箱 | ✅ | ✅ | ✅ |
| 状态/评论（localStorage）| ✅ | ✅ | ✅ |
| 上传（IndexedDB 存储）| ✅ | ✅ | ✅ |
| 上传（写入磁盘，FSA API）| ✅ | ❌ | ❌ |
| 删除（从磁盘删除，FSA API）| ✅ | ❌ | ❌ |

**推荐使用 Chrome**。Safari/Firefox 下上传和删除仍然生效（存 IndexedDB），但不会写入磁盘，需手动操作文件后运行 git 命令。

---

## 常见操作 Cheatsheet

```bash
# 查看当前状态
git status

# 上传新图片后同步
git add data/uploaded_images/ data/metadata.json
git commit -m "Add uploaded photos"
git push

# 删除图片后同步（网页会给出具体命令）
git rm images/{category}/{filename}
git rm thumbs/{category}/{filename}.jpg
git commit -m "Remove photo: {filename}"
git push

# 修改 index.html 后同步
git add index.html
git commit -m "Update gallery"
git push

# 强制刷新浏览器缓存（Mac Chrome）
Cmd + Shift + R
```

---

## 注意事项

1. **不要用 `file://` 以外的协议打开**（无需本地服务器），Chrome 直接双击打开即可
2. **DATA 是单行 JSON**，编辑时小心不要破坏 JSON 格式，每次修改后在浏览器控制台执行 `JSON.parse(DATA)` 验证
3. **localStorage 数据是每台电脑独立的**，状态和评论不会随 git push 同步给他人——如需共享状态，需要后端或将 metadata.json 作为状态存储
4. **IndexedDB 中的上传图片**在 FSA 写入磁盘成功后才会真正持久化到仓库，否则仅存在于当前浏览器
5. **删除原有 DATA 图片**只是在当前浏览器隐藏（localStorage），图片文件和 DATA 定义仍然存在，需手动 git rm
