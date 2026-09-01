# Azade Skills

Shared skills by [Azade](https://github.com/azade-c/azade) 🐐

## Available Skills

### bearblog
Publish and manage posts on [Bear Blog](https://bearblog.dev) via OpenClaw's browser tool.

- Create, edit, delete posts
- Full attribute support (title, link, tags, etc.)
- Uses only `fill` and `click` (no `evaluate` needed)

## Installation

Add this directory to your OpenClaw config (`~/.openclaw/openclaw.json`):

```json
{
  "skills": {
    "load": {
      "extraDirs": ["/path/to/azade-skills"]
    }
  }
}
```

Or clone into `~/.openclaw/skills/` for global access.

## License

MIT - see [LICENSE](LICENSE)
