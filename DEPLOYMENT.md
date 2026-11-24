# SemiIntel 上线与联调实操清单（Supabase + Vercel + 2C4G VPS）

本清单遵循「轻量启动、零冗余中间件」原则，按顺序约 15 分钟即可完成全链路生产部署。

---

## 第一步：初始化 Supabase（PostgreSQL + 私有证据桶）

1. 在 [Supabase](https://supabase.com/) 新建一个项目（区域建议选新加坡 / 东京以兼顾亚太与欧美信源延迟）。
2. 打开左侧 **SQL Editor**，按顺序依次新建查询并运行以下 3 个文件：
   - [0001_init_schema.sql](./supabase/migrations/0001_init_schema.sql)（建表、枚举、防幻觉约束、商业库预留表、RLS 策略、`public_evidence_citations` 视图）
   - [0002_review_rpcs.sql](./supabase/migrations/0002_review_rpcs.sql)（`spec` / `lifecycle` / `intel` 原子审核 RPC、`review_reject_claim`、`enqueue_due_source_jobs`）
   - [0001_seed_mvp_data.sql](./supabase/seed/0001_seed_mvp_data.sql)（35 家全球厂商、20 个精选信源、5 类规格字典、示例系列/SKU/产业链关系与演示冲突断言）
3. 打开左侧 **Storage** -> **New bucket**：
   - 名称：`raw-evidences`
   - **Public bucket**：**务必保持关闭（Private）**，防止原始抓取网页/PDF 快照被公网直接镜像访问。
4. 在 **Project Settings -> API** 与 **Database** 记录以下 4 个值：
   - `Project URL`（`NEXT_PUBLIC_SUPABASE_URL` / `SUPABASE_URL`）
   - `anon public key`（`NEXT_PUBLIC_SUPABASE_ANON_KEY`）
   - `service_role secret`（`SUPABASE_SERVICE_ROLE_KEY`，**仅放服务端/VPS，绝不暴露给浏览器**）
   - `Connection string (URI, Session Pooler / Direct)`（`DATABASE_URL`）

---

## 第二步：部署前端与审核台至 Vercel

1. 将 `semi-intel` 推送至你的 GitHub 私有仓库。
2. 在 [Vercel](https://vercel.com/) 导入仓库：
   - **Root Directory**：设为 `web`
   - **Framework Preset**：Next.js（自动识别）
3. 在 **Environment Variables** 配置（参考 [web/.env.example](./web/.env.example)）：
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `ADMIN_ACCESS_TOKEN`（自定义一个强口令，用于 `/admin/review` 与 `/admin/sources` 写操作鉴权）
   - `ADMIN_ACTOR_UUID`（默认 `00000000-0000-4000-8000-000000000001`，用于 `audit_logs` 记录审核人）
4. 点击 **Deploy**。上线后访问首页与 `/products/cxmt-ddr5`，确认顶部状态徽章已由「本地演示模式」切换为 **「Supabase 生产库已连接」**。

---

## 第三步：在 2C4G 轻量 VPS 启动采集与提取 Worker

1. 准备一台 2 vCPU / 4GB RAM / 40–80GB SSD 的 Ubuntu LTS 服务器，安装 Docker 与 Docker Compose 插件。
2. 将 `worker/` 目录同步至服务器（或直接 `git clone` 仓库后进入 `worker/`）：
   ```bash
   cd semi-intel/worker
   cp .env.example .env
   nano .env
   ```
3. 填写 `.env`：
   - `DATABASE_URL`（Supabase PostgreSQL 连接串）
   - `SUPABASE_URL` 与 `SUPABASE_SERVICE_ROLE_KEY`
   - `LLM_PROVIDER`：`openai` 或 `anthropic`
   - 对应填写 `OPENAI_API_KEY`（推荐 `gpt-4o-mini` 起步）或 `ANTHROPIC_API_KEY`（`claude-sonnet-4-6`）
   - `HTTP_USER_AGENT`：按 SEC EDGAR Fair Access 要求，填入真实联系邮箱（例如 `SemiIntelBot/0.1 (contact@yourdomain.com)`）
4. 启动后台守护容器（已内置 `mem_limit: 2560m` 防 OOM，且不暴露任何公网端口）：
   ```bash
   docker compose up -d --build
   docker compose logs -f worker
   ```

---

## 第四步：端到端 5 分钟验收（Smoke Test）

1. **验证产品档案与系列/SKU 隔离**：打开 `/products/cxmt-ddr5`，切换「产品系列总览 (Series)」与「SKU: DDR5-6400-RDIMM」标签，确认参数、可信度徽章与透明资料来源面板正常展示；再点击「加入规格对比」验证 `/products/compare`。
2. **验证自动调度与手动触发**：打开 `/admin/sources`，输入 `ADMIN_ACCESS_TOKEN`，对 `nvidia_newsroom` 或 `sec_edgar_filings` 点击「立即采集」，观察下方 `ingestion_jobs` 出现新任务并在 15 秒内由 VPS Worker 消费变为 `done`。
3. **验证人工投递 → AI 提取 → 护栏 → 审核签发闭环**：
   - 在 `/admin/review` 底部「人工证据投递通道」提交一篇官方公告或产品页 URL。
   - 等待 Worker 抓取消化后刷新 `/admin/review`，确认新提取的断言带着原文摘录匹配分出现在队列中。
   - 填写「审核/修改理由」并签发（或在种子冲突断言上测试「拆分口径并存发布」），去 Supabase Table Editor 核对 `verified_spec_facts` / `intel_items` 与 `audit_logs` 是否同步原子落库。
