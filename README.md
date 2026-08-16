# Kookerella website

Built with [Hugo](https://gohugo.io) — a static site generator. You write
blog posts in plain Markdown; Hugo turns them into HTML. No Node.js, no
build-your-own-webpack nonsense.

## Project layout

```
content/
  _index.md          <- homepage text (currently your About Us content)
  posts/*.md          <- blog posts, one file per post, in Markdown
layouts/               <- the HTML templates (you shouldn't need to touch these)
static/
  css/style.css        <- all the styling lives here
  CNAME                 <- tells GitHub Pages your custom domain is kookerella.com
hugo.toml               <- site config (title, nav menu, etc.)
.github/workflows/hugo.yml  <- auto-builds & deploys the site on every push
```

## 1. Install Hugo locally

**macOS:** `brew install hugo`
**Windows:** `winget install Hugo.Hugo.Extended` (or `choco install hugo-extended`)
**Linux:** `sudo apt install hugo` (or grab a release from
https://github.com/gohugoio/hugo/releases — get the "extended" build)

Check it worked:

```
hugo version
```

## 2. Preview the site locally

From the project folder:

```
hugo server -D
```

This starts a local server (usually http://localhost:1313) and **live-reloads**
in your browser as you edit files — save a Markdown file, the page updates
instantly. This is how you'll do all your day-to-day writing and tweaking.

Stop it with Ctrl+C.

## 3. Write a new blog post

Easiest way — let Hugo create the file with the right front matter for you:

```
hugo new posts/my-new-post.md
```

That creates `content/posts/my-new-post.md`. Open it — you'll see something like:

```markdown
---
title: "My New Post"
date: 2026-08-16
draft: true
---

Write your post here in Markdown.
```

The `-D` flag on `hugo server -D` means "show drafts too" so you can preview
it locally. Set `draft: false` when you're ready — drafts are excluded from
the real production build (the one GitHub Actions deploys) until you do.

Markdown basics: `# Heading`, `**bold**`, `` `code` ``, and fenced code blocks
with triple-backticks work exactly like on GitHub.

## 4. Edit the homepage / About text

That's `content/_index.md` — just Markdown, edit directly.

## 5. Change styling

Everything's in `static/css/style.css`. It's plain CSS with a few variables
at the top (`--accent`, `--bg`, etc.) for quick colour tweaks.

## 6. Publish

Once it's pushed to GitHub with Pages enabled (see below), **every push to
`main` automatically rebuilds and redeploys the site** — no manual steps.

```
git add .
git commit -m "Add new post about X"
git push
```

## First-time GitHub setup

1. Create a new repo under your company GitHub org (e.g. `kookerella/website`)
2. Push this folder to it:
   ```
   git init
   git add .
   git commit -m "Initial site"
   git branch -M main
   git remote add origin https://github.com/YOUR-ORG/YOUR-REPO.git
   git push -u origin main
   ```
3. In the repo: **Settings → Pages → Source → GitHub Actions**
4. Point your domain's DNS at GitHub Pages:
   - Add these 4 **A records** for `kookerella.com` (apex domain):
     ```
     185.199.108.153
     185.199.109.153
     185.199.110.153
     185.199.111.153
     ```
   - Or if you'd rather use `www.kookerella.com`, add a **CNAME record**
     pointing `www` at `YOUR-ORG.github.io` instead, and update the `CNAME`
     file in `static/` to say `www.kookerella.com`.
5. Push — the Action in `.github/workflows/hugo.yml` builds and deploys
   automatically. First deploy can take a few minutes; DNS can take longer
   to propagate.

## GitHub repos on the homepage

The GitHub link in the nav bar and footer currently points at
`https://github.com/kookerella` — update that in `hugo.toml` under
`[params] github = "..."` to your actual org URL.
