# Read URL Skill

> Send any URL → get the full content back in clean, readable format

## Trigger

When user sends a URL and wants to read its content:
- `/read [URL]`
- "帮我读一下这个链接"
- "去看看这篇文章"
- Any message containing a URL with reading intent

## Supported Platforms & Fetch Strategy

| Platform | URL Pattern | Strategy |
|----------|-------------|----------|
| X/Twitter Article | `x.com/*/status/*` (long-form, has `article` field) | FxTwitter API via curl → extract `article.content.blocks` |
| X/Twitter tweet | `x.com/*/status/*`, `twitter.com/*/status/*` | FxTwitter API (WebFetch) → Jina Reader |
| X/Twitter other | `x.com/*` | Jina Reader |
| WeChat article | `mp.weixin.qq.com/s/*` | Jina Reader → curl direct + HTML parse |
| Xiaohongshu | `xiaohongshu.com/*`, `xhslink.com/*` | Jina Reader |
| Bilibili | `bilibili.com/*`, `b23.tv/*` | Jina Reader |
| Any web page | `*` | Jina Reader |

## Pipeline

### Step 1: Identify Platform & Fetch Content

**For X/Twitter tweet URLs** (`x.com/<user>/status/<id>`):

First, fetch raw JSON via curl to detect if it's a long-form X Article:

```bash
curl -s "https://api.fxtwitter.com/<username>/status/<tweet_id>" | python3 -c "
import json, sys
data = json.load(sys.stdin)
article = data.get('tweet', {}).get('article')
if article:
    print('HAS_ARTICLE')
else:
    print(data['tweet'].get('text', ''))
"
```

**If `HAS_ARTICLE` — X/Twitter Article (long-form):**

Use curl + Python to extract the full article text from `article.content.blocks`:

```bash
curl -s "https://api.fxtwitter.com/<username>/status/<tweet_id>" | python3 -c "
import json, sys
data = json.load(sys.stdin)
blocks = data['tweet']['article']['content']['blocks']
for b in blocks:
    t = b.get('text', '').strip()
    btype = b.get('type', '')
    if not t:
        continue
    if btype == 'header-one':
        print(f'\n# {t}\n')
    elif btype == 'header-two':
        print(f'\n## {t}\n')
    elif btype == 'header-three':
        print(f'\n### {t}\n')
    elif btype == 'blockquote':
        print(f'> {t}\n')
    elif btype == 'unordered-list-item':
        print(f'- {t}')
    elif btype == 'ordered-list-item':
        print(f'1. {t}')
    else:
        print(f'{t}\n')
"
```

> **Why curl instead of WebFetch?** WebFetch always processes content through an AI model that summarizes it. For long articles, you must use Bash `curl` to get the raw JSON and extract text blocks directly.

**If regular tweet — try in order, stop at first success:**

1. **FxTwitter API** (best for tweets — full text, no truncation):
   ```
   WebFetch: https://api.fxtwitter.com/<username>/status/<tweet_id>
   Prompt: Return the full tweet content including any article text.
   ```

2. **Jina Reader** (fallback — works for all web pages):
   ```
   WebFetch: https://r.jina.ai/<original_url>
   Prompt: Return the full article content.
   ```

**For WeChat article URLs** (`mp.weixin.qq.com/s/*`):

Try in order, stop at first success:

1. **Jina Reader** (fast path):
   ```
   WebFetch: https://r.jina.ai/<original_url>
   Prompt: Return the full article content in its original language.
   ```

2. **curl direct + HTML parse** (fallback when Jina returns CAPTCHA/403):
   ```bash
   curl -s \
     -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
     -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8" \
     -H "Accept-Language: zh-CN,zh;q=0.9" \
     "<wechat_url>" | python3 -c "
   import sys, re
   html = sys.stdin.read()
   # Extract title
   title = re.search(r'<h1[^>]*class=\"rich_media_title\"[^>]*>(.*?)</h1>', html, re.DOTALL)
   if title:
       print('# ' + title.group(1).strip())
       print()
   # Extract content — stop before script tags to avoid JS pollution
   content = re.search(r'id=\"js_content\"[^>]*>(.*?)<script', html, re.DOTALL)
   if content:
       text = re.sub(r'<[^>]+>', '', content.group(1))
       text = re.sub(r'&amp;', '&', text)
       text = re.sub(r'&nbsp;', ' ', text)
       text = re.sub(r'&lt;', '<', text)
       text = re.sub(r'&gt;', '>', text)
       text = re.sub(r'\n{3,}', '\n\n', text).strip()
       print(text)  # No truncation — print full content
   "
   ```
   > **Key fixes**: match up to `<script` (not `</div>`) to avoid JS code leaking into output; no `[:N]` truncation.

**For all other URLs**:

Use Jina Reader directly:
```
WebFetch: https://r.jina.ai/<original_url>
Prompt: Return the full article content in its original language.
```

### Step 2: Present Content

Format the extracted content clearly:

```markdown
## [Title]

**Author**: [author/source if available]
**Platform**: [twitter/wechat/web/...]
**URL**: [original URL]

---

[Full content in original language, well-formatted with markdown]
```

### Rules

1. **Preserve original language** — if the article is in Chinese, present it in Chinese. Do not translate unless asked.
2. **Show full content** — do not truncate or over-summarize. Present the complete article.
3. **Clean formatting** — convert to readable markdown, preserve structure (headings, lists, quotes).
4. **If content is very long** (>3000 words), present it in full but add a brief TL;DR at the top.
5. **If fetch fails**, tell the user which methods were tried and suggest alternatives (e.g., "try opening in browser").

## Error Handling

| Situation | Action |
|-----------|--------|
| FxTwitter `article` field present but WebFetch summarizes | Use Bash `curl` + Python to extract raw `article.content.blocks` |
| FxTwitter returns empty | Fall back to Jina Reader |
| WeChat: Jina Reader (WebFetch) returns CAPTCHA | Fall back to `curl` + Jina Reader URL |
| WeChat: `curl` + Jina Reader returns 403 | Fall back to `curl` direct to WeChat URL with browser User-Agent + Python HTML parse |
| Jina Reader returns garbage/login wall (other platforms) | Inform user, suggest `x-reader login <platform>` |
| URL is behind paywall | Inform user, show whatever partial content is available |
| URL is invalid | Tell user the URL doesn't look right |
