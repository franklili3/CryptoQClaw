# 开发日志 - Day 4: AI 驱动的知识工程 — YouTube 字幕批量提取与结构化解析

## 背景

作为 CryptoClaw 量化交易 AI 助手项目的一部分，我们需要建立一套高质量的导师知识库。5 位核心导师（Andreas Clenow、Andrew Lo、Ernest Chan、Marcos López de Prado、Nassim Taleb）共 25 个 YouTube 视频，需要将原始字幕转化为结构化的中文解析文档，供后续量化交易 RAG 系统使用。

## 决策过程

### 方案一：Gemini 解析 ❌

**计划：** 将视频链接粘贴到 Google Gemini，利用其原生 YouTube 视频理解能力直接生成结构化解析。

**执行过程：**
1. 通过 Chrome CDP（远程调试端口 9222）自动化操作 Gemini 网页版
2. 尝试无痕模式登录 → 提示需要 Google 账号
3. 切换 VPN 节点（日本/美国）→ 仍提示"不支持所在国家"
4. 修改 Google 付款资料国家/地区 → 失败
5. 尝试注册新 Google 账号（用非中国手机号）→ +86 手机号暴露注册地区

**失败原因：** Google 账号地区由注册手机号决定，非 IP 地址。即使 VPN 切换到海外节点，+86 中国手机号注册的账号仍被识别为中国用户，Gemini 和 AI Studio 均不可用。

**教训：** 平台地域限制是账号级别的，不是网络级别的。代理只解决网络层问题，无法绕过账号级别的策略。

### 方案二：自行提取字幕 + AI 解析 ✅

**计划：** 通过浏览器 CDP 协议操作 YouTube 页面提取字幕文本，然后用 AI（GLM-5）生成结构化解析。

#### 子问题 1：如何访问 YouTube？

| 方案 | 结果 |
|------|------|
| 命令行代理 (v2rayA SOCKS5) | ❌ 超时，分流规则将 YouTube 走直连 |
| Python requests + 代理 | ❌ 同上 |
| yt-dlp 下载字幕 | ❌ 版本过旧 (2024.04.09)，pip 无法升级 |
| 第三方字幕 API | ❌ 大多返回 JS 渲染内容，静态抓取无效 |
| **浏览器 ZeroOmega 扩展** | ✅ 代理扩展独立于系统代理，正常访问 |

**关键发现：** ZeroOmega 浏览器扩展使用独立的代理规则和节点配置，与 v2rayA 命令行代理走不同的路由路径。浏览器能访问的外网资源，命令行工具不一定能访问。

#### 子问题 2：如何通过浏览器提取字幕？

**失败尝试：**
- `fetch()` 请求 YouTube timedtext API → 超时（CSP/CORS 限制）
- `XMLHttpRequest` 同源请求 → 同样超时
- CDP `Fetch.enable` 拦截请求 → 能拦截但获取响应体复杂
- CDP `Page.getResourceContent` → 无法获取动态内容

**最终方案：DOM 操作**
1. 通过 CDP `Page.navigate` 导航到 YouTube 视频页面
2. 从 `window.ytInitialPlayerResponse.captions` 获取字幕 track 信息
3. 模拟点击"字幕"按钮 → 点击"内容转文字"展开转写文稿面板
4. 用 `querySelectorAll('ytd-transcript-segment-renderer')` 提取已渲染的字幕 DOM
5. 每个 segment 包含时间戳和文本，拼接为完整字幕

**教训：** 当 API 层面被限制时，DOM 层面往往还有路径。YouTube 页面已经在浏览器中渲染了字幕内容，直接从 DOM 提取是最可靠的方式。

#### 子问题 3：如何批量解析 150 万字符字幕？

- 每个子智能体处理 1 个视频（平均 70K 字符）
- 5 个子智能体并行运行（OpenClaw 子智能体并发上限 5 个）
- 分 5 批处理 21 个视频，总耗时约 25 分钟
- 长字幕需要分段读取（prompt 文件截取 15K 字符，再从原始文件补充）
- timeout 设为 600 秒（默认 300 秒对长字幕不够）

## 最终结果

| 导师 | 视频数 | 成功 | 失败 |
|------|--------|------|------|
| Andreas Clenow | 5 | 5 | 0 |
| Andrew Lo | 5 | 5 | 0 |
| Ernest Chan | 5 | 3 | 2 |
| Marcos López de Prado | 5 | 4 | 1 |
| Nassim Taleb | 5 | 4 | 1 |
| **总计** | **25** | **21** | **4** |

4 个失败视频均为 YouTube 未提供自动字幕（非技术问题）。

## 产出

1. **21 个结构化解析文档** — 每个包含 6 大板块：核心主题、关键观点、具体示例、方法论建议、重要引用（附时间戳）、实践启示
2. **youtube-transcript 技能** — 可复用的 YouTube 字幕提取工具，已安装到 OpenClaw 全局技能目录
3. **1.5MB 原始英文字幕** — 保存在 `/tmp/youtube_transcripts/`

## 技术栈

- Chrome DevTools Protocol (CDP) — 浏览器自动化
- Python websocket-client — CDP WebSocket 连接
- OpenClaw 子智能体 — 并行 AI 解析
- GLM-5 Turbo — 中文结构化内容生成

---

*开发日志：#BuildInPublic Day 4*
*仓库：github.com/franklili3/CryptoClaw*
*关注：@cryptoclaw88*
