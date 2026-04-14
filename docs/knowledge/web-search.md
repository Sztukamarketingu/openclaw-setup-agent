# Web Search Configuration

## Overview

OpenClaw can fetch content from known URLs (WebFetch) but cannot search the web by default. To enable search — finding relevant sources without knowing the URL — you need a search provider.

**Recommended:** Perplexity Sonar Pro via OpenRouter.

| Function | Model | Cost |
|----------|-------|------|
| `web_search` | `perplexity/sonar-pro` | ~$3 / 1000 queries |

---

## Why OpenRouter instead of Perplexity directly

OpenRouter gives you one API key for hundreds of models. If you want to switch search providers later or add other models — you only need one key. With OpenRouter already set up for models, web search is just one more model on the same account.

---

## Setup

### Step 1: Add OpenRouter API key to Hostinger Environment

In Hostinger panel → VPS → **Environment**, add:

```
OPENROUTER_API_KEY=sk-or-YOUR_KEY
```

Click **Deploy**. (Skip if already set for model configuration.)

Get key: https://openrouter.ai/keys

### Step 2: Configure web_search in openclaw.json

Send to the agent:

```
Configure web_search via OpenRouter/Perplexity. The API key is already set in the OPENROUTER_API_KEY environment variable — you don't need to enter it anywhere. Just add to openclaw.json (in the tools section):

{
  "tools": {
    "web": {
      "search": {
        "provider": "perplexity",
        "perplexity": {
          "model": "perplexity/sonar-pro"
        }
      }
    }
  }
}

Save the file. The config reloads automatically.
```

### Step 3: Verify

```
Search the web: what is the latest version of OpenClaw?
```

If the agent responds with current data and source citations — web search is working.

---

## Usage

The agent uses web search automatically when it needs current information. You can also trigger it explicitly:

```
Search the web for: [your query]
```

---

## Cost control

- `sonar-pro` is ~$3 / 1000 queries
- At normal usage: a few dollars per month
- Monitor spending in your OpenRouter dashboard
- To disable temporarily: set `"web.search.provider": null` in config
