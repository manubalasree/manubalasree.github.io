# VirtMicro Brand Guide

Version 1.0 · November 2026 · virtmicro.com

## 1. The idea

VirtMicro is about running small things inside well-built hosts: virtualization and microservices.

The mark is an **octagon (the host) with a smaller solid octagon inside (the workload)**. The outline is the boundary, the isolation, the guardrail. The solid core is the small unit of work it protects. Octagons also echo a stop sign, which suits an SRE brand: calm, deliberate, reliability-first.

The tone is **calm and precise**, with enough colour to feel alive.

## 2. Logo

| Version | Use for | File |
|---|---|---|
| Horizontal | Default. Site header, documents, slides | `logo/svg/virtmicro-horizontal-color.svg` |
| Stacked | Square or narrow spaces, title slides | `logo/svg/virtmicro-stacked-color.svg` |
| Icon (mark only) | Avatars, favicons, small spaces | `logo/svg/virtmicro-icon-color.svg` |
| App icon | App tiles, bookmarks, PWA | `logo/svg/virtmicro-app-icon-navy.svg` |

Each version comes in four colourways:

- **color**: navy outline, blue core. Use on white or light backgrounds.
- **reversed**: white outline, sky core. Use on navy, deep blue or dark photography.
- **black / white**: single colour. Use for stamps, engraving, fax-style print, or any place colour is not available.

SVG is the master format. Use PNG only where SVG is not accepted. The wordmark in every SVG is converted to outlines, so it renders correctly without installing any font.

### Clear space

Keep an empty margin around the logo equal to **half the height of the mark** on all sides. Nothing else (text, edges, other logos) enters that zone.

### Minimum size

- Horizontal logo: 120 px wide (or 30 mm in print)
- Stacked logo: 80 px wide
- Mark alone: 24 px; below that, use the files in `favicon/`, which have a heavier outline built for 16 to 48 px

### Please don't

- Recolour the mark or wordmark outside the palette
- Stretch, rotate, outline or add shadows or gradients
- Retype the wordmark in a font; always use the supplied files
- Place the colour logo on a busy photo or a mid-tone blue; switch to reversed or single colour
- Separate "Virt" and "Micro", or change the weight split (Virt regular, Micro bold)

## 3. Colour

| Name | Hex | RGB | Role |
|---|---|---|---|
| Ink Navy | `#0F2557` | 15, 37, 87 | Primary. Outline, wordmark, dark backgrounds |
| Signal Blue | `#2F6FDE` | 47, 111, 222 | Accent on light. Core octagon, links, buttons |
| Sky | `#5CA2FF` | 92, 162, 255 | Accent on dark |
| Deep Blue | `#1A3A82` | 26, 58, 130 | Secondary dark surface, hover on navy |
| Mist | `#E8F0FE` | 232, 240, 254 | Light tint for page and card backgrounds |
| White | `#FFFFFF` | 255, 255, 255 | Reversed text, light backgrounds |

Ready-to-use tokens are in `colors/colors.css` and `colors/colors.json`.

### Contrast (WCAG)

Measured contrast ratios for the pairings you are most likely to use:

| Text on background | Ratio | Verdict |
|---|---|---|
| Navy on white | 14.8:1 | AAA for all text |
| Navy on Mist | 12.9:1 | AAA for all text |
| White on navy | 14.8:1 | AAA for all text |
| White on Deep Blue | 10.7:1 | AAA for all text |
| Sky on navy | 5.7:1 | AA for all text |
| Signal Blue on white | 4.7:1 | AA for all text |
| White on Signal Blue | 4.7:1 | AA for all text |
| Signal Blue on Mist | 4.1:1 | Large text (24 px+ or 19 px bold) and graphics only |
| Sky on Deep Blue | 4.1:1 | Large text and graphics only |

Rule of thumb: body text is navy on light or white on navy. Signal Blue is for links, buttons and the logo core, not for small text on Mist.

## 4. Typography

**Space Grotesk** (SIL Open Font License, free for commercial use). Weights 400, 500 and 700 are included in `fonts/`.

- Headings: 700, tight tracking (about -2%)
- Body and UI: 400 or 500
- Code and terminal examples: any monospace; JetBrains Mono or system mono pair well

Web usage:

```css
@font-face {
  font-family: 'Space Grotesk';
  src: url('/fonts/space-grotesk-latin-400-normal.woff2') format('woff2');
  font-weight: 400;
  font-display: swap;
}
@font-face {
  font-family: 'Space Grotesk';
  src: url('/fonts/space-grotesk-latin-700-normal.woff2') format('woff2');
  font-weight: 700;
  font-display: swap;
}
body { font-family: var(--vm-font); color: var(--vm-navy); }
```

## 5. Using the files

| Folder | What is inside |
|---|---|
| `logo/svg/` | Master vector logos: horizontal, stacked, icon (4 colourways each) and 2 app icons |
| `logo/png/` | Transparent PNG exports: horizontal at 2000 and 800 px wide, stacked at 1200 px wide, icon at 1024 and 256 px, app icons at 1024 and 512 px |
| `favicon/` | `favicon.svg` (switches to light-on-dark automatically), `favicon.ico`, PNGs at 16, 32, 48, 180 (Apple touch), 192 and 512 (PWA) |
| `social/` | Avatar (1024 px, works inside a circle crop), Open Graph card (1200 x 630), LinkedIn banner (1584 x 396), each with an SVG source |
| `colors/` | CSS variables and JSON tokens |
| `fonts/` | Space Grotesk WOFF2 files and licence |

### Favicon snippet

```html
<link rel="icon" href="/favicon.ico" sizes="48x48">
<link rel="icon" href="/favicon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon-180.png">
```

Rename `apple-touch-icon-180.png` to `apple-touch-icon.png` if you prefer the default path. For PWA manifests, use `icon-192.png` and `icon-512.png`.

### Social card snippet

```html
<meta property="og:image" content="https://virtmicro.com/og-image-1200x630.png">
<meta name="twitter:card" content="summary_large_image">
```

## 6. Quick checks before you publish

1. Is the logo on a background with enough contrast, using the right colourway?
2. Is there clear space of half a mark height around it?
3. Is it large enough (120 px wide, or the mark alone at 24 px)?
4. Is any text using a pairing from the contrast table that passes for its size?
