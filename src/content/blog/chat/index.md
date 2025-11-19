---
title: 零成本部署 LobeChat 服务端版（支持 OpenRouter + 多端云同步）
publishDate: 2025-11-19 11:27:00
description: '本文介绍如何利用 Vercel 部署 LobeChat 的服务端数据库版本。'
tags:
  - LLM
  - environment
heroImage: { src: './thumbnail.png', color: '#2e609aff' }
language: 'Chinese'
---

本文介绍如何利用 Vercel 部署 LobeChat 的服务端数据库版本。相比普通的一键部署，本方案的优势在于：
1.  **多端实时同步**：利用 Postgres 数据库同步聊天记录，电脑和手机无缝切换。
2.  **聚合 AI 模型**：直接接入 OpenRouter，无需中转即可使用 GPT-4、Claude 3、Gemini 等所有模型。
3.  **完全免费**：利用各个大厂的免费额度（Neon, Clerk, Cloudflare R2）搭建企业级服务。


## 第一步：准备三个核心外部服务

由于 Vercel 是无状态环境，我们需要分别准备数据库（存文字）、对象存储（存图片）和身份验证服务。

### 1. 数据库：Neon (Postgres)
*   注册 [Neon](https://neon.tech/) 并创建一个新项目。
*   在 Dashboard 复制 **Connection String**。
*   **用途**：作为 `DATABASE_URL`，存储聊天记录。

### 2. 身份验证：Clerk
*   注册 [Clerk](https://clerk.com/) 并创建一个 Application。
*   在 **API Keys** 页面获取：
    *   `Publishable Key`
    *   `Secret Key`
*   **用途**：管理用户登录，防止通过 Vercel 链接白嫖你的服务。

### 3. 对象存储：Cloudflare R2
LobeChat 服务端模式必须配置 S3 存储才能上传图片/头像。
*   登录 Cloudflare Dashboard，进入 **R2**。
*   **创建存储桶**：比如命名为 `lobechat`。
*   **开启公开访问**：在设置中开启 Public Access，复制 `Public R2.dev URL`。
*   **配置 CORS**：在设置的 CORS Policy 中添加规则，允许所有来源（Origins 填 `*`，Methods 全选），确保前端能跨域上传。
*   **获取密钥**：点击右上角“管理 R2 API 令牌” -> “创建 API 令牌”。
    *   **权限必须选**：**管理员读和写 (Object Read & Write)**。
    *   记录生成的：`Access Key ID`, `Secret Access Key`, `Endpoint`。

---

## 第二步：GitHub 项目设置

为了方便后续自动更新，不要直接使用一键部署按钮。

1.  访问 [LobeChat GitHub 仓库](https://github.com/lobehub/lobe-chat)。
2.  点击右上角 **Fork**，将项目复制到你自己的账号下。
3.  **重要**：确保你 Fork 的分支是 **`main`** (稳定版)，不要用 `next` (开发版)，否则极其容易报错。

---

## 第三步：在 Vercel 导入并配置环境变量

1.  登录 Vercel，点击 **Add New Project**，导入刚才 Fork 的仓库。
2.  在 **Environment Variables** 中，填入以下 **15 个关键变量**（缺一不可）：

### 1. 基础应用配置
| 变量名                     | 值/说明                                              |
| :------------------------- | :--------------------------------------------------- |
| `NEXT_PUBLIC_SERVICE_MODE` | `server` (必填，开启服务端数据库模式)                |
| `APP_URL`                  | `https://你的域名.vercel.app`                        |
| `KEY_VAULTS_SECRET`        | 随机生成一串字符 (如 `openssl rand -base64 32` 生成) |

### 2. 数据库 (Neon)
| 变量名         | 值/说明                                 |
| :------------- | :-------------------------------------- |
| `DATABASE_URL` | Neon 第一步获取的 `postgres://...` 链接 |

### 3. 身份验证 (Clerk)
| 变量名                              | 值/说明                                |
| :---------------------------------- | :------------------------------------- |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clerk 的 Publishable Key               |
| `CLERK_SECRET_KEY`                  | Clerk 的 Secret Key                    |
| `CLERK_WEBHOOK_SECRET`              | **(暂空，第四步部署失败一次后回来填)** |

### 4. AI模型 (OpenRouter)
| 变量名               | 值/说明               |
| :------------------- | :-------------------- |
| `ENABLED_OPENROUTER` | `1`                   |
| `OPENROUTER_API_KEY` | 你的 `sk-or-...` 密钥 |

### 5. 对象存储 (Cloudflare R2) - *易错点*
| 变量名                 | 值/说明                                                                                                                                                          |
| :--------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `S3_BUCKET`            | 你的存储桶名字 (如 `lobechat`)                                                                                                                                   |
| `S3_REGION`            | `auto`                                                                                                                                                           |
| `S3_PUBLIC_DOMAIN`     | R2 的公开链接 (如 `https://pub-xxx.r2.dev`，**末尾不要带 `/`**)                                                                                                  |
| `S3_ACCESS_KEY_ID`     | R2 的 Access Key ID                                                                                                                                              |
| `S3_SECRET_ACCESS_KEY` | R2 的 Secret Access Key                                                                                                                                          |
| `S3_ENDPOINT`          | **核心易错点**：填入 R2 的 Endpoint，**必须删掉末尾的存储桶名**。<br>正确示范：`https://xxx.r2.cloudflarestorage.com`<br>错误示范：`https://xxx....com/lobechat` |

---

## 第四步：初次部署与 Webhook 修复

1.  点击 **Deploy**。初次部署可能会失败，或者部署成功但无法注册用户，因为缺少 Webhook。
2.  **配置 Clerk Webhook**：
    *   进入 Clerk 后台 -> Webhooks -> Add Endpoint。
    *   URL 填：`https://你的Vercel域名.app/api/webhooks/clerk`。
    *   **Subscribe to events**：勾选 `user` 和 `session` 开头的所有事件。
    *   创建成功后，在右下角找到 **Signing Secret** (以 `whsec_` 开头)。
3.  **回填变量**：
    *   回到 Vercel 的 Environment Variables，添加/更新 `CLERK_WEBHOOK_SECRET`。
4.  **Redeploy**：去 Deployments 页面点击 Redeploy。

---

## 第五步：前端最终设置

部署成功并登录后，可能会发现无法发送消息，提示 "OpenAI API Key Missing"。

1.  进入 LobeChat **设置 -> 语言模型**。
2.  **OpenRouter**：确认开关已开启，点击“检查连通性”。
3.  **OpenAI**：建议直接**关闭**该选项卡，防止 UI 默认调用 OpenAI 官方接口。
4.  回到聊天界面，点击顶部的模型名称，向下滚动找到 **OpenRouter 分组**，选择该分组下的模型（如 Gemini Pro, GPT-4o 等）。

**至此，部署完成。** 你现在拥有了一个在网页端、手机端（PWA）完全同步历史记录，且支持图片上传的私人 AI 助手。