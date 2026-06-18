---
name: bearblog
description: Create and manage blog posts on Bear Blog (bearblog.dev). Supports extended Markdown, custom attributes, and browser-based publishing.
metadata: {"openclaw":{"emoji":"🐻","homepage":"https://bearblog.dev","requires":{"config":["browser.enabled"]}}}
---

# Bear Blog Skill

Create, edit, and manage posts on [Bear Blog](https://bearblog.dev) — a minimal, fast blogging platform.

## Authentication

Bear Blog requires browser-based authentication. Log in once via the browser tool, and cookies will persist.

```
browser action:navigate url:https://bearblog.dev/accounts/login/
```

## Creating a Post

### Step 1: Navigate to the post editor

```
browser action:navigate url:https://bearblog.dev/<subdomain>/dashboard/posts/new/
```


### Step 2: Fill the editor

Bear Blog uses a **plain text header format**.

The editor fields are:
- `div#header_content` (contenteditable): attributes (one per line)
- `textarea#body_content`: Markdown body

#### ⚠️ Filling the fields (hard-won, verified 2026-06-17)

**Body** (`#body_content`) is a real `<textarea>` → `fill`/`type` work, OR set `.value` via `evaluate`.

**Header** (`#header_content`) is a **special contenteditable** that Playwright sees as "generic", NON-fillable:
- `fill` and `type` **FAIL** on it (`type` does an internal re-snapshot that invalidates the ref; `click` works but re-renders and changes the ref).
- CSS selectors are NOT accepted as refs; `paste`/`send-keys` don't exist in the browser CLI (only in the MCP tool).
- **MUST fill it via `evaluate --fn`.** And critically: use **`innerText`**, NOT `textContent`. With `textContent`, the `\n` line breaks get flattened → Bear Blog reads it all as ONE line → takes the whole header as the title → giant broken slug + 404.

```js
// header fill, passed to: openclaw browser evaluate --fn "<code>"  (must be an arrow function)
() => {
  const h = document.querySelector('#header_content');
  h.focus();
  h.innerHTML = '';
  h.innerText = headerString;   // innerText preserves real \n in a contenteditable
  h.dispatchEvent(new Event('input',  {bubbles:true}));
  h.dispatchEvent(new Event('change', {bubbles:true}));
  return h.innerText;
}
```

#### ⚠️ `evaluate --fn` takes INLINE code, not a file path (verified 2026-06-18)

`openclaw browser evaluate --fn "..."` evaluates the **string you pass as JS code**. Passing a file path like `--fn /tmp/x.js` makes it try to run the *path* as code → `Invalid regular expression flags` (the `/` are read as a regex). So:

- The value must be an **arrow function**: `--fn "() => document.title"`.
- For multi-line / complex JS, write it to a file then **expand it into the arg**, don't pass the path:
  ```bash
  JS=$(cat /tmp/my.js); openclaw browser evaluate --fn "$JS"
  ```
- Avoid literal regex `/.../ ` inside the function when going through the CLI; build patterns with `new RegExp(...)` or string ops if needed.

Then publish by clicking the button directly:
```js
[...document.querySelectorAll('button')].find(b=>b.textContent.trim()==='Publish').click();
```

#### ⏰ `published_date` is UTC — generate it, never type it by hand (hard rule)

Bear Blog interprets `published_date` as **UTC** (the public page renders `datetime="...Z"`). The blog server runs in **UTC**, but the OpenClaw runtime reports **Europe/Paris**, so any mental conversion is a trap and has produced wrong dates (e.g. an evening post landing the next morning).

**Hard rule: do not type the date manually. Always generate it with the shell** so it matches the real publish moment, then paste that exact string into `published_date`:

```bash
date -u "+%Y-%m-%d %H:%M"   # current UTC, the value to put in published_date
```

- The server is already in UTC, so `date -u` and plain `date` give the same value here, but always use `-u` to stay explicit.
- **Sanity check before publishing:** an evening Paris post (~22:00–23:00 local) must show UTC **20:00–21:00 of the SAME calendar day** (summer, UTC+2) — never the next day, never early-morning hours. If `published_date` shows next-day or 03:00–08:00 for an evening write-up, it is wrong: regenerate with `date -u`.
- After publishing, verify the live page: `curl -s -A Mozilla/5.0 <post-url> | grep -o 'datetime="[^"]*"'` and confirm the day/hour is plausible for when it was actually written.
- To fix a wrong date afterward: open the editor, rewrite only the `published_date` line in the header via `evaluate` (innerText), then click **Publish** again to re-save.

> ⚠️ Past incident (2026-06-17): a post written ~23:00 Paris was saved as `2026-06-18 05:00` UTC and appeared as a *next-day morning* article. Root cause: hand-typed/guessed time instead of `date -u`. Fixed to `2026-06-17 21:00`.

**Header format:**
```
title: Your Post Title
link: custom-slug
published_date: 2026-01-05 14:00
tags: tag1, tag2, tag3
make_discoverable: true
is_page: false
class_name: custom-css-class
meta_description: SEO description for the post
meta_image: https://example.com/image.jpg
lang: en
canonical_url: https://original-source.com/post
alias: alternative-url
```

**Body format:** Standard Markdown with extensions (see below).

The separator `___` (three underscores) is used in templates to separate header from body.

### Step 3: Publish

Click the publish button or submit the form with `publish: true`.

## Post Attributes Reference

| Attribute | Description | Example |
|-----------|-------------|---------|
| `title` | Post title (required) | `title: My Post` |
| `link` | Custom URL slug | `link: my-custom-url` |
| `published_date` | Publication date/time | `published_date: 2026-01-05 14:30` |
| `tags` | Comma-separated tags | `tags: tech, ai, coding` |
| `make_discoverable` | Show in discovery feed | `make_discoverable: true` |
| `is_page` | Static page vs blog post | `is_page: false` |
| `class_name` | Custom CSS class (slugified) | `class_name: featured` |
| `meta_description` | SEO meta description | `meta_description: A post about...` |
| `meta_image` | Open Graph image URL | `meta_image: https://...` |
| `lang` | Language code | `lang: fr` |
| `canonical_url` | Canonical URL for SEO | `canonical_url: https://...` |
| `alias` | Alternative URL path | `alias: old-url` |

## Extended Markdown

Bear Blog uses [Mistune](https://github.com/lepture/mistune) with plugins:

### Text Formatting
- `~~strikethrough~~` → ~~strikethrough~~
- `^superscript^` → superscript
- `~subscript~` → subscript
- `==highlighted==` → highlighted (mark)
- `**bold**` and `*italic*` — standard

### Footnotes
```markdown
Here's a sentence with a footnote.[^1]

[^1]: This is the footnote content.
```

### Task Lists
```markdown
- [x] Completed task
- [ ] Incomplete task
```

### Tables
```markdown
| Header 1 | Header 2 |
|----------|----------|
| Cell 1   | Cell 2   |
```

### Code Blocks
````markdown
```python
def hello():
    print("Hello, world!")
```
````

Syntax highlighting via Pygments (specify language after ```).

### Math (LaTeX)
- Inline: `$E = mc^2$`
- Block: `$$\int_0^\infty e^{-x^2} dx$$`

### Abbreviations
```markdown
*[HTML]: Hypertext Markup Language
The HTML specification is maintained by the W3C.
```

### Admonitions
```markdown
.. note::
   This is a note admonition.

.. warning::
   This is a warning.
```

### Table of Contents
```markdown
.. toc::
```

## Dynamic Variables

Use `{{ variable }}` in your content:

### Blog Variables
- `{{ blog_title }}` — Blog title
- `{{ blog_description }}` — Blog meta description
- `{{ blog_created_date }}` — Blog creation date
- `{{ blog_last_modified }}` — Time since last modification
- `{{ blog_last_posted }}` — Time since last post
- `{{ blog_link }}` — Full blog URL
- `{{ tags }}` — Rendered tag list with links

### Post Variables (in post templates)
- `{{ post_title }}` — Current post title
- `{{ post_description }}` — Post meta description
- `{{ post_published_date }}` — Publication date
- `{{ post_last_modified }}` — Time since modification
- `{{ post_link }}` — Full post URL
- `{{ next_post }}` — Link to next post
- `{{ previous_post }}` — Link to previous post

### Post Listing
```markdown
{{ posts }}
{{ posts limit:5 }}
{{ posts tag:"tech" }}
{{ posts tag:"tech,ai" limit:10 order:asc }}
{{ posts description:True image:True content:True }}
```

Parameters:
- `tag:` — filter by tag(s), comma-separated
- `limit:` — max number of posts
- `order:` — `asc` or `desc` (default: desc)
- `description:True` — show meta descriptions
- `image:True` — show meta images
- `content:True` — show full content (only on pages)

### Email Signup (upgraded blogs only)
```markdown
{{ email-signup }}
{{ email_signup }}
```

## Links

### Standard Links
```markdown
[Link text](https://example.com)
[Link with title](https://example.com "Title text")
```

### Open in New Tab
Prefix URL with `tab:`:
```markdown
[External link](tab:https://example.com)
```

### Heading Anchors
Headings automatically get slugified IDs:
```markdown
## My Section Title
```
Links to: `#my-section-title`

## Typography

Automatic replacements:
- `(c)` → ©
- `(C)` → ©
- `(r)` → ®
- `(R)` → ®
- `(tm)` → ™
- `(TM)` → ™
- `(p)` → ℗
- `(P)` → ℗
- `+-` → ±

## Raw HTML

HTML is supported directly in Markdown:

```html
<div class="custom-class" style="text-align: center;">
  <p>Centered content with custom styling</p>
</div>
```

**Note:** `<script>`, `<object>`, `<embed>`, `<form>` are stripped for free accounts. Iframes are whitelisted (YouTube, Vimeo, Spotify, etc.).

## Whitelisted Iframe Sources

- youtube.com, youtube-nocookie.com
- vimeo.com
- soundcloud.com
- spotify.com
- codepen.io
- google.com (docs, drive, maps)
- bandcamp.com
- apple.com (music embeds)
- archive.org
- And more...

## Dashboard URLs

Replace `<subdomain>` with your blog subdomain:

- **Blog list:** `https://bearblog.dev/dashboard/`
- **Dashboard:** `https://bearblog.dev/<subdomain>/dashboard/`
- **Posts list:** `https://bearblog.dev/<subdomain>/dashboard/posts/`
- **New post:** `https://bearblog.dev/<subdomain>/dashboard/posts/new/`
- **Edit post:** `https://bearblog.dev/<subdomain>/dashboard/posts/<uid>/`
- **Styles:** `https://bearblog.dev/<subdomain>/dashboard/styles/`
- **Navigation:** `https://bearblog.dev/<subdomain>/dashboard/nav/`
- **Analytics:** `https://bearblog.dev/<subdomain>/dashboard/analytics/`
- **Settings:** `https://bearblog.dev/<subdomain>/dashboard/settings/`

## Example: Complete Post

**Header content:**
```
title: Getting Started with AI Assistants
link: ai-assistants-intro
published_date: 2026-01-05 15:00
meta_description: A beginner's guide to working with AI assistants
tags: ai, tutorial, tech
is_page: false
lang: en
```

**Body content:**
```markdown
AI assistants are changing how we work. Here's what you need to know.

## Why AI Assistants?

They help with:
- [x] Writing and editing
- [x] Research and analysis
- [ ] Making coffee (not yet!)

> "The best tool is the one you actually use." — Someone wise

## Getting Started

Check out [OpenAI](tab:https://openai.com) or [Anthropic](tab:https://anthropic.com) for popular options.

---

*What's your experience with AI? Let me know!*

{{ previous_post }} {{ next_post }}
```

## Tips

1. **Preview before publishing** — Use the preview button to check formatting
2. **Use templates** — Set up a post template in dashboard settings for consistent headers
3. **Schedule posts** — Set `published_date` in the future
4. **Draft mode** — Don't click publish to keep as draft
5. **Custom CSS** — Add `class_name` and style in your blog's CSS
6. **SEO** — Always set `meta_description` and `meta_image`

## Troubleshooting

- **Post not showing?** Check `publish` status and `published_date`
- **Tags not working?** Use comma separation, no quotes
- **Styling issues?** Check `class_name` is slugified (lowercase, hyphens)
- **Date format error?** Use `YYYY-MM-DD HH:MM` — and remember it's **UTC** (see filling section).
- **Post 404 / weird long slug?** The header was filled with `textContent` (flattened) instead of `innerText`. Re-fill the header via `evaluate` with `innerText`.
- **Wrong publish time / post shows on the wrong day?** Never hand-type the time. Generate `published_date` with `date -u "+%Y-%m-%d %H:%M"` at publish time, and sanity-check: an evening Paris post must read UTC 20:00–21:00 of the *same* day, not next-day morning. See the `published_date` section.
- **`evaluate` says “Invalid regular expression flags”?** You passed a file path (or literal regex) to `--fn`. It wants inline arrow-function code: `--fn "() => ..."`. Expand files with `JS=$(cat f.js); --fn "$JS"`.
