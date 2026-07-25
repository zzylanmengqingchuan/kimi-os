# 盘点 Cloudflare 账号下的域名与托管网站：一次 AI 辅助排查实录

> 日期：2026-07-25
> 背景：只记得"有个网站托管在 Cloudflare 上"，但不确定是哪个、也不知道去哪看。
> 结果：账号下实际有 **3 个域名**（一直以为只有 2 个），其中 1 个跑着 Cloudflare Pages 博客、1 个挂着 Worker 模板站、1 个只做了 DNS 解析（网站实际在 Vercel）。

## 排查方法

没有用 Cloudflare API Token，也没有翻邮箱找账号密码，而是让 AI Agent（Kimi Code CLI + web-access skill）通过 CDP（Chrome DevTools Protocol）直连日常使用的 Chrome 浏览器操作：

1. 在本机 Chrome 开启远程调试（`chrome://inspect/#remote-debugging`），Agent 通过 CDP Proxy 新开一个后台 tab，**天然携带已登录的 Cloudflare 会话**，全程不接触账号密码。
2. 依次查看 Dash 的三个关键页面：
   - **账户主页**：概览流量数据（哪个域名有访问量，一眼就暴露"有东西在跑"）。
   - **域名列表**（Domains）：列出账号下所有 zone 及状态。
   - **Workers 和 Pages**：列出所有托管项目（Cloudflare 上托管的网站只会以这两种形态存在）。
3. 逐个域名看 **DNS 记录**：DNS 记录就是"域名指向哪里"的账本——CNAME 到 `*.pages.dev` 说明是 Pages 托管，指向 Worker 说明是 Worker 托管，A/CNAME 指向外部 IP/服务商说明 Cloudflare 只做 DNS。
4. 最后逐个 `curl` 实测访问，用返回的 HTTP 状态码和 `<title>` 确认每个站点真实可访问的内容，不看控制台猜。

## 域名占用情况（2026-07-25 时点）

账号 ID：`d2cdf2b32d9d20b6000b794dbc829e22`，共 3 个活动 zone，均为 Free 计划。

| 域名 | 指向 | 托管位置 | 实际内容 | 状态 |
| --- | --- | --- | --- | --- |
| `zzylanmengqingchuan.uk` | CNAME → `achuanblog.pages.dev` | Cloudflare Pages（项目名 achuanblog） | 阿川 Blog：Agent 学习与内容创作记录 | ✅ 正常，三域名中流量最大 |
| `story-mkfast.uk` | 根域名 → Worker `story-mkfast` | Cloudflare Worker | TanStarter 空模板（TanStack 脚手架，无实际内容） | ✅ 可访问，但是空壳，准备替换为新站 |
| `xiaoxiaole.space` | A → 216.198.79.1，www → vercel-dns | **Vercel**（Cloudflare 仅 DNS） | 成语消消乐在线小游戏 | ✅ 正常 |

附带的其他记录：

- `tank.xiaoxiaole.space` → A 记录指向 152.136.211.81（自购腾讯云服务器），实测 HTTPS 不通，服务未运行或不在 443 端口。
- `xiaoxiaole.space` 下还有 Vercel 域名验证 TXT 和 google-site-verification TXT，属正常配置残留。

## Workers / Pages 免费子域名下的项目

不占用自有域名，挂在 `*.pages.dev` / `*.workers.dev` 下的项目共 9 个，实测结果：

| 项目 | 地址 | 实测 |
| --- | --- | --- |
| achuanblog | achuanblog.pages.dev | ✅ 阿川 Blog（与 zzylanmengqingchuan.uk 同站） |
| story-mkfast | story-mkfast.uk（Worker） | ✅ TanStarter 空模板 |
| weread-proxy | weread-proxy.zhuzhaoyu73.workers.dev | ✅ 微信读书代理 API（直接访问返回 405 属正常，只接受特定接口请求） |
| code0720 | code0720.pages.dev | ✅ AI工具集 · 智能工具导航平台 |
| code0720nm1 | code0720nm1.pages.dev | ✅ AI工具集（另一版本） |
| slide-2 | slide-2.pages.dev | ❌ 404（部署内容已删除） |
| code0719 | code0719.pages.dev | ❌ 404 |
| lanmengqingchuan | lanmengqingchuan.pages.dev | ❌ 404 |
| aicode | aicode-e8p.pages.dev | ❌ 404 |

## 结论与后续

- "记得托管在 Cloudflare 上的那个网站"= **阿川 Blog**（zzylanmengqingchuan.uk，Cloudflare Pages）。
- 一直以为只买了 2 个域名，实际有 3 个 zone——`story-mkfast.uk` 和多出来的那个 `.uk` 疑似低价/活动价注册，建议核对注册商账单。
- `story-mkfast.uk` 计划替换：新站是 Next.js 15.5 静态导出博客（`output: "export"`，产物在 `out/`，pnpm 构建），部署目标是 Cloudflare Pages 静态托管 + 绑定自定义域名，并解绑/删除旧的空壳 Worker。
- 4 个 404 的 Pages 项目（slide-2、code0719、lanmengqingchuan、aicode）可在控制台清理。

## 经验：自己手动排查该看哪三处

不借助 AI，在 Cloudflare Dash 里按这个顺序看即可：

1. **账户主页**：看每个域名的流量概览，有流量的必然跑着东西。
2. **Workers 和 Pages**：Cloudflare 上托管的网站只有这两种形态，列表即全部。
3. **每个域名的 DNS 记录**：CNAME 到 `*.pages.dev` / 指向 Worker = 托管在 Cloudflare；指向外部 IP 或其他服务商 = Cloudflare 只做 DNS 解析，网站在别人家。

> 教训：买过的域名随手记一笔（注册商、日期、用途），不然两年后只能靠排查才能搞清楚自己到底有几个域名。
