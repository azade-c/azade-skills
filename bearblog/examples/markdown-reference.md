# Bear Blog Markdown & Attributes Reference

Lookup tables for `SKILL.md`. Consult a section when a step points you here; you do not need to read it end to end.

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
