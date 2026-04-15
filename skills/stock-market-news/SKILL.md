---
name: stock-market-news
description: Use this skill when the user asks about stock market news, financial news, market updates, stock prices, or any publicly traded company performance. Trigger this skill for questions like "what's happening in the market?", "any news on [ticker]?", "how is the stock market doing?", or "what's the latest on [company] stock?"
---

# Stock Market News Skill

## Overview

Use the `web_search` tool to retrieve live stock market and financial news. Always fetch fresh data — never rely on training knowledge for current prices or news.

## When to Trigger

- User asks about stock market news or updates
- User asks about a specific stock, ticker, or publicly traded company
- User asks about market indices (S&P 500, Nasdaq, Dow Jones, etc.)
- User asks about financial news, earnings reports, or analyst ratings

## Workflow

### 1. Identify the Query Scope

Determine what the user is asking about:
- **Broad market**: Search for general market news
- **Specific stock/ticker**: Search for that company's news
- **Sector**: Search for sector-specific news (e.g., tech, energy, healthcare)

### 2. Run Web Searches

Use the `web_search` tool with targeted queries. Examples:

- General market: `"stock market news today [current date]"`
- Specific stock: `"[TICKER] stock news today"` or `"[Company Name] stock latest news"`
- Index: `"S&P 500 today"` or `"Nasdaq market update"`
- Earnings: `"[Company] earnings report [quarter/year]"`

Run **multiple searches** if the user asks about several topics or companies.

### 3. Synthesize and Present Results

Structure your response clearly:

- **Lead with the most important/recent news**
- Group by company or topic if multiple subjects are covered
- Include the **source and publication date** for each item as an inline markdown link
- Note any significant price movements, earnings beats/misses, or major corporate events
- Flag if information may be delayed or if markets are currently closed

### 4. Caveats to Always Include

- State that stock prices and news are sourced from public web results and may have a delay
- Do not provide investment advice or recommendations
- If the user asks for a price, note that it reflects the time of the search result, not necessarily real-time

## Response Format

```
### Market Overview (or [Company Name])
**[Headline]** — Brief summary of the news item. [Source Name](URL) — Published: [date]

**[Headline]** — Brief summary. [Source Name](URL) — Published: [date]

---
*Data sourced from public web results. May not reflect real-time prices. Not investment advice.*
```

## Example Searches

| User asks | Search query |
|-----------|-------------|
| "How's the market today?" | `"stock market news April 14 2026"` |
| "What's happening with Apple?" | `"AAPL Apple stock news today"` |
| "Any Tesla updates?" | `"TSLA Tesla stock news latest"` |
| "How's the S&P doing?" | `"S&P 500 performance today April 2026"` |
| "Any big earnings this week?" | `"earnings reports this week April 2026"` |
