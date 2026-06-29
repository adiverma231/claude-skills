---
name: scrape-reddit
description: This skill should be used when the user wants to "scrape Reddit", "search Reddit for posts", "research Reddit", "find Reddit discussions about X", "analyze what people think on Reddit", "pull Reddit posts", "scrape a subreddit", or wants to gather sentiment, pain points, customer language, or feature feedback from Reddit communities. Uses Reddit's unofficial JSON endpoints — no API key required.
---

# Reddit Scraper

Scrape Reddit posts and comments without the Reddit API or any credentials — just Reddit's built-in `.json` endpoints and the `requests` library.

## Overview

Reddit exposes a `.json` version of every page (e.g. `reddit.com/r/startups.json`). The tool `tools/scrape_reddit.py` wraps this into a clean CLI. No API key, no OAuth, no account needed.

## Setup (first time only)

Add this dependency to your project and install it:

```
requests>=2.31.0
```

```bash
pip install requests
```

Then add `tools/scrape_reddit.py` to your project. The script outputs to `.tmp/reddit_posts.json` (created automatically).

## Step 1 — Gather inputs

Before running, confirm with the user:

| Input | Required | Notes |
|-------|----------|-------|
| Subreddit | At least one of these | Without `r/` prefix, e.g. `startups` |
| Search query | At least one of these | e.g. `cold email outreach` |
| Post limit | No (default 25) | Keep ≤100 for reasonable speed |
| Include comments | No (default off) | Richer data but ~1 extra request per post |
| Research goal | Yes | Shapes how to analyze the output |

**Valid combinations:**
- Subreddit only → front page of that subreddit
- Query only → site-wide search
- Both → scoped search within that subreddit

## Step 2 — Run the tool

Execute from your project root:

```bash
# Subreddit front page
python tools/scrape_reddit.py --subreddit entrepreneur --limit 50

# Scoped search with comments (most useful for research)
python tools/scrape_reddit.py --subreddit startups --query "cold email" --limit 25 --comments

# Site-wide search
python tools/scrape_reddit.py --query "saas pricing" --limit 40

# Site-wide with comments
python tools/scrape_reddit.py --query "email marketing tool" --limit 20 --comments
```

Expected runtime: posts only ≈ a few seconds; with comments ≈ 1 second per post.

## Step 3 — Analyze the output

Read `.tmp/reddit_posts.json`. Each post has: `title`, `selftext` (body), `score`, `upvote_ratio`, `num_comments`, `author`, `flair`, `permalink`, and a `comments` array (each with `body`, `score`, `author`).

**Tailor analysis to the user's goal:**

- **Pain points / unmet needs** → scan `title` + `selftext` + high-scoring comment `body` for frustrations, blockers, complaints
- **Feature sentiment** → for each feature mentioned, split comments into liked vs disliked; use `score` as signal for how widely held a view is
- **Customer language** → pull verbatim quotes from high-score comments; these are gold for ad copy and landing pages
- **Competitor mentions** → search post and comment text for brand/product names
- **Demand signals** → look for "I wish", "why doesn't", "looking for", "anyone know of a tool"

## Step 4 — Deliver findings

Structure the output by **theme, not by post**. For each theme:
- State the finding clearly
- Support it with 1-2 verbatim quotes (include the comment score so the user knows how representative it is)
- Note which subreddit/posts it came from if doing a site-wide search

## Performance

- Script sleeps 1.1s between requests (~54 req/min, under Reddit's ~60 limit)
- 25 posts, no comments: ~5 seconds
- 50 posts with comments: ~60 seconds
- If the user asks for >100 posts with comments, warn them it will take several minutes

## Edge cases

- **Private/quarantined subreddit** → tool exits with a clear 403 error; try a different subreddit
- **Zero results** → broaden the query or remove the subreddit restriction to search site-wide
- **`[deleted]`/`[removed]` content** → normalized to `null` in the JSON; skip these in analysis
- **`comments: []`** → tool was run without `--comments`; re-run with the flag if comment data is needed
