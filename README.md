# Warayut's Homepage (Zola)

Personal homepage and blog, built with [Zola](https://www.getzola.org/) and the
[Linkita](https://github.com/salif/linkita) theme, edited with
[Sveltia CMS](https://sveltiacms.app/).

Content ported from the previous Hugo + PaperMod site
([Lezgend/homepage](https://github.com/Lezgend/homepage)).

## Requirements

- Zola `>= 0.23.4` (Linkita's `min_version`; this site was built with 0.23.4)

## Local development

```bash
zola serve
```

Then open <http://127.0.0.1:1111>.

```bash
zola build           # writes to public/
zola check           # validates internal + external links
```

## Layout

```
zola.toml                     site + theme configuration
content/
  _index.md                   home page (profile mode)
  posts/
    _index.md                 archive page (template = archive.html)
    <slug>/index.md           a post, as a page bundle
static/
  admin/index.html            Sveltia CMS entry point
  admin/config.yml            Sveltia CMS configuration
  banner.webp                 animated home page banner (lossless WebP)
  _headers                    cache rules (Cloudflare/Netlify only)
  icons/                      profile avatar + social icons
  uploads/                    CMS media uploads
templates/
  home.html                   home page override: banner + pinned posts
  robots.txt                  overrides Zola's default; disallows /admin/
  components/_generator.html  overrides the theme's footer credit (empty)
  injects/head_end.html       animated wallpaper CSS
themes/linkita/               theme, as a git submodule
```

Posts are **page bundles** (`content/posts/my-post/index.md`) so images can sit
next to the post they belong to and be referenced by bare filename.

## Writing a post

```toml
+++
title = "Post title"
description = "Shown on the home page card and in search engines."
date = 2026-09-11
# updated = 2026-09-12
# draft = true

[taxonomies]
tags = ["helpful"]

[extra]
# pinned = true        # stick this post to the top of the home page
# pin_order = 1        # order within the pinned group, lowest first
# math = true
# mermaid = true
# comment = true

[extra.cover]
# image = "cover.png"   # a file inside this post's folder
# alt = ""
+++

Post body in Markdown.
```

All front matter keys are optional except what you want rendered. See the
[Linkita front matter reference](https://salif.github.io/linkita/extra-frontmatter/).

## Pinned posts

Linkita has no pinning of its own, so the home page uses
[`templates/home.html`](templates/home.html) (set via `template = "home.html"` in
`content/_index.md`). It extends the theme's `index.html` and overrides only the
`main` block, so the theme stays a clean, updatable submodule.

Pin a post in its front matter:

```toml
[extra]
pinned = true
pin_order = 1
```

Pinned posts render in a **Pinned** group at the top of the home page, ordered by
`pin_order` ascending, and are left out of the list below so they never appear
twice. Everything else keeps normal date order. Both fields are editable from the
CMS under *Extra options*.

Currently pinned: `useful-links` (1), `free-illustrations` (2), `more` (3).

The Archive page at `/posts/` is unaffected and always lists every post by year.

## Animated wallpaper

[`templates/injects/head_end.html`](templates/injects/head_end.html) adds a slow
drifting gradient behind all content, via Linkita's `head_end` inject point (so
again, no theme fork). It is pure CSS with no JavaScript and no images.

How the layering works: Linkita paints `body { background: var(--bg) }`, and an
opaque body background would cover anything at a negative `z-index`. So the base
colour is moved onto `html`, `body` is made transparent, and a fixed
`body::before` at `z-index: -1` paints between the two.

To tune it, edit the `--wp-*` variables at the top of that file — they are
defined twice, once on `:root` for light mode and once on `:root.dark` for dark
mode. Raise the alpha values to make it more obvious, lower them to calm it
down. The drift speed is the `32s` in the `animation` line.

The animation is GPU-composited (`transform` only, with `will-change`) and is
switched off automatically under `prefers-reduced-motion: reduce`.

Kept deliberately faint because Linkita's post cards (`.block-bg`) are only 3%
opaque — a strong wallpaper would show straight through them and hurt text
contrast.

## Footer credit

[`templates/components/_generator.html`](templates/components/_generator.html)
overrides the theme component of the same name so the footer keeps
**"Powered by Zola"** but drops the **"✎ Linkita"** theme credit. The copyright
line beside it is untouched.

Linkita is MIT licensed, which requires the copyright notice to travel with the
source — that is `themes/linkita/LICENSE`, still present — not a credit in the
rendered page.

## Banner image

The home page shows a banner above the profile name, rendered by
[`templates/home.html`](templates/home.html) and configured in
`content/_index.md`:

```toml
[extra.banner]
image = "banner.svg"
alt = "Animated dusk skyline"
```

That is the whole configuration. Any image format works — **APNG, GIF, animated
SVG or a plain PNG**. Delete the `[extra.banner]` block and the banner
disappears; nothing else changes.

Details that make it drop-in:

- `image` resolves relative to `static/`, and a leading slash is tolerated, so
  both `banner.svg` and a CMS upload at `/uploads/my-banner.png` work.
- The image's real pixel size is read off the file with `get_image_metadata`, so
  `width`/`height` are emitted automatically and the page never jumps on load.
  You do **not** need to state them. Front matter `width`/`height` still
  override if you ever want to.
- A missing file does not break the build — the banner just renders without
  dimensions.
- The banner is contained to the content column and only shows on paginator
  page 1.

Editable from the CMS under *Pages → Home page → Extra options → Banner image*.

### Swapping in your own art

`static/banner.svg` is a placeholder: an animated dusk skyline done as an
animated SVG, so it needs no image tooling and honours `prefers-reduced-motion`.
It is flat vector shapes, not real pixel art.

To use your own image, drop it in `static/` and change one line:

```toml
image = "my-banner.png"
```

Sizing is automatic, so that is the only edit needed.

## Performance

Lighthouse findings and what was done about each.

### Fixed: image delivery — 513 KB to 111 KB

The banner was a 513 KB animated GIF, larger than everything else on the page
combined. It is now `static/banner.webp`, a **lossless** animated WebP at
111 KB — a 78% cut with pixel-identical output.

Lossy WebP is the wrong tool here and made things *worse*: at quality 80 the same
image encoded to 1014 KB, and at 70 to 865 KB. Pixel art has hard edges and a
small palette, which GIF's LZW handles well and lossy DCT handles badly. WebP
**lossless** beats GIF on the same content because of better entropy coding.

If you replace the art, re-encode it the same way:

```bash
npm install sharp
node -e "require('sharp')('in.gif',{animated:true}).webp({lossless:true,effort:4}).toFile('static/banner.webp')"
```

The original GIF is kept at `assets-src/banner.gif`. That directory is outside
`static/`, so it stays in the repo but is never deployed. It is 516 KB and
serves no purpose except re-encoding, so it is safe to delete — the source is:

    https://i.pinimg.com/originals/19/6a/d9/196ad9d3122098b297d7b99ce9ff209f.gif

### Fixed: LCP request discovery

The banner is the LCP element, but it lives in `<main>`, so the browser only
found it after the CSS had loaded. [`templates/injects/head.html`](templates/injects/head.html)
now preloads it — that inject point sits *before* the theme's stylesheets — and
the `<img>` carries `fetchpriority="high"`.

The preload and the `<img>` must resolve to a byte-identical URL or the file is
fetched twice; both go through the same `?h=<hash>` cachebust, so they match.

### Partly fixable: cache lifetimes

`static/_headers` sets long immutable caching for the fingerprinted assets, and
is read by **Cloudflare Pages and Netlify**.

**GitHub Pages ignores it.** Pages serves a fixed 10-minute cache and exposes no
configuration, so this audit cannot be fixed there — it is a hosting limit, not
a site problem. Moving to Cloudflare Pages would resolve it.

### Not worth fixing: render-blocking requests

This is the theme's own `main.min.css`, a 44 KB Tailwind bundle. The other two
sheets (`icons.css`, `admonition.css`) are about 1 KB each, so dropping them
would save one multiplexed HTTP/2 request and nothing measurable — while
forking the theme's `_stylesheets.html` means silently missing any stylesheet
the theme adds later. Bad trade.

Also note the 240 ms figure is inflated if you measured against `zola serve`,
which sends no compression. Both GitHub Pages and Cloudflare Pages serve
gzip/brotli, taking that 44 KB to roughly 10 KB on the wire. Re-measure on the
deployed site before spending anything more here.

## How a CMS edit reaches the live site

Verified end to end by simulating exactly what Sveltia commits — a page bundle
with TOML front matter, the date as a **quoted string** (what the datetime
widget writes), and an uploaded image beside the post:

```
Sveltia saves
  -> commit to main:  content/posts/<slug>/index.md  +  content/posts/<slug>/shot.png
  -> push triggers .github/workflows/zola_build.yaml
  -> zola build     reads content/ and static/, writes ./public
  -> upload-pages-artifact packages ./public
  -> deploy-pages   publishes it
```

`static/` is not uploaded on its own — Zola copies it into `public/`, and
`public/` is the artifact. So both your posts and your static files arrive the
same way.

Confirmed a simulated post landed in every output: its own page, the uploaded
image at `/posts/<slug>/shot.png`, the home page, the archive, its tag page,
`rss.xml`, `sitemap.xml` and the search index. Setting **Draft** in the CMS
correctly excluded it from all of them.

One thing worth knowing: Zola accepts `date = "2026-09-11"` as a quoted string,
which is the format the CMS writes. If it did not, every CMS-created post would
break the build — so this is the single most important compatibility point
between Sveltia and Zola, and it works.

## Sass

Not used, and removed. The Linkita theme ships prebuilt CSS as static files
(`main.min.css`) and sets `compile_sass = false` in its own config; there are
zero `.scss` files in the theme. The empty `sass/` directory and
`compile_sass = true` have been deleted — verified by diffing the whole `public/`
tree before and after, which came out byte-identical.

If you ever want your own Sass, recreate `sass/` and set `compile_sass = true`
again.

## Sveltia CMS

The CMS is served from `/admin/` and stores content directly in this repo as
Zola TOML front matter (`format: toml-frontmatter`).

### Editing locally — no setup needed

1. `zola serve`
2. Open <http://127.0.0.1:1111/admin/> in a **Chromium** browser
   (Chrome / Edge / Brave — this uses the File System Access API).
3. Click **Work with Local Repository** and pick this project folder.

Saves are written straight to your working tree.

### Editing from the live site — two TODOs

Both live in `static/admin/config.yml`:

1. **`backend.repo`** — currently `Lezgend/myblog`. Set it to the real
   `owner/repo` once this project is pushed to GitHub.
2. **`backend.base_url`** — GitHub OAuth needs an auth relay. Deploy
   [sveltia-cms-auth](https://github.com/sveltia/sveltia-cms-auth) (a free
   Cloudflare Worker), then uncomment `base_url` and point it at your worker.

Until step 2 is done, **Sign In with GitHub** on the hosted `/admin/` page will
not complete. **Sign In Using Access Token** works without a worker if you paste
a GitHub personal access token with `repo` scope.

### What the CMS can edit

| Collection | Target |
| ---------- | ------ |
| Posts | `content/posts/<slug>/index.md` — create, edit, delete |
| Pages → Home page | `content/_index.md` |
| Pages → Archive page | `content/posts/_index.md` |

Uploads for a post go into that post's own folder. Site-wide media goes to
`static/uploads/` and is served from `/uploads/`.

Theme settings (menu, profile, social links, footer) live in `zola.toml` and are
intentionally **not** exposed to the CMS — Sveltia cannot edit a Zola config file.

## Notes

- `zola check` reports a few external links from the original content as broken.
  Most are sites that reject automated requests (403) rather than genuinely dead
  links; `zola build` is unaffected.
- The theme is a submodule. Update it with:
  `git submodule update --remote themes/linkita`
  and check its [CHANGELOG](https://github.com/salif/linkita/blob/main/CHANGELOG.md)
  for breaking changes.
