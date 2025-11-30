# TrendRadar - Project Documentation

A lightweight Chinese news trend aggregation and monitoring tool that consolidates trending topics from 15+ major platforms.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            TrendRadar System                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Config     │    │   Crawler    │    │   Filter     │    │   Output     │
│              │    │              │    │              │    │              │
│ config.yaml  │───▶│ DataFetcher  │───▶│ WordMatcher  │───▶│ HTML Report  │
│ frequency_   │    │              │    │              │    │ Email/Push   │
│ words.txt    │    │ newsnow API  │    │ Scoring      │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MAIN FLOW                                       │
└─────────────────────────────────────────────────────────────────────────────┘

     main.py
        │
        ▼
┌───────────────┐
│ NewsAnalyzer  │──────────────────────────────────────────────────────────┐
│   .run()      │                                                          │
└───────┬───────┘                                                          │
        │                                                                  │
        ▼                                                                  │
┌───────────────────────────────────────────────────────────────────────┐  │
│ STEP 1: LOAD CONFIG                                                   │  │
│                                                                       │  │
│  config/config.yaml ──▶ Platforms list (toutiao, baidu, weibo...)    │  │
│                     ──▶ Notification settings (email, feishu...)     │  │
│                     ──▶ Report mode (daily/current/incremental)      │  │
│                                                                       │  │
│  config/frequency_words.txt ──▶ Keyword groups for filtering         │  │
└───────────────────────────────────────────────────────────────────────┘  │
        │                                                                  │
        ▼                                                                  │
┌───────────────────────────────────────────────────────────────────────┐  │
│ STEP 2: CRAWL DATA                                                    │  │
│                                                                       │  │
│  For each platform in config:                                         │  │
│  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │ DataFetcher.fetch_data(platform_id)                             │ │  │
│  │      │                                                          │ │  │
│  │      ▼                                                          │ │  │
│  │ GET https://newsnow.busiyi.world/api/s?id={platform}&latest     │ │  │
│  │      │                                                          │ │  │
│  │      ▼                                                          │ │  │
│  │ Response: { status, data: [{title, url, mobileUrl, extra}...] } │ │  │
│  └─────────────────────────────────────────────────────────────────┘ │  │
│                                                                       │  │
│  Output: Raw news data from all platforms                             │  │
│  Saved to: output/YYYY年MM月DD日/txt/HH时MM分.txt                      │  │
└───────────────────────────────────────────────────────────────────────┘  │
        │                                                                  │
        ▼                                                                  │
┌───────────────────────────────────────────────────────────────────────┐  │
│ STEP 3: FILTER BY KEYWORDS                                            │  │
│                                                                       │  │
│  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │ frequency_words.txt format:                                     │ │  │
│  │                                                                 │ │  │
│  │   比特币        ◄── Normal word: match if title contains       │ │  │
│  │   Bitcoin                                                       │ │  │
│  │   BTC                                                           │ │  │
│  │                  ◄── Blank line separates groups                │ │  │
│  │   以太坊                                                        │ │  │
│  │   +DeFi         ◄── Required word (+): must ALSO contain       │ │  │
│  │   !广告         ◄── Filter word (!): EXCLUDE if contains       │ │  │
│  └─────────────────────────────────────────────────────────────────┘ │  │
│                                                                       │  │
│  matches_word_groups(title, word_groups, filter_words):               │  │
│  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │ 1. Check filter words (!) ──▶ If match, REJECT                  │ │  │
│  │ 2. For each word group:                                         │ │  │
│  │    a. Check required words (+) ──▶ ALL must match               │ │  │
│  │    b. Check normal words ──▶ ANY must match                     │ │  │
│  │ 3. If any group passes ──▶ ACCEPT                               │ │  │
│  └─────────────────────────────────────────────────────────────────┘ │  │
└───────────────────────────────────────────────────────────────────────┘  │
        │                                                                  │
        ▼                                                                  │
┌───────────────────────────────────────────────────────────────────────┐  │
│ STEP 4: CALCULATE WEIGHTS & RANK                                      │  │
│                                                                       │  │
│  calculate_news_weight(rank, frequency, hotness):                     │  │
│  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │ Composite Score = (rank_weight × 0.6)                           │ │  │
│  │                 + (frequency_weight × 0.3)                      │ │  │
│  │                 + (hotness_weight × 0.1)                        │ │  │
│  │                                                                 │ │  │
│  │ • rank_weight: Higher rank (1st, 2nd) = higher score            │ │  │
│  │ • frequency_weight: Appears on more platforms = higher score   │ │  │
│  │ • hotness_weight: Platform-reported hotness value               │ │  │
│  └─────────────────────────────────────────────────────────────────┘ │  │
│                                                                       │  │
│  News items sorted by composite score (highest first)                 │  │
└───────────────────────────────────────────────────────────────────────┘  │
        │                                                                  │
        ▼                                                                  │
┌───────────────────────────────────────────────────────────────────────┐  │
│ STEP 5: GENERATE REPORTS                                              │  │
│                                                                       │  │
│  generate_html_report() ──▶ output/YYYY年MM月DD日/html/当日汇总.html   │  │
│                                                                       │  │
│  Report includes:                                                     │  │
│  • Matched news items (filtered by keywords)                          │  │
│  • Platform source and ranking                                        │  │
│  • Links to original articles                                         │  │
│  • New items since last crawl (highlighted)                           │  │
└───────────────────────────────────────────────────────────────────────┘  │
        │                                                                  │
        ▼                                                                  │
┌───────────────────────────────────────────────────────────────────────┐  │
│ STEP 6: SEND NOTIFICATIONS                                            │  │
│                                                                       │  │
│  send_to_notifications():                                             │  │
│  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │ • Email (SMTP) ──▶ send_to_email()                              │ │  │
│  │ • Feishu ──▶ send_to_feishu()                                   │ │  │
│  │ • DingTalk ──▶ send_to_dingtalk()                               │ │  │
│  │ • WeChat Work ──▶ send_to_wework()                              │ │  │
│  │ • Telegram ──▶ send_to_telegram()                               │ │  │
│  │ • ntfy ──▶ send_to_ntfy()                                       │ │  │
│  └─────────────────────────────────────────────────────────────────┘ │  │
└───────────────────────────────────────────────────────────────────────┘  │
                                                                           │
◀──────────────────────────────────────────────────────────────────────────┘
```

## Keyword Filtering Logic (Important!)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KEYWORD MATCHING ALGORITHM                                │
└─────────────────────────────────────────────────────────────────────────────┘

frequency_words.txt is parsed into GROUPS separated by blank lines.

Example file:
┌────────────────────┐
│ 比特币             │ ─┐
│ Bitcoin            │  ├── Group 1: ANY of these matches
│ BTC                │ ─┘
│                    │ ◄── blank line = new group
│ 以太坊             │ ─┐
│ Ethereum           │  ├── Group 2: ANY of these matches
│ ETH                │ ─┘
│                    │
│ DeFi               │ ─── Group 3
│ +Ethereum          │ ─── Required: title must ALSO contain "Ethereum"
│ !scam              │ ─── Filter: EXCLUDE titles containing "scam"
└────────────────────┘

MATCHING LOGIC:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Title: "比特币价格突破10万美元"                                              │
│                                                                             │
│  Step 1: Check filter words (!)                                             │
│          └── No filter words matched ──▶ Continue                           │
│                                                                             │
│  Step 2: Check each group:                                                  │
│          Group 1: [比特币, Bitcoin, BTC]                                    │
│                   └── "比特币" found in title ──▶ MATCH!                    │
│                                                                             │
│  Result: ✓ ACCEPTED (appears in report)                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Title: "华为发布新手机"                                                     │
│                                                                             │
│  Step 1: Check filter words (!)                                             │
│          └── No filter words matched ──▶ Continue                           │
│                                                                             │
│  Step 2: Check each group:                                                  │
│          Group 1: [比特币, Bitcoin, BTC] ──▶ No match                       │
│          Group 2: [以太坊, Ethereum, ETH] ──▶ No match                      │
│          Group 3: [DeFi] ──▶ No match                                       │
│          ... (no groups match)                                              │
│                                                                             │
│  Result: ✗ REJECTED (filtered out)                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

SPECIAL PREFIXES:
┌────────┬────────────────────────────────────────────────────────────────────┐
│ Prefix │ Meaning                                                            │
├────────┼────────────────────────────────────────────────────────────────────┤
│ (none) │ Normal word - title must contain this OR other words in group     │
│   +    │ Required word - title MUST contain this (AND logic)                │
│   !    │ Filter word - title must NOT contain this (global exclusion)       │
└────────┴────────────────────────────────────────────────────────────────────┘
```

## Report Modes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           REPORT MODES                                       │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ MODE: daily (当日汇总模式)                                                   │
│                                                                             │
│ • Push timing: Scheduled (default: hourly)                                  │
│ • Content: ALL matched news from today + new items section                  │
│ • Use case: Daily summary, comprehensive view of trending topics            │
│                                                                             │
│ Timeline: ════════════════════════════════════════════════════════════════  │
│           8:00    9:00    10:00   11:00   ...                               │
│             │       │        │       │                                      │
│             ▼       ▼        ▼       ▼                                      │
│           [Push]  [Push]   [Push]  [Push]  (all include full day's data)   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ MODE: current (当前榜单模式)                                                 │
│                                                                             │
│ • Push timing: Scheduled (default: hourly)                                  │
│ • Content: Current ranking matches + new items section                      │
│ • Use case: Real-time hot topic tracking                                    │
│                                                                             │
│ Timeline: ════════════════════════════════════════════════════════════════  │
│           8:00    9:00    10:00   11:00   ...                               │
│             │       │        │       │                                      │
│             ▼       ▼        ▼       ▼                                      │
│           [Push]  [Push]   [Push]  [Push]  (only current rankings)         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ MODE: incremental (增量监控模式)                                             │
│                                                                             │
│ • Push timing: Only when new matches detected                               │
│ • Content: Only newly appeared matching news                                │
│ • Use case: Avoid repeated notifications, alert on new items only           │
│                                                                             │
│ Timeline: ════════════════════════════════════════════════════════════════  │
│           8:00    9:00    10:00   11:00   ...                               │
│             │               │                                               │
│             ▼               ▼                                               │
│           [Push]          [Push]           (only when new items found)     │
│           (3 new)         (1 new)                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
TrendRadar/
├── main.py                     # Core application (4500+ lines)
│   ├── DataFetcher            # Fetches data from newsnow API
│   ├── NewsAnalyzer           # Main orchestrator class
│   ├── load_frequency_words() # Parses keyword config
│   ├── matches_word_groups()  # Keyword matching logic
│   ├── count_word_frequency() # Statistics and filtering
│   ├── generate_html_report() # HTML report generation
│   └── send_to_*()            # Notification functions
│
├── config/
│   ├── config.yaml            # Main configuration
│   │   ├── platforms          # List of news sources to monitor
│   │   ├── notification       # Email/webhook settings
│   │   ├── report.mode        # daily/current/incremental
│   │   └── weight             # Scoring weights
│   │
│   └── frequency_words.txt    # Keywords for filtering
│
├── output/                    # Generated reports
│   └── YYYY年MM月DD日/
│       ├── txt/              # Raw crawled data
│       │   └── HH时MM分.txt
│       └── html/             # HTML reports
│           ├── HH时MM分.html
│           └── 当日汇总.html
│
├── mcp_server/               # MCP server for AI integration
│   ├── server.py             # FastMCP 2.0 server
│   └── tools/                # Query and analytics tools
│
└── .github/workflows/        # GitHub Actions automation
    └── crawler.yml           # Daily scheduled runs
```

## Configuration Reference

### config.yaml

```yaml
# Report mode
report:
  mode: "daily"          # daily | current | incremental
  rank_threshold: 5      # Highlight items ranked <= this

# Scoring weights (must sum to 1.0)
weight:
  rank_weight: 0.6       # Platform ranking importance
  frequency_weight: 0.3  # Cross-platform frequency
  hotness_weight: 0.1    # Platform-reported hotness

# Platforms to monitor
platforms:
  - id: "toutiao"        # API identifier (must match newsnow)
    name: "今日头条"      # Display name (customizable)
  - id: "weibo"
    name: "微博"
```

### frequency_words.txt

```
# Group 1: Bitcoin-related
比特币
Bitcoin
BTC

# Group 2: Ethereum-related
以太坊
Ethereum
ETH

# Group 3: DeFi (requires Ethereum context)
DeFi
+以太坊          # Required: must also contain 以太坊

# Global exclusions
!广告            # Exclude all titles containing 广告
!scam
```

## Deployment Options

| Method | Command | Use Case |
|--------|---------|----------|
| Local | `python main.py` | Development/testing |
| Venv | `.venv/bin/python main.py` | Isolated environment |
| Docker | `docker-compose up` | Production |
| GitHub Actions | Automatic | Free, scheduled runs |
