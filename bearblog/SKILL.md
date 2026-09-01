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

#### 🚧 Cloudflare Turnstile guards the login (verified 2026-09-01)

`bearblog.dev` sits behind Cloudflare. When the session cookie has expired, any dashboard URL
redirects to `/accounts/login/` and serves a **"Verify you are human"** checkbox instead of the form.
The page snapshot at that point contains only Cloudflare links plus a `Ray ID` — that is the tell.

**Do not click that checkbox.** It exists to tell a human apart from a bot, and the agent is the bot.
Hand the browser to the human instead: they solve the challenge and log in on the live screen
(see the `host-infra` skill for this host's screen-sharing URL), then say when it is done.
The session persists afterwards, so this costs one handoff, not one per post.

Never ask for the Bear Blog password in chat — the human types it directly into the browser.

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

Then click the button you want directly — see **Step 3** for which one:
```js
[...document.querySelectorAll('button')].find(b=>b.textContent.trim()==='Save as draft').click();
[...document.querySelectorAll('button')].find(b=>b.textContent.trim()==='Publish').click();
```

#### ⚠️ Shell quoting eats apostrophes when generating the JS (verified 2026-09-01)

Building the `evaluate` payload with `node -e '...'` wraps the whole program in **single quotes**, so
no `'` can appear inside it. A French header written that way ships silently mangled — a real
`meta_description` went out as *"Mon identifiant nest plus main : cest mon nom"* and was only caught
on re-read. The body is safe (it is read from a file); the danger is any string typed inline.

Use the typographic apostrophe `’` in prose, or move the text out of the inline program:

```bash
node - <<'EOF' > /tmp/fill.js
const fs = require("fs");
const body = fs.readFileSync("post.md", "utf8").trim();
// JSON.stringify handles every quote, newline and backtick safely
console.log('() => { document.querySelector("#body_content").value = ' + JSON.stringify(body) + '; }');
EOF
JS=$(cat /tmp/fill.js); openclaw browser evaluate --fn "$JS"
```

Always `JSON.stringify` the strings into the generated JS — never interpolate raw text, and never use
a JS template literal (the body may contain ``` code fences, which are backticks).

**Re-read the header after filling.** `evaluate` returns whatever you make it return: end the function
with `return h.innerText` and actually look at it.

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

**Exception — a text that states its own date.** When the article says *"nous sommes le 31 août"*, the
publish moment and the written date differ, and the date the reader must see is the **written** one:
publishing it as "1 September" contradicts its own first line. In that case date the post from when it
was written, still generated rather than guessed (`date -u -d '2026-08-31 22:00 CEST' "+%Y-%m-%d %H:%M"`),
and say so explicitly to the human — a deliberate choice announced is not the same as a silent drift.
The rule being enforced is *never guess a date*, not *always use now*.

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

### Step 3: Save as draft, or publish

The editor has **two** save buttons. There is no `publish: true` form field — click the one you want.

| Button | id | Effect |
|--------|----|--------|
| `Save as draft` | `#save-button` | Persists the post **unpublished**. Use this whenever the human has not yet read it. |
| `Publish` | `#publish-button` | Makes it live at `https://<subdomain>.bearblog.dev/<link>/`. |

Both are `type=submit` and safe to click repeatedly: they save the current field contents. A draft is
published by reopening it and clicking `Publish`; nothing needs to be retyped.

**The button row tells you the post's state** — read it instead of guessing (verified 2026-09-01):

| State | Buttons present |
|-------|-----------------|
| New, never saved | `← Back` · `Publish` · `Save as draft` |
| Saved draft | + `View draft` · `Delete` |
| Published | `← Back` · `Publish` · `View` · `Delete` · **`Unpublish`** (`#unpublish-button`) |

So `Save as draft` / `View draft` present ⇒ still a draft; `Unpublish` present ⇒ live. Note that
`#save-button` **does not exist on a published post** — it is replaced by `#unpublish-button`, so a
lookup by id there fails misleadingly. Prefer matching on button text, or check the id exists first.

After the first save the URL becomes `/<subdomain>/dashboard/posts/<uid>/`; keep that uid to reopen it.

**Bear Blog normalises the header on save:** it reorders the keys and drops attributes left at their
default (`is_page: false`, `make_discoverable: true` disappear). This is not data loss — do not
re-add them in a panic.

#### Verifying the result — check the right URL

```bash
curl -s -o /dev/null -w "%{http_code}\n" -A Mozilla/5.0 https://<subdomain>.bearblog.dev/<link>/
curl -s -A Mozilla/5.0 https://<subdomain>.bearblog.dev/blog/ | grep -o 'href="/<link>/"'
```

- Post URL returns **404** → still a draft. Returns **200** → live.
- The post index is at **`/blog/`**, not at `/`. The blog root is a landing page that may list no posts
  at all, so "grep the homepage" proves nothing in either direction. Use the post URL status code.
- Confirm the rendered date too: `curl -s ... | grep -o 'datetime="[^"]*"'`.

## Reference material

Post attributes, extended Markdown (footnotes, admonitions, math, tables), dynamic `{{ variables }}`,
typography substitutions, raw HTML and the iframe allowlist all live in
[`examples/markdown-reference.md`](examples/markdown-reference.md). Open it when you need a specific
syntax; the workflow above does not require it.

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
4. **Draft mode** — Click `Save as draft`; it is a real button next to `Publish` (see Step 3)
5. **Custom CSS** — Add `class_name` and style in your blog's CSS
6. **SEO** — Always set `meta_description` and `meta_image`

## Troubleshooting

- **Post not showing?** Check it was actually published (`Save as draft` ≠ `Publish`) and check `published_date`. Look at `/blog/`, not at the blog root — the root is a landing page and may list nothing.
- **Tags not working?** Use comma separation, no quotes
- **Styling issues?** Check `class_name` is slugified (lowercase, hyphens)
- **Date format error?** Use `YYYY-MM-DD HH:MM` — and remember it's **UTC** (see filling section).
- **Post 404 / weird long slug?** The header was filled with `textContent` (flattened) instead of `innerText`. Re-fill the header via `evaluate` with `innerText`.
- **Wrong publish time / post shows on the wrong day?** Never hand-type the time. Generate `published_date` with `date -u "+%Y-%m-%d %H:%M"` at publish time, and sanity-check: an evening Paris post must read UTC 20:00–21:00 of the *same* day, not next-day morning. See the `published_date` section.
- **`evaluate` says “Invalid regular expression flags”?** You passed a file path (or literal regex) to `--fn`. It wants inline arrow-function code: `--fn "() => ..."`. Expand files with `JS=$(cat f.js); --fn "$JS"`.
- **Apostrophes missing from the published text?** The JS was generated inside `node -e '...'`, whose single quotes silently dropped every `'`. See the shell-quoting section; re-read the header after filling.
- **Login page shows “Verify you are human”?** Cloudflare Turnstile — the agent does not click it. Hand the browser to the human (see Authentication).
