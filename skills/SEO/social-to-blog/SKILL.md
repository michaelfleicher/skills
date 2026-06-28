---
name: social-to-blog
description: Download the user's own social media posts (LinkedIn supported; other platforms asked but not yet implemented) into a dedicated `linkedin-posts/` (or similar) directory at the repo root as self-contained per-post folders. Each post folder gets a `post.md` with YAML frontmatter plus co-located images. Tracks already-imported post IDs in a state file so re-runs only pull new posts. Use when the user wants to "sync my LinkedIn posts", "download my LinkedIn posts", "save my LinkedIn posts as markdown", "back up my social posts to the repo", or any similar download workflow. **The skill never edits the project's codebase** - it only creates standalone files under the import directory and updates the state file.
---

# social-to-blog

Downloads posts from the user's social profile (LinkedIn today) into the current repo as self-contained per-post markdown folders. Maintains a state file of imported post IDs so re-running only picks up new posts.

**This skill is download-only.** It never edits the project's source code, blog data files, or any existing application files. The user (or another skill) is responsible for turning the downloaded markdown into rendered blog pages.

## Output layout

```
<repo-root>/
  linkedin-posts/                       # configurable per repo; see step 1
    <slug>/
      post.md                           # YAML frontmatter + markdown body
      image-1.jpg                       # co-located media
      image-2.jpg
  .social-blog-sync.json                # state file (commit this)
```

`post.md` frontmatter:

```yaml
---
sourceId: "urn:li:activity:7418536383646097408"
sourceUrl: "https://www.linkedin.com/feed/update/urn:li:activity:7418536383646097408/"
source: "linkedin"
datePublished: "2026-01-18"
title: "First line of the post, trimmed to ~100 chars"
tags:
  - "AI"
  - "Hebrew"
images:
  - "image-1.jpg"
---

(post body as plain markdown — paragraphs separated by blank lines, URLs as
inline `[label](url)` links)

![](image-1.jpg)
```

## Quick start

1. Pick the output directory (default: `linkedin-posts/`). Confirm with user.
2. Confirm the source platform - only **LinkedIn** is implemented today.
3. Pick scope: **all** or **latest only**.
4. Verify Playwright MCP is installed; if not, instruct the user and stop.
5. Fetch via Playwright MCP, filter against `.social-blog-sync.json`.
6. Download media, write per-post folders, update state.

## Workflow

### 1. Pick output directory

Default to `linkedin-posts/` at the repo root. If the user already has a similar dir (e.g. `social-imports/`, `imports/linkedin/`), suggest reusing it. **Do not write inside `src/`, `app/`, `content/`, `public/`, or any directory that looks like part of the running app.** This skill is download-only - the user wires it into their site separately.

### 2. Load state

State file: `.social-blog-sync.json` at repo root. Commit it - teammates share the same import history. Schema in [REFERENCE.md](REFERENCE.md#state-file-schema). Initialize empty structure if missing.

### 3. Pick source

Ask user. **LinkedIn** is the only implemented source today. If user names X/Twitter, Substack, Medium, etc., tell them it's not implemented yet, point to [REFERENCE.md](REFERENCE.md#adding-new-source-platforms) for how to extend, and ask whether to proceed with LinkedIn or stop.

### 4. Pick scope

- **All** - scroll-load entire activity history, filter by state.
- **Latest only** - extract first rendered batch (~10-20 posts), filter.

### 5. Verify Playwright MCP

Check `.mcp.json` in repo root for a `playwright` entry. If missing, tell user:

```bash
claude mcp add playwright -s project -- npx -y @playwright/mcp@latest --user-data-dir ~/.pw-social
```

The `--user-data-dir` flag persists the LinkedIn login between sessions. After adding, the user must restart Claude Code, then re-invoke this skill.

### 6. Fetch (LinkedIn)

Full extraction recipe in [REFERENCE.md](REFERENCE.md#linkedin-fetch). Summary:

- Navigate to `https://www.linkedin.com/in/<handle>/recent-activity/all/` (the `/shares/` tab redirects to `/all/` on current LinkedIn). Filter to original posts by checking the actor name matches the profile and the header is not "X reposted this".
- If redirected to login on first run, stop and ask user to log in manually in the spawned browser, then resume.
- Scroll-load until height stabilises (skip for "latest only"). LinkedIn caps at ~50 most-recent activity items per scroll session - this is a platform limit, not a bug.
- Use `browser_evaluate` to return JSON array of `{ id, url, text, images, hasVideo, hasDocument, hasPoll, hasArticle }`.
- `id` is the URN from `data-urn` (e.g. `urn:li:activity:7012345678901234567`) - this is the canonical, stable identifier; use it as the dedup key.
- The post's posted-at timestamp is encoded in the URN: `Number(BigInt(id_digits) >> 22n)` gives ms since Unix epoch. Use this instead of the "3mo •" relative string on the page.
- `images` is an array of source URLs (highest-resolution variant from `srcset`); empty when the post has no images. Videos, documents, and polls are not yet downloaded - record their presence in frontmatter (`hasVideo: true` etc.) so the reader knows the original had more than text.
- Inside `browser_evaluate`, the LinkedIn DOM emits raw U+2028 / U+2029 separators inside post text. Never use them as literal characters in your script source (they break JS parsing). Build them with `String.fromCharCode(0x2028)` / `String.fromCharCode(0x2029)` and `.split().join('\n')` to normalise.

### 7. Filter, download media, write markdown

A post is **skipped** (not written) if its `id` is in EITHER:
- `sources.linkedin.posts` (already imported - the user may since have deleted, edited, or replaced the folder; we still don't re-fetch).
- `sources.linkedin.excluded` (manually opted-out - the user decided this post should never be imported).

For each remaining post:

- **slug**: kebab-case from first ASCII-bearing line (strip Hebrew/non-Latin to get a usable slug), max 60 chars, dedup with `-2`, `-3` suffix. Always prefix with `linkedin-` so the source is obvious.
- **post folder**: `<output-dir>/<slug>/`
- **images**: for each URL in `images`, download to `<output-dir>/<slug>/image-<index>.<ext>`. Default to `.jpg` extension when the CDN URL has no extension. Skip download if the target file already exists.
- **post.md**: write YAML frontmatter (fields shown in "Output layout" above) followed by a blank line, the markdown body, then a blank line and each image as `![](image-N.ext)`. Body rules:
  - Paragraphs separated by blank lines.
  - Convert bare URLs (`https://...` and `lnkd.in/...`) to `[url](url)` markdown links.
  - If `hasVideo`/`hasDocument`/`hasPoll`/`hasArticle` is true, append a blockquote line like `> _Originally posted with video / document on LinkedIn. See [original](<sourceUrl>)._`
  - Do not transliterate Hebrew or other non-Latin text; preserve the original content verbatim.

### 8. Update state

After every successful post write, append to `.social-blog-sync.json`:

```jsonc
{
  "sources": {
    "linkedin": {
      "lastSyncAt": "<ISO timestamp>",
      "posts": {
        "urn:li:activity:7418536383646097408": {
          "fetchedAt": "<ISO timestamp>",
          "postedAt": "2026-01-18",
          "blogPath": "linkedin-posts/<slug>/post.md",
          "sourceUrl": "https://www.linkedin.com/feed/update/urn:li:activity:7418536383646097408/",
          "media": [
            { "localPath": "linkedin-posts/<slug>/image-1.jpg", "type": "image" }
          ]
        }
      },
      "excluded": []
    }
  }
}
```

Once a post is recorded in `posts` or `excluded`, the user can freely delete, rename, or rewrite the resulting folder - it will not be re-fetched on the next run. To intentionally allow re-import, the user manually removes the `id` from both lists.

### 9. Report - do NOT modify the project's code

Hand off cleanly:

- Print a one-line summary: "Imported N new posts to `<output-dir>/`. Updated `.social-blog-sync.json`."
- Print the path of one example post folder so the user can open it.
- **Do not** touch `src/`, `app/`, `content/`, `data/`, or any rendered blog file.
- **Do not** import these markdown files into the user's blog data structure. That is a follow-up task the user (or another skill) drives explicitly.
- Offer to commit the new folder + state file as a single commit (do not push without explicit ask).

## Caveats

- **LinkedIn ToS** prohibits automated scraping. This skill is for the user's own profile/own posts. For a one-time bulk export, mention LinkedIn's official "Get a copy of your data" tool as the sanctioned alternative.
- **LinkedIn caps activity history at ~50 items** in the rendered timeline. If the user needs older posts, point them at the official data export.
- **Selectors break** when LinkedIn updates the DOM. If extraction returns empty arrays, take a `browser_snapshot` and re-derive selectors from the accessibility tree rather than guessing class names.
- **Headless detection** - always run headed with persistent `--user-data-dir` and human-paced scrolling. Don't bulk-refetch repeatedly in short windows.
- **No app-code edits.** If the user asks the skill to also wire the posts into their site, refuse and tell them that wiring is a separate task. The skill writes plain markdown only.

## Reference

- [REFERENCE.md](REFERENCE.md) - state schema, LinkedIn fetch recipe, slug rules, extending to new sources.
