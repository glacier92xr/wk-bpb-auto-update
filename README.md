# wk-bpb-auto-update

## 🚀 快速开始（适合 Fork）

1. 点击右上角 Fork 本仓库到你的 GitHub 账号。

2. 打开你的仓库，进入 Actions 页面，点击 Enable workflows（启用 GitHub Actions）。

3. **配置 Cloudflare 凭证（Secrets）：** 在仓库 **Settings → Secrets and variables → Actions** 中添加以下敏感信息（详细说明请参阅"配置说明"章节）：

   **必需配置：**

   | 类型    | 名称            | 说明                 |
   | ------- | --------------- | -------------------- |
   | Secrets | `CF_API_TOKEN`  | Cloudflare API Token |
   | Secrets | `CF_ACCOUNT_ID` | Cloudflare 账户 ID   |

   **可选配置（如需自定义可添加）：**

   | 类型    | 名称              | 说明                                |
   | ------- | ----------------- | ----------------------------------- |
   | Secrets | `CF_VAR_UUID`     | 自定义 UUID 值                      |
   | Secrets | `CF_VAR_TR_PASS`  | 自定义密码                          |
   | Secrets | `CF_ROUTE_DOMAIN` | 自定义域名（如 `shop.example.com`） |

4. 你可以手动点击 Run workflow，也可以等待每天定时自动检查。

> 注意：确保你的仓库默认分支为 main，否则推送时可能失败。

🌟 如果觉得这个项目对你有帮助，欢迎顺手点个 Star 支持一下！

## 功能介绍

- 每天自动检查 bia-pain-bache/BPB-Worker-Panel 仓库的最新 Release
- 自动下载最新版本的 `_worker.js`
- 自动创建/复用 Cloudflare KV 命名空间并绑定到 Worker
- 同步更新本地 version.txt
- 自动提交并推送到本仓库
- **更新成功后，自动复用或创建 GitHub Issue 进行通知**

## 工作流程

GitHub Actions 会每日 **UTC 16:00（北京时间 00:00）** 自动运行，或通过手动触发执行：

1. **Checkout 仓库**：检出仓库代码到工作环境。

2. **设置 Node.js 环境**：配置最新稳定版 Node.js。

3. **安装系统依赖**：安装 `jq`、`curl`、`unzip` 用于后续操作。

4. **获取最新版本号**：根据 `RELEASE_TYPE` 环境变量获取上游仓库对应类型的最新版本：
   - `release`：获取最新正式发布版本（当前默认）
   - `prerelease`：获取最新预发布版本

5. **读取当前版本号**：从本地 `version.txt` 获取已记录的版本。

6. **版本比较**：对比最新版本与当前版本，若一致则终止流程。

7. **下载并替换 `_worker.js`**（仅版本不一致时执行）：
   - 从上游仓库下载对应版本的 `worker.js`
   - 替换本地 `_worker.js` 文件
   - 更新 `version.txt` 为最新版本号

8. **提交并推送更新**（仅版本不一致时执行）：
   - 配置 Git 用户信息（`github-actions[bot]`）
   - 提交 `_worker.js` 和 `version.txt` 更改
   - 推送到远程仓库主分支

9. **发送更新通知**（仅更新成功时执行）：
   - 查找带有 `auto-update-status-issue` 标签的现有 Issue
   - 若存在，在该 Issue 下添加评论（含更新时间、版本类型、版本号）
   - 若不存在，创建新 Issue 并添加标签

10. **初始化环境**（仅版本不一致时执行）：
    - 执行 `npm install` 安装 `wrangler` 依赖

11. **KV 命名空间创建/绑定**（仅版本不一致时执行）：
    - 执行 `scripts/step-kv.sh` 脚本
    - 检查同名 KV 是否已存在，若存在则复用，若不存在则创建
    - 自动更新 `wrangler.toml` 中的 KV 命名空间 ID

12. **更新 wrangler.toml 变量**（仅版本不一致时执行）：
    - 若配置了 `CF_VAR_UUID` Secrets，更新 `UUID` 变量
    - 若配置了 `CF_VAR_TR_PASS` Secrets，更新 `TR_PASS` 变量
    - 若配置了 `CF_ROUTE_DOMAIN` Secrets，添加 `[[routes]]` 配置并设置 `workers_dev = false`

13. **部署到 Cloudflare Workers**（仅版本不一致时执行）：
    - 执行 `npm run deploy` 部署 Worker

## 📂 目录结构

```text
/
├── _worker.js              # Cloudflare Worker 主文件（上游同步）
├── version.txt             # 当前版本记录
├── wrangler.toml           # Cloudflare Workers 配置
├── package.json            # 项目依赖配置
├── scripts/                # 自动化脚本目录
│   ├── step-kv.sh         # KV 命名空间创建/绑定脚本 (Linux/macOS)
│   ├── step-kv.ps1        # KV 命名空间创建/绑定脚本 (Windows)
│   └── backups/           # 配置文件备份目录（自动生成）
├── .github/
│   └── workflows/
│       └── update_worker.yml    # GitHub Actions 工作流
├── .gitignore              # Git 忽略配置
├── .gitattributes          # Git 属性配置（脚本换行符处理）
├── README.md               # 项目主文档
```

## ⚙️ 配置说明

### 1. GitHub Secrets（敏感信息）

在仓库设置中添加以下 Secrets：

| 名称              | 必填 | 说明                                                                                                                                                            |
| ----------------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CF_API_TOKEN`    | 是   | Cloudflare API Token，需包含 `账户:Workers 脚本:编辑`、`账户:Workers KV存储:编辑`、`区域:Workers 路由:编辑`、`用户:用户详细信息:读取` 权限和 自定义域名编辑权限 |
| `CF_ACCOUNT_ID`   | 是   | Cloudflare 账户 ID，可在 Dashboard URL 中获取                                                                                                                   |
| `CF_VAR_UUID`     | 否   | 自定义 UUID 值，用于更新 wrangler.toml 中的 `UUID` 变量                                                                                                         |
| `CF_VAR_TR_PASS`  | 否   | 自定义密码，用于更新 wrangler.toml 中的 `TR_PASS` 变量                                                                                                          |
| `CF_ROUTE_DOMAIN` | 否   | 自定义域名（如 `shop.example.com`），配置后会自动设置 `workers_dev=false` 并添加 `[[routes]]`                                                                   |

> **提示**：`CF_API_TOKEN` 权限说明：
>
> - `Workers:Edit`：部署和更新 Worker
> - `Workers Routes:Edit`：配置自定义路由
> - `Workers KV:Write`：创建和管理 KV 命名空间

### 2. wrangler.toml 配置说明

项目根目录下的 `wrangler.toml` 是 Cloudflare Workers 的核心配置文件，包含以下关键配置项：

| 配置项               | 说明                                                                  |
| -------------------- | --------------------------------------------------------------------- |
| `name`               | Worker 名称                                                           |
| `main`               | 入口文件路径                                                          |
| `compatibility_date` | 兼容性日期，指定 Worker 运行的运行时版本                              |
| `workers_dev`        | 是否启用 workers.dev 子域名                                           |
| `preview_urls`       | 是否启用预览 URL                                                      |
| `[vars]`             | 环境变量配置，如 `UUID`（节点 ID）、`TR_PASS`（密码）等               |
| `[[kv_namespaces]]`  | KV 命名空间绑定，`binding` 为代码中引用的名称，`id` 为实际命名空间 ID |
| `[observability]`    | 可观测性配置，包含日志（logs）和调用追踪（traces）                    |

> **注意**：`wrangler.toml` 中的 KV 命名空间 ID 会在工作流首次运行时自动创建并更新，无需手动配置。
>
> **环境变量说明**：
>
> - `UUID`：节点唯一标识符，建议使用随机生成的值
> - `TR_PASS`：认证密码，用于 Worker 身份验证
>
> **KV 命名空间脚本特性**（`scripts/step-kv.sh` 和 `scripts/step-kv.ps1`）：
>
> - **自动复用**：会检查是否存在同名的 KV 命名空间，避免重复创建
> - **配置备份**：更新 `wrangler.toml` 前自动备份现有配置到 `scripts/backups/` 目录
> - **重试机制**：KV 操作失败时自动重试最多 3 次，采用指数退避策略
> - **跨平台**：提供 Bash（Linux/macOS）和 PowerShell（Windows）两个版本

### 3. 环境变量配置

工作流中定义了以下环境变量（硬编码在 workflow 文件中）：

| 变量名         | 值                              | 说明                                                      |
| -------------- | ------------------------------- | --------------------------------------------------------- |
| `GITHUB_REPO`  | bia-pain-bache/BPB-Worker-Panel | 上游仓库地址                                              |
| `RELEASE_TYPE` | release                         | 更新版本类型，`release` 为正式版，`prerelease` 为预发布版 |
| `KV_NAME`      | wb-bp-kv                        | KV 命名空间名称                                           |

> 如需修改 `RELEASE_TYPE` 或 `KV_NAME`，请直接编辑 `.github/workflows/update_worker.yml` 文件中的 `env` 区块。

### 4. 更新类型配置

当前工作流**固定更新正式发布版本**（`RELEASE_TYPE: release`）。

如需更新到预发布版本，需手动修改 workflow 文件中的 `RELEASE_TYPE` 环境变量值为 `prerelease`。

### 5. 更新成功通知

- 工作流在成功更新并提交代码后，会尝试复用一个特定的 GitHub Issue 进行通知。
- 该 Issue 的标题统一为 `_worker.js 自动更新通知`，并带有 `auto-update-status-issue` 标签。
- 如果该 Issue 已存在，新的更新信息将作为评论添加到该 Issue 中，这样可以保持通知集中在一个地方。
- 如果该 Issue 不存在，工作流会创建一个新的 Issue。
- 您可以通过关注该仓库的 Issue 动态来接收通知。

### 6. 本地开发

```bash
# 安装依赖
npm install

# 本地开发
npm run dev

# 部署到 Cloudflare
npm run deploy
```

## 📜 开源协议

本项目使用 MIT License 开源。

您可以自由地使用、复制、修改和分发本项目，前提是附带原始许可证声明。

## ⚠️ 免责声明

1. 本项目（"wk-bpb-auto-update"）仅供**教育、科学研究及个人安全测试**之目的。
2. 使用者在下载或使用本项目代码时，必须严格遵守所在地区的法律法规。
3. 作者 **glacier92xr** 对任何滥用本项目代码导致的行为或后果均不承担任何责任。
4. 本项目不对因使用代码引起的任何直接或间接损害负责。
5. 建议在测试完成后 24 小时内删除本项目相关部署。

## 📢 特别说明

- 本仓库同步的内容来源于 [BPB-Worker-Panel](https://github.com/bia-pain-bache/BPB-Worker-Panel)。
- 原项目版权归原作者所有，本项目仅用于自动同步更新，不对原内容进行修改。

## 🛠 代码引用

- [byJoey/wk-Auto-update](https://github.com/byJoey/wk-Auto-update)

## Star History

![Star History Chart](https://api.star-history.com/svg?repos=glacier92xr/wk-bpb-auto-update&type=Timeline)
