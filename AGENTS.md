# AGENTS.md — CryptoQClaw 项目指南

> 本文档面向 AI 编码助手（zcode 等），内容浓缩自 `docs/` 三大核心文档（PRD / 设计 / 技术规范），作为跨工具迁移开发的单一入口。
>
> ⚠️ 命名说明：设计文档中 OpenClaw Agent 工作区内的 `workspace/AGENTS.md`（Agent 运行时行为规范）与本文档（开发者 AI 助手项目指南）是**两个不同的文件，互不相关**。

---

## 1. 项目概述

**CryptoQClaw**（曾用名 CryptoClaw，见分支 `9b6b9eb` 更名提交）是一个 AI 驱动的加密货币量化交易助手：

- **产品形态**：对话式 AI 代理（Telegram/WhatsApp）+ 轻量 Electron 桌面客户端（仅敏感信息管理）
- **核心价值**：自然语言编写策略 → 一键回测 → 模拟/实盘交易，全部通过对话完成
- **商业模式**：免费使用，盈利分润 10%（高水位机制，亏损回本期不收费）
- **隐私架构**：本地优先（Local-First），API Key 本地 AES-256 加密，交易数据永不上云
- **目标平台**：macOS（Apple Silicon / Intel）与 Linux 完全支持；**Windows 不支持**
- GitHub: `franklili3/CryptoClaw` · 官网: `cryptoclaw.pro` · X: `@cryptoclawai`

---

## 2. 仓库与分支状态（迁移必读）

| 分支 | 内容 | 说明 |
|------|------|------|
| `main` | **仅文档** | `docs/`（PRD/设计/技术规范，中英双语）、README、images（回测图表）、assets（Logo）、LICENSE |
| `feature/week1-installer` | **MVP 实际代码（约 2 万行，171 个文件）** | Gateway、桌面客户端、Skills、安装脚本、Docker、CI。**尚未合并回 main** |
| `fix/remove-stale-lock-files` | main 的清理分支 | 删除 stale lock 文件 + .gitignore 加固，领先 main 2 个提交，待合并 |

**迁移后第一件事**：确认 `feature/week1-installer` 分支已随仓库迁移——MVP 代码全在这里，丢了它等于从文档阶段重来。

**工作区遗留物（未纳入 git）**：
- `.env` — 本地测试配置（含 LLM/Telegram/Binance/OKX 密钥变量名），**严禁提交**
- `client/dist/` — Electron 构建产物（含 CryptoClaw-1.0.0.AppImage），应保持 ignored
- `client/node_modules/`、`gateway/node_modules/` — 依赖残留，目录结构参考用

---

## 3. 系统架构

```
Telegram/WhatsApp Bot ──┐
(对话交互，全部功能)      ├──► OpenClaw Gateway (Daemon)
Electron 桌面客户端 ─────┘        │  ├─ Agent Runtime（SOUL.md / AGENTS.md / USER.md / TOOLS.md）
(仅 API Key 管理)                │  ├─ Skills：freqtrade / billing / trading-signals
                                 │  │        + hyperliquid / defillama（feature 分支新增）
                                 │  └─ Tools：exec / read / write / edit / browser / cron
                                 ▼
                          Freqtrade 量化引擎
                          (策略执行 / 数据下载 / 回测 / 交易)
                                 │
                                 ▼
                          本地存储层
                          SQLite + AES-256 加密密钥 + Freqtrade user_data
```

**云端极简原则**（Supabase + Serverless）：只做用户注册（下发 Supabase Key）、交易信号存储、账单确认存档、链上支付确认、逾期禁用/补付启用 Key、软件更新分发。
**云上明确不做**：交易记录存储、账单计算、用户 API Key 管理——全在本地。

**OpenClaw 五层架构**：Soul → Agent → Skill → Tool → Script。技能加载位置：内置 `/usr/lib/node_modules/openclaw/skills/`、托管 `~/.openclaw/skills/`、工作区 `<workspace>/skills/`。

---

## 4. 目录结构（feature/week1-installer 分支规划）

```
CryptoQClaw/
├── gateway/                  # TypeScript/Express 网关（端口 HOST_PORT=3000，OpenClaw 19001）
│   ├── src/server.ts         # API 路由：/health、skills 列表、SecureVault 初始化
│   ├── src/secure-vault/     # 密钥加密存储（local / TEE 两种实现）
│   ├── src/skills-loader.ts  # 技能扫描加载
│   ├── skills/               # defillama / freqtrade / hyperliquid / trading-signals
│   ├── Dockerfile / start.sh / tsconfig.json / package.json
├── client/                   # Electron 桌面客户端（API Key 管理，appId: pro.cryptoqclaw.client）
│   ├── src/main/             # 主进程 + preload
│   ├── src/renderer/         # 渲染进程（index.html / welcome.html）
│   └── electron-builder.yml / build.sh
├── skills/                   # OpenClaw 技能（billing / freqtrade / hyperliquid / defillama…）
├── .agents/skills/           # Binance Skills Hub 系列（algo/alpha/spot/margin/衍生品等）
├── scripts/                  # install.sh / install.ps1 / init-config.sh / init-db.sh / build-docker.sh
├── templates/config.json     # Freqtrade 配置模板
├── tests/test_installer.sh   # 安装测试（327 行）
├── docker-compose.yml        # + docker-compose.hostnet.yml / docker-bake.hcl
├── .github/workflows/docker-build.yml
├── .env.example              # 环境变量模板
├── docs/                     # 核心文档（见 §8 索引）
├── images/                   # V9 策略回测图表（freqtrade-bitcoin-strategy，100 USDT fixed stake）
└── README_ADMIN.md           # 管理员文档（仅 feature 分支）
```

**运行时目录**（用户机器，Docker 卷挂载）：

```
~/.cryptoclaw/
├── config/            # .env（敏感）/ openclaw.yaml / docker-compose.yml
├── user_data/         # Freqtrade 标准：config.json、config-private.json（密钥）、strategies/、data/、hyperopts/
├── workspace/         # OpenClaw 工作区：SOUL.md、AGENTS.md、USER.md、TOOLS.md、MEMORY.md、skills/
├── tradesv3.sqlite    # 实盘交易库（.dryrun.sqlite 为模拟）
├── cryptoclaw.db      # 业务库：用户信息、高水位、账单、支付
└── logs/
```

---

## 5. 技术栈与版本要求

| 层级 | 技术 | 版本/备注 |
|------|------|-----------|
| AI Agent | OpenClaw | 多模型支持、技能系统 |
| 量化引擎 | Freqtrade | Python 3.11+ |
| 内置策略 | freqtrade-bitcoin-strategy | BTC/ETH 均值回归（本机 `~/freqtrade-bitcoin-strategy/`） |
| 网关 | Node.js + TypeScript + Express | Node ≥ 18（开发环境 20+） |
| 桌面客户端 | Electron 30 + electron-builder 25 | electron-store / electron-updater / axios |
| 本地存储 | SQLite + AES-256-GCM | 密钥由主密码 PBKDF2（100k 迭代）派生 |
| 消息渠道 | Telegram Bot API / WhatsApp (Baileys) | |
| 云服务 | Supabase + Vercel/Cloudflare Pages + Serverless | 极简（见 §3） |
| 容器 | Docker 24+ | 镜像 `cryptoclaw/cryptoclaw`，分发主渠道 |
| LLM | OpenAI / Claude / DeepSeek / 智谱 GLM | `.env` 中 LLM_PROVIDER 配置 |

**开发命令**（feature 分支）：

```bash
# Gateway（gateway/ 目录）
npm run dev      # ts-node src/server.ts
npm run build    # tsc → dist/
npm start        # node dist/server.js
npm test         # jest
npm run lint     # eslint src/**/*.ts

# 桌面客户端（client/ 目录）
npm start        # electron .
npm run build:linux   # electron-builder（另有 build:mac / build:win / pack）

# 安装/部署（scripts/ 目录）
./install.sh             # 一键安装（macOS/Linux）
bash tests/test_installer.sh  # 安装测试
docker compose up -d     # 容器方式运行
```

---

## 6. 核心领域概念

### 6.1 高水位计费（High Watermark）

```
计费基础 = max(0, 当前累计利润 − 历史最高累计利润)
应付费用 = 计费基础 × 10%
```

- 只有创造**新**利润才收费；亏损后回本不收费
- 月度结算（每月 1 日 00:00 本地计算），用户对话确认账单后上传服务器存档
- 支付：USDT/USDC（TRC-20/ERC-20），账单生成后 7 天内支付
- 逾期：暂停实盘交易 + 禁用 Supabase Key；补付后重新启用

### 6.2 密钥层级

```
Level 0   Master Key     用户密码派生（PBKDF2），不存储
Level 1   DEK            每用户唯一，加密实际数据
Level 1.5 Supabase Key   服务器签发，逾期禁用/支付启用（业务控制手段）
Level 2   通信密钥       HMAC 请求签名 / JWT Session，定期轮换
```

### 6.3 数据模型概览（完整 Schema 见 technical-spec.md §1）

- **客户端 SQLite（15 张表）**：本地配置、用户信息、API Key（加密）、交易记录、高水位记录、策略配置、支付记录
- **服务端（8 张表）**：云用户信息、收费规则同意记录、账单确认记录、支付确认记录、支付地址、支付订单/记录
- 上传时机仅三种：用户注册、收费规则同意、账单确认（只传摘要，不传交易明细）

### 6.4 内置策略参数与回测目标

| 参数 | 值 | 回测指标 | 目标 |
|------|-----|---------|------|
| timeframe | 1d | 年化收益 | > 30% |
| stake_amount | 100 | 最大回撤 | < 15% |
| max_open_trades | 2 | 夏普比率 | > 1.5 |
| stoploss | −0.13 | 胜率 | > 55% |
| take_profit | 5（500%） | | |

策略列表：BTC 均值回归、ETH 均值回归、BTC/ETH 组合。交易信号从 Supabase 下载。

### 6.5 环境变量（.env.example，值见本地 .env，勿入库）

`TZ` / `HOST_PORT`(3000) / `GATEWAY_PORT`(19001) / `OPENCLAW_TOKEN` / `TELEGRAM_BOT_TOKEN` / `LLM_PROVIDER` / `LLM_API_KEY` / `LLM_MODEL` / `BINANCE_API_KEY` / `BINANCE_API_SECRET` / `OKX_API_KEY` / `OKX_API_SECRET` / `OKX_PASSPHRASE` / `FREQTRADE_URL`（可选 TEE_* 系列）

---

## 7. 关键流程

1. **收费规则同意**：首次启动实盘前必须同意 → 记录上传服务器 → 解锁实盘
2. **月度计费**：本地定时计算 → 对话展示账单 → 用户确认 → 上传存档 → 生成支付二维码
3. **链上支付确认**：服务端监听链上支付 → 核对金额 → 更新用户状态
4. **自动更新**：容器启动检查版本 → Telegram 通知用户确认 → `docker pull` + restart（卷挂载保数据）
5. **桌面客户端首启 7 步向导**：欢迎 → 检查 Docker → 下载/启动容器 → 注册/登录 → 配置 Telegram Bot → 配置 LLM API → 完成

---

## 8. 文档索引（docs/）

| 文档 | 内容 | 规模 |
|------|------|------|
| [requirement.md](docs/requirement.md) | PRD：产品定位、商业模式、用户旅程、功能模块、技术选型 | 1984 行 |
| [design.md](docs/design.md) | 架构设计：OpenClaw 映射、组件、数据流、安全、部署、开发计划 | 844 行 |
| [technical-spec.md](docs/technical-spec.md) | 技术规范：SQL Schema、API JSON、客户端/后端代码、Docker/CI 配置 | 2162 行 |
| `*_en.md` | 上述三份的英文版 | |
| github-discussion-2026-03-18*.md | GitHub 讨论存档 | 营销/公告类 |
| announcement-post*.md、x-*.md、github-description.md | 发布公告、X 推文、仓库介绍文案 | 同上 |

**检索建议**：改需求先查 requirement.md 对应章节；设计/架构问题查 design.md；写代码前查 technical-spec.md 的 Schema（§1）与代码模板（§3/§4）。

---

## 9. 开发约定与注意事项

- **密钥安全**：`.env`、`user_data/config-private.json` 永不入库（.gitignore 已覆盖，含 lock 文件规则）；不要在文档/日志中输出密钥值
- **Freqtrade 兼容**：`user_data/` 必须保持 Freqtrade CLI 标准结构；本机参考源码 `~/freqtrade/`、策略库 `~/freqtrade-bitcoin-strategy/`
- **本地优先红线**：新增功能不得将交易明细、API Key、账单明细上传云端；云端只收摘要
- **平台支持**：不为 Windows 编写 `.bat/.ps1` 之外的适配（install.ps1 仅为 Docker 检测）；主分发渠道是 Docker 镜像
- **提交规范**：沿用现有风格（`feat:` / `fix:` / `docs:` / `content:` 前缀，中英混用）
- **命名**：对外品牌已从 CryptoClaw 迁移到 **CryptoQClaw**（npm 包名 `cryptoqclaw-gateway` / `cryptoqclaw-client`，appId `pro.cryptoqclaw.client`）；历史文档中 "CryptoClaw" 均指本项目
- **测试**：安装流程测试在 `tests/test_installer.sh`；高水位计算单测模板见 technical-spec.md §4.4

---

## 10. 开发路线图

- [x] 需求文档与架构设计（main）
- [~] **Phase 1 MVP（Week 1-4）**：Gateway + Agent、freqtrade skill、Telegram 集成、基础对话 ← *feature/week1-installer 进行中*
- [ ] Phase 2 Trading（Week 5-7）：Freqtrade 交易集成、模拟/实盘、交易通知
- [ ] Phase 3 Payment（Week 8-9）：收费同意流程、月度计费、支付二维码、链上确认
- [ ] Phase 4 Polish（Week 10+）：性能优化、更多策略、用户反馈迭代

---

*本文档基于 2026-09-11 仓库状态生成；三大核心文档最后更新于 2026-03-18（v1.1）。内容与 docs/ 冲突时，以 docs/ 为准并更新本文档。*
