# MagicBlog / Terry Homepage

基于 SvelteKit 构建的个人主页与博客系统。项目包含首页信息卡片、博客文章、足迹地图及图片、友情链接、GitHub 项目展示、Cloudflare 访问统计和 WakaTime 编程统计。

## 目录

- [MagicBlog / Terry Homepage](#magicblog--terry-homepage)
  - [目录](#目录)
  - [特性](#特性)
  - [开发指南](#开发指南)
    - [常用命令](#常用命令)
  - [配置说明](#配置说明)
  - [Cloudflare Pages 部署](#cloudflare-pages-部署)
  - [Cloudflare Analytics Worker 部署](#cloudflare-analytics-worker-部署)
  - [WakaTime 编程统计](#wakatime-编程统计)
  - [Markdown 博客更新流程](#markdown-博客更新流程)
  - [博客自动化部署](#博客自动化部署)
    - [1. 自动化监听](#1-自动化监听)
    - [2. 精简部署](#2-精简部署)
  - [鸣谢](#鸣谢)

---

## 特性

- **技术栈**：SvelteKit、Svelte 5、TypeScript、Tailwind CSS
- **博客系统**：Markdown 文章、分类索引、搜索索引、RSS、Sitemap
- **个人主页**：个人资料、社交链接、GitHub 项目展示
- **足迹地图**：地点图钉、图片轮播、点击查看地点图片
- **统计模块**：Cloudflare Analytics 访问统计、WakaTime 编程统计
- **部署方式**：Cloudflare Pages 静态部署，支持自定义域名

## 开发指南

项目依赖 Node.js 环境

### 常用命令

```bash
# 启动开发服务器
npm run dev

# 生成 Markdown 博客索引
npm run gen-blog

# 构建生产版本
npm run build

# 本地预览生产构建
npm run preview

# 类型检查
npm run check
```

## 配置说明

项目使用环境变量进行配置。请参考 [CONFIGURATION.md](./CONFIGURATION.md) 查看详细变量列表。

常用变量如下：

| 变量名 | 说明 |
| :--- | :--- |
| `VITE_SITE_NAME` | 网站名称 |
| `VITE_SITE_URL` | 线上站点地址，例如 `https://luckluck-dong.cn` |
| `VITE_GITHUB_USERNAME` | GitHub 用户名，用于展示 GitHub 项目 |
| `VITE_AVATAR_URL` | 自定义头像地址；留空时回退到 GitHub 头像 |
| `VITE_CF_ANALYTICS_WORKER_URL` | Cloudflare Analytics Worker 地址 |
| `VITE_WAKATIME_EMBED_URL` | WakaTime Coding Activity JSON 地址 |
| `VITE_WAKATIME_LANGUAGES_URL` | WakaTime Languages JSON 地址 |
| `VITE_REPO_URL` | 当前项目仓库地址 |

## Cloudflare Pages 部署

Cloudflare Pages 推荐配置：

| 配置项 | 值 |
| :--- | :--- |
| 构建命令 | `npm run gen-blog && npm run build` |
| 构建输出目录 | `build` |
| 根目录 | `/` 或留空 |
| 部署命令 | 可留空；如果页面要求必填，可填 `echo ok` |

修改代码或 Markdown 后，正常流程是：

```bash
npm run gen-blog
npm run build
git add .
git commit -m "Update site"
git push
```

推送到 GitHub 后，Cloudflare Pages 会自动重新构建并部署。

## Cloudflare Analytics Worker 部署

首页访问统计模块依赖一个独立的 Cloudflare Worker 代理 Analytics API。代码位于 `cf-analytics-worker/` 目录。

```bash
# 安装 Wrangler CLI
npm install -g wrangler

# 登录 Cloudflare 账户
wrangler login

# 进入 Worker 目录
cd cf-analytics-worker

# 配置密钥和变量（按提示输入）
wrangler secret put CF_API_TOKEN
wrangler secret put CF_ZONE_ID

# 部署
wrangler deploy
```

部署成功后，将 Worker URL 填入主站环境变量：

```env
VITE_CF_ANALYTICS_WORKER_URL=https://cf-analytics-worker.xxxxx.workers.dev
```

Worker 里还需要配置：

```env
ALLOWED_ORIGIN=https://luckluck-dong.cn
```

如果线上页面显示 `Failed to load`，优先检查：

- Worker URL 是否可以直接打开并返回 JSON
- `ALLOWED_ORIGIN` 是否和当前访问域名完全一致
- Cloudflare Pages 环境变量修改后是否重新部署

## WakaTime 编程统计

编程统计来源于WakaTime编辑器插件，不是 GitHub 仓库语言比例。

配置流程：

1. 在 VS Code / Cursor 安装 `WakaTime.vscode-wakatime` 插件
2. 在命令面板执行 `WakaTime: Api Key`
3. 粘贴 WakaTime API Key
4. 在 WakaTime 网站生成 Embeddable Charts
5. 将 JSON 分享链接填入 Cloudflare Pages 环境变量

常用配置：

```env
VITE_WAKATIME_EMBED_URL=https://wakatime.com/share/xxx/coding-activity.json
VITE_WAKATIME_LANGUAGES_URL=https://wakatime.com/share/xxx/languages.json
```

其中：

- `VITE_WAKATIME_EMBED_URL` 用于展示 Coding Activity
- `VITE_WAKATIME_LANGUAGES_URL` 用于展示语言分布

## Markdown 博客更新流程

博客文章放在 `static/posts/` 目录下。新增、删除或修改 Markdown 后，需要重新生成索引：

```bash
npm run gen-blog
```

该命令会更新：

- `static/posts/all.json`
- `static/posts/categories.json`
- `static/posts/search.json`
- `static/posts/rss.xml`
- `static/sitemap.xml`

如果只修改了 Markdown 文件但没有执行 `npm run gen-blog`，页面上的目录、分类、搜索和 RSS 可能不会同步更新。

## 博客自动化部署

支持在服务器上实现"上传即发布"：

### 1. 自动化监听

建议使用 `systemd` 守护进程运行 `scripts/watch-posts.js` 实现递归监控

**服务文件配置** (`/etc/systemd/system/blog-watcher.service`):

```ini
[Unit]
Description=Recursive Blog Post Watcher
After=network.target

[Service]
Type=simple
WorkingDirectory=/path/to/project
ExecStart=/path/to/node scripts/watch-posts.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**管理命令**:

```bash
systemctl daemon-reload
systemctl enable --now blog-watcher.service
```

### 2. 精简部署

服务器可仅保留以下必要文件以运行自动化脚本：

- `scripts/generate-blog-index.js`
- `scripts/watch-posts.js`
- `package.json`
- `.env`

在目标目录运行 `npm install --production` 即可

## 鸣谢

参考的开源项目：

- [FuyaoHomepage](https://github.com/skyrocketingHong/FuyaoHomepage): 博客项目
