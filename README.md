# Warayut's Homepage

Personal homepage and blog, built with [Zola](https://www.getzola.org/) and the
[Linkita](https://github.com/salif/linkita) theme, edited with
[Sveltia CMS](https://sveltiacms.app/) and deployed to GitHub Pages.

Live at <https://www.warayut.xyz>.

This repo previously ran Hugo + PaperMod; the Zola rewrite landed in the
`Migrate to Zola` commit, so the old site is still in this repo's git history.

## Requirements

- Zola `0.23.4` or newer — that is Linkita's `min_version`, and the version CI
  installs (`ZOLA_VERSION` in the deploy workflow).

## Local development

```bash
zola serve
```

Then open <http://127.0.0.1:1111>.

```bash
zola build           # writes to public/ (gitignored)
zola check           # validates internal + external links
```

## Layout

```
zola.toml                       site + theme configuration
content/
  _index.md                     home page (profile mode, banner)
  posts/
    _index.md                   archive page (template = archive.html)
    <slug>/index.md             a post, as a page bundle
static/
  admin/index.html              Sveltia CMS entry point
  admin/config.yml              Sveltia CMS configuration
  banner.webp                   animated home page banner (lossless WebP)
  _headers                      cache rules (Cloudflare/Netlify only)
  icons/cat-the-box.svg         profile avatar
  icons/logo.svg                Open Graph preview image
  favicon.ico                   favicons
  android-icon.png
  apple-touch-icon.png
  uploads/                      CMS media uploads
templates/
  home.html                     home page override: banner + pinned posts
  robots.txt                    overrides Zola's default; disallows /admin/
  components/_generator.html    overrides the theme's footer credit
  injects/head.html             preloads the banner (before the theme's CSS)
  injects/head_end.html         animated wallpaper CSS
themes/linkita/                 the theme, vendored in-tree
.github/workflows/zola_build.yaml   build + deploy to GitHub Pages
```

Posts are **page bundles** (`content/posts/my-post/index.md`) so images can sit
next to the post they belong to and be referenced by bare filename.

Every file under `templates/` is an override or an inject point, never a fork of
a theme file, so `themes/linkita/` stays a clean, replaceable copy.

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
`main` block.

Pin a post in its front matter:

```toml
[extra]
pinned = true
pin_order = 1
```

Pinned posts render in a **Pinned** group at the top of the home page, ordered by
`pin_order` ascending, and are left out of the list below so they never appear
twice. Everything else keeps normal date order. Both fields are editable from the
CMS under _Extra options_.

Currently pinned: `useful-links` (1), `free-illustrations` (2), `more` (3).

The Archive page at `/posts/` is unaffected and always lists every post by year.

## Banner image

The home page shows a banner above the profile name, rendered by
[`templates/home.html`](templates/home.html) and configured in
`content/_index.md`:

```toml
[extra.banner]
image = "banner.webp"
alt = "Pixel art night street scene with a lit convenience store and cherry blossom trees"
```

That is the whole configuration. Any image format works — **WebP, APNG, GIF,
animated SVG or a plain PNG**. Delete the `[extra.banner]` block and the banner
disappears; nothing else changes.

Details that make it drop-in:

- `image` resolves relative to `static/`, and a leading slash is tolerated, so
  both `banner.webp` and a CMS upload at `/uploads/my-banner.png` work.
- The image's real pixel size is read off the file with `get_image_metadata`, so
  `width`/`height` are emitted automatically and the page never jumps on load.
  You do **not** need to state them. Front matter `width`/`height` still
  override if you ever want to.
- A missing file does not break the build — the banner just renders without
  dimensions.
- The banner is contained to the content column and only shows on paginator
  page 1.

Editable from the CMS under _Pages → Home page → Extra options → Banner image_.

To use your own image, drop it in `static/` and change one line:

```toml
image = "my-banner.png"
```

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

## Deployment

[`.github/workflows/zola_build.yaml`](.github/workflows/zola_build.yaml) builds
and publishes to GitHub Pages on every push to `main`, and can also be run
manually from the Actions tab.

```
push to main
  -> install Zola (ZOLA_VERSION, currently 0.23.4)
  -> zola build --base-url <the URL configure-pages reports>
  -> upload-pages-artifact packages ./public
  -> deploy-pages publishes it
```

Two things worth knowing:

- The build passes `--base-url`, so the `base_url` in `zola.toml` is overridden
  at deploy time by whatever the repo's Pages settings resolve to (the custom
  domain, when one is configured).
- `TZ: Asia/Bangkok` is set for the build step so post dates render in local
  time rather than UTC.

`static/` is not uploaded on its own — Zola copies it into `public/`, and
`public/` is the artifact. So posts and static files arrive the same way.

## Theme

`themes/linkita/` is **vendored**: the theme's files are committed directly into
this repo, not pulled in as a live git submodule. `git submodule status` returns
nothing here, and `git submodule update --remote themes/linkita` does nothing.

> [!NOTE]
> `.gitmodules` still declares `themes/linkita` as a submodule. It is a leftover
> and has no effect. Either delete it, or convert the directory into a real
> submodule, if you want the two to agree.

To update the theme, replace the directory with a newer copy of
[salif/linkita](https://github.com/salif/linkita) and check its
[CHANGELOG](https://github.com/salif/linkita/blob/main/CHANGELOG.md) for
breaking changes — especially the `min_version` in its `theme.toml`, which must
stay `<=` the Zola version pinned in the workflow.

### Fixed: image delivery — 513 KB to 111 KB

The banner was a 513 KB animated GIF, larger than everything else on the page
combined. It is now `static/banner.webp`, a **lossless** animated WebP at
111 KB — a 78% cut with pixel-identical output.

Lossy WebP is the wrong tool here and made things _worse_: at quality 80 the same
image encoded to 1014 KB, and at 70 to 865 KB. Pixel art has hard edges and a
small palette, which GIF's LZW handles well and lossy DCT handles badly. WebP
**lossless** beats GIF on the same content because of better entropy coding.

If you replace the art, re-encode it the same way:

```bash
npm install sharp
node -e "require('sharp')('in.gif',{animated:true}).webp({lossless:true,effort:4}).toFile('static/banner.webp')"
```

The original GIF is no longer kept in the repo.

### Partly fixable: cache lifetimes

[`static/_headers`](static/_headers) sets long immutable caching for the
fingerprinted assets, and is read by **Cloudflare Pages and Netlify**.
**GitHub Pages ignores it**, which is where this site is deployed today — it
serves a fixed 10-minute cache and exposes no configuration, so this audit is a
hosting limit rather than a site problem. Moving to Cloudflare Pages would
resolve it, and `_headers` is already in place for that.

## Sveltia CMS

The CMS is served from `/admin/` and stores content directly in this repo as
Zola TOML front matter (`format: toml-frontmatter`).

### Editing locally — no setup needed

1. `zola serve`
2. Open <http://127.0.0.1:1111/admin/> in a **Chromium** browser
   (Chrome / Edge / Brave — this uses the File System Access API).
3. Click **Work with Local Repository** and pick this project folder.

Saves are written straight to your working tree.

### What the CMS can edit

| Collection           | Target                                                 |
| -------------------- | ------------------------------------------------------ |
| Posts                | `content/posts/<slug>/index.md` — create, edit, delete |
| Pages → Home page    | `content/_index.md`                                    |
| Pages → Archive page | `content/posts/_index.md`                              |

Uploads for a post go into that post's own folder. Site-wide media goes to
`static/uploads/` and is served from `/uploads/`.

Theme settings (menu, profile, social links, footer) live in `zola.toml` and are
intentionally **not** exposed to the CMS — Sveltia cannot edit a Zola config file.

## Licence

[MIT](LICENSE) © Warayut Poomiwatracanont. The bundled theme keeps its own
licence at `themes/linkita/LICENSE`.
