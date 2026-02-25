---
title: "Getting Hugo Running with Tranquilpeak on Modern Hugo (v0.156+)"
draft: false
date: 2026-01-06
tags:
  - hugo
  - tranquilpeak
  - blogging
  - static-site
categories:
  - Blueprint
thumbnailImage: /images/posts/hugo-tranquilpeak-cover.png
summary: "Tranquilpeak is still one of the most elegant Hugo themes around, but it was built for an older Hugo. Here are all the fixes you need to get it running on Hugo v0.156+, collected in one place so you don't have to go hunting."
---
Tranquilpeak is still one of the most elegant Hugo themes around. Clean typography, a beautiful cover image sidebar, and a thoughtful reading experience. The problem? It was built for an older Hugo, and modern Hugo (v0.114 onwards in particular) introduced several breaking changes that the theme hasn't kept pace with.

When I set up this blog I hit every one of these issues in sequence. This post collects all the fixes in one place so you don't have to go hunting.

---

## Prerequisites

- Hugo extended v0.100+ (tested on v0.156.0)
- Git
- Basic command line comfort

---

## Step 1: Create Your Hugo Site and Make an Initial Commit

Start by creating a new Hugo site as you normally would:

```bash
hugo new site quickstart
cd quickstart
git init
```

Here is the first trap: **`git subtree` requires at least one commit in the repository**. It needs `HEAD` to exist as a valid reference. If you try to add the theme before making any commit, you will get this error:

```
fatal: ambiguous argument 'HEAD': unknown revision or path not in the working tree.
fatal: working tree has modifications. Cannot add.
```

Fix it by making an initial commit first. If your repo has no files yet, an empty commit works fine:

```bash
git commit --allow-empty -m "Initial commit"
```

---

## Step 2: Add Tranquilpeak via git subtree

Using `git subtree` rather than `git submodule` means the theme files live directly in your repo. No separate clone step needed for collaborators or CI/CD pipelines, and you can freely modify the theme files and commit them — which you will need to do.

```bash
git subtree add --prefix themes/tranquilpeak \
  git@github.com:kakawait/hugo-tranquilpeak-theme.git \
  master --squash
```

Note the prefix carefully: **`themes/tranquilpeak` not `./themes/tranquilpeak`**. The leading `./` causes a subtle path error:

```
error: invalid path './themes/tranquilpeak/.eslintignore'
```

Drop the `./` and it works cleanly.

---

## Step 3: Minimum Viable hugo.toml

Before building, set up a working configuration. Create or replace `hugo.toml` with this baseline:

```toml
baseURL = "https://yourdomain.com/"
languageCode = "en-us"
title = "Ebby Builds"
theme = "tranquilpeak"
paginate = 7

[author]
  name = "Ebby Peter"
  gravatarEmail = ""

[params]
  subtitle = "Blueprint. Build. Ship. Repeat."
  authorName = "Ebby Peter"
  gravatarEmail = ""
  disqusShortname = ""
  description = "Cloud architecture, enterprise design, and open source"

[menu]
  [[menu.main]]
    identifier = "home"
    name = "Home"
    url = "/"
    weight = 1
  [[menu.main]]
    identifier = "about"
    name = "About"
    url = "/about/"
    weight = 2
```

The `[author]` block and the params fields are important — we will explain why in the next steps.

---

## Step 4: Fix `.Site.Author` — Removed in Modern Hugo

If you try to build now you will hit this:

```
execute of template failed: template: _partials/head.html:13:12:
executing "_partials/head.html" at <.Site.Author.gravatarEmail>:
can't evaluate field Author in type page.Site
```

Hugo removed `.Site.Author` as a top-level accessor. It no longer works. The fix is to replace all references in the theme templates to use `.Site.Params` instead.

First, see how many references you are dealing with:

```bash
grep -rn "\.Site\.Author" themes/tranquilpeak/layouts/
```

Then replace them all in one pass:

```bash
find themes/tranquilpeak/layouts/ -type f -name "*.html" \
  -exec sed -i \
  's/\.Site\.Author\.gravatarEmail/.Site.Params.gravatarEmail/g;
   s/\.Site\.Author\.name/.Site.Params.authorName/g' {} +
```

Verify no references remain:

```bash
grep -rn "\.Site\.Author" themes/tranquilpeak/layouts/
```

---

## Step 5: Fix `.Site.DisqusShortname` — Also Removed

A similar error will appear for Disqus:

```
execute of template failed: template: _partials/post/actions.html:65:104:
executing "_partials/post/actions.html" at <.Site.DisqusShortname>:
can't evaluate field DisqusShortname in type page.Site
```

Same pattern, same fix:

```bash
find themes/tranquilpeak/layouts/ -type f -name "*.html" \
  -exec sed -i \
  's/\.Site\.DisqusShortname/.Site.Params.disqusShortname/g' {} +
```

If you are not using Disqus, just leave `disqusShortname = ""` in your params as set in Step 3.

---

## Step 6: Scan for Any Remaining Deprecated References

At this point the most common errors are fixed, but it is worth a broader scan to catch anything else before it bites you:

```bash
grep -rn "\.Site\." themes/tranquilpeak/layouts/ \
  | grep -v "\.Site\.Params\
           \|\.Site\.Title\
           \|\.Site\.BaseURL\
           \|\.Site\.Language\
           \|\.Site\.Pages\
           \|\.Site\.RegularPages\
           \|\.Site\.Data\
           \|\.Site\.Menus\
           \|\.Site\.Home\
           \|\.Site\.Taxonomies\
           \|\.Site\.IsMultiLingual\
           \|\.Site\.IsServer\
           \|\.Site\.Hugo"
```

Any results here are candidates for the same `Params`-based fix pattern.

---

## Step 7: Build and Verify

Start the development server:

```bash
hugo server -D
```

You should see output like:

```
Start building sites …
Built in 12 ms
Web Server is available at http://localhost:1313/
```

Navigate to `http://localhost:1313` and you should have a working Tranquilpeak site.

---

## Step 8: Commit the Patched Theme

Because you added the theme via `git subtree`, the patched template files live directly in your repo. Commit them so the fixes are preserved:

```bash
git add themes/tranquilpeak/
git commit -m "fix: patch Tranquilpeak templates for Hugo v0.156+ compatibility

- Replace .Site.Author.* with .Site.Params equivalents
- Replace .Site.DisqusShortname with .Site.Params.disqusShortname
- Fixes compatibility with Hugo v0.114+ breaking changes"
```

---

## Summary of Fixes

| Error | Root Cause | Fix |
|---|---|---|
| `fatal: ambiguous argument 'HEAD'` | No initial commit before `git subtree add` | `git commit --allow-empty -m "Initial commit"` |
| `error: invalid path './themes/...'` | Leading `./` in subtree prefix | Use `themes/tranquilpeak` not `./themes/tranquilpeak` |
| `can't evaluate field Author in type page.Site` | `.Site.Author` removed in Hugo v0.114+ | Replace with `.Site.Params.authorName` / `.Site.Params.gravatarEmail` |
| `can't evaluate field DisqusShortname in type page.Site` | `.Site.DisqusShortname` removed | Replace with `.Site.Params.disqusShortname` |

---

## Wrapping Up

None of these errors are difficult to fix once you know what they are — the frustration is hitting them one by one and having to piece the solution together from scattered GitHub issues and forum posts. Hopefully this post saves you that afternoon.

If you are customising the theme further (custom cover images, menu configuration, social links), the Tranquilpeak documentation is still reasonably accurate for those areas — the breaking changes are mostly confined to the deprecated `.Site.*` accessors covered here.

*Have you hit a different Tranquilpeak error on modern Hugo? Drop a comment below.*
