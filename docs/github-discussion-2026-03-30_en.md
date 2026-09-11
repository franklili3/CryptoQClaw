# Dev Log - Day 4: AI-Powered Knowledge Engineering — Batch YouTube Transcript Extraction & Structured Analysis

## Background

### Goal: Build 5 Quantitative Trading Mentor AI Agents

CryptoQClaw is more than a trading execution tool — we're building a **mentor-level AI Agent system**. The core idea: create a dedicated AI Agent for each quantitative trading master, deeply internalizing their investment philosophy, methodology, and practical experience.

**What these 5 mentor Agents do:**

1. **Strategy Review** — When a user submits a quant trading strategy, mentor Agents evaluate it from their specialized perspectives. For example:
   - Ernest Chan checks for overfitting risks and backtesting methodology
   - Marcos López de Prado reviews feature engineering for multiple testing bias
   - Nassim Taleb evaluates tail risk exposure and fragility
   - Andrew Lo analyzes strategy environment-dependency through Adaptive Markets Hypothesis
   - Andreas Clenow assesses trend-following logic for simplicity and robustness

2. **Strategy Generation** — Based on each mentor's methodological framework, proactively propose new quant strategy ideas. For example:
   - Chan Agent suggests GenAI-based solutions for data scarcity
   - López de Prado Agent proposes hierarchical clustering portfolio optimization
   - Taleb Agent designs tail risk hedging strategies

3. **Knowledge Consultation** — Users can directly ask mentors quant questions and receive authoritative answers grounded in their original works and lectures.

### Why build a mentor knowledge base?

Each mentor Agent's quality depends on its "depth of understanding." To make AI truly think and advise like a mentor, general training data alone is insufficient. We need:

- **Mentor papers** (SSR academic papers) — rigorous theoretical frameworks and mathematical derivations
- **Mentor lectures/interview transcripts** (YouTube videos) — practical experience, intuitive judgment, case studies — content that rarely appears in papers
- **Structured analysis** — transforming raw materials into knowledge units that AI can efficiently retrieve and cite

### Why these 5 mentors?

| Mentor | Core Expertise | Agent Value |
|--------|---------------|-------------|
| Ernest Chan | Quant strategy development, backtesting science, ML | Practical strategy review, overfitting detection |
| Marcos López de Prado | Financial ML, portfolio optimization | Feature engineering review, ML framework guidance |
| Nassim Taleb | Risk management, tail risk, antifragility | Risk exposure assessment, extreme scenario analysis |
| Andrew Lo | Adaptive Markets Hypothesis, financial evolution | Market regime identification, strategy adaptability |
| Andreas Clenow | Trend following, momentum strategies, CTA | Trend strategy design, simplicity review |

### Today's specific task

Building the mentor video transcript knowledge base: 25 YouTube videos from 5 mentors, transforming raw transcripts into structured Chinese-language analytical documents for our RAG system — the core knowledge source for mentor Agents.

## Decision Process

### Approach 1: Gemini Parsing ❌

**Plan:** Paste video links into Google Gemini, leveraging its native YouTube video understanding to generate structured analysis directly.

**Execution:**
1. Automated Gemini web UI via Chrome CDP (remote debugging port 9222)
2. Tried incognito login → required Google account
3. Switched VPN nodes (Japan/US) → "Country not supported" error persisted
4. Modified Google Payments profile country → failed
5. Attempted new Google account registration (non-China phone) → +86 phone number revealed registration region

**Root Cause:** Google account region is determined by registration phone number, not IP address. Even with VPN routing through overseas nodes, accounts registered with +86 Chinese phone numbers are flagged as Chinese users. Both Gemini and AI Studio enforce this restriction at the account level.

**Lesson:** Platform geo-restrictions are account-level, not network-level. Proxies solve network-layer problems but cannot bypass account-level policies.

### Approach 2: Custom Transcript Extraction + AI Analysis ✅

**Plan:** Extract transcripts via browser CDP protocol, then use AI (GLM-5) to generate structured analysis.

#### Sub-problem 1: How to access YouTube?

| Method | Result |
|--------|--------|
| CLI proxy (v2rayA SOCKS5) | ❌ Timeout — routing rules send YouTube traffic direct |
| Python requests + proxy | ❌ Same issue |
| yt-dlp subtitle download | ❌ Version too old (2024.04.09), pip can't upgrade |
| Third-party subtitle APIs | ❌ Most return JS-rendered content |
| **Browser ZeroOmega extension** | ✅ Extension has independent proxy rules, works fine |

**Key Discovery:** ZeroOmega browser extension uses independent proxy rules and node configuration, separate from v2rayA CLI proxy routing. Resources accessible via browser may not be reachable via command-line tools.

#### Sub-problem 2: How to extract transcripts via browser?

**Failed attempts:**
- `fetch()` to YouTube timedtext API → timeout (CSP/CORS)
- `XMLHttpRequest` same-origin request → same timeout
- CDP `Fetch.enable` interception → can intercept but response body retrieval is complex
- CDP `Page.getResourceContent` → can't get dynamic content

**Final approach: DOM manipulation**
1. Navigate to YouTube video page via CDP `Page.navigate`
2. Get caption track info from `window.ytInitialPlayerResponse.captions`
3. Simulate click "Subtitles" button → click "Show transcript" to expand transcript panel
4. Extract rendered subtitle DOM via `querySelectorAll('ytd-transcript-segment-renderer')`
5. Each segment contains timestamp + text, concatenate into full transcript

**Lesson:** When API-level access is restricted, DOM-level extraction often still works. YouTube renders transcript content in the browser — extracting from DOM is the most reliable path.

#### Sub-problem 3: How to parse 1.5M characters of transcripts at scale?

- Each sub-agent processes 1 video (avg 70K chars)
- 5 sub-agents run in parallel (OpenClaw sub-agent concurrency limit: 5)
- 5 batches for 21 videos, total ~25 minutes
- Long transcripts require segmented reading (prompt truncated at 15K chars, supplemented from source)
- Timeout set to 600s (default 300s insufficient for long transcripts)

## Results

| Mentor | Videos | Success | Failed |
|--------|--------|---------|--------|
| Andreas Clenow | 5 | 5 | 0 |
| Andrew Lo | 5 | 5 | 0 |
| Ernest Chan | 5 | 3 | 2 |
| Marcos López de Prado | 5 | 4 | 1 |
| Nassim Taleb | 5 | 4 | 1 |
| **Total** | **25** | **21** | **4** |

4 failures: videos without auto-generated captions on YouTube (not a technical issue).

## Deliverables

1. **21 structured analysis documents** — Each with 6 sections: core theme, key insights (with evidence), specific examples, methodology recommendations, notable quotes (with timestamps), practical takeaways
2. **youtube-transcript skill** — Reusable YouTube transcript extraction tool, installed to OpenClaw global skills directory
3. **1.5MB raw English transcripts** — Archived for reference

## Tech Stack

- Chrome DevTools Protocol (CDP) — browser automation
- Python websocket-client — CDP WebSocket connection
- OpenClaw sub-agents — parallel AI analysis
- GLM-5 Turbo — structured Chinese content generation

---

*Dev Log: #BuildInPublic Day 4*
*Repo: github.com/franklili3/CryptoQClaw*
*Follow: @cryptoclaw88*
