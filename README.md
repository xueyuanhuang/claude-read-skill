# claude-read-skill

A Claude Code plugin that reads and extracts content from any URL — tweets, articles, WeChat posts, and more.

## Features

- Read any URL — just paste the link
- Full support for X/Twitter tweets (including long-form articles)
- Works with WeChat, Xiaohongshu, Bilibili, and any web page
- Preserves original language (no unwanted translation)
- Clean, readable markdown output
- No API keys required

## How It Works

| Platform | Method |
|----------|--------|
| X/Twitter | FxTwitter API (full text, no truncation) -> Jina Reader fallback |
| WeChat, Xiaohongshu, Bilibili | Jina Reader |
| Any web page | Jina Reader |

The plugin uses free, public APIs — no configuration needed.

## Getting Started

### Install the plugin

**Option A: Plugin install (recommended)**

```
/plugin marketplace add xueyuanhuang/claude-read-skill
/plugin install claude-read-skill@claude-read-skill
```

**Option B: Manual install**

```bash
mkdir -p ~/.claude/skills/read
curl -o ~/.claude/skills/read/SKILL.md \
  https://raw.githubusercontent.com/xueyuanhuang/claude-read-skill/main/skills/read/SKILL.md
```

### Use it

In Claude Code, type `/read` followed by a URL:

```
/read https://x.com/elonmusk/status/123456789
```

```
/read https://mp.weixin.qq.com/s/abc123
```

Or just paste a URL and say "read this":

```
帮我读一下 https://x.com/lanhubiji/status/2028376806378930211
```

Claude will automatically fetch the content and present it in clean, readable format.

## Examples

**Read a tweet:**
```
/read https://x.com/lanhubiji/status/2028376806378930211
```

**Read a WeChat article:**
```
/read https://mp.weixin.qq.com/s/some-article-id
```

**Read any web page:**
```
/read https://example.com/some-interesting-article
```

## License

MIT
