# wk-bpb-auto-update

## 🚀 快速开始（适合 Fork）

1. 点击右上角 Fork 本仓库到你的 GitHub 账号。

2. 打开你的仓库，进入 Actions 页面，点击 Enable workflows（启用 GitHub Actions）。

3. **配置 Cloudflare 凭证（Secrets）：**

   | 类型    | 名称            | 说明                                                                |
   | ------- | --------------- | ------------------------------------------------------------------- |
   | Secrets | `CF_API_TOKEN`  | Cloudflare API Token，需包含 Workers Edit、Workers Routes Edit 权限 |
   | Secrets | `CF_ACCOUNT_ID` | Cloudflare 账户 ID                                                  |

4. 你可以手动点击 Run workflow，也可以等待每天定时自动检查。

> 注意：确保你的仓库默认分支为 main，否则推送时可能失败。

🌟 如果觉得这个项目对你有帮助，欢迎顺手点个 Star 支持一下！

## 功能介绍

- 每天自动检查 bia-pain-bache/BPB-Worker-Panel 仓库的最新 Release

- **支持选择更新正式版或预发布版本：通过手动触发或 `update_type.txt` 文件配置。**

- 自动下载最新版本的 `_worker.js`

- 自动创建/复用 Cloudflare KV 命名空间并绑定到 Worker

- 同步更新本地 version.txt

- 自动提交并推送到本仓库

- **如果 `update_type.txt` 文件不存在，将自动创建并默认设置为更新正式版。**

- **更新成功后，自动复用或创建 GitHub Issue 进行通知。**

## 工作流程

GitHub Actions 会每日 **UTC 16:00（北京时间 00:00）** 自动运行：

1. **检查 `update_type.txt` 文件：如果文件不存在，会自动创建并写入 `1`（表示正式版）。**

2. **根据 `update_type.txt` 或手动输入确定更新类型（正式版或预发布版）。**

3. 获取上游仓库的最新 Release 版本号（根据所选类型）。

4. 比较本地 version.txt 的记录。

5. 若版本不同，则自动下载并替换 `_worker.js`，并更新 `version.txt`。

6. 自动提交并推送到主分支（main）。

7. **如果 `update_type.txt` 文件是自动创建的，也会一并提交到仓库。**

8. **初始化环境（安装 wrangler）。**

9. **执行 KV 命名空间创建/绑定流程。**

10. **更新 wrangler.toml 中的变量（如需配置）。**

11. **部署到 Cloudflare Workers。**

12. **如果更新成功并提交了更改，工作流会首先查找一个名为 `_worker.js 自动更新通知` 且带有 `auto-update-status-issue` 标签的现有 Issue。如果找到，则在该 Issue 下添加一条评论，包含更新时间、版本类型和版本号；如果未找到，则创建一个新的 Issue 并添加该标签。**

> 若版本一致，则不执行任何操作。

## 📂 目录结构

```text
/
├── _worker.js              # Cloudflare Worker 主文件（上游同步）
├── version.txt             # 当前版本记录
├── update_type.txt         # 更新类型配置 (1=正式版, 0=预发布版)
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
└── README-exp.md           # 衍生项目文档参考
```

## ⚙️ 配置说明

### 1. GitHub Secrets（敏感信息）

在仓库设置中添加以下 Secrets：

| 名称              | 必填 | 说明                                                                                          |
| ----------------- | ---- | --------------------------------------------------------------------------------------------- |
| `CF_API_TOKEN`    | 是   | Cloudflare API Token，需包含 `Workers:Edit`、`Workers Routes:Edit`、`Workers KV:Write` 权限   |
| `CF_ACCOUNT_ID`   | 是   | Cloudflare 账户 ID，可在 Dashboard URL 中获取                                                 |
| `CF_VAR_UUID`     | 否   | 自定义 UUID 值，用于更新 wrangler.toml 中的 `UUID` 变量                                       |
| `CF_VAR_TR_PASS`  | 否   | 自定义密码，用于更新 wrangler.toml 中的 `TR_PASS` 变量                                        |
| `CF_ROUTE_DOMAIN` | 否   | 自定义域名（如 `shop.example.com`），配置后会自动设置 `workers_dev=false` 并添加 `[[routes]]` |

> **提示**：`CF_API_TOKEN` 权限说明：
>
> - `Workers:Edit`：部署和更新 Worker
> - `Workers Routes:Edit`：配置自定义路由
> - `Workers KV:Write`：创建和管理 KV 命名空间

### 2. 工作流输入参数

手动触发时可配置以下参数：

| 参数           | 说明                                                 | 默认值    |
| -------------- | ---------------------------------------------------- | --------- |
| `release_type` | 更新类型：release（正式版）或 prerelease（预发布版） | release   |
| `kv-name`      | KV 命名空间名称，用于创建或查找已有的 KV 命名空间    | cf-bpb-kv |

### 3. wrangler.toml 配置说明

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

### 4. 更新类型配置（`update_type.txt`）

在仓库根目录下创建或修改 `update_type.txt` 文件：

- 文件内容为 `1`：表示定时任务将更新到**最新正式发布版本**。
- 文件内容为 `0`：表示定时任务将更新到**最新预发布版本**。
- **如果 `update_type.txt` 文件不存在，工作流会自动创建它并默认设置为 `1`（正式版）。**
- **手动触发时，您可以通过 GitHub Actions 界面选择更新类型，此选择将覆盖 `update_type.txt` 的设置。**

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

[![Star History Chart](https://api.star-history.com/svg?repos=glacier92xr/wk-bpb-auto-update&type=Timeline)](https://api.star-history.com/#glacier92xr/wk-bpb-auto-update&Timeline)
