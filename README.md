# SemiIntel · 全球半导体与 AI 硬件情报数据库（MVP）

面向全球半导体与 AI 硬件产业链的可追溯情报与数据底座。  
遵循 **轻量启动、模块化设计、按需扩容** 架构原则：**Supabase + Vercel + 单台 2C4G VPS + 按量 AI API**。

## 核心铁律
1. **每条重要信息**，都能找到原始来源。
2. **每个重要参数**，都能追溯原文摘录、页码/位置并严格区分 **Series（产品系列）** 与 **SKU（具体型号）**。
3. **每个市场推测/判断**，均锁死 `D 级` 并解释依据；AI 绝不能自行将推测升级为官方事实。

## 目录结构

```text
semi-intel/
├── supabase/
│   ├── migrations/0001_init_schema.sql         # 完整三库 DDL + 证据链约束 + PG 任务表 + 商业库预留 + RLS
│   ├── migrations/0002_review_rpcs.sql         # 规格/生命周期/情报原子审核 RPC + 自动信源调度函数
│   └── seed/0001_seed_mvp_data.sql             # 35 家全球重点厂商 + 别名词典 + 20 精选高质量信源 + 示例档案
├── worker/                                     # 部署至 2C4G 轻量 VPS（Docker Compose + Python 3.11）
│   ├── src/
│   │   ├── collectors/rss.py                   # 官方 RSS 采集 + URL 规范化 + SHA-256 去重
│   │   ├── collectors/cninfo.py                # 中国巨潮资讯官方监管披露（S 级 PDF 入队）
│   │   ├── collectors/sec_edgar.py             # 美国 SEC EDGAR 官方 10-K/10-Q/8-K/6-K 披露（S 级）
│   │   ├── collectors/web_fetcher.py           # 单链接 HTML/PDF 抓取 + robots.txt 遵守 + VLM 关键页渲染
│   │   ├── parsers/pdf_parser.py               # 轻量 PyMuPDF 解析 + 复杂页按需转图走 VLM
│   │   ├── extractors/llm_extractor.py         # 统一支持 OpenAI Structured Outputs & Claude Tool-Use + VLM
│   │   ├── guardrails.py                       # 确定性防幻觉护栏 / 可信度天花板 / 冲突检测
│   │   ├── db.py                               # PostgreSQL FOR UPDATE SKIP LOCKED 任务队列（免 Redis）
│   │   └── main.py                             # Worker 定时调度与消费主循环
│   ├── tests/                                  # 护栏与 SEC 解析单元测试
│   └── docker-compose.yml                      # 4GB 内存安全限额（无多余公网端口）
└── web/                                        # 部署至 Vercel（Next.js 15 App Router）
    └── src/app/
        ├── intel/page.tsx                      # 模块一：情报中心（默认 S/A/B，可选展开 C/D）
        ├── products/[slug]/page.tsx            # 模块二：产品档案页 + 透明资料来源面板 + 生命周期时间轴
        ├── search/page.tsx                     # 全局证据与型号检索
        ├── admin/review/page.tsx               # 模块三：全类型审核台（规格冲突/生命周期/情报发布 + 人工投递）
        └── admin/sources/page.tsx              # 采集源启停、手动触发采集与 PG 任务队列监控
```

## 快速启动指南

### 1. 初始化 Supabase 数据库
1. 创建 Supabase 项目，打开 SQL Editor，按顺序依次执行：
   - `supabase/migrations/0001_init_schema.sql`
   - `supabase/migrations/0002_review_rpcs.sql`
   - `supabase/seed/0001_seed_mvp_data.sql`
2. 在 Supabase Storage 创建一个名为 `raw-evidences` 的 **Private Bucket**（严禁设为 Public）。

### 2. 部署/运行轻量 VPS Worker（2C4G Docker）
```bash
cd worker
cp .env.example .env
# 填入 DATABASE_URL / SUPABASE_SERVICE_ROLE_KEY / OPENAI_API_KEY
docker compose up -d --build
```
本地运行护栏核心单元测试：
```bash
cd worker
python -m unittest discover -s tests -v
```

### 3. 启动 / 部署前端（Vercel）
```bash
cd web
cp .env.example .env.local
npm install
npm run dev
```
直接关联 Git 仓库至 Vercel，设置 Root Directory 为 `web` 并配置 `.env.example` 中的环境变量即可上线。
