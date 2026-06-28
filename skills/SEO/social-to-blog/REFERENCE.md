# social-to-blog reference

This skill is **download-only.** It writes self-contained `<output-dir>/<slug>/post.md` folders and updates a state file. It never edits source code, blog data files, or rendered application files.

## State file schema

Path: `.social-blog-sync.json` at repo root.

```json
{
  "version": 1,
  "sources": {
    "linkedin": {
      "profile": "https://www.linkedin.com/in/<handle>",
      "lastSyncAt": "2026-05-12T15:30:00Z",
      "posts": {
        "urn:li:activity:7012345678901234567": {
          "fetchedAt": "2026-05-12T15:30:00Z",
          "postedAt": "2026-04-08",
          "blogPath": "linkedin-posts/<slug>/post.md",
          "sourceUrl": "https://www.linkedin.com/feed/update/urn:li:activity:7012345678901234567/",
          "media": [
            {
              "type": "image",
              "localPath": "linkedin-posts/<slug>/image-1.jpg"
            }
          ]
        }
      },
      "excluded": [
        "urn:li:activity:7011111111111111111"
      ]
    }
  }
}
```

Rules:
- Skip during fetch if `id` exists in EITHER `posts` (already imported) OR `excluded` (opted out).
- Never delete entries from `posts` - they remain the do-not-refetch record even after the markdown folder is deleted, renamed, or rewritten.
- `excluded` is an array of source IDs the user chose to skip; populate it from user choices during the run or via manual edits to the state file.
- The map key in `posts` IS the canonical ID; use it as the dedup key.
- One entry per source platform under `sources`.
- If the JSON file does not exist, initialise with `{ "version": 1, "sources": {} }`.

### Optional richer exclusion form

If the user wants reasons attached, allow `excluded` to be either an array of strings OR an array of objects:

```json
"excluded": [
  "urn:li:activity:7011111111111111111",
  {
    "id": "urn:li:activity:7022222222222222222",
    "reason": "rewritten as long-form blog post",
    "excludedAt": "2026-05-12T15:30:00Z"
  }
]
```

When reading, normalise both shapes to a `Set` of IDs for filtering.

## Output layout

```
<repo-root>/
  <output-dir>/                       # default: linkedin-posts/
    <slug>/
      post.md
      image-1.jpg
      image-2.jpg
```

### Choosing `<output-dir>`

Default: `linkedin-posts/` at the repo root.

Do **not** write inside any of these directories - they belong to the running app:

| Directory | Framework |
|---|---|
| `src/`, `app/`, `components/` | most JS frameworks |
| `src/content/`, `content/` | Astro / Next content collections |
| `public/`, `static/` | static-asset roots |
| `pages/` | Next.js / Astro routes |
| `_posts/`, `_drafts/` | Jekyll |
| `data/` | data layers across many frameworks |

If the user already has a similar download dir from a previous run (`social-imports/`, `imports/linkedin/`, `posts-raw/`), reuse it. Otherwise default to `linkedin-posts/`.

## post.md format

```markdown
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
hasVideo: false
hasDocument: false
hasPoll: false
hasArticle: false
---

(post body as plain markdown — paragraphs separated by blank lines, URLs as
inline `[label](url)` links)

> _Originally posted with a video on LinkedIn. See [original](<sourceUrl>)._

![](image-1.jpg)
```

Image references in the body use **filenames only** (`image-1.jpg`, not `./image-1.jpg` and not `linkedin-posts/<slug>/image-1.jpg`). The image lives in the same folder as `post.md`, so a relative bare filename is the most portable form across markdown renderers.

Omit `hasVideo` / `hasDocument` / `hasPoll` / `hasArticle` fields when false to keep frontmatter readable.

## LinkedIn fetch

### Navigation

```
https://www.linkedin.com/in/<handle>/recent-activity/all/
```

`/recent-activity/shares/` currently 302-redirects to `/recent-activity/all/` on logged-in sessions. The `/all/` page mixes original posts, reshares, and "X reposted this" entries - filter at extraction time (see below).

If you don't know the handle, ask the user once.

### First-run login

After `browser_navigate`, if the URL or page content indicates login is required, stop and tell the user:

> A LinkedIn login window opened. Please log in there, then say "continue" once you see your activity feed.

When they confirm, navigate back to the activity URL.

### Scroll-load loop (scope = all)

```js
async () => {
  let lastHeight = 0;
  let stable = 0;
  let iter = 0;
  while (stable < 3 && iter < 50) {
    window.scrollTo(0, document.body.scrollHeight);
    await new Promise(r => setTimeout(r, 1500));
    const h = document.body.scrollHeight;
    if (h === lastHeight) stable++;
    else { stable = 0; lastHeight = h; }
    iter++;
  }
  return document.querySelectorAll('[data-urn^="urn:li:activity:"]').length;
}
```

LinkedIn caps the rendered activity timeline at roughly the most-recent ~50 entries. If the user wants older posts, point them at LinkedIn's official "Get a copy of your data" export.

For scope = latest only, skip the loop entirely.

### Extraction

The post text on LinkedIn contains raw U+2028 / U+2029 line/paragraph separators. **Never type them literally into your `browser_evaluate` source** - they tokenise as line terminators and break JS parsing. Build them with `String.fromCharCode` instead:

```js
() => {
  const LSEP = String.fromCharCode(0x2028);
  const PSEP = String.fromCharCode(0x2029);
  const TARGET_ACTOR = 'Michael Fleicher'; // pass in or derive from profile

  const pickFromSrcset = (srcset) => {
    if (!srcset) return null;
    let bestUrl = null;
    let bestWidth = 0;
    for (const part of srcset.split(',')) {
      const m = part.trim().match(/^(\S+)\s+(\d+)w$/);
      if (m && Number(m[2]) > bestWidth) {
        bestWidth = Number(m[2]);
        bestUrl = m[1];
      }
    }
    return bestUrl;
  };

  const out = [];
  const seen = new Set();
  for (const el of document.querySelectorAll('.feed-shared-update-v2')) {
    const id = el.getAttribute('data-urn');
    if (!id || seen.has(id)) continue;
    seen.add(id);

    // Filter: original posts by the target actor only.
    const headerText = el.querySelector(
      '.update-components-header__text-view, .update-components-actor__sub-description, .update-components-actor__description'
    )?.textContent?.trim() ?? '';
    const actorName = el.querySelector(
      '.update-components-actor__title span[aria-hidden="true"]'
    )?.textContent?.trim() ?? '';
    if (actorName !== TARGET_ACTOR) continue;
    if (/reposted this/i.test(headerText)) continue;

    const textEl = el.querySelector(
      '.update-components-text, .feed-shared-update-v2__commentary, .feed-shared-text'
    );
    let text = textEl?.innerText?.trim() ?? '';
    text = text.split(LSEP).join('\n').split(PSEP).join('\n\n');

    // Images: prefer highest-res from srcset, fall back to src or data-delayed-url.
    const images = [];
    const seenImgs = new Set();
    for (const img of el.querySelectorAll(
      '.update-components-image img, .feed-shared-image img, [class*="update-components-image"] img'
    )) {
      let src = img.getAttribute('data-delayed-url')
        || pickFromSrcset(img.getAttribute('srcset'))
        || img.getAttribute('src');
      if (!src || /profile-displayphoto|profile-framedphoto|ghost/.test(src)) continue;
      if (seenImgs.has(src)) continue;
      seenImgs.add(src);
      images.push(src);
    }

    out.push({
      id,
      url: `https://www.linkedin.com/feed/update/${id}/`,
      text,
      images,
      hasVideo: !!el.querySelector('.update-components-video, video'),
      hasDocument: !!el.querySelector('.document-s-container, .feed-shared-document-v2'),
      hasPoll: !!el.querySelector('.update-components-poll, .feed-shared-poll-v2'),
      hasArticle: !!el.querySelector('.feed-shared-article, .update-components-article')
    });
  }
  return out;
}
```

If the array is empty, LinkedIn has changed its DOM. Take a `browser_snapshot`, find post containers in the accessibility tree, and re-derive selectors before retrying.

### URN → timestamp

The post's posted-at timestamp is encoded in the first 41 bits of the URN's numeric ID. Use this instead of the unreliable "3mo • Edited •" relative string on the page:

```js
function urnToDate(urn) {
  const idStr = urn.replace('urn:li:activity:', '');
  return new Date(Number(BigInt(idStr) >> 22n));
}
```

The result is the **post creation** time (not edit time) in ms since the Unix epoch.

## Media download (images)

LinkedIn image URLs are served from `media.licdn.com` and are publicly accessible (no auth required). A direct `curl` download works:

```bash
curl -L --fail --silent --show-error \
  -A "Mozilla/5.0" \
  -H "Referer: https://www.linkedin.com/" \
  -o "<output-dir>/<slug>/image-<n>.jpg" \
  "<image-url>"
```

If a direct download returns 401/403, fall back to fetching inside the authenticated browser session via `browser_evaluate` and a `fetch(url, { credentials: 'include' })`, then decoding the base64 to disk.

### Filename convention

```
<output-dir>/<slug>/image-<index>.<ext>
```

- `index` starts at 1 (`image-1.jpg`, `image-2.jpg`).
- `ext` from URL or `Content-Type` (`image/jpeg` → `jpg`, `image/png` → `png`, `image/webp` → `webp`, `image/gif` → `gif`).
- Default to `jpg` when no signal is available; LinkedIn feed images are JPEG in practice.
- If the target file exists, skip the download and reuse it.

### Non-image media (videos, documents, polls, linked articles)

Not downloaded. Add a single blockquote to the body and set the relevant `hasVideo` / `hasDocument` / `hasPoll` / `hasArticle` field in frontmatter so the reader knows the original post had richer media:

```markdown
> _Originally posted with a video on LinkedIn. See [original](<sourceUrl>)._
```

## Slug generation

```
firstLine
  → lowercase
  → strip Hebrew / Cyrillic / CJK ranges so the slug is Latin-only and usable
  → replace [^a-z0-9 ] with " "
  → collapse whitespace to "-"
  → trim leading/trailing "-"
  → slice to 60 chars
  → trim trailing "-"
prefix with "linkedin-"
```

If a post has no ASCII content (pure emoji, pure Hebrew, etc.) fall back to `linkedin-post-<index>` where `<index>` is the post's position in newest-first order.

If the resulting slug collides with an existing folder, append `-2`, `-3`, etc.

## Body conversion

LinkedIn post text → markdown:

- Preserve blank-line paragraph breaks.
- Convert bare URLs (`https://...` and `lnkd.in/...`) to `[url](url)`.
- Keep hashtags as plain text (`#topic`); don't link.
- Preserve non-Latin text verbatim (Hebrew, etc.) - do not transliterate.
- Strip LinkedIn-specific UI text if it leaks in (e.g. "...see more", "Activate to view larger image").
- Append downloaded images on their own lines after the body: `![](image-1.jpg)`.

## Reshares

Default: skip. The header text "X reposted this" or an actor name that doesn't match the user's profile name marks a reshare. The user can opt-in to include reshares with their own commentary; if so, the body should be the user's commentary only (not the reshared author's post).

## Adding new source platforms

Architecture is platform-agnostic from the "filter, download, write" step onward. To add X/Twitter, Substack, Medium, Bluesky, etc.:

1. Add a new key under `sources.<platform>` in the state file with the same shape (`profile`, `lastSyncAt`, `posts`, `excluded`).
2. Implement a fetcher that returns the same shape: `[{ id, url, text, postedAt?, images, hasVideo?, hasDocument?, hasPoll?, hasArticle? }]`.
   - The `id` must be stable and unique on that platform (Tweet ID, Substack post slug, Medium post ID).
   - `images` is a list of source URLs (highest-resolution variant available); empty if none.
3. Reuse the filter / download / write / state-update steps unchanged.

Possible fetchers:
- **X/Twitter**: same Playwright MCP approach against `x.com/<handle>` with `data-testid="tweet"` selectors.
- **Substack**: public RSS at `<sub>.substack.com/feed` - no browser needed, just `fetch` and parse XML.
- **Medium**: RSS at `medium.com/feed/@<handle>` - same approach as Substack.
- **Bluesky**: public AT Protocol API `https://bsky.social/xrpc/com.atproto.repo.listRecords` - no auth needed for public posts.

RSS-based sources are simpler than browser scraping - prefer them when available.

## What this skill explicitly does NOT do

- Edit `src/`, `app/`, `content/`, `data/`, or any source-of-truth blog file.
- Generate JSX, TSX, MDX, or framework-specific page components.
- Update routing, sitemaps, RSS feeds, or static-asset manifests.
- Run the project's build, dev server, lint, or typecheck.
- Push commits or open PRs.

If the user asks for any of the above as a follow-up, treat it as a separate task driven by a different skill or an explicit hand-written change.
