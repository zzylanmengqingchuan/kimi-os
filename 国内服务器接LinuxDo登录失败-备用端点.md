# 国内服务器接 LinuxDo 登录失败：换备用端点 `connect.linuxdo.org`

> 日期：2026-07-24
> 场景：应用部署在**国内服务器**（如腾讯云，无翻墙出口），接入 LINUX DO Connect 第三方登录时，用户在浏览器里授权成功，回跳后却登录失败。
> 一句话方案：**服务端调用的 token / 用户信息两个端点，把域名 `connect.linux.do` 换成 `connect.linuxdo.org`，其他什么都不用改。**

## 问题现象

- 本地开发一切正常（本地挂了 VPN / 代理）。
- 部署到国内腾讯云后，登录流程卡住：用户点"同意授权"后跳回站点，提示登录失败或超时。
- 服务端日志表现为请求 `connect.linux.do` 超时 / 连接被重置。

## 原因

LinuxDo 登录（OAuth2）分两条网络链路，走的网络完全不同：

1. **浏览器侧（用户授权）**：用户浏览器跳转授权页 `connect.linux.do/oauth2/authorize`，走的是**用户自己电脑的网络**——用户能翻墙，所以这步没问题。
2. **服务端侧（换凭证）**：你的服务器拿到授权码 code 后，要**由服务器自己**请求两个接口完成登录：
   - `POST /oauth2/token`：用 code 换 access_token
   - `GET /api/user`：用 token 拉用户信息

`connect.linux.do` 域名在国内被墙/被污染，国内服务器没有翻墙出口，第 2 步直接失败——所以"浏览器里明明授权成功了，回跳却登录不上"。

## 解决方案：官方备用端点

官方（linux.do 论坛帖子 [LINUX DO Connect 服务端添加备用接口](https://linux.do/t/topic/1144530)）为此添加了国内可达的备用域名 `connect.linuxdo.org`，**专供服务端调用**：

| 用途 | 原地址（国内不通） | 备用地址（国内可用） |
|------|------|------|
| 换 token | `https://connect.linux.do/oauth2/token` | `https://connect.linuxdo.org/oauth2/token` |
| 取用户信息 | `https://connect.linux.do/api/user` | `https://connect.linuxdo.org/api/user` |

## 改动清单

在你的应用 OAuth 配置里：

- ✅ **token 端点** → 改为 `https://connect.linuxdo.org/oauth2/token`
- ✅ **userinfo / 用户信息端点** → 改为 `https://connect.linuxdo.org/api/user`
- ❌ **授权地址（authorize URL）不用改** —— 它跑在用户浏览器里，走用户自己的网络
- ❌ `client_id`、`client_secret`、`redirect_uri` 都不用动

## 注意

- 备用域名只给**服务端调用**用；官方未提供授权页的备用地址，说明授权页预期仍走原域名（用户侧网络自理）。
- 排查同类"本地行、服务器不行"的第三方登录问题，先分清是浏览器侧还是服务端侧的请求失败——服务端出网受限是国内服务器的常见坑。
