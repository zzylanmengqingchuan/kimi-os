# 用 Kimi Code 复刻 zhenhua.lu 博客并部署到 Cloudflare Pages（全程实战）

> 日期：2026-07-24 ～ 2026-07-25
> 成果：https://story-mkfast.uk
> 工具：Kimi Code CLI + wrangler + Cloudflare Pages

本文记录一个完整过程：我喜欢 [zhenhua.lu](https://zhenhua.lu) 这个博客的设计，想让 Kimi 用它的样式套我自己的博客数据（18 篇文章、5 个开源作品、学习经历），最后部署到 Cloudflare Pages 并绑定自己的域名。从零到上线不到两天，其中大部分时间还是在等授权和 DNS。

## 一、关键发现：目标站点本身是开源的

开始之前以为要"看着截图手写仿制"，调研后发现 **zhenhua.lu 的源码在 GitHub 上以 MIT 协议开源**（[luzhenhua/zhenhua.lu](https://github.com/luzhenhua/zhenhua.lu)）。这直接改变了方案：

- ~~手写仿制~~ → **克隆官方源码 + 替换数据层**，视觉效果天然 1:1
- MIT 协议允许自由使用修改，只需保留 LICENSE；我在页脚加了一行「Theme: zhenhua.lu」署名

**教训：复刻一个网站之前，先查它有没有开源。**

## 二、技术栈

原站技术栈（也是最终栈）：

- Next.js 15（App Router）+ React 18 + TypeScript
- Tailwind CSS + shadcn/ui + Magic UI（BlurFade 动效、毛玻璃 Dock 导航）
- motion + next-themes（亮暗主题）
- 静态导出（`output: "export"`），产物纯 HTML/CSS/JS，不需要 Node 服务器

原站自带的特色功能全部保留：CLI 终端页（/cli）、Matrix 矩阵雨页（/matrix）、Live 年龄计数器、GitHub 贡献图、中英双语、Loading 遮罩。

## 三、数据迁移与二次开发

我原来的博客是 Astro 项目，内容是 Markdown collections。迁移和改动：

1. **内容迁移**：18 篇文章 md 文件原样复制到 `content/articles/`，封面图 51 张进 `public/covers/`，头像进 `public/images/`
2. **个人数据**：全部集中在 `src/data/resume.tsx`（中英两份），替换简介、社交链接、标签、作品、学习经历
3. **新增文章功能**（原站没有博客功能）：
   - `src/lib/blog.ts`：gray-matter 解析 frontmatter
   - `/articles` 列表页 + `/articles/[slug]` 详情页，首页加「最新文章」区块，Dock 加入口
   - 按原站视觉语言写的卡片组件，风格统一
4. **清理原作者数据**：文章、作品、备案号、Clarity 统计脚本、CLI 页 ASCII 字和提示符用户名、Matrix 字符雨里的名字

### 踩坑 1：react-markdown 同步管线不支持异步插件

文章页用 react-markdown + rehype-pretty-code（Shiki 代码高亮），预渲染报错：

```
Error: `runSync` finished async. Use `run` instead
```

原因：react-markdown 走 unified 的同步管线，而 rehype-pretty-code 加载 Shiki 高亮器是异步的。
解法：文章正文改为**服务端用 unified 异步管线渲染成 HTML 字符串**（remark-parse → remark-gfm → remark-rehype → rehype-pretty-code → rehype-stringify），前端 `dangerouslySetInnerHTML` 注入。

### 踩坑 2：静态导出的兼容处理

- `headers()` / `redirects()` 在 `output: "export"` 下不生效 → 删掉，安全头交给 CDN/nginx 层
- 动态 OG 图路由 `api/og/route.tsx` 不支持静态导出 → 删除
- `robots.ts` / `sitemap.ts` 需要显式 `export const dynamic = 'force-static'`
- Next 15 的动态路由 `params` 是 Promise，必须 `await params`

## 四、部署到 Cloudflare Pages

选 Cloudflare 而不是国内服务器的原因：**免备案**。备案要求针对中国大陆境内服务器/CDN，Cloudflare 国际版服务器在境外，域名 NS 托管过去即可。

部署步骤：

```bash
pnpm build                                            # 产物在 out/
npx wrangler login                                    # OAuth 浏览器授权
npx wrangler pages project create story-mkfast --production-branch=main
npx wrangler pages deploy out --project-name=story-mkfast --branch=main
```

域名根域名原来指向一个旧的空 Worker，先解绑（直接删服务，自定义域名绑定随之解除）：

```bash
curl -X DELETE -H "Authorization: Bearer $TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/<account_id>/workers/services/story-mkfast"
```

再通过 API 把域名绑到 Pages 项目：

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"story-mkfast.uk"}' \
  "https://api.cloudflare.com/client/v4/accounts/<account_id>/pages/projects/story-mkfast/domains"
```

### 踩坑 3：Pages 自定义域名卡在 "CNAME record not set"

绑定后域名状态一直 pending，报错 `CNAME record not set`。Pages 不会自动建 DNS 记录，需要手动补一条：

```
类型 CNAME | 名称 @ | 目标 story-mkfast.pages.dev | 已代理（橙云）
```

### 踩坑 4：wrangler OAuth token 没有 DNS 编辑权限

用 wrangler 登录的 OAuth token 调 DNS API 报 `Authentication error`（code 10000）——它的 scope 列表里只有 `zone:read`，没有 `dns:write`。
解法：单独建一个 API Token（Dashboard → My Profile → API Tokens），权限给 **Zone.DNS:Edit** 并限定单个域名。注意新版令牌界面的权限搜索框只认 `DNS` 这种关键词，搜 `Zone.DNS:Edit` 旧格式会显示"未找到匹配"。

### 踩坑 5：本机代理 fake-ip 干扰验证

本地 `dig story-mkfast.uk` 返回 `198.18.0.233`（198.18.0.0/15 是保留段）——这是本机代理的 fake-ip 模式拦截了 DNS，导致本地 curl 全部 000。
解法：绕开系统 DNS，用 DoH 验证真实解析结果：

```bash
curl -s -H 'accept: application/dns-json' "https://1.1.1.1/dns-query?name=story-mkfast.uk&type=A"
```

## 五、结果与感受

- https://story-mkfast.uk 全站 200，首页/文章/CLI/Matrix 全部正常
- 静态导出 + Cloudflare 边缘缓存，首屏直接命中最近节点，速度明显比原来的动态博客快
- 以后更新：加一篇 md → `pnpm build && npx wrangler pages deploy out --project-name=story-mkfast --branch=main`

整个过程 Kimi 负责了全部代码改动和部署命令，我只做了三件事：回答几个选择题、浏览器里点授权、建了一个 API token。
