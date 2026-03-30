# Dev Log - Day 4: AI-Powered Knowledge Engineering — Batch YouTube Transcript Extraction & Structured Analysis

## Background

As part of the CryptoClaw quantitative trading AI assistant project, we needed to build a high-quality mentor knowledge base. 25 YouTube videos from 5 core mentors (Andreas Clenow, Andrew Lo, Ernest Chan, Marcos López de Prado, Nassim Taleb) needed to be transformed from raw transcripts into structured Chinese-language analytical documents for our future RAG system.

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
*Repo: github.com/franklili3/CryptoClaw*
*Follow: @cryptoclaw88*
