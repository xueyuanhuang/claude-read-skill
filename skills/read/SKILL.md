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
| X/Twitter tweet | `x.com/*/status/*`, `twitter.com/*/status/*` | FxTwitter API → Jina Reader → Playwright |
| X/Twitter other | `x.com/*` | Jina Reader → Playwright |
| WeChat article | `mp.weixin.qq.com/s/*` | curl direct + HTML parse → Jina Reader → Playwright |
| Login-required pages | any URL requiring auth | Playwright directly |
| Xiaohongshu | `xiaohongshu.com/*`, `xhslink.com/*` | Jina Reader → Playwright |
| Bilibili | `bilibili.com/*`, `b23.tv/*` | Jina Reader → Playwright |
| Any web page | `*` | Jina Reader → Playwright |

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

**If regular tweet — try in order, stop at first success:**

1. **FxTwitter API**:
   ```
   WebFetch: https://api.fxtwitter.com/<username>/status/<tweet_id>
   Prompt: Return the full tweet content including any article text.
   ```

2. **Jina Reader**:
   ```
   WebFetch: https://r.jina.ai/<original_url>
   Prompt: Return the full article content.
   ```

3. **Playwright** (fallback):
   ```
   browser_navigate to <original_url>
   browser_snapshot to extract page content
   ```

---

**For WeChat article URLs** (`mp.weixin.qq.com/s/*`):

Try in order, stop at first success:

1. **curl direct + HTML parse** (most reliable for WeChat):
   ```bash
   curl -s \
     -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
     -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8" \
     -H "Accept-Language: zh-CN,zh;q=0.9" \
     "<wechat_url>" | python3 -c "
   import sys, re
   html = sys.stdin.read()
   title = re.search(r'<h1[^>]*class=\"rich_media_title\"[^>]*>(.*?)</h1>', html, re.DOTALL)
   if title:
       print('# ' + title.group(1).strip())
       print()
   content = re.search(r'id=\"js_content\"[^>]*>(.*?)<script', html, re.DOTALL)
   if content:
       text = re.sub(r'<[^>]+>', '', content.group(1))
       text = re.sub(r'&amp;', '&', text)
       text = re.sub(r'&nbsp;', ' ', text)
       text = re.sub(r'&lt;', '<', text)
       text = re.sub(r'&gt;', '>', text)
       text = re.sub(r'\n{3,}', '\n\n', text).strip()
       print(text)
   "
   ```

2. **Jina Reader** (fallback):
   ```
   WebFetch: https://r.jina.ai/<original_url>
   Prompt: Return the full article content in its original language.
   ```

3. **Playwright** (final fallback):
   ```
   browser_navigate to <original_url>
   browser_snapshot to extract page content
   ```

---

**For login-required pages** (detected when Jina/curl returns login wall):

Skip directly to Playwright:
```
browser_navigate to <original_url>
browser_snapshot — if login wall detected, inform user and stop
```

---

**For all other URLs**:

Try in order, stop at first success:

1. **Jina Reader**:
   ```
   WebFetch: https://r.jina.ai/<original_url>
   Prompt: Return the full article content in its original language.
   ```

2. **Playwright** (fallback):
   ```
   browser_navigate to <original_url>
   browser_snapshot to extract page content
   ```

---

### Step 2: Present Content

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
5. **If fetch fails**, tell the user which methods were tried and suggest alternatives.

## Error Handling

| Situation | Action |
|-----------|--------|
| FxTwitter returns empty | Fall back to Jina Reader → Playwright |
| WeChat: curl returns CAPTCHA/empty | Fall back to Jina Reader → Playwright |
| Jina returns login wall / CAPTCHA | Fall back to Playwright |
| Playwright hits login wall | Inform user, ask if they want to provide credentials |
| URL is behind paywall | Inform user, show whatever partial content is available |
| URL is invalid | Tell user the URL doesn't look right |
