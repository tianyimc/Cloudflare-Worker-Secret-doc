# Cloudflare Worker Secret doc

极简、[开源](https://github.com/fzxx/Cloudflare-Worker-Secret-doc)端到端加密的阅后即焚文档。

## 功能特点

- 🔒 **安全加密**：使用 AES -GCM 加密保护文档内容
- ⏰ **自动销毁**：支持设置查看次数和有效期
- 📝 **Markdown**：支持 Markdown 格式的文档
- 🎨 **夜间模式**：夜间护眼不可少
- 🔄 **实时倒计时**：显示剩余查看次数、剩余时间
- 📋 **一键复制**：自动复制分享链接到剪贴板
- 💥 **手动销毁**：支持手动立即销毁文档
- 🌐 **边缘计算**：基于 Cloudflare Worker，全球极速访问
- 🛡️ **校验链接**：防止机器自动扫描
- 📌 **自定义分享链接**：支持自定义分享链接路径和类型

## 使用方法

1. **访问并创建 [Cloudflare](https://www.cloudflare.com) 账号**，Compute - Workers和Pages - 创建 - 创建应用程序，从 Hello World! 开始，名称保持默认或者随意设置，点击部署，然后编辑代码，把原来的代码删除，然后把仓库里 `Cloudflare Worker Secret doc.js` 的内容复制并粘贴到代码处，点击部署；
2. **如果想自定义分享路径、分享ID类型等，见代码头部的注释；**
3. **返回Cloudflare首页**，存储和数据库 - Workers KV，新建一个KV，名称随意，进入刚才创建的workers，点击设置 - 绑定 - 添加 - KV命名空间，变量名称填 `Worker_Secret_doc` 找到刚新建的KV，KV命名空间选择刚才创建的KV，点击保存即可使用Cloudflare提供的默认域名访问；
4. 自定义域名：**添加域名到Cloudflare后**，Workers和Pages - 找到秘密文档的 workers - 设置 - 域和路由 - 添加 - 自定义域，添加自己域名即可；例如：你的域名是 doc.com 那么可以添加 mimi.doc.com ；
5. 阻止恶意访问需要在Cloudflare防火墙设置规则；
6. 如果想删数据，进入KV按文档ID删除，或者删除整个KV。

## Cloudflare Zero Trust（写域名保护）

通过 Cloudflare Zero Trust（Access）可以将**写操作域名**（创建 / 删除文档的 `GET /` 和 `POST /submit`、`POST /delete/` 端点）设置为仅限授权用户访问，同时保持**读操作域名**（`GET /s/<id>`）完全公开。

### 工作原理

```
read.yourdomain.com   →  GET /s/<id>            公开，无需认证
write.yourdomain.com  →  GET /  POST /submit
                           POST /delete/         需通过 CF Access 认证
```

1. **Cloudflare 网络层**：Cloudflare Access 在请求到达 Worker 之前拦截写域名的流量，强制完成身份验证，并在通过验证的请求头中注入 `Cf-Access-Jwt-Assertion`。
2. **Worker 层（纵深防御）**：Worker 验证写域名的主机名，并使用 Cloudflare 公钥对 JWT 进行签名验证，双重保障。

### 配置步骤

#### 1. 修改 Worker 代码头部的 `Config`

```js
var Config = {
  // … 其他配置保持不变 …
  WriteDomain: "write.yourdomain.com",           // 写域名的主机名
  CfTeamDomain: "your-team",                     // Zero Trust 团队名（your-team.cloudflareaccess.com）
  CfAccessAudience: "your-application-aud-tag",  // Access 应用的 Audience 标签
};
```

#### 2. 部署 Worker 并绑定两个自定义域名

参考 `wrangler.toml`，将写域名和读域名都路由到同一个 Worker。你也可以在 Cloudflare 控制台 → Workers 和 Pages → 设置 → 域和路由中手动添加。

#### 3. 在 Cloudflare Zero Trust 中创建 Access 应用

1. 进入 **Cloudflare Zero Trust 控制台** → **Access** → **Applications** → **Add an application**；
2. 选择 **Self-hosted**；
3. **Application domain** 填写写域名（`write.yourdomain.com`）；
4. 配置身份验证策略（如仅允许特定邮箱 / 邮箱后缀 / GitHub 组织等）；
5. 创建完成后，在应用详情页复制 **Application Audience (AUD) Tag**，填入上方 `CfAccessAudience`；
6. 在应用的 **Overview** 页面确认 **Team domain**，填入 `CfTeamDomain`（仅填团队名，不含 `.cloudflareaccess.com`）。

#### 4. 验证效果

- 访问 `read.yourdomain.com/s/<id>` → 无需登录即可查看文档；
- 访问 `write.yourdomain.com/` → 跳转至 Cloudflare Access 登录页，认证后才能创建文档；
- 直接对写域名发起未经 Access 认证的 `POST` 请求 → 返回 `403 Forbidden`。

> **向后兼容**：若 `WriteDomain` 留空，Worker 行为与原版完全相同，方便本地调试或单域名部署。



#### 带预览链接功能的即时通讯软件、邮件，会导致链接被机器访问而失效，如何解决？

- 限制次数设置为2次或更多
- 用Cloudflare防火墙把它们拦截下来（根据UA、地区等）

#### 限制文本大小是？

- 由于免费的KV空间只有1G，因此限制每个文档的大小是100KB

#### KV空间满了，如何清理？

- 进入KV命名空间删除整个KV，**重新创建并再次绑定**；不设置自动清理是因为Api计次的，自动清理会耗尽次数。

## 更新日志

### v1.5

- 增加文档ID校验、签名验证，防止恶意请求
- 增加缓存机制，优化UI逻辑

### v1.4

- 增加可选的用户密码
- 限制文档大小，美化UI
- 分享页修改为复制Markdown格式文档

### v1.3

- 增强3秒盾，使用AES-GCM端到端加密文档，防止服务器查看，分享页倒计时增加单位秒
- 调整UI，修复BUG

### v1.2

- 修改文档ID生成方式，使其不重复
- 增加Markdown格式文档，夜间模式，最大次数、最长时间设置
- 分享页倒计时，立刻销毁功能

### v1.1

- 生成链接后自动复制


### v1.0

- 阅后即焚

- 自定义分享链接路径、长度

- 简单防扫描3秒盾
