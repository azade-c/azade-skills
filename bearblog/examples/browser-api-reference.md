# Bear Blog Browser API Reference

Complete reference for interacting with Bear Blog via OpenClaw's browser tool.

## Prerequisites

1. Browser enabled: `DISPLAY=:99` in `~/.openclaw/.env`
2. Browser started: `POST http://127.0.0.1:18791/start`
3. Logged in (session persists in cookies)

## Authentication

### Login
```bash
# Navigate to login
POST /navigate {"url": "https://bearblog.dev/accounts/login/"}

# Fill credentials (use snapshot to get refs)
POST /act {"kind": "fill", "fields": [
  {"ref": "emailRef", "type": "text", "value": "email@example.com"}
]}
POST /act {"kind": "fill", "fields": [
  {"ref": "passwordRef", "type": "text", "value": "password"}
]}

# Click submit
POST /act {"kind": "click", "ref": "submitButtonRef"}
```

## Post Operations

### Create Post

Bear Blog uses:
- `div#header_content` (contenteditable) - attributes display & input (ref=e14)
- `input#hidden_header_content` (hidden) - auto-filled on submit by JS
- `textarea#body_content` - post content (ref=e15)

> ⚠️ **Outdated claim removed (re-verified 2026-09-01).** This file used to say `fill` works on the
> contenteditable header. It does not. `#header_content` is a bare `<div contenteditable>` with no
> `role` and no `aria-label`, so the accessibility snapshot never assigns it a ref — there is nothing
> for `fill` to target. **The header must be written with `evaluate` + `innerText`**; see the
> "Filling the fields" section of `SKILL.md`, which is authoritative. Only the body is a real
> `<textarea>` and genuinely fillable.
>
> Do not hardcode refs either (`e14`, `e15`, `e10` below were true on one page load in January and
> are wrong today). Snapshot and read the refs, or select by id in `evaluate`.

```bash
# Navigate to new post
POST /navigate {"url": "https://bearblog.dev/<subdomain>/dashboard/posts/new/"}

# Get snapshot to read the CURRENT refs — never reuse refs from these docs
GET /snapshot

# Fill header — evaluate + innerText, NOT fill (see warning above)
POST /act {
  "kind": "evaluate",
  "fn": "() => { const h = document.querySelector('#header_content'); h.focus(); h.innerHTML=''; h.innerText = 'title: My Post\nlink: my-slug\ntags: tag1, tag2'; h.dispatchEvent(new Event('input',{bubbles:true})); return h.innerText; }"
}

# Fill body (textarea)
POST /act {
  "kind": "fill",
  "fields": [{"ref": "e15", "type": "text", "value": "# Content\n\nMarkdown here..."}]
}

# Publish (the Publish button is a submit button)
POST /act {"kind": "click", "ref": "e10"}
```

### Edit Post

```bash
# Navigate to edit page
POST /navigate {"url": "https://bearblog.dev/<subdomain>/dashboard/posts/<uid>/"}

# Read current content
POST /act {
  "kind": "evaluate",
  "fn": "() => ({
    header: document.getElementById('header_content').innerText,
    body: document.getElementById('body_content').value
  })"
}

# Update content
POST /act {
  "kind": "evaluate",
  "fn": "() => {
    document.getElementById('body_content').value = 'Updated content...';
    return 'updated';
  }"
}

# Save (click Publish)
POST /act {
  "kind": "evaluate",
  "fn": "() => { document.getElementById('publish-button').click(); return 'saved'; }"
}
```

### List Posts

```bash
# Navigate to posts list
POST /navigate {"url": "https://bearblog.dev/<subdomain>/dashboard/posts/"}

# Get all posts
POST /act {
  "kind": "evaluate",
  "fn": "() => Array.from(document.querySelectorAll('a'))
    .filter(a => a.href.includes('/dashboard/posts/') && a.href.length > 45)
    .map(a => ({title: a.innerText, href: a.href}))"
}
```

### Delete Post

```bash
# Navigate to post edit page
POST /navigate {"url": "https://bearblog.dev/<subdomain>/dashboard/posts/<uid>/"}

# Override confirm and click delete
POST /act {
  "kind": "evaluate",
  "fn": "() => {
    window.confirm = () => true;
    const btn = Array.from(document.querySelectorAll('button'))
      .find(b => b.innerText.toLowerCase() === 'delete');
    if (btn) { btn.click(); return 'deleted'; }
    return 'not found';
  }"
}
```

### Save as Draft

```bash
POST /act {
  "kind": "evaluate",
  "fn": "() => { document.getElementById('save-button').click(); return 'saved as draft'; }"
}
```

### Unpublish

```bash
# Find and click Unpublish button (only visible on published posts)
POST /act {
  "kind": "evaluate",
  "fn": "() => {
    const btn = Array.from(document.querySelectorAll('button'))
      .find(b => b.innerText.toLowerCase().includes('unpublish'));
    if (btn) { btn.click(); return 'unpublished'; }
    return 'not found';
  }"
}
```

## Header Attributes Reference

```
title: Post Title
link: custom-slug
alias: old-url-redirect
canonical_url: https://original-source.com/post
published_date: 2026-01-05 15:30
is_page: false
meta_description: SEO description
meta_image: https://example.com/image.jpg
lang: en
tags: tag1, tag2, tag3
make_discoverable: true
```

## Tips

1. **Use `fill` for everything**: Playwright supports `fill` on `[contenteditable]` - no `evaluate` needed!
2. **Newlines work**: `fill` handles `\n` correctly in both contenteditable and textarea
3. **Check refs after navigation**: Refs like e14, e15 are usually stable but verify with `GET /snapshot`
4. **Confirmation dialogs**: Override `window.confirm` before clicking Delete (requires `evaluate`)
5. **Screenshots**: Use `POST /screenshot` to debug visual state
6. **Cache issues**: Browser may cache 404s - use `location.reload(true)` if needed

## Common Issues

- **Attributes not saving**: Make sure you're filling the contenteditable div (e14), not a hidden field
- **Delete not working**: Must override `window.confirm` via `evaluate`
- **Login loop**: Check cookies are persisting (browser profile)
- **404 on subdomain URLs**: Use bearblog.dev/<subdomain>/dashboard/... format
- **404 after publish**: Browser cache - wait a moment or hard refresh
