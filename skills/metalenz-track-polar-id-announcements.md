---
name: Track Metalenz Polar ID and meta-optics announcements
description: >-
  Read the Metalenz newsroom end to end from its public WordPress REST API — every press release,
  its taxonomy, its featured image and its embeddable card — to follow Polar ID, Polar 3D,
  PolarEyes and the foundry partnerships without scraping HTML.
api: openapi/metalenz-press-releases-api-openapi.yml
operations:
  - search
  - listPressReleases
  - getPressRelease
  - listCategories
  - listTags
  - getTag
  - getMedia
  - getOembed
---

# Track Metalenz Polar ID and meta-optics announcements

Metalenz publishes no developer program, but the marketing site at `metalenz.com` runs WordPress
with its REST API open, so the whole newsroom is available as JSON. Base URL for every call:

```
https://metalenz.com/wp-json
```

No credential. No key. `Allow: GET` — anonymous access is read-only.

## 1. Size the archive first

```
GET /wp/v2/posts?per_page=1&_fields=id
```

Read `X-WP-Total` and `X-WP-TotalPages` from the response headers. At capture the archive held
**26** press releases going back to the February 2021 launch out of stealth. Note that the WordPress
post type here is labelled **"Press Releases"**, not "Blog" — `listTypes` confirms it.

## 2. Page the archive, do not scrape it

```
GET /wp/v2/posts?per_page=20&_fields=id,date,slug,link,title,excerpt,categories,tags,featured_media
```

- Always send `_fields`. An unfiltered record is ~11KB because `content.rendered` carries the whole
  press release as HTML.
- Follow the `Link: …; rel="next"` header rather than incrementing `page` yourself.
- `title.rendered` and `excerpt.rendered` are HTML with entity escaping. Decode before display.
- Responses are `cache-control: public, max-age=604800` behind Fastly, so you may be reading a
  week-old edge copy. That is fine for an archive; do not use this surface as a real-time feed.

## 3. Filter to what you care about

Date windows and tags are the two useful filters:

```
GET /wp/v2/posts?after=2026-01-01T00:00:00&_fields=id,date,title,link
GET /wp/v2/tags?per_page=100&_fields=id,name,slug,count
GET /wp/v2/posts?tags=28&_fields=id,date,title,link
```

The taxonomy is thin — one category (`uncategorized`, carrying all 26 posts) and two tags
(`polar-3d`, `new-products`). Do not build a classifier on it; it is not maintained. Use
`search` instead when you want topical recall:

```
GET /wp/v2/search?search=Polar+ID&per_page=20&_fields=id,title,url,type,subtype
```

**Watch out:** WordPress search is a substring `LIKE` match, not ranked full text. Searching `SEMI`
returns 26 hits because every one of them contains the word *semiconductor*. Treat a match count as
a string statistic, never as a topical signal.

## 4. Pull one release in full

```
GET /wp/v2/posts/{id}
```

`getPressRelease` returns `content.rendered` (the full body HTML), `date`/`modified`, `author`
(an integer user id), `featured_media` (an attachment id) and the `_links` block. Add `?_embed` to
inline the author, featured media and terms in one round trip instead of three.

## 5. Get the artwork and the share card

```
GET /wp/v2/media/{featured_media}
GET /oembed/1.0/embed?url={url-encoded permalink}
```

`getMedia` gives you `source_url` plus every generated size in `media_details.sizes`.
`getOembed` gives you a ready-made rich card — title, thumbnail and iframe HTML.

## Rules to hold to

- **`per_page` is 1..100.** Anything outside returns `400 rest_invalid_param` with the exact bound
  in `data.params`. See `errors/metalenz-problem-types.yml`.
- **A 401 is not a retry signal.** There is no public credential path on this surface. `401
  rest_forbidden` and its capability-specific variants mean "not part of the public API" — stop,
  do not attempt to authenticate.
- **No rate-limit headers exist.** Nothing tells you when to back off, so self-throttle. See
  `rate-limits/metalenz-rate-limits.yml`.
- **Do not touch `/wp/v2/users`.** It answers anonymously and returns real people. It is documented
  in the spec for completeness and is deliberately excluded from this skill.
- **This is an unmanaged surface.** No versioning commitment, no deprecation policy, no status
  page, no support channel. Metalenz would not consider a breaking change to be one. See
  `lifecycle/metalenz-lifecycle.yml`.
