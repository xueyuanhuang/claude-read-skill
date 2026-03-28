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
| Feishu/Lark | `*.feishu.cn/*`, `feishu.cn/*`, `*.larksuite.com/*` | Feishu Open API (preferred) → Jina Reader → Playwright |
| X/Twitter Article | `x.com/*/status/*` (long-form, has `article` field) | FxTwitter API via curl → extract `article.content.blocks` |
| X/Twitter tweet | `x.com/*/status/*`, `twitter.com/*/status/*` | FxTwitter API → Jina Reader → Playwright |
| X/Twitter other | `x.com/*` | Jina Reader → Playwright |
| WeChat article | `mp.weixin.qq.com/s/*` | curl direct + HTML parse → Jina Reader → Playwright |
| Login-required pages | any URL requiring auth | Playwright directly |
| Seeking Alpha | `seekingalpha.com/news/*`, `seekingalpha.com/article/*` | SA API (cookie + JSON API) |
| Xiaohongshu | `xiaohongshu.com/*`, `xhslink.com/*` | Jina Reader → Playwright |
| Bilibili | `bilibili.com/*`, `b23.tv/*` | Jina Reader → Playwright |
| Any web page | `*` | Jina Reader → Playwright |

## Pipeline

### Step 1: Identify Platform & Fetch Content

**For Feishu/Lark URLs** (`*.feishu.cn/*`, `feishu.cn/*`, `*.larksuite.com/*`):

Feishu documents are JS-rendered and require authentication. Multiple strategies available depending on setup.

**URL patterns and document types:**

| URL Path | Doc Type | API Token Name |
|----------|----------|----------------|
| `*/docx/<token>` | DocX (new doc) | document_id |
| `*/docs/<token>` | Doc (legacy) | doc_token |
| `*/wiki/<token>` | Wiki node | node_token (needs resolution) |
| `*/sheets/<token>` | Spreadsheet | spreadsheet_token |
| `*/base/<token>` | Bitable | app_token |
| `*/mindnote/<token>` | MindNote | mindnote_token |
| `*/slides/<token>` | Slides | presentation_token |

**Domain variants** (all valid): `feishu.cn`, `my.feishu.cn`, `<tenant>.feishu.cn`, `larksuite.com`, `<tenant>.larksuite.com`

**Extract token from URL:**
```python
import re
match = re.search(r'/(docx|docs|sheets|base|wiki|file|mindnote|slides)/([A-Za-z0-9_-]+)', url)
doc_type, token = match.group(1), match.group(2)
```

**Strategy A: Feishu Open API (preferred, if credentials configured)**

Check if Feishu credentials exist:
```bash
# Check for env vars or config file
echo "${FEISHU_APP_ID:-}" "${FEISHU_APP_SECRET:-}"
cat ~/.config/feishu/config.json 2>/dev/null || true
```

If credentials are available (`FEISHU_APP_ID` + `FEISHU_APP_SECRET`):

```bash
# Step 1: Get tenant_access_token
TOKEN=$(curl -s -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d "{\"app_id\":\"${FEISHU_APP_ID}\",\"app_secret\":\"${FEISHU_APP_SECRET}\"}" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['tenant_access_token'])")

# Step 2a: For docx — get plain text content
curl -s 'https://open.feishu.cn/open-apis/docx/v1/documents/<document_id>/raw_content' \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['content'])"

# Step 2b: For wiki — first resolve node, then fetch actual document
NODE_INFO=$(curl -s "https://open.feishu.cn/open-apis/wiki/v2/spaces/get_node?token=<node_token>" \
  -H "Authorization: Bearer $TOKEN")
# Extract obj_type and obj_token from NODE_INFO, then call the appropriate API

# Step 2c: For docx — get structured blocks (richer content with formatting)
curl -s 'https://open.feishu.cn/open-apis/docx/v1/documents/<document_id>/blocks?page_size=500' \
  -H "Authorization: Bearer $TOKEN"
```

**Strategy A2: feishu-docx CLI tool (if installed)**

```bash
# Check if feishu-docx is installed
which feishu-docx 2>/dev/null && feishu-docx export "<feishu_url>" -o /tmp/feishu-read-output
```

If installed, read the exported markdown file from the output directory.

**Strategy B: Jina Reader (for publicly shared documents)**

Some Feishu documents are publicly accessible ("anyone with the link can view"). Try Jina Reader:
```
WebFetch: https://r.jina.ai/<original_feishu_url>
Prompt: Return the full document content in its original language. Do not summarize.
```

If Jina returns a login page or empty content, fall through to Strategy C.

**Strategy C: Playwright (fallback, requires user login)**

```
browser_navigate to <original_feishu_url>
browser_snapshot — check if content is loaded or login wall appears
```

If login wall: inform user that Feishu requires authentication and suggest:
1. **Recommended**: Set up Feishu Open API credentials for seamless access:
   - Create app at https://open.feishu.cn/app (自建应用)
   - Get `app_id` + `app_secret`
   - Add permissions: `docx:document:readonly`, `wiki:wiki:readonly`
   - Set env vars: `export FEISHU_APP_ID=xxx FEISHU_APP_SECRET=xxx`
   - Or install `pip install feishu-docx` and run `feishu-docx config set --app-id XXX --app-secret XXX`
2. **Quick**: Log in manually in the Playwright browser, then re-navigate
3. **Alternative**: Copy-paste the document content directly

---

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

**For Seeking Alpha URLs** (`seekingalpha.com/news/*`, `seekingalpha.com/article/*`):

SA 直接访问和 Jina Reader 均返回 403/CAPTCHA，必须通过其内部 API 抓取：

```bash
# Step 1: 获取 session cookie
SA_COOKIES=$(curl -sL -c - "https://seekingalpha.com/" \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  -H "Accept: text/html" | grep -E "machine_cookie" | awk '{printf "%s=%s; ", $6, $7}')

# Step 2: 从 URL 提取文章类型和 ID
# seekingalpha.com/news/4569089-xxx    → type=news,    id=4569089
# seekingalpha.com/article/4886149-xxx → type=articles, id=4886149
# 注意：news URL 用 /api/v3/news/{id}，article URL 用 /api/v3/articles/{id}

# Step 3: 调用 API
curl -s "https://seekingalpha.com/api/v3/{type}/{id}" \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  -H "Accept: application/json" \
  -H "Cookie: $SA_COOKIES"
```

**返回 JSON 关键字段：**
- `data.attributes.title` → 标题
- `data.attributes.content` → HTML 全文（需剥离标签提取纯文本）
- `data.attributes.publishOn` → 发布时间
- `data.attributes.isPaywalled` / `data.attributes.metered` → 是否付费墙
- `data.attributes.themes` → 主题标签

**URL → API 路径映射：**

| URL 模式 | API 路径 |
|----------|----------|
| `seekingalpha.com/news/{id}-slug` | `/api/v3/news/{id}` |
| `seekingalpha.com/article/{id}-slug` | `/api/v3/articles/{id}` |
| Press Release | `/api/v3/press_releases/{id}` |

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
| Feishu: no API credentials configured | Try Jina Reader → Playwright → suggest setting up Feishu Open API |
| Feishu: API returns permission denied | Document not shared with app; suggest user share document with the app or use Playwright |
| Feishu: API token expired | Re-fetch tenant_access_token and retry |
| Feishu: wiki node resolution needed | Call `/wiki/v2/spaces/get_node` first, then fetch actual doc by `obj_type` + `obj_token` |
| Feishu: redirect to login page | Auth required; guide user to set up credentials or log in via Playwright |
| FxTwitter returns empty | Fall back to Jina Reader → Playwright |
| WeChat: curl returns CAPTCHA/empty | Fall back to Jina Reader → Playwright |
| Jina returns login wall / CAPTCHA | Fall back to Playwright |
| Playwright hits login wall | Inform user, ask if they want to provide credentials |
| Seeking Alpha: cookie 获取失败 | 重试一次首页请求；若仍失败，提示用户网络问题 |
| Seeking Alpha: API 返回 403 | cookie 可能过期，重新获取 cookie 后重试 |
| Seeking Alpha: `isPaywalled: true` | 基于可见部分给出内容，标注"付费墙截断，内容不完整" |
| URL is behind paywall | Inform user, show whatever partial content is available |
| URL is invalid | Tell user the URL doesn't look right |
