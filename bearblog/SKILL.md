---
name: bearblog
description: Create and manage blog posts on Bear Blog (bearblog.dev). Supports extended Markdown, custom attributes, and browser-based publishing.
metadata: {"openclaw":{"emoji":"🐻","homepage":"https://bearblog.dev","requires":{"config":["browser.enabled"]}}}
---

# Bear Blog

Publishing runs through the browser tool. The body is Mistune Markdown, so footnotes, task lists, tables, fenced code with Pygments highlighting, `$math$`, abbreviations, `==mark==`, `^sup^` and `~sub~` all behave as expected.

## URLs

`https://bearblog.dev/<subdomain>/dashboard/` and under it `posts/`, `posts/new/`, `posts/<uid>/`, plus `styles/`, `nav/`, `analytics/`, `settings/`. Published posts are at `https://<subdomain>.bearblog.dev/<link>/`.

## Login

Cookies persist, so this is rare. When the session expires, every dashboard URL redirects to `/accounts/login/` behind Cloudflare Turnstile: the snapshot shows a "Verify you are human" checkbox and a `Ray ID`.

Do not click it. Hand the browser to the human on the live screen, they solve it and log in, one handoff per expiry. Never ask for the password in chat.

## Write

Two fields on `posts/new/`:

- `textarea#body_content`: `fill` works, or set `.value`.
- `div#header_content`: contenteditable with no role and no aria-label, so the snapshot gives it no ref. `fill` and `type` cannot reach it. Write it with `evaluate` and **`innerText`**: `textContent` flattens the newlines, Bear Blog then reads the whole header as the title and the post 404s under a giant slug.

```js
() => {
  const h = document.querySelector('#header_content');
  h.focus();
  h.innerHTML = '';
  h.innerText = headerString;
  h.dispatchEvent(new Event('input', {bubbles:true}));
  return h.innerText;
}
```

`openclaw browser evaluate --fn` takes inline arrow-function code, never a path: `--fn /tmp/x.js` reads the slashes as a regex and fails on "Invalid regular expression flags". Expand the file instead, `JS=$(cat /tmp/x.js); openclaw browser evaluate --fn "$JS"`.

Generate that JS with `JSON.stringify` for every string, from a heredoc. `node -e '...'` wraps the program in single quotes and silently swallows apostrophes (a `meta_description` once shipped as "Mon identifiant nest plus main"), and a template literal breaks on the backticks of a code fence.

```bash
node - <<'EOF' > /tmp/fill.js
const body = require("fs").readFileSync("post.md", "utf8").trim();
console.log('() => { document.querySelector("#body_content").value = ' + JSON.stringify(body) + '; }');
EOF
JS=$(cat /tmp/fill.js); openclaw browser evaluate --fn "$JS"
```

Read back what the function returns. Header format is `key: value`, one per line:

```
title: Your Post Title
link: custom-slug
published_date: 2026-01-05 14:00
tags: tag1, tag2
meta_description: SEO description
lang: en
```

Bear Blog reorders the keys on save and drops attributes left at their default. Nothing was lost, do not re-add them.

The full attribute set: `title`, `link`, `alias`, `canonical_url`, `published_date`, `is_page`, `class_name` (slugified), `meta_description`, `meta_image`, `lang`, `tags`, `make_discoverable`. An unknown key is ignored in silence.

## Bear Blog's own syntax

- `[text](tab:https://example.com)` opens in a new tab.
- `{{ posts }}` lists posts, with `tag:"tech,ai"`, `limit:5`, `order:asc`, `description:True`, `image:True`, `content:True` (pages only).
- Other variables: `{{ blog_title }}`, `{{ blog_description }}`, `{{ blog_link }}`, `{{ blog_created_date }}`, `{{ blog_last_modified }}`, `{{ blog_last_posted }}`, `{{ tags }}`, and in post templates `{{ post_title }}`, `{{ post_description }}`, `{{ post_published_date }}`, `{{ post_last_modified }}`, `{{ post_link }}`, `{{ previous_post }}`, `{{ next_post }}`. `{{ email-signup }}` needs a paid blog.
- Admonitions are reStructuredText style, not Markdown: `.. note::`, `.. warning::`, and `.. toc::` for a table of contents, content indented under them.
- Raw HTML passes through, but free accounts strip `<script>`, `<object>`, `<embed>` and `<form>`. Iframes work only from the allowlist: YouTube, Vimeo, SoundCloud, Spotify, Bandcamp, Apple Music, CodePen, archive.org, Google docs/drive/maps.

## Dates

`published_date` is UTC. Generate it, never type it: `date -u "+%Y-%m-%d %H:%M"`. The runtime reports Europe/Paris, so a mental conversion drifts, and an evening post has landed on the next morning.

When the text states its own date ("nous sommes le 31 aout"), date the post from when it was written, still generated (`date -u -d '2026-08-31 22:00 CEST' "+%Y-%m-%d %H:%M"`), and tell the human. The rule is never guess a date, not always use now.

## Publish

`#publish-button` puts it live, `#save-button` ("Save as draft") saves it unpublished. Both are submits, both safe to click again. Skipping the submit discards the post.

The button row is the post's state: `Save as draft` present means draft, `Unpublish` present means live. `#save-button` does not exist on a published post, the slot holds `#unpublish-button`, so a lookup by id there fails misleadingly.

After the first save the URL carries the uid: `/dashboard/posts/<uid>/`. Keep it to reopen the post.

## Verify

```bash
curl -s -o /dev/null -w "%{http_code}\n" -A Mozilla/5.0 https://<subdomain>.bearblog.dev/<link>/
curl -s -A Mozilla/5.0 https://<subdomain>.bearblog.dev/<link>/ | grep -o 'datetime="[^"]*"'
```

404 means draft, 200 means live. The index is `/blog/`, the root is a landing page that may list nothing, so grepping the root proves neither way.
