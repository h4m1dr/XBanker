<div align="center">
<br>
<img width="200" src="https://raw.githubusercontent.com/Yu9191/sub-store-workers/main/png/cloudflare4.png" alt="Sub-Store Workers">
<br>
<br>
<h2 align="center">Sub-Store Workers</h2>
</div>

<p align="center" color="#6a737d">
A port of Cloudflare Workers to the Sub-Store backend
</p>

<p align="center">
<a href="https://deploy.workers.cloudflare.com/?url=https://github.com/Yu9191/sub-store-workers">
<img src="https://deploy.workers.cloudflare.com/button" alt="Deploy to Cloudflare">
</a>
<br><br>
<a href="mydocs/codemap/project-overview.md">
<img src="https://img.shields.io/badge/Architecture-Project_Overview-blue?style=flat-square" alt="Project Overview">
</a>
</p>

> **Note:** The one-click deployment button is for reference only. Since the project requires local building (esbuild + Sub-Store source code), deployment cannot actually be completed directly using this button. Please refer to the following...[手动部署步骤](#部署)。
>
> **Automatic Deployment:** This repository has a built-in GitHub Actions workflow that automatically detects and deploys updates to the upstream Sub-Store to Cloudflare daily. Enabling this only requires configuring Secrets; no local intervention is needed. See details. [GitHub Actions 自动同步上游](#同步更新)。

## Introduction

Will [Sub-Store](https://github.com/sub-store-org/Sub-Store) The backend is deployed to Cloudflare Workers/Pages, requiring no server and is free to use.

- **Zero Servers:** Running on the Cloudflare edge network
- **KV Persistence**: Data is stored in Cloudflare KV.
- **Full Functionality:** Reuses all original backend business logic (subscription management, format conversion, download, preview, etc.)
- **Pre-compiled parser:** Peggy grammars are compiled at build time, avoiding runtime `eval()`.

## Table of contents

- [部署](#部署)(Core process, it is recommended to start here)
- [进阶配置 / 平台说明](#进阶配置--平台说明)(Push notifications, environment variables, local development, etc. have been collapsed)
- [FAQ](#faq)
- [同步更新](#同步更新)
- [Surge 面板脚本](#surge-面板脚本)
- [致谢](#致谢) ｜ [许可证](#许可证)

<details>
<summary><b>Architecture Description</b>(Skip this if you only want to deploy)</summary>

```
sub-store-workers/src/        ← Workers 适配层（6 个文件）
Sub-Store/backend/src/        ← 原始后端源码（直接复用）
esbuild.js                    ← 构建脚本，通过插件桥接两者
```

Only the platform-related layers were replaced; the core logic remained unchanged.

Workers Files | Purpose |
|---|---|
| `vendor/open-api.js` | KV replaces fs, fetch replaces undici |
| `vendor/express.js` | Workers fetch handler 替换 Node express |
| `core/app.js` | Import Workers version of OpenAPI |
| `utils/env.js` Environmental Monitoring |
| `restful/token.js` | Allow Workers to issue tokens |
| `index.js` | Workers Entrance |

For more detailed project master plans, please see [`mydocs/codemap/project-overview.md`](mydocs/codemap/project-overview.md)。

</details>

## deploy

> Deployment Overview: **1. Preparation → 2. Upload Workers/Pages → 3. Set Password (Required) → 4. Connect to Frontend**
>
> Why do we need both departments?`*.workers.dev` It is blocked in the country by the Great Firewall (GFW).`*.pages.dev` Using Cloudflare CDN usually allows for direct connection.
>
> - **Custom domain available:** Workers is sufficient.
> - **No custom domain: Pages provides an API to the outside world, and Workers run Cron in the background.

### 1. Cloning Repository

```bash
# 目录结构必须如下：
# parent/
#   ├── Sub-Store/          ← 原始后端源码
#   └── sub-store-workers/  ← 本项目

git clone https://github.com/sub-store-org/Sub-Store.git
git clone https://github.com/Yu9191/sub-store-workers.git

cd sub-store-workers
npm install
```

### 2. Log in to Cloudflare

```bash
npx wrangler login
```

### 3. Create a key-value namespace

```bash
npx wrangler kv namespace create SUB_STORE_DATA
```

Will return `id` Fill in `wrangler.toml`：

```toml
[[kv_namespaces]]
binding = "SUB_STORE_DATA"
id = "你的KV命名空间ID"
```

### 4. Building & Deploying

Both need to be deployed:

| Deployment method | Domain name | Purpose |
|---|---|---|
| **Workers** | `*.workers.dev` Or a custom domain | API + Cron scheduled synchronization |
| **Pages** | `*.pages.dev` | API (Direct connection available within China) |

> ⚠️ **Before executing the deployment commands below, please ensure you have completed the following three steps: "1. Clone the repository", "2. Log in to Cloudflare", and "3. Create a KV namespace", and carefully read the following sections: "5. Connect to the frontend" and "6. API authentication".** **After deployment, the Worker is public until a password is set; anyone can manage your data.**

```bash
# Workers 部署（含 Cron Triggers）
npm run deploy

# Pages 部署（国内可用，一条命令）
npm run deploy:pages
```

> **It is strongly recommended to set an authentication password immediately after deployment**, otherwise anyone can manage your Sub-Store data.
>
> ```bash
> # Windows
> npm run rotate-secret
>
> # Linux / macOS
> npm run rotate-secret:sh
> ```
>
> The script generates a random URL-safe password, writes it to the Cloudflare Worker Secret, and copies it to the clipboard. See "6. API Authentication" below for details.

> After Pages is deployed, you also need to configure it in the Cloudflare Dashboard:
>
> 1. Bind key-value namespaces `SUB_STORE_DATA`
> 2. Set the authentication password (Secret) `SUB_STORE_FRONTEND_BACKEND_PATH`
>
> Detailed steps with pictures are below. [6. API 鉴权 → 方式 A.2 Pages 端](#6-api-鉴权强烈建议--已在第-4-步完成可跳过)You must run it again after configuration. `npm run deploy:pages` Make the binding effective.

<details>
<summary><b>Important notes for custom domains (if you have bound your own domain)</b></summary>

- The SSL/TLS encryption mode must be set to **Full** (Cloudflare Dashboard → Domain → SSL/TLS → Overview)
- Cloudflare free SSL certificates only cover **first-level subdomains**.`*.example.com`Multi-level subdomains are not supported (e.g., `a.b.example.com`）
  - correct `substore.example.com`
  - mistake `substore.sub.example.com`(will lead to) `ERR_CONNECTION_CLOSED`）

</details>

### 5. Connect to the front end

Open [Sub-Store 前端](https://sub-store.vercel.app)Backend address format:

```text
你的域名/你的密码
```

For example:

```text
https://sub-store-workers.your.workers.dev/aBc123XyZ
https://sub-store.your.pages.dev/aBc123XyZ
```

> Note: ** at the end `/密码` ** cannot be omitted, otherwise `/api/...` All will be 401.

Access after deployment `https://你的域名/你的密码/api/utils/worker-status`The response should be:

```json
{ "kv": { "bound": true }, "auth": { "backendPathConfigured": true } }
```

### 6. API Authentication (Strongly recommended/Can be skipped if already completed in step 4)

> The default API has no password protection. **Anyone can manage your subscription without a password.** Step 4. `npm run rotate-secret` If you have already set this up, you can skip this section.

> We recommend using **Worker/Pages Secret** (encrypted storage). **Do not** write to `wrangler.toml [vars]` Inside—that's plaintext, it will be leaked with commits, and `wrangler deploy` It will overwrite the Secret with the same name, which conflicts with the process below.

#### Method A (Recommended): Use Cloudflare Secret

##### A.1 Workers side (project's built-in script)

The repository provides a key rotation script that completes the generation, writing, and copying to the clipboard in a single command:

```bash
# Windows
npm run rotate-secret

# Linux / macOS
npm run rotate-secret:sh
```

The script will:

- Generate a 32-bit URL-safe password using encrypted random numbers.
- Automatic addition `/` prefix
- Write Cloudflare Worker Secret via pipeline `SUB_STORE_FRONTEND_BACKEND_PATH`(Password is not written to disk, not displayed on the screen, and not accessed in shell history)
- Copy to clipboard for easy pasting into front-end configuration.

After successful execution, please update accordingly:

1. Front-end and back-end addresses:`https://xxx.workers.dev<剪贴板里的新密码>`
2. If you are using GitHub Actions for automated deployment, you also need to update the repository secret (e.g., `SUB_STORE_PASSWORD_VALUE`）

> It is recommended to clear the clipboard after use:
> - PowerShell：`Set-Clipboard -Value $null`
> - macOS：`pbcopy </dev/null`
> - Linux：`wl-copy --clear` or `xsel --clipboard --clear`

You can also set it manually (fill in the input when prompted). `/` (Password at the beginning)

```bash
npx wrangler secret put SUB_STORE_FRONTEND_BACKEND_PATH
```

##### A.2 Pages (Configured in Dashboard)

`wrangler.toml` of `[[kv_namespaces]]` and `[vars]` **This only applies to Workers.** Pages projects must have their key-value pairs (KVs) separately bound and passwords set in the Cloudflare Dashboard; otherwise, the API will return a 500 error due to the missing KVs, and anyone can access the management API.

**① Go to Workers & Pages and click on the sub-store project.**

![Enter the project](png/1.png)

**② Go to Settings → Bindings, click + to add a KV namespace**

![Add binding](png/2.png)

**③ Select the KV namespace**

![Select KV](png/3.png)

**④ Fill in the variable name `SUB_STORE_DATA`Select the corresponding KV and save.

![Save binding](png/4.png)

**⑤ Go to Settings → Variables and Secrets, and click + to add**

![Add variables](png/5.png)

**⑥ Fill in the variable name `SUB_STORE_FRONTEND_BACKEND_PATH`, fill in the value `/你的密码`(Must be brought) `/` (Start with the first option), select Secret (encrypted) as the type, and save.

![Fill in the variables](png/6.png)

> You must redeploy after saving. `npm run deploy:pages` Only then will it take effect.

> Pages cannot share Worker Secrets across projects. It is recommended to separate Workers' and Pages' secrets. `SUB_STORE_FRONTEND_BACKEND_PATH` Set the same value for easy switching between front-end components.

#### Method B (not recommended): Use `wrangler.toml [vars]` plaintext variables

```toml
[vars]
SUB_STORE_FRONTEND_BACKEND_PATH = "/你的密码"
```

> Use only for temporary debugging. Writing repository files in plaintext is prone to leakage with commits; at the same time... `wrangler deploy` This will overwrite the Secret, disrupting the CI/Secret process of method A. Please use method A in production environments.

> Shared links (download/preview) are not subject to authentication and can be accessed without a password.

> **Share Button**: The "Share" button in the subscription list is only displayed when accessed via a password prefix (as opposed to upstream Docker/Node deployments). `be_merge` (Consistent behavior) Deployments without a configured password prefix will not display the share button.

---

## Advanced Configuration / Platform Description

<details>
<summary><b>Local development</b></summary>

```bash
npm run dev
```

access `http://127.0.0.1:3000`。

</details>

<details>
<summary><b>Push notifications (Bark / Pushover)</b></summary>

Supports HTTP URL push method. `wrangler.toml` Medium configuration:

```toml
[vars]
SUB_STORE_PUSH_SERVICE = "https://api.day.app/你的BarkKey/[推送标题]/[推送内容]"
```

Pages requires you to manually add an environment variable with the same name to the Dashboard. shoutrrr (command-line tool) is not supported.

</details>

<details>
<summary><b>A list of environment variables</b></summary>

| Variable | Description | Required |
|---|---|---|
| `SUB_STORE_FRONTEND_BACKEND_PATH` | API path prefix password, recommended to use Worker Secret for management | No (required in production environment) |
| `SUB_STORE_PUSH_SERVICE` | HTTP URL Push Address | No |

</details>

<details>
<summary><b>Status check interface/Script Operator limitations</b></summary>

```text
https://你的域名/你的密码/api/utils/worker-status
```

Description of returned fields:

- `kv.bound`Has the key-value pair (KV) been correctly bound?
- `auth.backendPathConfigured`: Has authentication been configured?
- `auth.managementApiPublic`Is the management API public?
- `capabilities`: Capabilities that are currently supported/not supported in the deployment (script operations, Gist backups, Cron, etc.)

**Script Operator Restriction:** Cloudflare Workers are prohibited. `eval` / `new Function`This repository will include upstream resources during the construction phase. `createDynamicFunction` Instead, explicitly state the error. Alternatives: built-in filters/operators, mihomo YAML patch, or execute a script within an external trusted service.

</details>

<details>
<summary><b>esbuild plugin</b></summary>

| Plugin | Function |
|---|---|
| `路径别名解析` | Analysis `@/` Import, prioritize Workers coverage |
| `eval 重写` Replace the eval() call with a static expression.
| `peggy 预编译` | Compile PEG grammars at build time, eliminating runtime eval |
| `Node 模块存根` | Unavailable modules such as stub fs/crypto |

</details>

<details>
<summary><b>Cron scheduled synchronization</b></summary>

The Workers version includes a built-in Cron Trigger, which automatically syncs artifacts to Gist every day at **23:55 (Beijing time)**.

Available `wrangler.toml` Modification frequency:

```toml
[triggers]
crons = ["55 15 * * *"]  # UTC 时间，+8 即北京时间
```

> Prerequisite: Configure your GitHub username and Gist Token in the front-end Settings.

</details>

<details>
<summary><b>KV read/write optimization</b></summary>

Cloudflare KV free quota: 100,000 reads/day, **1,000 writes/day**.

This project has implemented two layers of optimization:

- **Dirty flag**: Only used when calling `$.write()` / `$.delete()` Mark the dirty bit; read requests do not trigger writes.
- **Content Comparison:** Before writing, the current data is compared with the snapshot taken at the time of loading. If the content is the same, the writing is skipped (to prevent...). `$.write()` Write back the same data)
- **Edge Buffer**: Key-Value Reads set to 60 seconds. `cacheTtl`Multiple requests hitting the edge cache within a short period of time are not counted in the key-value read count.

| Operations | Key-Value Read | Key-Value Write |
|---|---|---|
| Open frontend to browse data (~8 requests) | 1 time (the rest hit the cache) | 0 times |
| Edit subscription/settings | 0~1 times | 1 time |
| Cron scheduled synchronization | 1 time | 1 time |

For personal use, there is absolutely no need to worry about exceeding the limit.

</details>

<details>
<summary><b>Workers platform limitations/unsupported features</b></summary>

### Platform restrictions

| Limitations | Explanation |
|---|---|
| **Request Timeout 30 Seconds** | The maximum time for a single request to the wall clock is 30 seconds. Subscriptions will time out if the source response is slow.
| **Outbound IP is overseas** | Subscriptions are pulled from Cloudflare nodes; some subscription sources that restrict access to domestic IPs cannot be pulled.
Push notifications only support HTTP URLs (Bark, Pushover, etc.), not push notifications.

> If your subscription source restricts access to China or has slow response times, it is recommended to use a VPS-hosted Node.js version.

### Node-specific features (not available in Workers)

| Function | Reason |
|---|---|
| Front-end static file hosting | Required `express.static` + `fs`No local file system |
| Front-end proxy middleware | Required `http-proxy-middleware`Node Exclusive |
| MMDB IP Query | Requires reading a local MMDB file (`@maxmind/geoip2-node`） |
Scheduled MMDB downloads are required. `fs.writeFile` Write to local file |
| DATA_URL Startup Recovery | Requires Node.js `fs` Write file |
| Scheduled Gist Backup Downloads | Cron script for downloading and restoring backups from Gist (also works when manually triggered) |
| `ip-flag-node.js` Script | Depends on local MMDB, available `ip-flag.js`(HTTP API) Alternative |
| jsrsasign TLS fingerprint | Global scope restrictions |
| shoutrrr push| Need `child_process` Execute command-line tools |
| Proxy Request | Workers outbound traffic uses the Cloudflare network and does not support custom HTTP/SOCKS5 proxies |

</details>

---

## FAQ

<details>
<summary><b>Frequently Asked Questions</b></summary>

**Q: Front-end prompts** `找不到 Sub-Store Artifacts Repository`**
A: This is normal; you haven't created a synchronization configuration yet. It will be automatically generated after you create the first synchronization.

**Q: Subscription pull timed out**
A: Workers have a maximum request time of 30 seconds per request. If the subscription source is slow to respond, it will time out and fail. You can try a different subscription link.

**Q: Some subscription sources return empty or error**
A: The outbound IP of the Workers is a Cloudflare node located outside of China, which means that some subscription sources that are restricted to domestic IPs cannot be fetched.

Q: How do I update to the latest version?
A: See the "Simultaneous Updates" section below.

Q: What if I forget the password I set?
A: The Worker Secret is not visible in the Dashboard and cannot be retrieved. [Directly...] `npm run rotate-secret` Simply reset it.

</details>

---

## Synchronous updates

<details>
<summary><b>Update sub-store-workers (this project)</b></summary>

```bash
cd sub-store-workers
git pull
npm run deploy
```

> The Worker Secret will not be overwritten by deploy, and the password will remain unchanged.

</details>

<details>
<summary><b>Update the Sub-Store original repository</b></summary>

When a new version is available in the original repository, execute the following manually:

```bash
cd Sub-Store
git pull

cd ../sub-store-workers
npm run deploy
```

esbuild will build from `Sub-Store/backend/src/` Read the latest source code and rebuild to include the new features.

</details>

<details>
<summary><b>GitHub Actions automatically syncs with upstream systems (recommended)</b></summary>

The warehouse has built-in `.github/workflows/sync-upstream.yml` The workflow automatically detects and deploys updates to the Sub-Store upstream daily.

#### Workflow

```
每天 00:00（北京时间）自动触发
  ↓
拉取上游最新 commit，对比已部署版本
  ↓ 有更新
安装依赖 → 运行上游测试套件
  ↓ 测试通过
构建 → 部署 Workers → 部署 Pages → 健康检查
  ↓ 全部成功
记录已部署版本 + Bark 通知

任何环节失败 → Bark 通知"同步失败"，线上版本不受影响
```

#### Configuration steps

**1. Create a Cloudflare API Token**

Open [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens) → Create Token → **Custom Token**, add the following permissions:

| Resources | Permissions | Levels |
|---|---|---|
| Account → Workers Script | Edit | Your Account |
| Account → Cloudflare Pages | Edit | Your Account |
| Account → Workers KV Storage | Edit | Your Account |
| User → User Details | Read | -- |

Account Resources: Select **Include → Your Account**.

**2. Add GitHub Repository Secrets**

Open the repository settings → Secrets and variables → Actions → New repository secret, and add the following in sequence:

| Secret Name | Value | Description |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | Token created in the previous step | Cloudflare deployment authentication |
| `CLOUDFLARE_ACCOUNT_ID` Your Account ID | Visible on the right side of the Cloudflare Dashboard homepage |
| `KV_NAMESPACE_ID` | Key-Value Namespace ID | The ID returned when creating the key-value store |
| `PAGES_PROJECT_NAME` | Pages Project Name | For example `sub-store` |
| `WORKERS_SUBDOMAIN` | Workers subdomain | For example `sub-store2`(Right now `*.sub-store2.workers.dev` (Part of the text)
| `BARK_KEY` | Bark Push Key | Optional, used for success/failure notifications |

**3. Manually trigger verification**

Open the repository Actions page → Sync Upstream Sub-Store → Run workflow → Check the box `force = true` → Run。

Once all colors are green, the configuration is successful, and it will run automatically every day thereafter.

#### Risk and failure scenarios

| Scenario | Consequences | Handling Method |
|---|---|---|
Upstream test failed. | Deployment will not proceed; production will not be affected. | Automatic retry will occur after upstream fix.
**Build failed** (Upstream introduced an API incompatible with Workers) | Deployment issues | Manual adaptation required `src/` Overlay layer, raise an issue |
**Cloudflare API Timeout/Rate Limiting** | Deployment Interrupted | Retry Automatically Next Time |
| **API Token expired or insufficient permissions** | Deployment failed | Recreate the token and update the GitHub Secret |
**Major Upstream Version Refactoring** (Directory Structure Changes) | Build Failed | Requires Manual Update of esbuild Configuration |
**Health check failed** | Workers/Pages deployed but version tag not updated | Manually check if it's working properly online |

> **Security Tip:** Please manage your Cloudflare API Token and Account ID through GitHub Secrets, and **do not** write them to any files or commit them to a repository.

> **Manual Trigger:** You can manually run the workflow at any time on the Actions page.`force = true` It will skip version checks and force deployment.

</details>

---

## Surge panel script

<details>
<summary><b>Expand to view</b></summary>

`surge/` The directory contains a Surge Panel script that allows you to monitor Cloudflare Worker usage in real time within the Surge panel.

### Function

- Workers / Pages Request Count and Percentage
- KV read/write count
- Sub-Store subscription quantity, backend version (optional)
- Chinese/English switching

### How to use

Installing modules in Surge:

```
https://raw.githubusercontent.com/Yu9191/sub-store-workers/main/surge/SubStorePanel.sgmodule
```

After installation, edit the module parameters and enter:

| Parameters | Description |
|---|---|
| `ID` | Cloudflare Account ID |
| `Token` | Cloudflare API Token |
| `Limit` Daily request limit, default `100000` |
| `SubStoreURL` | Sub-Store backend address (optional, e.g.) `https://example.com/your-path`） |
| `Lang` Language,`en` or `cn`,default `en` |

### API Token Permissions

exist [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens) To create a **Custom Token** on the page, you only need to enable **1 permission**:

Permissions | Level |
|---|---|
| Account Analytics | Read |

We recommend selecting "No expiration date" for the expiration time.

</details>

---

## Acknowledgments

based on [Sub-Store](https://github.com/sub-store-org/Sub-Store) This project is a thank you to the original author and all contributors.

## license

GPL V3


<div align="center">
<br>
<img width="200" src="https://raw.githubusercontent.com/Yu9191/sub-store-workers/main/png/cloudflare4.png" alt="Sub-Store Workers">
<br>
<br>
<h2 align="center">کارگران فروشگاه‌های فرعی</h2>
</div>

<p align="center" color="#6a737d">
انتقال Workerهای Cloudflare به بک‌اند Sub-Store
</p>

<p align="center">
<a href="https://deploy.workers.cloudflare.com/?url=https://github.com/Yu9191/sub-store-workers">
<img src="https://deploy.workers.cloudflare.com/button" alt="Deploy to Cloudflare">
</a>
<br><br>
<a href="mydocs/codemap/project-overview.md">
<img src="https://img.shields.io/badge/Architecture-Project_Overview-blue?style=flat-square" alt="Project Overview">
</a>
</p>

> **توجه:** دکمه‌ی استقرار با یک کلیک فقط برای مرجع است. از آنجایی که پروژه نیاز به ساخت محلی (esbuild + کد منبع Sub-Store) دارد، استقرار در واقع نمی‌تواند مستقیماً با استفاده از این دکمه تکمیل شود. لطفاً به موارد زیر مراجعه کنید...[手动部署步骤](#部署)。
>
> **استقرار خودکار:** این مخزن دارای یک گردش کار داخلی GitHub Actions است که به طور خودکار به‌روزرسانی‌ها را روزانه در Sub-Store بالادستی Cloudflare شناسایی و مستقر می‌کند. فعال کردن این قابلیت فقط نیاز به پیکربندی Secrets دارد؛ هیچ مداخله محلی لازم نیست. جزئیات را ببینید. [GitHub Actions 自动同步上游](#同步更新)。

## مقدمه

اراده [Sub-Store](https://github.com/sub-store-org/Sub-Store) بک‌اند روی Cloudflare Workers/Pages مستقر شده است، نیازی به سرور ندارد و استفاده از آن رایگان است.

- **بدون سرور:** در حال اجرا بر روی شبکه لبه کلودفلر
- **پایداری KV**: داده‌ها در Cloudflare KV ذخیره می‌شوند.
- **عملکرد کامل:** از تمام منطق کسب‌وکار اصلی backend (مدیریت اشتراک، تبدیل قالب، دانلود، پیش‌نمایش و غیره) دوباره استفاده می‌کند.
- **پارسِر از پیش کامپایل‌شده:** گرامرهای پگی در زمان ساخت کامپایل می‌شوند و از `eval()` در زمان اجرا اجتناب می‌کنند.

## فهرست مطالب

- [部署](#部署)(فرآیند اصلی، توصیه می‌شود از اینجا شروع کنید)
- [进阶配置 / 平台说明](#进阶配置--平台说明)(اعلان‌های فشاری، متغیرهای محیطی، توسعه محلی و غیره از بین رفته‌اند)
- [FAQ](#faq)
- [同步更新](#同步更新)
- [Surge 面板脚本](#surge-面板脚本)
- [致谢](#致谢) ｜ [许可证](#许可证)

<details>
<summary><b>شرح معماری</b>(اگر فقط می‌خواهید مستقر شوید، از این مرحله صرف نظر کنید)</summary>

```
sub-store-workers/src/        ← Workers 适配层（6 个文件）
Sub-Store/backend/src/        ← 原始后端源码（直接复用）
esbuild.js                    ← 构建脚本，通过插件桥接两者
```

فقط لایه‌های مربوط به پلتفرم جایگزین شدند؛ منطق اصلی بدون تغییر باقی ماند.

پرونده‌های کارگران | هدف |
|---|---|
| `vendor/open-api.js` | KV جایگزین fs و fetch جایگزین undici می‌شود.
| `vendor/express.js` | workerها، هندلر fetch را اجرا می‌کنند. Node express
| `core/app.js` | وارد کردن نسخه Worker از OpenAPI |
| `utils/env.js` پایش محیطی |
| `restful/token.js` | اجازه دادن به کارگران برای صدور توکن |
| `index.js` | ورودی کارگران |

برای جزئیات بیشتر در مورد طرح‌های جامع پروژه، لطفاً مراجعه کنید به [`mydocs/codemap/project-overview.md`](mydocs/codemap/project-overview.md)。

</details>

## مستقر کردن

> مرور کلی استقرار: **۱. آماده‌سازی → ۲. آپلود Workerها/صفحات → ۳. تنظیم رمز عبور (الزامی) → ۴. اتصال به Frontend**
>
> چرا به هر دو بخش نیاز داریم؟`*.workers.dev` توسط فایروال بزرگ (GFW) در کشور مسدود شده است.`*.pages.dev` استفاده از CDN کلودفلر معمولاً امکان اتصال مستقیم را فراهم می‌کند.
>
> - **دامنه سفارشی موجود است:** تعداد کارگران کافی است.
> - **بدون دامنه سفارشی: صفحات یک API به دنیای خارج ارائه می‌دهند و کارگران Cron را در پس‌زمینه اجرا می‌کنند.

### ۱. مخزن کلونینگ

```bash
# 目录结构必须如下：
# parent/
#   ├── Sub-Store/          ← 原始后端源码
#   └── sub-store-workers/  ← 本项目

git clone https://github.com/sub-store-org/Sub-Store.git
git clone https://github.com/Yu9191/sub-store-workers.git

cd sub-store-workers
npm install
```

### ۲. وارد کلودفلر شوید

```bash
npx wrangler login
```

### ۳. ایجاد یک فضای نام کلید-مقدار

```bash
npx wrangler kv namespace create SUB_STORE_DATA
```

باز خواهد گشت `id` پر کنید `wrangler.toml`:

```toml
[[kv_namespaces]]
binding = "SUB_STORE_DATA"
id = "你的KV命名空间ID"
```

### ۴. ساخت و استقرار

هر دو باید مستقر شوند:

| روش استقرار | نام دامنه | هدف |
|---|---|---|
| **کارگران** | `*.workers.dev` یا یک دامنه سفارشی | همگام‌سازی زمان‌بندی‌شده API + Cron |
| **صفحات** | `*.pages.dev` | API (اتصال مستقیم در داخل چین موجود است) |

> ⚠️ **قبل از اجرای دستورات استقرار زیر، لطفاً مطمئن شوید که سه مرحله زیر را انجام داده‌اید: «۱. کلون کردن مخزن»، «۲. ورود به Cloudflare» و «۳. ایجاد فضای نام KV» و بخش‌های زیر را با دقت بخوانید: «۵. اتصال به فرانت‌اند» و «۶. احراز هویت API».** **پس از استقرار، Worker تا زمانی که رمز عبور تنظیم نشود، عمومی است؛ هر کسی می‌تواند داده‌های شما را مدیریت کند.**

```bash
# Workers 部署（含 Cron Triggers）
npm run deploy

# Pages 部署（国内可用，一条命令）
npm run deploy:pages
```

> **اکیدا توصیه می‌شود بلافاصله پس از استقرار، یک رمز عبور احراز هویت تنظیم کنید**، در غیر این صورت هر کسی می‌تواند داده‌های فروشگاه فرعی شما را مدیریت کند.
>
> ```bash
> # Windows
> npm run rotate-secret
>
> # Linux / macOS
> npm run rotate-secret:sh
> ```
>
> این اسکریپت یک رمز عبور تصادفی و ایمن برای URL ایجاد می‌کند، آن را در Cloudflare Worker Secret می‌نویسد و در کلیپ‌بورد کپی می‌کند. برای جزئیات بیشتر به بخش «۶. احراز هویت API» در زیر مراجعه کنید.

> پس از استقرار Pages، باید آن را در داشبورد Cloudflare نیز پیکربندی کنید:
>
> ۱. اتصال فضاهای نام کلید-مقدار `SUB_STORE_DATA`
> ۲. رمز عبور احراز هویت (مخفی) را تنظیم کنید `SUB_STORE_FRONTEND_BACKEND_PATH`
>
> مراحل دقیق به همراه تصاویر در ادامه آمده است. [6. API 鉴权 → 方式 A.2 Pages 端](#6-api-鉴权强烈建议--已在第-4-步完成可跳过)بعد از پیکربندی باید دوباره آن را اجرا کنید. `npm run deploy:pages` اتصال را مؤثر کنید.

<details>
<summary><b>نکات مهم برای دامنه‌های سفارشی (اگر دامنه خود را محدود کرده‌اید)</b></summary>

- حالت رمزگذاری SSL/TLS باید روی **کامل** تنظیم شود (داشبورد کلودفلر → دامنه → SSL/TLS → نمای کلی)
- گواهینامه‌های SSL رایگان کلودفلر فقط زیر دامنه‌های سطح اول را پوشش می‌دهند.`*.example.com`زیر دامنه‌های چند سطحی پشتیبانی نمی‌شوند (مثلاً `a.b.example.com`)
  - درست `substore.example.com`
  - اشتباه `substore.sub.example.com`(منجر خواهد شد به) `ERR_CONNECTION_CLOSED`)

</details>

### ۵. به قسمت جلویی (front end) متصل شوید

باز [Sub-Store 前端](https://sub-store.vercel.app)قالب آدرس‌های بک‌اند:

```text
你的域名/你的密码
```

برای مثال:

```text
https://sub-store-workers.your.workers.dev/aBc123XyZ
https://sub-store.your.pages.dev/aBc123XyZ
```

> توجه: ** در پایان `/密码` ** قابل حذف نیست، در غیر این صورت `/api/...` همه ۴۰۱ خواهند بود.

دسترسی پس از استقرار `https://你的域名/你的密码/api/utils/worker-status`پاسخ باید این باشد:

```json
{ "kv": { "bound": true }, "auth": { "backendPathConfigured": true } }
```

### ۶. احراز هویت API (اکیدا توصیه می‌شود/اگر در مرحله ۴ تکمیل شده است، می‌توانید از آن صرف نظر کنید)

> API پیش‌فرض هیچ رمز عبوری ندارد. **هر کسی می‌تواند اشتراک شما را بدون رمز عبور مدیریت کند.** مرحله ۴. `npm run rotate-secret` اگر قبلاً این مورد را تنظیم کرده‌اید، می‌توانید از این بخش صرف نظر کنید.

> توصیه می‌کنیم از **Worker/Pages Secret** (حافظه رمزگذاری شده) استفاده کنید. **در آن** ننویسید. `wrangler.toml [vars]` درون - این متن ساده است، با کامیت‌ها فاش خواهد شد، و `wrangler deploy` این کار باعث می‌شود که فایل مخفی با همان نام قبلی بازنویسی شود که با فرآیند زیر در تضاد است.

#### روش الف (توصیه شده): استفاده از Cloudflare Secret

##### الف.۱ سمت کارگرها (اسکریپت داخلی پروژه)

این مخزن یک اسکریپت چرخش کلید ارائه می‌دهد که تولید، نوشتن و کپی کردن در کلیپ‌بورد را در یک دستور واحد تکمیل می‌کند:

```bash
# Windows
npm run rotate-secret

# Linux / macOS
npm run rotate-secret:sh
```

اسکریپت:

- با استفاده از اعداد تصادفی رمزگذاری شده، یک رمز عبور 32 بیتی ایمن برای URL ایجاد کنید.
- جمع خودکار `/` پیشوند
- نوشتن رمز عبور کارگر Cloudflare از طریق pipeline `SUB_STORE_FRONTEND_BACKEND_PATH`(رمز عبور روی دیسک نوشته نمی‌شود، روی صفحه نمایش داده نمی‌شود و در تاریخچه پوسته قابل دسترسی نیست)
- برای چسباندن آسان در پیکربندی front-end، آن را در کلیپ‌بورد کپی کنید.

پس از اجرای موفقیت‌آمیز، لطفاً موارد زیر را به‌روزرسانی کنید:

1. آدرس‌های فرانت‌اند و بک‌اند:`https://xxx.workers.dev<剪贴板里的新密码>`
2. اگر از GitHub Actions برای استقرار خودکار استفاده می‌کنید، باید رمز مخزن (repository secret) را نیز به‌روزرسانی کنید (مثلاً `SUB_STORE_PASSWORD_VALUE`)

> توصیه می‌شود پس از استفاده، کلیپ‌بورد را پاک کنید:
> - پاورشل：`Set-Clipboard -Value $null`
> - مک‌او‌اس：`pbcopy </dev/null`
> - لینوکس：`wl-copy --clear` یا `xsel --clipboard --clear`

همچنین می‌توانید آن را به صورت دستی تنظیم کنید (در صورت درخواست، ورودی را پر کنید). `/` (رمز عبور در ابتدا)

```bash
npx wrangler secret put SUB_STORE_FRONTEND_BACKEND_PATH
```

##### صفحات A.2 (پیکربندی شده در داشبورد)

`wrangler.toml` از `[[kv_namespaces]]` و `[vars]` **این فقط برای Workerها اعمال می‌شود.** پروژه‌های Pages باید جفت‌های کلید-مقدار (KV) خود را به طور جداگانه به هم متصل کرده و رمزهای عبور را در داشبورد Cloudflare تنظیم کنند؛ در غیر این صورت، API به دلیل فقدان KVها خطای ۵۰۰ را برمی‌گرداند و هر کسی می‌تواند به API مدیریت دسترسی پیدا کند.

**① به بخش کارگران و صفحات بروید و روی پروژه زیرفروشگاه کلیک کنید.**

![وارد پروژه شوید](png/1.png)

**② به تنظیمات → پیوندها بروید، برای افزودن فضای نام KV روی + کلیک کنید**

![افزودن اتصال](png/2.png)

**③ فضای نام KV را انتخاب کنید**

![انتخاب KV](png/3.png)

**④ نام متغیر را وارد کنید `SUB_STORE_DATA`KV مربوطه را انتخاب کرده و ذخیره کنید.

![ذخیره اتصال](png/4.png)

**⑤ به تنظیمات → متغیرها و رازها بروید و برای افزودن روی + کلیک کنید**

![متغیرها را اضافه کنید](png/5.png)

**⑥ نام متغیر را وارد کنید `SUB_STORE_FRONTEND_BACKEND_PATH`، مقدار را وارد کنید `/你的密码`(حتماً باید آورده شود) `/` (با اولین گزینه شروع کنید)، نوع (رمزگذاری شده) را Secret (رمزگذاری شده) انتخاب کنید و ذخیره کنید.

![متغیرها را پر کنید](png/6.png)

> بعد از ذخیره باید دوباره مستقر شوید. `npm run deploy:pages` تنها در این صورت است که اثر خواهد کرد.

> صفحات نمی‌توانند اسرار کارگر را در بین پروژه‌ها به اشتراک بگذارند. توصیه می‌شود اسرار کارگران و صفحات را از هم جدا کنید. `SUB_STORE_FRONTEND_BACKEND_PATH` برای جابجایی آسان بین اجزای front-end، مقدار یکسانی را تنظیم کنید.

#### روش ب (توصیه نمی‌شود): استفاده `wrangler.toml [vars]` متغیرهای متن ساده

```toml
[vars]
SUB_STORE_FRONTEND_BACKEND_PATH = "/你的密码"
```

> فقط برای اشکال‌زدایی موقت استفاده شود. نوشتن فایل‌های مخزن به صورت متن ساده، مستعد نشت اطلاعات با کامیت‌ها است؛ در عین حال... `wrangler deploy` این کار باعث بازنویسی رمز (Secret) می‌شود و فرآیند CI/Secret روش A را مختل می‌کند. لطفاً در محیط‌های عملیاتی از روش A استفاده کنید.

> لینک‌های اشتراک‌گذاری‌شده (دانلود/پیش‌نمایش) نیازی به احراز هویت ندارند و بدون رمز عبور قابل دسترسی هستند.

> **دکمه اشتراک‌گذاری**: دکمه «اشتراک‌گذاری» در فهرست اشتراک‌ها فقط زمانی نمایش داده می‌شود که از طریق پیشوند رمز عبور قابل دسترسی باشد (برخلاف استقرارهای Docker/Node در بالادست). `be_merge` (رفتار ثابت) پیاده‌سازی‌های بدون پیشوند رمز عبور پیکربندی‌شده، دکمه اشتراک‌گذاری را نمایش نمی‌دهند.

---

## پیکربندی پیشرفته / توضیحات پلتفرم

<details>
<summary><b>توسعه محلی</b></summary>

```bash
npm run dev
```

دسترسی `http://127.0.0.1:3000`。

</details>

<details>
<summary><b>اعلان‌های فشاری (Bark / Pushover)</b></summary>

از روش ارسال URL از طریق HTTP پشتیبانی می‌کند. `wrangler.toml` پیکربندی متوسط:

```toml
[vars]
SUB_STORE_PUSH_SERVICE = "https://api.day.app/你的BarkKey/[推送标题]/[推送内容]"
```

صفحات از شما می‌خواهند که به صورت دستی یک متغیر محیطی با همین نام را به داشبورد اضافه کنید. shoutrr (ابزار خط فرمان) پشتیبانی نمی‌شود.

</details>

<details>
<summary><b>فهرستی از متغیرهای محیطی</b></summary>

| متغیر | توضیحات | الزامی |
|---|---|---|
| `SUB_STORE_FRONTEND_BACKEND_PATH` | رمز عبور پیشوند مسیر API، توصیه می‌شود از Worker Secret برای مدیریت استفاده شود | خیر (در محیط عملیاتی الزامی است) |
| `SUB_STORE_PUSH_SERVICE` | آدرس ارسال URL HTTP | خیر |

</details>

<details>
<summary><b>محدودیت‌های رابط/عملگر اسکریپت برای بررسی وضعیت</b></summary>

```text
https://你的域名/你的密码/api/utils/worker-status
```

شرح فیلدهای برگشتی:

- `kv.bound`آیا جفت کلید-مقدار (KV) به درستی مقید شده است؟
- `auth.backendPathConfigured`آیا احراز هویت پیکربندی شده است؟
- `auth.managementApiPublic`آیا API مدیریت عمومی است؟
- `capabilities`قابلیت‌هایی که در حال حاضر در استقرار پشتیبانی می‌شوند/پشتیبانی نمی‌شوند (عملیات اسکریپت، پشتیبان‌گیری از Gist، Cron و غیره)

**محدودیت اپراتور اسکریپت:** استفاده از کارگران Cloudflare ممنوع است. `eval` / `new Function`این مخزن شامل منابع بالادستی در طول مرحله ساخت و ساز خواهد بود. `createDynamicFunction` در عوض، خطا را صریحاً بیان کنید. جایگزین‌ها: فیلترها/عملگرهای داخلی، وصله YAML mihomo، یا اجرای یک اسکریپت در یک سرویس قابل اعتماد خارجی.

</details>

<details>
<summary><b>افزونه esbuild</b></summary>

| افزونه | تابع |
|---|---|
| `路径别名解析` | تحلیل `@/` واردات، اولویت‌بندی پوشش کارگران |
| `eval 重写` فراخوانی eval() را با یک عبارت استاتیک جایگزین کنید.
| `peggy 预编译` | کامپایل گرامرهای PEG در زمان ساخت، حذف ارزیابی زمان اجرا |
| `Node 模块存根` | ماژول‌های غیرقابل دسترس مانند stub fs/crypto |

</details>

<details>
<summary><b>همگام‌سازی زمان‌بندی‌شده‌ی کرون</b></summary>

نسخهٔ Worker شامل یک Cron Trigger داخلی است که به‌طور خودکار هر روز ساعت **۲۳:۵۵ (به وقت پکن)** مصنوعات را با Gist همگام‌سازی می‌کند.

موجود است `wrangler.toml` فرکانس اصلاح:

```toml
[triggers]
crons = ["55 15 * * *"]  # UTC 时间，+8 即北京时间
```

> پیش‌نیاز: نام کاربری GitHub و توکن Gist خود را در تنظیمات front-end پیکربندی کنید.

</details>

<details>
<summary><b>بهینه‌سازی خواندن/نوشتن KV</b></summary>

سهمیه رایگان کلودفلر کی‌وی: ۱۰۰۰۰۰ بازدید در روز، **۱۰۰۰ بازدید در روز**.

این پروژه دو لایه بهینه‌سازی را پیاده‌سازی کرده است:

- **پرچم کثیف**: فقط هنگام تماس استفاده می‌شود `$.write()` / `$.delete()` بخش کثیف را علامت بزنید؛ درخواست‌های خواندن باعث نوشتن نمی‌شوند.
- **مقایسه محتوا:** قبل از نوشتن، داده‌های فعلی با اسنپ‌شات گرفته شده در زمان بارگذاری مقایسه می‌شوند. اگر محتوا یکسان باشد، نوشتن رد می‌شود (برای جلوگیری از...). `$.write()` همان داده‌ها را دوباره بنویسید)
- **بافر لبه**: زمان خواندن کلید-مقدار روی ۶۰ ثانیه تنظیم شده است. `cacheTtl`درخواست‌های متعددی که در مدت زمان کوتاهی به حافظه پنهان لبه (edge ​​cache) می‌رسند، در شمارش خواندن کلید-مقدار محاسبه نمی‌شوند.

| عملیات | خواندن کلید-مقدار | نوشتن کلید-مقدار |
|---|---|---|
| باز کردن frontend برای مرور داده‌ها (حدود ۸ درخواست) | ۱ بار (بقیه به حافظه پنهان (cache) ارسال شد) | ۰ بار |
| ویرایش اشتراک/تنظیمات | ۰~۱ بار | ۱ بار |
| همگام‌سازی زمان‌بندی‌شده‌ی کرون | ۱ بار | ۱ بار |

برای استفاده شخصی، مطلقاً نیازی به نگرانی در مورد تجاوز از حد مجاز نیست.

</details>

<details>
<summary><b>محدودیت‌های پلتفرم Worker/ویژگی‌های پشتیبانی نشده</b></summary>

### محدودیت‌های پلتفرم

| محدودیت‌ها | توضیحات |
|---|---|
| **مهلت درخواست ۳۰ ثانیه** | حداکثر زمان برای یک درخواست به ساعت دیواری ۳۰ ثانیه است. در صورت کند بودن پاسخ منبع، اشتراک‌ها با وقفه مواجه می‌شوند.
| **آی‌پی خروجی خارج از کشور است** | اشتراک‌ها از گره‌های Cloudflare حذف می‌شوند؛ برخی از منابع اشتراک که دسترسی به آی‌پی‌های داخلی را محدود می‌کنند، قابل حذف نیستند.
اعلان‌های فشاری فقط از URLهای HTTP (مانند Bark، Pushover و غیره) پشتیبانی می‌کنند، نه اعلان‌های فشاری.

> اگر منبع اشتراک شما دسترسی به چین را محدود می‌کند یا زمان پاسخگویی کندی دارد، توصیه می‌شود از نسخه Node.js میزبانی شده توسط VPS استفاده کنید.

### ویژگی‌های خاص گره (در Workerها موجود نیست)

| عملکرد | دلیل |
|---|---|
| میزبانی فایل استاتیک front-end | الزامی `express.static` + `fs`بدون سیستم فایل محلی |
| میان‌افزار پروکسی front-end | الزامی `http-proxy-middleware`اختصاصی نود |
| پرس و جوی IP MMDB | نیاز به خواندن یک فایل MMDB محلی دارد (`@maxmind/geoip2-node`） |
دانلودهای زمان‌بندی‌شده‌ی MMDB الزامی است. `fs.writeFile` نوشتن در فایل محلی |
| بازیابی راه‌اندازی DATA_URL | به Node.js نیاز دارد `fs` نوشتن فایل |
| دانلودهای زمان‌بندی‌شده‌ی پشتیبان‌گیری از Gist | اسکریپت Cron برای دانلود و بازیابی پشتیبان‌ها از Gist (همچنین در صورت فعال‌سازی دستی نیز کار می‌کند) |
| `ip-flag-node.js` اسکریپت | بستگی به MMDB محلی دارد، در دسترس است `ip-flag.js`جایگزین (HTTP API) |
| اثر انگشت TLS با jsrsasign | محدودیت‌های دامنه سراسری |
| فریاد بزن فشار بده | نیاز `child_process` اجرای ابزارهای خط فرمان |
| درخواست پروکسی | ترافیک خروجی کارگران از شبکه Cloudflare استفاده می‌کند و از پروکسی‌های سفارشی HTTP/SOCKS5 پشتیبانی نمی‌کند.

</details>

---

## سوالات متداول

<details>
<summary><b>سوالات متداول</b></summary>

**س: درخواست‌های front-end** `找不到 Sub-Store Artifacts Repository`**
الف) این طبیعی است؛ شما هنوز پیکربندی همگام‌سازی ایجاد نکرده‌اید. این پیکربندی پس از ایجاد اولین همگام‌سازی، به طور خودکار ایجاد خواهد شد.

**س: مهلت اشتراک به پایان رسید**
الف) حداکثر زمان درخواست برای هر درخواست توسط workerها 30 ثانیه است. اگر منبع اشتراک در پاسخ دادن کند باشد، زمان آن تمام می‌شود و از کار می‌افتد. می‌توانید لینک اشتراک دیگری را امتحان کنید.

**س: برخی از منابع اشتراک، خالی یا خطا برمی‌گردانند**
الف) آی‌پی خروجی کارگران، یک گره Cloudflare است که در خارج از چین قرار دارد، به این معنی که برخی از منابع اشتراک که محدود به آی‌پی‌های داخلی هستند، قابل دریافت نیستند.

س: چگونه به آخرین نسخه به‌روزرسانی کنم؟
پاسخ: به بخش «به‌روزرسانی‌های همزمان» در زیر مراجعه کنید.

س: اگر رمز عبوری را که تنظیم کرده‌ام فراموش کنم، چه می‌شود؟
الف) راز کارگر در داشبورد قابل مشاهده نیست و قابل بازیابی نیست. [مستقیماً...] `npm run rotate-secret` به سادگی آن را دوباره تنظیم کنید.

</details>

---

## به‌روزرسانی‌های همزمان

<details>
<summary><b>به‌روزرسانی کارگران فرعی فروشگاه (این پروژه)</b></summary>

```bash
cd sub-store-workers
git pull
npm run deploy
```

> رمز کارگر با استقرار رونویسی نمی‌شود و رمز عبور بدون تغییر باقی می‌ماند.

</details>

<details>
<summary><b>مخزن اصلی Sub-Store را به‌روزرسانی کنید</b></summary>

وقتی نسخه جدیدی در مخزن اصلی موجود است، موارد زیر را به صورت دستی اجرا کنید:

```bash
cd Sub-Store
git pull

cd ../sub-store-workers
npm run deploy
```

esbuild از روی آن خواهد ساخت `Sub-Store/backend/src/` آخرین کد منبع را بخوانید و برای گنجاندن ویژگی‌های جدید، آن را بازسازی کنید.

</details>

<details>
<summary><b>GitHub Actions به طور خودکار با سیستم‌های بالادستی همگام‌سازی می‌شود (توصیه می‌شود)</b></summary>

انبار داخلی دارد `.github/workflows/sync-upstream.yml` این گردش کار به طور خودکار به‌روزرسانی‌ها را روزانه در فروشگاه فرعی بالادستی شناسایی و اعمال می‌کند.

#### گردش کار

```
每天 00:00（北京时间）自动触发
  ↓
拉取上游最新 commit，对比已部署版本
  ↓ 有更新
安装依赖 → 运行上游测试套件
  ↓ 测试通过
构建 → 部署 Workers → 部署 Pages → 健康检查
  ↓ 全部成功
记录已部署版本 + Bark 通知

任何环节失败 → Bark 通知"同步失败"，线上版本不受影响
```

#### مراحل پیکربندی

**۱. یک توکن API کلودفلر ایجاد کنید**

باز [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens) → ایجاد توکن → **توکن سفارشی**، مجوزهای زیر را اضافه کنید:

| منابع | مجوزها | سطوح |
|---|---|---|
| حساب کاربری → اسکریپت کارگران | ویرایش | حساب کاربری شما |
| حساب کاربری → صفحات کلودفلر | ویرایش | حساب کاربری شما |
| حساب کاربری → انبار KV کارگران | ویرایش | حساب کاربری شما |
| کاربر → جزئیات کاربر | خواندن | -- |

منابع حساب: **شامل شدن → حساب شما** را انتخاب کنید.

**۲. اضافه کردن اسرار مخزن گیت‌هاب**

تنظیمات مخزن → رازها و متغیرها → اقدامات → راز مخزن جدید را باز کنید و موارد زیر را به ترتیب اضافه کنید:

| نام مخفی | مقدار | توضیحات |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | توکن ایجاد شده در مرحله قبل | احراز هویت استقرار Cloudflare |
| `CLOUDFLARE_ACCOUNT_ID` شناسه حساب شما | در سمت راست صفحه اصلی داشبورد کلودفلر قابل مشاهده است |
| `KV_NAMESPACE_ID` | شناسه‌ی فضای نام کلید-مقدار | شناسه‌ای که هنگام ایجاد مخزن کلید-مقدار بازگردانده می‌شود |
| `PAGES_PROJECT_NAME` | نام پروژه صفحات | به عنوان مثال `sub-store` |
| `WORKERS_SUBDOMAIN` | زیردامنه کارگران | برای مثال `sub-store2`(همین الان `*.sub-store2.workers.dev` (بخشی از متن)
| `BARK_KEY` | کلید فشاری پارس | اختیاری، برای اعلان‌های موفقیت/شکست استفاده می‌شود |

**۳. فعال‌سازی دستی تأیید**

صفحه اقدامات مخزن را باز کنید → همگام‌سازی زیر-فروشگاه بالادستی → اجرای گردش کار → کادر را علامت بزنید `force = true` → دویدن.

زمانی که همه رنگ‌ها سبز شدند، پیکربندی موفقیت‌آمیز بوده و پس از آن هر روز به طور خودکار اجرا خواهد شد.

#### سناریوهای ریسک و شکست

| سناریو | پیامدها | روش برخورد |
|---|---|---|
تست بالادستی ناموفق بود. | استقرار ادامه نخواهد یافت؛ تولید تحت تأثیر قرار نخواهد گرفت. | پس از رفع مشکل بالادستی، تلاش مجدد خودکار انجام خواهد شد.
**ساخت ناموفق بود** (Upstream یک API ناسازگار با Workerها معرفی کرد) | مشکلات استقرار | نیاز به تطبیق دستی `src/` لایه رویی، طرح یک مشکل |
**محدودیت زمان/سرعت API کلودفلر** | استقرار متوقف شد | دفعه بعد به صورت خودکار دوباره امتحان کنید |
| **نشانه API منقضی شده یا مجوزهای کافی ندارد** | استقرار ناموفق بود | نشانه را دوباره ایجاد کنید و راز گیت‌هاب را به‌روزرسانی کنید |
**بازسازی اساسی نسخه بالادستی** (تغییرات ساختار دایرکتوری) | ساخت ناموفق | نیاز به به‌روزرسانی دستی پیکربندی esbuild دارد |
**بررسی سلامت ناموفق بود** | Workerها/صفحات مستقر شدند اما برچسب نسخه به‌روزرسانی نشد | به‌صورت دستی بررسی کنید که آیا به‌صورت آنلاین به‌درستی کار می‌کند یا خیر |

> **نکته امنیتی:** لطفاً توکن API کلودفلر و شناسه حساب خود را از طریق GitHub Secrets مدیریت کنید و **آنها را** در هیچ فایلی ننویسید یا در مخزنی ثبت نکنید.

> **راه‌اندازی دستی:** شما می‌توانید گردش کار را در هر زمانی در صفحه اقدامات به صورت دستی اجرا کنید.`force = true` این کار بررسی نسخه را رد می‌کند و استقرار را اجباری می‌کند.

</details>

---

## اسکریپت پنل سرج

<details>
<summary><b>برای مشاهده، گسترش دهید</b></summary>

`surge/` این دایرکتوری حاوی یک اسکریپت Surge Panel است که به شما امکان می‌دهد میزان استفاده از Cloudflare Worker را به صورت بلادرنگ در پنل Surge نظارت کنید.

### عملکرد

- تعداد و درصد درخواست‌های کارگران/صفحات
- تعداد خواندن/نوشتن KV
- تعداد اشتراک زیر فروشگاه، نسخه پشتیبان (اختیاری)
- تعویض چینی/انگلیسی

### نحوه استفاده

نصب ماژول‌ها در Surge:

```
https://raw.githubusercontent.com/Yu9191/sub-store-workers/main/surge/SubStorePanel.sgmodule
```

پس از نصب، پارامترهای ماژول را ویرایش کرده و وارد کنید:

| پارامترها | توضیحات |
|---|---|
| `ID` | شناسه حساب کلودفلر |
| `Token` | توکن API کلودفلر |
| `Limit` محدودیت درخواست روزانه، پیش‌فرض `100000` |
| `SubStoreURL` | آدرس بک‌اند فروشگاه فرعی (اختیاری، مثلاً) `https://example.com/your-path`） |
| `Lang` زبان،`en` یا `cn`، پیش‌فرض `en` |

### مجوزهای توکن API

وجود داشته باشد [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens) برای ایجاد یک **توکن سفارشی** در صفحه، فقط باید **1 مجوز** را فعال کنید:

مجوزها | سطح |
|---|---|
| تحلیل حساب کاربری | مطالعه |

توصیه می‌کنیم برای تاریخ انقضا، گزینه «بدون تاریخ انقضا» را انتخاب کنید.

</details>

---

## تقدیرنامه‌ها

مبتنی بر [Sub-Store](https://github.com/sub-store-org/Sub-Store) این پروژه تشکری از نویسنده اصلی و همه مشارکت‌کنندگان است.

## مجوز

مجوز عمومی نسخه ۳ (GPL V3)

