# Codex Blog

Diary-style Hugo site with automated deploys to GitHub Pages via GitHub Actions.

## What is configured

- Hugo site structure and starter content
- Diary theme integration (`hugo-theme-diary`)
- CI deploy workflow at `.github/workflows/hugo.yml`

## One-time setup in GitHub

1. Push this repository to GitHub.
2. Go to `Settings` -> `Pages` -> `Build and deployment` and set `Source` to `GitHub Actions`.
3. (If needed) Go to `Settings` -> `Actions` -> `General` and ensure workflow permissions are allowed to deploy Pages.

## Required edits before first deploy

Update these placeholders in `hugo.toml`:

- `baseURL = "https://YOUR_GITHUB_USERNAME.github.io/codexBlog/"`
- `[params.social] github = "YOUR_GITHUB_USERNAME"`

If this repo is your user/org site (named `YOUR_GITHUB_USERNAME.github.io`), set:

- `baseURL = "https://YOUR_GITHUB_USERNAME.github.io/"`

## How deployment works

On every push to `main` or `master`, GitHub Actions will:

1. Install Hugo extended
2. Clone `hugo-theme-diary` into `themes/diary`
3. Build the site
4. Publish `public/` to GitHub Pages

## Create a new post

Add a new Markdown file under `content/posts/` with front matter similar to:

```toml
+++
title = "Post title"
date = "2026-02-25T12:00:00-08:00"
draft = false
tags = ["tag1"]
category = "general"
+++
```
# personalBlog
