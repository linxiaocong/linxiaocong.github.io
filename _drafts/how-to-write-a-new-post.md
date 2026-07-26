---
title: How to Write a New Post
date: 2026-07-24 11:00:00 +0800
categories: [Guide]
tags: [writing, jekyll]
---

You don't need to write any code to publish a post — you only write Markdown.
Here's the whole workflow.

## 1. Create a file in the `_posts` folder

Name the file using this exact pattern:

```text
YYYY-MM-DD-your-post-title.md
```

For example: `2026-08-01-my-first-real-post.md`. The date and the hyphens matter,
so keep the format.

## 2. Add the front matter at the top

Every post starts with a small block between two `---` lines. This tells the site
the title, date, and how to file the post:

```yaml
---
title: My First Real Post
date: 2026-08-01 09:00:00 +0800
categories: [Life]
tags: [example, notes]
---
```

- **title** — shown as the headline.
- **date** — publish date and time (`+0800` is the timezone offset).
- **categories** — one or two broad buckets. They show under the *Categories* tab.
- **tags** — as many as you like, always lowercase. They show under the *Tags* tab.

## 3. Write your content in Markdown

Below the front matter, just write normally:

```markdown
## A heading

Some **bold** text, some *italic* text, and a [link](https://example.com).

- a list item
- another item

> A nice quote or callout.
```

## 4. Publish

Save the file, commit it, and push to GitHub. GitHub automatically rebuilds and
deploys the site within a minute or two — your post appears at
`https://linxiaocong.github.io`.

> Tip: to work on a draft you're not ready to publish, put the file in a
> `_drafts` folder (no date needed in the filename) and it won't appear on the
> live site until you move it into `_posts`.
{: .prompt-tip }

## Handy extras

Chirpy supports colored callout boxes:

> An informational note.
{: .prompt-info }

> A warning worth noticing.
{: .prompt-warning }

And images:

```markdown
![description](/assets/img/your-image.png)
```

That's it — happy writing!
