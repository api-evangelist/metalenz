---
name: Map the Metalenz product line, technology and company facts
description: >-
  Build a structured picture of Metalenz — the Polar ID, Polar 3D, PolarEyes, Orion and Gemini
  product pages, the meta-optics technology explainers, the market pages and the company pages —
  from the public WordPress REST API behind metalenz.com, without rendering a single page of HTML.
api: openapi/metalenz-pages-api-openapi.yml
operations:
  - getRouteIndex
  - listTypes
  - listPages
  - getPage
  - search
  - getMedia
  - getOembed
---

# Map the Metalenz product line, technology and company facts

Metalenz is a fabless meta-optics semiconductor company: it sells flat metasurface lenses and
full-stack biometric solutions to device makers and foundries. Its real product documentation is
behind a Microsoft Entra ID login at `docs.metalenz.com` and you will not get in. Everything
public lives on the marketing site — and that site's WordPress REST API is open.

Base URL:

```
https://metalenz.com/wp-json
```

## 1. Orient yourself on the surface

```
GET /
```

`getRouteIndex` returns the site name, tagline, the 13 registered namespaces and all 212 routes.
Cache it — it is ~230KB and it is the only map of this API that exists. Then:

```
GET /wp/v2/types
```

This is how you learn that on *this* site the core `post` type is relabelled **"Press Releases"**,
which changes how you read the archive.

## 2. Enumerate the page tree

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title,parent,modified
```

29 published pages at capture. Send `_fields` — an unfiltered page record is ~22KB because
`content.rendered` is the entire page.

Pages **nest**, and the nesting is meaningful here: `parent` links the Polar ID, Polar ID (zh) and
Polar 3D pages under the PolarEyes page, which is why they live at
`/polareyes-polarization-imaging-system/polar-id/`. PolarEyes is the platform; Polar ID and Polar
3D are products on it. Build your product tree from `parent`, not from the URL string.

The pages you will care about:

| Group | Slugs |
|---|---|
| Products | `all-products`, `polareyes-polarization-imaging-system` (+ `polar-id`, `polar-3d`), `orion-pattern-projectors`, `gemini-dual-pattern-projectors` |
| Technology | `our-technology`, `what-are-meta-optics` |
| Markets | `consumer-electronics`, `automotive`, `smart-home-and-industrial-robotics` |
| Company | `about-us`, `leadership`, `careers`, `events`, `video-gallery`, `contact-us` |
| Legal | `terms-and-conditions-of-sale` |

## 3. Read the ones that matter in full

```
GET /wp/v2/pages/{id}
```

`content.rendered` is HTML, not structured product data — there is no spec sheet endpoint, no
datasheet JSON and no parts catalogue. If you need numbers, the site gates them: the
`/request-product-brief/` page is a lead form, not a download.

## 4. Fall back to search when you do not know the slug

```
GET /wp/v2/search?search=metasurface&per_page=20&_fields=id,title,url,type,subtype
```

`search` spans press releases and pages together and returns `type`/`subtype` so you can route each
hit back to `getPage` or `getPressRelease`. Remember it is a substring match, not ranked full text.

## 5. Collect imagery

```
GET /wp/v2/media?per_page=100&search=logo&_fields=id,slug,source_url,media_details
```

418 attachments at capture — product imagery, metasurface diagrams, event photography, brand and
sub-brand logos. **Pagination gotcha:** the default `orderby=date&order=desc` left page 1 of a
`per_page=1` query empty while `X-WP-Total` reported 418. Page by `Link: rel="next"`, and do not
conclude a collection is empty from one page.

## Rules to hold to

- **Everything here is marketing copy from a CMS.** It is authoritative for what Metalenz *says*
  about itself and nothing more. Do not present `content.rendered` as a specification.
- **`per_page` is 1..100**; violations return `400 rest_invalid_param`.
- **401 means stop, not authenticate.** `/wp/v2/settings`, `/wp/v2/themes`, `/wp/v2/menus`,
  `/wp/v2/templates` and every plugin namespace are credential-gated with no public issuance path.
- **`POST /batch/v1` is blocked at the edge** with a plain-text `403 Request has been blocked.` —
  not a JSON error. Handle a non-JSON 4xx body.
- **Do not touch `/wp/v2/users`.** Real people; excluded from this skill by policy.
- **No rate-limit signal exists.** Self-throttle.
