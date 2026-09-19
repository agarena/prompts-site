# 优质提示词聚合网站（Prompt Folio）

「不吃鲸B」出品的提示词卡片盒：每张卡片写清使用场景、提示词正文与使用示例，看到合适的展开即可整体复制。支持搜索、标签筛选、三种排序、点赞、分享深链、投稿与反馈。

- 线上地址：<https://prompts.agarena.xyz>
- 品牌与引流入口见页面底部署名（运行时由后端 `/api/site` 下发）

## 架构

```
浏览器（本仓库，纯静态单文件 index.html）
   │  GET  /api/prompts            拉取已发布提示词（缓存 60s）
   │  POST /api/prompts/like       点赞/取消（按访客去重）
   │  POST /api/prompts/submit     投稿（先审后显）
   │  POST /api/prompts/feedback   反馈（评价/建议/问题）
   │  POST /api/prompts/log        关键节点行为日志（批量）
   │  POST /api/collect            页面访问与停留时长（复用主站统计）
   ▼
api.agarena.xyz（Cloudflare Worker，仓库 agarena/analytics-worker）
   ▼
Cloudflare D1（表：prompts / prompt_likes / prompt_feedback / pf_logs + 主站 visits/events/…）
```

- 前端零构建：单文件 `index.html`，直接部署到 Cloudflare Pages 项目 `prompts-site`。
- 接口不可达时自动降级为内置的 12 条兜底数据（与 `analytics-worker/seed.sql` 保持同步，**改一处必须同步另一处**）。

## 功能与关键节点日志

所有关键行为都写入后端 `pf_logs` 表（后台 `/admin` 可见）：

| 行为 | 日志 type | 备注 |
|---|---|---|
| 页面访问 | page_view（pf_logs）+ visits 表 | visits 走 /api/collect，后台图表可看流量/来源/地域 |
| 搜索 | search | 记录关键词，防抖去重 |
| 展开全文 | expand | |
| 复制提示词 | copy | |
| 复制作者账号 | copy_account | |
| 点赞/取消 | like / unlike | 服务端记录，前端不重复记 |
| 分享 | share | via: share / copy |
| 分享深链打开 | share_open | found: true/false |
| 投稿 | submit | 服务端记录 |
| 反馈 | feedback | 服务端记录 |

服务端错误与管理操作（admin_publish / admin_hide / admin_delete）同样入 pf_logs。

## 分享深链

每张卡片有「分享」按钮，链接格式：

```
https://prompts.agarena.xyz/?id=<卡片id>    如 https://prompts.agarena.xyz/?id=pf01
```

打开后自动清除筛选、滚动定位到该卡片、展开全文并高亮。卡片不存在或未公开时给出提示并展示完整列表。

## 开放投稿 API（可接 AI 工具自动投稿）

无需注册，公开接口，先审后显。滥用受频率限制（同 IP 3 次/分钟）与蜜罐防线。

### 提交投稿

```
POST https://api.agarena.xyz/api/prompts/submit
Content-Type: application/json
```

| 字段 | 必填 | 说明 |
|---|---|---|
| title | 是 | 标题，≤80 字 |
| scene | 是 | 使用场景，建议「当…时，请使用本提示词。」句式，≤300 字 |
| content | 是 | 提示词正文，≤6000 字 |
| example | 否 | 使用示例（输入/输出），≤2000 字 |
| platform | 否 | 引流平台，如 抖音 / Bilibili / 微信公众号 / 小红书 / 知乎 |
| account | 否 | 引流账号名，≤60 字（署名展示） |
| url | 否 | 主页链接，http(s):// 开头，非法值会被丢弃 |
| img | 否 | 配图 dataURL，仅接受 `data:image/(png|jpeg|jpg|webp|gif);base64,` 且 ≤200KB，建议最长边 1080px 宽 |
| hp | — | 蜜罐字段，**永远不要填**（填了会被当作机器人，假装成功但不入库） |

响应：`{"ok":true,"id":"u1789…"}`。投稿进入待审队列，站长在后台通过后公开。

```bash
curl -X POST https://api.agarena.xyz/api/prompts/submit \
  -H "Content-Type: application/json" \
  -d '{"title":"周报 60 秒生成器","scene":"当你周五要交周报时，请使用本提示词。","content":"（完整提示词正文）","platform":"小红书","account":"@你的账号","url":"https://www.xiaohongshu.com/user/profile/xxx"}'
```

### 公开数据

```
GET https://api.agarena.xyz/api/prompts
```

返回已发布提示词数组（含 id/no/title/author/platform/account/url/tags/scene/content/example/img/likes/ts），缓存 60 秒。

### 点赞（页面交互用）

```
POST https://api.agarena.xyz/api/prompts/like
{"id":"pf01","vid":"访客匿名id","liked":true}   →  {"ok":true,"liked":true,"likes":343}
```

## 内容管理与审核

1. 打开 `https://api.agarena.xyz/admin?key=<ADMIN_TOKEN>`（密码在 analytics-worker 仓库的 `.env`）。
2. 「提示词投稿审核」区块：查看待审投稿（标题/作者/场景/正文/配图），点「通过上架」自动分配 PF 编号并公开；也可「隐藏」或「删除」。
3. 同页可看提示词反馈、关键节点日志（按类型计数 + 最近 100 条）。
4. 也可走管理 API：`GET /api/admin/prompts?status=pending`、`POST /api/admin/prompt {"id":"…","action":"publish|hide|delete"}`（`X-Admin-Key` 认证）。

## 发布

```bash
# 任意目录（需 Cloudflare 授权：export CLOUDFLARE_API_TOKEN=… 或 wrangler login）
npx wrangler pages deploy . --project-name prompts-site --branch main
```

首次搭建（新环境）：
1. 后端按 `agarena/analytics-worker` 仓库 README 初始化（D1 建表 + seed + deploy）。
2. Cloudflare 控制台给 Pages 项目 `prompts-site` 绑定自定义域 `prompts.agarena.xyz`（CNAME → prompts-site.pages.dev，已开橙云）。
3. 确认 Worker 的 `ALLOW_ORIGIN` 白名单含 `https://prompts.agarena.xyz`。
