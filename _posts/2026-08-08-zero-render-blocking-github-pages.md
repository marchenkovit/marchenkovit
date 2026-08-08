---
layout: post
title: "A GitHub Pages site with zero render-blocking requests"
description: "Dropping the Jekyll theme, inlining the CSS, and the cmark-gfm gotchas that only show up once the site is live."
date: 2026-08-08
tags: [performance, jekyll, github-pages]
permalink: /blog/zero-render-blocking-github-pages/
---

This site is a single Jekyll page served by GitHub Pages. It loads no external
stylesheet, no webfont, and no JavaScript framework. Here is what that took, and
the two rendering traps that only appear after deploying.

## Drop the theme, inline the CSS

The default `jekyll-theme-cayman` pulls a stylesheet over the network before the
browser can paint. For a page whose entire CSS budget is around 4 KB, that
round-trip costs more than the CSS itself.

Removing `theme:` from `_config.yml` and writing `_layouts/default.html` by hand
means the CSS can live in an include that is inlined into `<head>`:

```liquid
{% raw %}{% include critical-css.html %}{% endraw %}
```

One document, one request, nothing blocking the first paint.

## No webfonts

A font file is another blocking request, and a swap-in later causes a visible
reflow. The system stack costs nothing and looks native on every platform:

```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
             "Helvetica Neue", Arial, sans-serif;
```

## Reserve space for every image

Layout shift is not a rendering problem, it is a *measurement* problem: the
browser cannot reserve a box for an image whose dimensions it does not know yet.
Every `<img>` carries explicit `width`/`height` matching its SVG `viewBox`, so
the box exists before a single byte of the image arrives.

The hero also gets priority, while everything below the fold defers:

```html
<img src="hero.svg" width="1200" height="380" fetchpriority="high">
<img src="infra.svg" width="1200" height="880" loading="lazy">
```

## Push non-critical JS off the critical path

The floating table of contents is built from the page's own `<h2>` elements —
useful, but not worth a millisecond of first paint. `requestIdleCallback` runs it
only once the browser has nothing better to do:

```js
var idle = window.requestIdleCallback || function (cb) { return setTimeout(cb, 250); };
idle(buildToc, { timeout: 2000 });
```

## Trap 1: GitHub alerts do not exist on Pages

This renders as a nicely styled callout on github.com:

```markdown
> [!IMPORTANT]
> Copyright notice.
```

On GitHub Pages it renders as a plain blockquote containing the literal text
`[!IMPORTANT]`. The alert syntax is a github.com post-process, not part of
CommonMark, so `jekyll-commonmark-ghpages` (cmark-gfm) passes it straight
through. The fix is to write the markup you actually want:

```html
<blockquote class="markdown-alert-important" role="note" aria-label="Important">
  <p>Copyright notice.</p>
</blockquote>
```

Raw HTML survives because commonmark runs with the `UNSAFE` option — which is
also why the same file can mix Liquid, HTML, and markdown freely.

## Trap 2: excluded files are not just hidden, they are absent

`_config.yml` lists files that should not be published:

```yaml
exclude:
  - LICENSE
  - NOTICE
  - .github
```

A relative link such as `[LICENSE](./LICENSE)` still works on github.com, because
there the file is part of the repository view. On the built site it is a 404 —
the file was never copied into `_site`. Link to the canonical source instead:

```liquid
{% raw %}<a href="{{ site.github.repository_url }}/blob/main/LICENSE">LICENSE</a>{% endraw %}
```

## The common thread

Both traps share a shape: the markdown looked correct in the repository, and both
surfaces — github.com and the Pages site — render the *same file* differently.
Neither was catchable by reading the source. That is a good argument for building
the site in CI and checking the built output, rather than the input.
