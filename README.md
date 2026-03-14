<h1>ResonanceX</h1>
<p>Next.js 16、React 19 和 Chatterbox TTS 构建的 AI 驱动文本转语音与声音克隆系统</p>

产品链接：[ResonanceX](https://resonancex-production.up.railway.app/)

<br />

<p>
  <a href="https://cwa.run/clerk"><img src="https://img.shields.io/badge/Clerk-6C47FF?style=for-the-badge&logo=clerk&logoColor=white" alt="Clerk" /></a>&nbsp;
  <a href="https://cwa.run/railway"><img src="https://img.shields.io/badge/Railway-0B0D0E?style=for-the-badge&logo=railway&logoColor=white" alt="Railway" /></a>&nbsp;
  <a href="https://cwa.run/sentry"><img src="https://img.shields.io/badge/Sentry-362D59?style=for-the-badge&logo=sentry&logoColor=white" alt="Sentry" /></a>&nbsp;
  <a href="https://cwa.run/prisma"><img src="https://img.shields.io/badge/Prisma-2D3748?style=for-the-badge&logo=prisma&logoColor=white" alt="Prisma" /></a>
</p>

</div>

## Getting Started

### Prerequisites

- Node.js **20.9** 
- [Prisma Postgres] 数据库
- [Clerk] account （开启组织功能）
- [Cloudflare R2] 存储桶
- [Modal] account （用于 GPU 托管的 TTS）
  
### 1. Clone and install

```bash
git clone https://github.com/lihecen/ResonanceX.git
cd resonance
npm install
```

### 2. Configure environment

```bash
cp .env.example .env
```

在 `.env` 文件中填写空白的配置值。合理的默认值（如 Clerk 路由、APP_URL 等）已经预先填好

### 3. Set up the database

```bash
npx prisma migrate deploy
```

### 4. Deploy the TTS engine

由于 chatterbox_tts.py 基于 Modal 官方的 Chatterbox TTS 示例进行了改写，所以建议修改为直接从个人的 R2 存储桶中读取语音参考音频，而不是从 Modal Volume 中读取

在部署之前，请在 chatterbox_tts.py 中更新为个人的 R2 凭证信息

```python
R2_BUCKET_NAME = "<your-r2-bucket-name-here>"
R2_ACCOUNT_ID = "<your-r2-account-id-here>"
```

然后在你的 Modal 控制台中创建所需的密钥：
| Secret Name | Keys | Description |
|-------------|------|-------------|
| `cloudflare-r2` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | R2 API 凭证（用于挂载存储桶） |
| `chatterbox-api-key` | `CHATTERBOX_API_KEY` | 用于保护接口的 API 密钥（可以使用任意强随机字符串） |
| `hf-token` | `HF_TOKEN` | Hugging Face 令牌（用于下载 Chatterbox 模型权重）|

部署模型:

```bash
modal deploy chatterbox_tts.py
```

这会将 Chatterbox TTS 部署到 Modal 上的无服务器 NVIDIA A10G GPU。容器会以只读方式挂载你的 R2 存储桶，从而可以直接访问语音参考音频。将生成的 Modal URL 作为 CHATTERBOX_API_URL 填写到个人的 .env.local 文件中

> **Note:** 在一段时间未使用后发出的第一次请求可能会花费更长时间，因为 Modal 需要冷启动并配置 GPU 容器

部署完成后，根据 OpenAPI 规范生成类型安全的 Chatterbox 客户端：

```bash
npm run sync-api
```

### 5. Seed voices

```bash
npx prisma db seed
```

将 20 个内置声音初始化导入到数据库和 R2 中。系统语音的 WAV 文件已包含在仓库中，来源于 Modal 的语音示例包, 可以根据个人喜好和风格自主添加

### 6. Run

```bash
npm run dev
```

打开 [http://localhost:3000](http://localhost:3000) 即可看到效果

## Self-Hosting

Resonance 设计为自托管部署。需要准备：

1. **A PostgreSQL database**  - Prisma Postgres（推荐），或任何托管的 Postgres 数据库
2. **Cloudflare R2**  - 用于音频存储
3. **Modal**  - 用于无服务器 GPU 推理
4. **Clerk**  - 用于身份认证和多租户支持

将 Next.js 应用部署到任何支持 Node.js 的主机上（例如 Railway、Docker 等）

## Project Structure

```
src/
├── app/                        # Next.js 应用路由（App Router）
│   ├── (dashboard)/            # 受保护的路由（主页、TTS、语音）
│   ├── api/                    # 音频代理路由 + tRPC 处理器
│   ├── sign-in/                # Clerk 身份认证页面
│   └── sign-up/
├── components/                 # 共享 UI 组件（基于 shadcn/ui + 自定义组件）
├── features/
│   ├── dashboard/              # 主页，快捷操作
│   ├── text-to-speech/         # TTS 表单、音频播放器、设置、历史记录
│   ├── voices/                 # 语音库、语音创建、语音录制
│   └── billing/                # 使用量显示、结算（支付）
├── hooks/                      # 应用全局 Hooks
├── lib/                        # 核心模块：数据库（db）、R2 存储、Polar、环境变量（env）、Chatterbox 客户端
├── trpc/                       # tRPC 路由、客户端、服务器辅助工具
├── generated/                  # Prisma 客户端
└── types/                      # 生成的 API 类型定义
```

## Scripts

| Command | Description |
|---------|-------------|
| `npm run dev` | 启动开发服务器 |
| `npm run build` | 生产环境构建 |
| `npm run start` | 启动生产服务器 |
| `npm run lint` | 使用 ESLint 进行代码检查（Lint） |
| `npm run sync-api` | 根据 OpenAPI 规范重新生成 Chatterbox API 的类型定义 |
