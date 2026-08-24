# Post 1.0 — Type Format `https://estoc.dev/post/1.0`

**Status:** Draft — 2026-08-24. Companion to the structure layer, [spec.md](../spec.md) §3.1. This page defines only the vocabulary members of a post; everything structural (`format`, `id`, `content`, `files/`, hashing, bundles, cards) is inherited unchanged.

A **post** is an object whose principal bytes are a piece of authored text — an article, a note, a micro-post. It is the type behind a blog entry and behind the DIDComm share message; the same object serves both.

## 1. Members

All vocabulary members live in `index.json` beside the structural ones. Names are borrowed from [Activity Streams 2.0](https://www.w3.org/TR/activitystreams-vocabulary/); only the names are borrowed — a post is not JSON-LD and carries no `@context`.

| Member | Req. | Type | Meaning |
|---|---|---|---|
| `content` | REQUIRED | structural (§3.1 of the spec) | The body. `mediaType` MUST be a text markup type; this version names `text/x-djot` (djot) as the format the family renders natively, and `text/markdown` (CommonMark, raw HTML ignored) as an accepted alternative. |
| `name` | OPTIONAL | string | Title, plain text (no markup). A post without a `name` is a note. |
| `summary` | OPTIONAL | string | One-paragraph plain-text abstract. Used in listings and feeds; never a substitute for the body. |
| `published` | OPTIONAL | RFC 3339 date-time | When the author first published this entity. Stable across versions. |
| `updated` | OPTIONAL | RFC 3339 date-time | When this version was made. Absent means equal to `published`. Governs "latest-authenticated-wins" between versions sharing an `id` (spec §12). |
| `tag` | OPTIONAL | array of strings | Free-form plain-text labels. |
| `inLanguage` | OPTIONAL | BCP 47 tag | Language of the body (`en`, `zh-Hant`). |

Readers MUST ignore unknown members (spec §3.2). Writers MUST NOT emit `objects` (reserved).

## 2. What is deliberately absent

- **No `attributedTo` / author.** Who stands behind a post is answered by the bundle's card (spec §6), not restated as metadata — metadata that repeats testimony is a second source of truth that can disagree with the first. A body may of course name its authors in prose.
- **No `url` / `link`.** An object has no location; locations are transport hints (spec §11).
- **No rendered HTML alongside the source.** The body is the single fact; every HTML page is a projection produced by a renderer.
- **No front matter in the body.** The index is the metadata carrier; a body that also carries front matter would be a second one.

## 3. Body conventions

- Images and other in-tree assets are referenced by relative path from the object root: `![figure](files/images/fig1.jpg)`. A missing path is a broken link, not a malformed object (spec §4).
- Absolute `http(s)` links are external and MUST NOT be auto-loaded by renderers.
- djot bodies use only the core syntax; raw HTML blocks (`{=html}`) SHOULD be dropped by renderers.

## 4. Example

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03110-7c1e-7b3a-9f42-3d5e8a1b2c04",
  "name": "A Day at the Sea",
  "summary": "Notes from an afternoon watching the tide.",
  "published": "2026-08-24T10:30:00Z",
  "tag": ["sea", "journal"],
  "inLanguage": "en",
  "content": { "mediaType": "text/x-djot", "path": "files/body.dj" }
}
```
