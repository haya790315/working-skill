---
name: doc-html
description: 分析当前项目代码库，生成深色科技感 HTML 文档页面，帮助快速理解项目结构和数据流。当用户运行 /doc-html、想快速理解项目、需要代码结构可视化概览，或要求为某个模块/流程生成文档时使用。支持自由文本参数：路径（/doc-html src/app/Services）、主题（/doc-html 解释图片上传流程）或问题（/doc-html 为什么不把图片存进 SQL）。
disable-model-invocation: true
---

# me:doc-html — 项目文档生成器

## 快速开始

```
/doc-html                              # 整个项目总览
/doc-html src/app/Services             # 聚焦某个目录
/doc-html 解释图片上传到 GCS 的流程      # 主题聚焦
/doc-html 认证流程                      # 特定功能
```

---

## Step 1 — 解析参数意图

- **无参数** → 整个项目总览
- **路径参数**（`src/app/Services`）→ 聚焦分析该目录
- **自然语言**（`解释图片上传流程`）→ 识别相关模块，聚焦分析

---

## Step 2 — 技术栈探测 + 代码探索

### 2-A：技术栈探测（首先执行）

用 `find` 和 `ls` 判断项目类型，决定阅读哪些文件：

| 技术栈 | 判断依据 | 核心文件 |
|--------|---------|---------|
| **Laravel / PHP** | `artisan`、`composer.json` | routes/*.php、app/Http/Controllers/、app/Services/、app/Models/ |
| **Django / Python** | `manage.py`、`requirements.txt` | urls.py、views.py、serializers.py、models.py、tasks.py |
| **NestJS / Node.js** | `nest-cli.json`、`package.json` | src/**/*.module.ts、*.controller.ts、*.service.ts、*.entity.ts |
| **Express / Fastify** | `package.json` (no nest) | routes/、middleware/、models/、services/ |
| **React / Vue** | `src/App.tsx` 或 `src/App.vue` | src/pages/、src/components/、src/stores/、src/api/ |
| **Terraform / IaC** | `*.tf` 文件、`modules/` 目录 | modules/*/main.tf、modules/*/variables.tf、envs/*/main.tf |
| **Go** | `go.mod` | cmd/、internal/、pkg/、handler/、service/ |
| **Rails / Ruby** | `Gemfile`、`config/routes.rb` | app/controllers/、app/models/、app/services/ |

### 2-B：并行读取核心文件

根据探测到的技术栈，**并行**读取：

1. **路由 / 入口** — 了解系统对外暴露什么（routes、urls、controller 方法签名）
2. **业务逻辑层** — 完整内容（Services、use cases、interactors）
3. **数据层** — 完整内容（Models、Entities、Schema、Terraform resources）
4. **配置/环境** — 了解 dev vs prod 的差异（.env.example、config/*.yaml、envs/*/main.tf）
5. **README / CLAUDE.md / CONTEXT.md** — 项目背景和架构决策

### 2-C：识别关键信息

从阅读的文件中提取：
- **数据流方向** — 请求从哪进来，经过哪些层，最终怎么响应
- **模块依赖关系** — 哪个 Service 调用哪个 Repository，哪个 Module 导入哪个
- **配置参数的实际值** — 不是变量名，而是真实值（读 .env.example、config、envs/）
- **环境差异** — dev 和 prod 在规格、开关、限制上有什么不同
- **核心方法 / 关键逻辑** — 最重要的 1-2 个方法，用于代码片段展示

---

## Step 3 — 决定输出模式

### 多页模式 ★ 首选（适合复杂项目）

**触发条件**（满足任意一条）：
- 项目有 5 个以上有明确职责边界的模块/服务/功能区域
- 用户明确要"每个模块单独页面"或"整个项目的文档"
- Terraform 项目（天然适合多页）

**输出结构**：
```
guide/{topic}/
  index.html          ← 总览页（架构图 + 模块/服务卡片网格）
  {module-a}.html     ← 每个模块/服务一个独立页面（300-600 行）
  {module-b}.html
  ...
```

### 单页模式（适合聚焦主题）

**触发条件**：
- 用户只问某一个流程或功能（"解释图片上传流程"）
- 项目较小，模块数 < 5
- 聚焦某个目录（`src/app/Services`）

**输出**：`guide/{filename}.html` 一个自包含 HTML 文件，包含 Sidebar TOC + 多个 section。

---

## Step 4 — 生成 HTML

### 4-A：index.html 总览页（多页模式）

```
Sidebar（260px，固定）
  项目名称 / Logo
  按功能分组的导航（链接到各模块页）

主内容区
  Hero（渐变大标题 + 技术栈 Badge + 一段描述）
  Stats Bar（模块数、文件数、API 数、环境数等 4 格统计）
  架构流程图（分层节点，每层颜色不同，节点可点击跳转模块页）
  依赖关系可视化（模块依赖图，CSS-only，仅在 index.html 上显示）
  模块/服务卡片网格（hover 有顶部彩条 + translateY(-4px)，点击跳转）
```

**架构图节点必须可点击**：`<a href="module.html" class="flow-box blue">...</a>`，让用户从架构图直接跳转对应模块页。

**Sidebar 分组建议（按项目类型）**：

| Laravel / PHP | Django / Python | NestJS | Terraform/GCP |
|--------------|----------------|--------|--------------|
| 路由层 | URL 路由 | API 层 | 入口层 |
| Controller 层 | View 层 | Controller 层 | 计算层 |
| Service 层 | Serializer 层 | Service 层 | 数据层 |
| Model 层 | Model 层 | Entity 层 | 安全/认证 |
| Job / Command | Task / Command | Guard / Pipe | 可观测性 |
| Middleware | Middleware | Middleware | 集成 |

### 4-B：模块/服务详情页（多页模式，每个模块一页）★ 核心

**固定页面骨架**：

```html
<nav class="sidebar">
  ← 返回总览按钮（href="index.html"）
  模块/服务名称（字体加粗）
  文件路径 / 命名空间（monospace 小字）
  TOC（本页所有 section 的锚点链接，IntersectionObserver 自动高亮）
</nav>

<main class="main-content">
  Breadcrumb（首页 > 分类 > 当前页）
  
  module-hero（id="overview"）
    icon + 大标题 + 副标题（类型/文件路径）+ 描述 + tag 列表
    border-left: 3px solid var(--accent)  ← 该模块特色颜色
  
  section#why      ❓ 为什么这样设计？（设计决策 + alert-box）
  section#flow     🔀 数据流 / 处理流程（flow-diagram）
  section#api      🌐 对外接口（如有，列出方法/端点）
  section#config   ⚙️ 配置 / 参数（info-grid）
  section#...      （根据模块类型添加的专属 section，见下方）
  section#deps     🔗 依赖关系（调用了谁，被谁调用）
  section#source   📁 文件位置（最后一个 section）

  <!-- 页面底部 -->
  related-pages（相关模块链接，2-4 个卡片）
  prev-next-nav（上一页 / 下一页导航栏）
</main>

<script> IntersectionObserver TOC 自动高亮 + 复制按钮逻辑 </script>
```

**按模块类型添加专属 section**：

| 模块类型 | 专属 section |
|---------|-------------|
| Controller / View | API 路由表（method badge + path + handler + 说明），请求/响应结构 |
| Service / Usecase | 核心方法列表 + 最重要方法的代码片段 |
| Model / Entity / ORM | 字段表（名称/类型/说明），关联关系图 |
| Job / Task / Command | 执行计划（cron/触发条件），参数说明，重试策略 |
| Middleware / Guard | 触发条件，处理逻辑，异常处理 |
| Repository / DAO | 查询方法列表，索引使用，N+1 风险说明 |
| 认证 / 权限模块 | 角色权限表，token 生命周期，认证流程图 |
| 队列 / 消息 | 消息格式，消费者关系，死信队列处理 |
| Terraform 模块 | 资源列表，服务账号/IAM，环境差异（dev vs prd）|
| 外部集成 | 调用方式，认证机制，失败重试，超时处理 |

**必须包含的 section（无论什么类型）**：

1. **#why** — 解释 1-3 个关键设计决策（为什么用这个库/模式/架构，不是说明"做了什么"）
2. **#flow** — 端到端流程图，数据从哪来、怎么处理、去哪里
3. **#source** — 文件路径、关键类/方法名、调用入口

---

## 设计系统 ★ 最重要

### 基础颜色变量（每页必须）

```css
:root {
  --bg-primary: #0a0e1a;
  --bg-card:    #1a2235;
  --bg-card-hover: #1e2a42;
  --accent:     #4285f4;           /* 当前模块主色，每页不同 */
  --accent-light: rgba(66,133,244,0.15);
  --text-primary:   #e2e8f0;
  --text-secondary: #94a3b8;
  --text-muted:     #64748b;
  --border:     #1e3a5f;
  --sidebar-width: 260px;
}
```

**每个模块/服务页面选用不同的 --accent 颜色（让页面有独特感）**：

| 模块职责 | accent | rgba 前缀 |
|---------|--------|---------|
| API / Web 服务 / Controller | `#34a853` 绿 | `rgba(52,168,83,` |
| 批处理 / Job / Command / Task | `#fbbc04` 黄 | `rgba(251,188,4,` |
| 数据库 / ORM / Repository | `#4285f4` 蓝 | `rgba(66,133,244,` |
| 缓存 / 队列 / 消息 | `#ea4335` 红 | `rgba(234,67,53,` |
| 认证 / 权限 / IAM / WIF | `#a855f7` 紫 | `rgba(168,85,247,` |
| 外部集成 / FTP / Webhook | `#f97316` 橙 | `rgba(249,115,22,` |
| 监控 / 日志 / 告警 | `#ea4335` 红 | `rgba(234,67,53,` |
| 存储 / 文件 / GCS / S3 | `#f97316` 橙 | `rgba(249,115,22,` |
| 容器 / 部署 / CI/CD | `#06b6d4` 青 | `rgba(6,182,212,` |
| Middleware / Guard / Filter | `#a855f7` 紫 | `rgba(168,85,247,` |

### 背景光晕（body::before，必须）

```css
body::before {
  content:''; position:fixed; inset:0;
  background: radial-gradient(ellipse at 30% 60%, rgba({accent-rgb},0.05) 0%, transparent 50%);
  pointer-events:none; z-index:0;
}
/* accent-rgb 对应上表的 rgba 前缀，如绿色用 rgba(52,168,83, */
```

### 行内代码（必须）

页面所有 `<code>` 标签统一应用以下样式，让行内代码在文字中一眼可辨：

```css
code {
  font-family: 'Courier New', monospace;
  font-size: 0.85em;
  background: rgba(255,255,255,0.08);
  color: #7ab4ff;
  padding: 2px 6px;
  border-radius: 4px;
  border: 1px solid rgba(255,255,255,0.1);
}
```

**用法**（写在段落文字中）：
```html
<p>当 <code>retention_days = 30</code> 时，Cloud Logging 自动清理超过 30 天的日志。</p>
```

### Sidebar（必须）

```css
.sidebar {
  position:fixed; top:0; left:0;
  width:var(--sidebar-width); height:100vh;
  background:rgba(17,24,39,0.95);
  backdrop-filter:blur(20px);
  border-right:1px solid var(--border);
  overflow-y:auto; z-index:100;
}
.sidebar::-webkit-scrollbar { width:4px }
.sidebar::-webkit-scrollbar-thumb { background:var(--border); border-radius:2px }

.toc-item {
  display:flex; align-items:center; gap:8px;
  padding:7px 10px; border-radius:7px;
  text-decoration:none; color:var(--text-secondary);
  font-size:12px; transition:all .2s; margin-bottom:2px;
}
.toc-item:hover  { background:var(--bg-card); color:var(--text-primary) }
.toc-item.active { background:var(--accent-light); color:var(--accent) }
```

### Module Hero（必须，每页顶部）

```css
.module-hero {
  display:flex; align-items:flex-start; gap:20px;
  background:var(--bg-card); border:1px solid var(--border);
  border-radius:16px; padding:28px; margin-bottom:32px;
  border-left:3px solid var(--accent);    /* ← 该模块特色颜色边框 */
}
.hero-icon { width:56px; height:56px; border-radius:12px;
  background:var(--accent-light); display:flex; align-items:center;
  justify-content:center; font-size:28px; flex-shrink:0; }
.hero-title   { font-size:26px; font-weight:800; color:var(--text-primary) }
.hero-service { font-size:12px; color:var(--accent); font-family:monospace }
.hero-desc    { font-size:14px; color:var(--text-secondary); line-height:1.7 }
.hero-tags    { display:flex; flex-wrap:wrap; gap:6px; margin-top:12px }
.tag { font-size:11px; padding:3px 10px; border-radius:4px;
  background:rgba(255,255,255,0.06); color:var(--text-muted); border:1px solid rgba(255,255,255,0.1) }
```

### Section 通用

```css
.section { margin-bottom:36px; animation:fadeInUp .5s ease both }
.section-title {
  font-size:16px; font-weight:700; color:var(--text-primary);
  margin-bottom:18px; display:flex; align-items:center; gap:10px;
  padding-bottom:12px; border-bottom:1px solid var(--border);
}
```

### Info Grid — 配置值、参数展示

```css
.info-grid { display:grid; grid-template-columns:repeat(auto-fill,minmax(220px,1fr)); gap:16px }
.info-item { background:rgba(255,255,255,0.03); border:1px solid rgba(255,255,255,0.07);
  border-radius:10px; padding:16px; }
.info-label { font-size:10px; color:var(--text-muted); text-transform:uppercase;
  letter-spacing:.5px; margin-bottom:6px; }
.info-value        { font-size:14px; font-weight:600; color:var(--text-primary) }
.info-value.code   { font-family:monospace; font-size:13px; color:#7ab4ff }
.info-value.green  { color:#6ee7a0 }
.info-value.yellow { color:#fcd34d }
.info-value.red    { color:#f87171 }
```

**用法**：
```html
<div class="info-grid">
  <div class="info-item">
    <div class="info-label">Timeout</div>
    <div class="info-value yellow">30s</div>
  </div>
  <div class="info-item">
    <div class="info-label">Queue Driver</div>
    <div class="info-value code">redis</div>
  </div>
</div>
```

### Flow Diagram — 数据流、处理流程

```css
.flow-diagram { background:var(--bg-card); border:1px solid var(--border); border-radius:12px; padding:28px }
.flow-col     { display:flex; flex-direction:column; gap:8px }
.flow-row     { display:flex; align-items:center; gap:12px; margin-bottom:8px; flex-wrap:wrap }
.flow-down    { text-align:center; color:var(--text-muted); font-size:20px; padding:4px 0; margin-left:20px }
.flow-box {
  padding:10px 16px; border-radius:8px; font-size:13px; font-weight:500;
  border:1px solid; display:flex; align-items:center; gap:8px;
}
/* 正常路径颜色 */
.flow-box.blue   { background:rgba(66,133,244,0.1);  border-color:rgba(66,133,244,0.4);  color:#7ab4ff }
.flow-box.green  { background:rgba(52,168,83,0.1);   border-color:rgba(52,168,83,0.4);   color:#6ee7a0 }
.flow-box.yellow { background:rgba(251,188,4,0.1);   border-color:rgba(251,188,4,0.4);   color:#fcd34d }
.flow-box.orange { background:rgba(249,115,22,0.1);  border-color:rgba(249,115,22,0.4);  color:#fb923c }
.flow-box.cyan   { background:rgba(6,182,212,0.1);   border-color:rgba(6,182,212,0.4);   color:#67e8f9 }
.flow-box.purple { background:rgba(168,85,247,0.1);  border-color:rgba(168,85,247,0.4);  color:#c084fc }
/* 失败 / 错误路径（红色）*/
.flow-box.red    { background:rgba(234,67,53,0.1);   border-color:rgba(234,67,53,0.4);   color:#f87171 }
.flow-arrow { font-size:16px; color:var(--text-muted) }
/* 失败路径箭头（↘）颜色 */
.flow-arrow.error { color:#f87171 }
```

**用法（正常 + 失败分支）**：
```html
<div class="flow-diagram">
  <div class="flow-col">
    <!-- 正常路径 -->
    <div class="flow-row">
      <div class="flow-box blue">📥 HTTP Request</div>
      <div class="flow-arrow">→</div>
      <div class="flow-box green">🔐 AuthMiddleware</div>
      <div class="flow-arrow">→</div>
      <div class="flow-box green">🎯 Controller</div>
    </div>
    <div class="flow-down">↓ 认证成功</div>
    <div class="flow-row">
      <div class="flow-box yellow">⚙️ Service</div>
      <div class="flow-arrow">→</div>
      <div class="flow-box cyan">🗄️ Database</div>
    </div>

    <!-- 失败路径（认证失败时分叉） -->
    <div class="flow-row" style="margin-top:12px">
      <div class="flow-arrow error">↘ 认证失败</div>
      <div class="flow-box red">🚫 401 Unauthorized</div>
    </div>
  </div>
</div>
```

**失败路径规则**：失败/错误分支使用 `.flow-box.red` + `.flow-arrow.error`，配合 `↘` 方向箭头表示分叉。始终在正常路径绘制完后追加失败分支，保持视觉上的主从关系。

### Alert Box — 重要提示 / 设计决策说明

```css
.alert-box      { border-radius:10px; padding:16px 20px; margin-bottom:16px; display:flex; gap:12px }
.alert-box.info { background:rgba(66,133,244,0.08); border:1px solid rgba(66,133,244,0.25) }
.alert-box.tip  { background:rgba(52,168,83,0.08);  border:1px solid rgba(52,168,83,0.25) }
.alert-box.warn { background:rgba(251,188,4,0.08);  border:1px solid rgba(251,188,4,0.25) }
.alert-icon    { font-size:18px; flex-shrink:0 }
.alert-content { font-size:13px; color:var(--text-secondary); line-height:1.6 }
.alert-content strong { color:var(--text-primary) }
```

**用法**：
```html
<div class="alert-box info">
  <div class="alert-icon">💡</div>
  <div class="alert-content">
    <strong>为什么用 Redis 队列而不是 Database 队列？</strong><br>
    Database 队列需要轮询，高并发时会锁表。Redis 原子操作支持 BLPOP 阻塞消费，
    吞吐量高 10 倍，且不影响主数据库性能。
  </div>
</div>
```

### HTTP Method Badges — API 路由方法标识

```css
.method-badge {
  display:inline-block; font-family:monospace; font-size:11px; font-weight:700;
  padding:3px 8px; border-radius:4px; min-width:52px; text-align:center;
  letter-spacing:.5px;
}
.method-get    { background:rgba(52,168,83,0.15);  color:#6ee7a0; border:1px solid rgba(52,168,83,0.4)  }
.method-post   { background:rgba(66,133,244,0.15); color:#7ab4ff; border:1px solid rgba(66,133,244,0.4) }
.method-put    { background:rgba(251,188,4,0.15);  color:#fcd34d; border:1px solid rgba(251,188,4,0.4)  }
.method-delete { background:rgba(234,67,53,0.15);  color:#f87171; border:1px solid rgba(234,67,53,0.4)  }
.method-patch  { background:rgba(249,115,22,0.15); color:#fb923c; border:1px solid rgba(249,115,22,0.4) }
```

### API Table — Controller / 路由接口表

带 HTTP 方法徽章的接口一览表，适用于 Controller 类型的模块页面：

```css
.api-table-wrap { overflow-x:auto; border-radius:10px; border:1px solid var(--border) }
.api-table { width:100%; border-collapse:collapse; font-size:13px }
.api-table th {
  background:#0f172a; color:var(--text-muted); font-size:10px; text-transform:uppercase;
  letter-spacing:.5px; padding:10px 16px; text-align:left; border-bottom:1px solid var(--border);
}
.api-table td { padding:12px 16px; border-bottom:1px solid rgba(255,255,255,0.04); vertical-align:top }
.api-table tr:last-child td { border-bottom:none }
.api-table tr:hover td { background:rgba(255,255,255,0.02) }
.api-table .path { font-family:monospace; color:#c084fc }
.api-table .handler { font-family:monospace; font-size:11px; color:var(--text-muted) }
.api-table .desc { color:var(--text-secondary); font-size:12px }
```

**用法**：
```html
<div class="api-table-wrap">
  <table class="api-table">
    <thead>
      <tr>
        <th>Method</th><th>Path</th><th>Handler</th><th>説明</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><span class="method-badge method-get">GET</span></td>
        <td class="path">/api/users</td>
        <td class="handler">index()</td>
        <td class="desc">ユーザー一覧取得</td>
      </tr>
      <tr>
        <td><span class="method-badge method-post">POST</span></td>
        <td class="path">/api/users</td>
        <td class="handler">store()</td>
        <td class="desc">新規ユーザー作成</td>
      </tr>
      <tr>
        <td><span class="method-badge method-delete">DELETE</span></td>
        <td class="path">/api/users/{id}</td>
        <td class="handler">destroy()</td>
        <td class="desc">ユーザー削除</td>
      </tr>
    </tbody>
  </table>
</div>
```

### Data Table — 統一テーブルスタイル（フィールド表、設定比較など）

ゼブラストライプ + ホバー効果付きの汎用テーブル：

```css
.data-table-wrap { overflow-x:auto; border-radius:10px; border:1px solid var(--border); margin-bottom:16px }
.data-table { width:100%; border-collapse:collapse; font-size:13px }
.data-table th {
  background:#0f172a; color:var(--text-muted); font-size:10px; text-transform:uppercase;
  letter-spacing:.5px; padding:10px 16px; text-align:left; border-bottom:1px solid var(--border);
}
.data-table td { padding:11px 16px; border-bottom:1px solid rgba(255,255,255,0.04); color:var(--text-secondary) }
.data-table tr:nth-child(even) td { background:rgba(255,255,255,0.015) }
.data-table tr:hover td { background:rgba(255,255,255,0.03) }
.data-table tr:last-child td { border-bottom:none }
.data-table .mono { font-family:monospace; color:#7ab4ff }
.data-table .badge-green  { color:#6ee7a0 }
.data-table .badge-yellow { color:#fcd34d }
.data-table .badge-red    { color:#f87171 }
```

**用法**（フィールド一覧など）：
```html
<div class="data-table-wrap">
  <table class="data-table">
    <thead>
      <tr><th>フィールド名</th><th>型</th><th>NULL</th><th>説明</th></tr>
    </thead>
    <tbody>
      <tr>
        <td class="mono">user_id</td>
        <td class="mono">BIGINT</td>
        <td class="badge-red">NOT NULL</td>
        <td>ユーザーID（PK）</td>
      </tr>
      <tr>
        <td class="mono">email</td>
        <td class="mono">VARCHAR(255)</td>
        <td class="badge-red">NOT NULL</td>
        <td>メールアドレス（UNIQUE）</td>
      </tr>
    </tbody>
  </table>
</div>
```

### Item Card — 通用条目卡（规则、接口、策略等）

```css
.item-card { background:rgba(255,255,255,0.03); border:1px solid rgba(255,255,255,0.07);
  border-radius:10px; padding:16px; margin-bottom:10px }
.item-name   { font-family:monospace; font-size:13px; font-weight:600; color:#7ab4ff; margin-bottom:6px }
.item-detail { font-size:12px; color:var(--text-secondary); line-height:1.8 }
```

**用途**：路由规则、中间件配置、权限策略、防火墙规则、Cron 计划等任何"条目型"数据。

### Role/Permission Grid — 角色、权限、SA（适用任何认证体系）

```css
.role-grid { display:grid; grid-template-columns:repeat(auto-fill,minmax(280px,1fr)); gap:14px }
.role-card { border-radius:10px; padding:16px; border:1px solid }
.role-card.blue   { background:rgba(66,133,244,0.07); border-color:rgba(66,133,244,0.3) }
.role-card.green  { background:rgba(52,168,83,0.07);  border-color:rgba(52,168,83,0.3) }
.role-card.orange { background:rgba(249,115,22,0.07); border-color:rgba(249,115,22,0.3) }
.role-card.purple { background:rgba(168,85,247,0.07); border-color:rgba(168,85,247,0.3) }
.role-card.gray   { background:rgba(255,255,255,0.03); border-color:rgba(255,255,255,0.1) }
.role-name  { font-family:monospace; font-size:13px; font-weight:600; margin-bottom:8px }
.role-perms { font-size:11px; color:var(--text-secondary); line-height:1.9 }
```

**用途**：GCP SA 权限（sa-grid 改名）、Laravel 角色权限、OAuth scope、IAM 策略等。

### Job Card — 定时/批处理任务（适用任何语言的 Job/Task/Command）

```css
.job-card { background:rgba(251,188,4,0.05); border:1px solid rgba(251,188,4,0.2);
  border-radius:12px; padding:20px; margin-bottom:14px }
.job-header { display:flex; align-items:flex-start; gap:16px; margin-bottom:14px }
.job-num { width:32px; height:32px; border-radius:8px; background:rgba(251,188,4,0.2);
  display:flex; align-items:center; justify-content:center;
  font-size:14px; font-weight:800; color:#fbbc04; flex-shrink:0 }
.job-name    { font-family:monospace; font-size:14px; font-weight:700; color:#fcd34d; margin-bottom:4px }
.job-purpose { font-size:13px; color:var(--text-primary); font-weight:600 }
.cron-badge  { display:inline-block; background:#0f172a; border:1px solid rgba(251,188,4,0.3);
  border-radius:6px; padding:4px 10px; font-family:monospace; font-size:13px; color:#fcd34d; margin-bottom:10px }
.job-detail  { display:grid; grid-template-columns:auto 1fr; gap:6px 16px;
  font-size:12px; color:var(--text-secondary) }
.jd-label { color:var(--text-muted); text-transform:uppercase; font-size:10px; letter-spacing:.5px }
.jd-val   { font-family:monospace; color:#7ab4ff }
```

**用途**：Laravel Artisan Command + Scheduler、Django Celery Task + Crontab、NestJS Cron、GCP Cloud Scheduler 等所有定时任务。

### Step Card — ステップ・手順説明（順序付きの処理説明）

番号付きの縦並びステップコンポーネント。セットアップ手順、デプロイフロー、認証シーケンスなど「順序が重要な処理」の説明に使う：

```css
.step-list { display:flex; flex-direction:column; gap:0 }
.step-item { display:flex; gap:20px; position:relative }
.step-item:not(:last-child)::after {
  content:''; position:absolute; left:19px; top:44px; bottom:0;
  width:2px; background:linear-gradient(to bottom, var(--border), transparent);
}
.step-num {
  width:40px; height:40px; border-radius:50%; flex-shrink:0;
  background:var(--accent-light); border:2px solid var(--accent);
  display:flex; align-items:center; justify-content:center;
  font-size:15px; font-weight:800; color:var(--accent);
}
.step-body { padding:8px 0 28px }
.step-title  { font-size:14px; font-weight:700; color:var(--text-primary); margin-bottom:6px }
.step-detail { font-size:13px; color:var(--text-secondary); line-height:1.7 }
```

**用法**：
```html
<div class="step-list">
  <div class="step-item">
    <div class="step-num">1</div>
    <div class="step-body">
      <div class="step-title">GitHub Actions が WIF でGCP認証</div>
      <div class="step-detail">
        Workload Identity Federation を使ってサービスアカウントキーなしで認証。
        <code>roles/iam.workloadIdentityUser</code> を付与済み。
      </div>
    </div>
  </div>
  <div class="step-item">
    <div class="step-num">2</div>
    <div class="step-body">
      <div class="step-title">docker build &amp; push</div>
      <div class="step-detail">Artifact Registry にイメージをプッシュ</div>
    </div>
  </div>
</div>
```

### Code Block — 代码片段（带语法着色 + 语言标签 + 复制按钮）

```css
.code-wrap { position:relative; margin-bottom:16px }

/* 语言标签（左上角） */
.code-lang {
  position:absolute; top:10px; left:10px;
  font-family:monospace; font-size:10px; color:var(--text-muted);
  background:rgba(255,255,255,0.06); border:1px solid rgba(255,255,255,0.08);
  padding:2px 8px; border-radius:4px; letter-spacing:.5px; text-transform:uppercase;
  pointer-events:none;
}

/* 复制按钮（右上角） */
.copy-btn {
  position:absolute; top:10px; right:10px;
  background:rgba(255,255,255,0.06); border:1px solid rgba(255,255,255,0.1);
  color:var(--text-muted); border-radius:6px; padding:4px 8px;
  font-size:11px; cursor:pointer; transition:all .2s;
}
.copy-btn:hover { background:rgba(255,255,255,0.12); color:var(--text-primary) }
.copy-btn.copied { color:#6ee7a0; border-color:rgba(52,168,83,0.4) }

/* padding-top を広めに取り、左上の言語タグ・右上の複制ボタンとコード先頭行の重なりを防ぐ */
.code-block { background:#0f172a; border-radius:10px; padding:44px 20px 20px 20px;
  font-family:'Courier New',monospace; font-size:12.5px; line-height:1.7;
  color:#e2e8f0; overflow-x:auto; white-space:pre; margin:0 }
.code-comment { color:#64748b }  /* # // <!-- 注释 */
.code-key     { color:#c084fc }  /* 关键字、函数名、属性名 */
.code-val     { color:#6ee7a0 }  /* 变量值、数字、布尔 */
.code-str     { color:#fbbf24 }  /* 字符串 */
```

**复制按钮 JS（全页只写一次）**：
```html
<script>
function copyCode(btn) {
  const pre = btn.closest('.code-wrap').querySelector('.code-block');
  navigator.clipboard.writeText(pre.innerText).then(() => {
    btn.textContent = '✓ 复制成功';
    btn.classList.add('copied');
    setTimeout(() => { btn.textContent = '复制'; btn.classList.remove('copied'); }, 2000);
  });
}
</script>
```

**用法**：
```html
<div class="code-wrap">
  <span class="code-lang">Terraform</span>
  <button class="copy-btn" onclick="copyCode(this)">复制</button>
  <pre class="code-block"><span class="code-comment"># modules/gcp/repository/main.tf</span>
<span class="code-key">resource</span> <span class="code-str">"google_artifact_registry_repository"</span> <span class="code-str">"this"</span> {
  repository_id = <span class="code-val">var.repository_id</span>
  format        = <span class="code-str">"DOCKER"</span>
}</pre>
</div>
```

**代码语言标签约定**：
| 语言 | `.code-lang` 文字 |
|------|-----------------|
| Terraform | `Terraform` |
| PHP | `PHP` |
| TypeScript | `TypeScript` |
| Python | `Python` |
| SQL | `SQL` |
| Shell / Bash | `Shell` |
| YAML | `YAML` |
| Go | `Go` |

### 对比卡（Compare Grid）— 方案/环境对比

```css
.compare-grid { display:grid; grid-template-columns:1fr 1fr; gap:16px }
.compare-card { border-radius:10px; padding:18px; border:1px solid }
/* 颜色主题同 role-card */
.compare-title { font-size:13px; font-weight:600; margin-bottom:12px }
.compare-item  { display:flex; justify-content:space-between; padding:7px 0;
  border-bottom:1px solid rgba(255,255,255,0.05); font-size:12px }
.compare-item:last-child { border-bottom:none }
.compare-key { color:var(--text-muted) }
.compare-val { font-family:monospace; font-weight:600 }
```

**用途**：dev vs prod 参数对比、方案 A vs 方案 B 取舍对比。

### Dependency Visualization — 依赖关係図（index.html のみ）

モジュール間の依存関係を CSS のみで表示。index.html の「アーキテクチャ」セクションに配置する：

```css
.dep-graph { background:var(--bg-card); border:1px solid var(--border); border-radius:12px; padding:28px }
.dep-layer { display:flex; justify-content:center; gap:12px; margin-bottom:8px; flex-wrap:wrap }
.dep-label { font-size:10px; color:var(--text-muted); text-align:center; margin-bottom:16px; letter-spacing:.5px }
.dep-node {
  padding:10px 18px; border-radius:8px; font-size:12px; font-weight:600;
  border:1px solid; text-decoration:none; transition:all .2s; display:inline-block;
  text-align:center;
}
.dep-node:hover { transform:translateY(-2px); box-shadow:0 4px 16px rgba(0,0,0,0.3) }
/* 颜色继承 flow-box 同款 */
.dep-node.blue   { background:rgba(66,133,244,0.1);  border-color:rgba(66,133,244,0.4);  color:#7ab4ff }
.dep-node.green  { background:rgba(52,168,83,0.1);   border-color:rgba(52,168,83,0.4);   color:#6ee7a0 }
.dep-node.yellow { background:rgba(251,188,4,0.1);   border-color:rgba(251,188,4,0.4);   color:#fcd34d }
.dep-node.purple { background:rgba(168,85,247,0.1);  border-color:rgba(168,85,247,0.4);  color:#c084fc }
.dep-node.cyan   { background:rgba(6,182,212,0.1);   border-color:rgba(6,182,212,0.4);   color:#67e8f9 }
.dep-arrow-row   { display:flex; justify-content:center; gap:24px; color:var(--text-muted); font-size:18px; padding:4px 0 }
```

**用法**（レイヤー構造で上から下へ依存関係を表示）：
```html
<div class="dep-graph">
  <div class="dep-label">↓ 依存方向（上层调用下层）</div>

  <div class="dep-layer">
    <a href="controller.html" class="dep-node blue">🎯 OrderController</a>
    <a href="auth.html"       class="dep-node blue">🔐 AuthController</a>
  </div>
  <div class="dep-arrow-row">↓ &nbsp;&nbsp;&nbsp;&nbsp; ↓</div>

  <div class="dep-layer">
    <a href="order_service.html"  class="dep-node green">⚙️ OrderService</a>
    <a href="notify_service.html" class="dep-node green">📨 NotifyService</a>
  </div>
  <div class="dep-arrow-row">↓ &nbsp;&nbsp;&nbsp;&nbsp; ↓</div>

  <div class="dep-layer">
    <a href="order_model.html" class="dep-node yellow">🗄️ Order Model</a>
    <a href="user_model.html"  class="dep-node yellow">👤 User Model</a>
  </div>
</div>
```

**配置原则**：
- 每层代表一个架构层（Controller → Service → Model / Repository）
- 节点必须是 `<a href="module.html">` 可点击跳转
- 箭头行用 `dep-arrow-row` 居中对齐
- 仅在 index.html 上显示，详情页不需要

### Related Pages — 相关模块链接（详情页底部）

每个模块详情页底部放置 2-4 张关联模块的跳转卡：

```css
.related-pages { margin-top:40px; padding-top:24px; border-top:1px solid var(--border) }
.related-title { font-size:13px; color:var(--text-muted); margin-bottom:14px; text-transform:uppercase; letter-spacing:.5px }
.related-grid  { display:grid; grid-template-columns:repeat(auto-fill,minmax(200px,1fr)); gap:12px }
.related-card  {
  background:var(--bg-card); border:1px solid var(--border);
  border-radius:10px; padding:14px 16px; text-decoration:none;
  display:flex; align-items:center; gap:12px; transition:all .2s;
}
.related-card:hover { background:var(--bg-card-hover); border-color:var(--accent); transform:translateX(4px) }
.related-icon  { font-size:20px; flex-shrink:0 }
.related-info  { flex:1; min-width:0 }
.related-name  { font-size:13px; font-weight:600; color:var(--text-primary); margin-bottom:2px }
.related-type  { font-size:11px; color:var(--text-muted) }
```

**用法**（在 source section 之后、prev-next nav 之前）：
```html
<div class="related-pages">
  <div class="related-title">🔗 関連モジュール</div>
  <div class="related-grid">
    <a href="cloud_run.html" class="related-card">
      <div class="related-icon">🚀</div>
      <div class="related-info">
        <div class="related-name">Cloud Run</div>
        <div class="related-type">コンピュート層</div>
      </div>
    </a>
    <a href="secret_manager.html" class="related-card">
      <div class="related-icon">🔐</div>
      <div class="related-info">
        <div class="related-name">Secret Manager</div>
        <div class="related-type">セキュリティ層</div>
      </div>
    </a>
  </div>
</div>
```

### Prev/Next Navigation — 前後ページナビゲーション

モジュール詳情页の最底部（ページフッターの直前）に配置する：

```css
.prev-next-nav {
  display:flex; justify-content:space-between; gap:16px;
  margin-top:32px; padding-top:24px; border-top:1px solid var(--border);
}
.nav-btn {
  display:flex; align-items:center; gap:12px;
  background:var(--bg-card); border:1px solid var(--border);
  border-radius:10px; padding:14px 20px; text-decoration:none;
  flex:1; max-width:45%; transition:all .2s;
}
.nav-btn:hover { background:var(--bg-card-hover); border-color:var(--accent) }
.nav-btn.next  { flex-direction:row-reverse; text-align:right }
.nav-btn.disabled { opacity:.3; pointer-events:none }
.nav-direction { font-size:10px; color:var(--text-muted); text-transform:uppercase; letter-spacing:.5px; margin-bottom:2px }
.nav-title     { font-size:13px; font-weight:600; color:var(--text-primary) }
.nav-arrow     { font-size:20px; color:var(--accent); flex-shrink:0 }
```

**用法**：
```html
<div class="prev-next-nav">
  <a href="repository.html" class="nav-btn prev">
    <div class="nav-arrow">←</div>
    <div>
      <div class="nav-direction">前のページ</div>
      <div class="nav-title">Artifact Registry</div>
    </div>
  </a>
  <a href="logging.html" class="nav-btn next">
    <div>
      <div class="nav-direction">次のページ</div>
      <div class="nav-title">Cloud Logging</div>
    </div>
    <div class="nav-arrow">→</div>
  </a>
</div>
```

**规则**：
- 顺序与 index.html sidebar 的模块排列顺序一致
- 第一页的 `prev` 按钮加 `.disabled` class
- 最后一页的 `next` 按钮加 `.disabled` class

### Module Cards（index.html 总览网格）

```css
.module-card {
  background:var(--bg-card); border:1px solid var(--border);
  border-radius:14px; padding:24px; display:block; text-decoration:none;
  transition:all .3s ease; position:relative; overflow:hidden;
}
.module-card::before {
  content:''; position:absolute; top:0; left:0; right:0; height:2px;
  background:linear-gradient(to right, transparent, var(--card-color, var(--accent-blue)), transparent);
  opacity:0; transition:opacity .3s;
}
.module-card:hover {
  background:var(--bg-card-hover); border-color:var(--card-color, #4285f4);
  transform:translateY(-4px); box-shadow:0 12px 30px rgba(0,0,0,0.3);
}
.module-card:hover::before { opacity:1 }
/* 用法：<a class="module-card" href="service.html" style="--card-color:#34a853"> */
```

### Source Info（最后一个 section 的固定格式）

```css
.source-card { background:var(--bg-card); border:1px solid var(--border); border-radius:12px; padding:24px }
.source-row  { display:grid; grid-template-columns:1fr 1fr; gap:16px; margin-bottom:16px }
.source-block { background:rgba(255,255,255,0.03); border:1px solid rgba(255,255,255,0.07);
  border-radius:10px; padding:16px }
.sb-label { font-size:10px; color:var(--text-muted); text-transform:uppercase; letter-spacing:.5px; margin-bottom:6px }
.sb-value { font-size:14px; font-weight:600; color:var(--text-primary) }
.sb-value.mono { font-family:monospace; font-size:13px; color:#7ab4ff }
.tag-list { display:flex; flex-wrap:wrap; gap:8px }
.tag-item { background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.09);
  border-radius:6px; padding:5px 12px; font-family:monospace; font-size:12px; color:#c084fc }
```

### 动画（必须）

```css
@keyframes fadeInDown { from{opacity:0;transform:translateY(-15px)} to{opacity:1;transform:translateY(0)} }
@keyframes fadeInUp   { from{opacity:0;transform:translateY(15px)}  to{opacity:1;transform:translateY(0)} }
/* Hero 用 fadeInDown，各 section 用 fadeInUp + animation-delay 依次错开 */
.page-header { animation:fadeInDown .5s ease }
.section { animation:fadeInUp .5s ease both }
/* 多个 section 可设 style="animation-delay:.1s/.2s/.3s" */
```

### TOC IntersectionObserver（每页必须）

```html
<script>
const items = document.querySelectorAll('.toc-item');
const observer = new IntersectionObserver(entries => {
  entries.forEach(e => {
    if(e.isIntersecting) {
      items.forEach(i => i.classList.remove('active'));
      const t = document.querySelector(`.toc-item[href="#${e.target.id}"]`);
      if(t) t.classList.add('active');
    }
  });
}, {threshold:0.3});
document.querySelectorAll('[id]').forEach(el => observer.observe(el));
</script>
```

---

## 内容质量标准 ★ 决定文档好坏的核心

### ✅ 合格页面的 5 条必须（无论什么技术栈）

1. **WHY section** — 解释 1-3 个关键设计决策。  
   不是说"做了什么"，而是"为什么这样做，有哪些取舍"。  
   举例：「为什么选 Redis 而不是 DB Queue」「为什么 Service 层不直接访问 Model 而是经过 Repository」

2. **端到端流程图** — 用 `flow-diagram` 展示数据/请求的生命周期。  
   从外部进入系统→经过哪些层→如何响应/持久化，必须有方向感和颜色区分。  
   **如果存在失败路径（认证失败、网络错误、超时），必须用 `.flow-box.red` + `.flow-arrow.error` 展示**。

3. **具体值，而非变量名** — info-grid 中展示的应该是实际值（`30s`、`redis`、`3次重试`），  
   不是 `$config['timeout']` 或 `var.retention_days`。读源码或配置文件获取真实值。

4. **比较/对比** — 至少有一处环境对比（dev vs prod）、方案对比或版本对比。  
   用 `compare-grid` 或 `data-table` 展示。

5. **代码片段** — 每个 Service/核心模块至少有一段实际代码（不超过 25 行），  
   使用 `code-wrap` + `code-lang` + `copy-btn` + `code-block` 展示最关键的方法。

### ❌ 低质量页面的特征（必须避免）

- 只有 overview + 几个 info-grid + source：内容太浅，没有 WHY 和流程
- 配置值全是变量名或占位符，没有从代码/配置文件读取真实值
- 没有 alert-box：说明没有解释重要的约束、注意事项或设计取舍
- 页面少于 250 行（通常意味着内容不足）
- 只描述"是什么"，不解释"为什么"和"怎么工作"
- 代码块缺少语言标签或复制按钮
- Controller 页面没有 API 路由表（api-table + method-badge）
- 详情页底部缺少 related-pages 或 prev-next-nav
- index.html 架构图节点不可点击（缺少 `href`）

---

## 设计原则（硬性规定）

| 原则 | 要求 |
|------|------|
| **零外部依赖** | 所有 CSS/JS 内联，无 CDN，无外部字体 |
| **深色主题 Only** | 固定深色，不添加 Light/Dark 切换按钮 |
| **文档语言** | 内容（标题、描述、说明）使用**中文**（除非项目文档是其他语言） |
| **响应式** | `@media(max-width:900px)` 时隐藏 Sidebar，主内容全宽 |
| **平滑滚动** | `html{scroll-behavior:smooth}` + `.section{scroll-margin-top:20vh}` |
| **字体** | `'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif` |
| **代码背景** | `#0f172a`（比主背景略深，形成对比） |
| **行内代码** | 所有 `<code>` 统一应用行内代码样式（见上方 Inline Code 节） |

---

## Step 5 — 在浏览器打开

```bash
# 多页模式（打开总览页）
open guide/{topic}/index.html

# 单页模式
open guide/{filename}.html
```
