---
name: red-hat-branding
description: >-
  Red Hat brand standards for documentation and other artifacts: colors,
  typography, and usage rules. Use when the user asks to Red Hat–brand something
  (docs sites, Angular apps, slides, diagrams, UI chrome); when they mention Red
  Hat branding, brand standards, Red Hat red, or Red Hat Text/Display/Mono. Do
  not apply by default to MkDocs or Angular work.
---

Apply this skill only when the user asks for Red Hat branding (or clearly wants an artifact to look like Red Hat). Do not brand MkDocs sites, Angular apps, or other work by default.

Use the official brand standards when creating branded documentation or related artifacts:

- https://www.redhat.com/en/about/brand/standards
- https://www.redhat.com/en/about/brand/standards/color
- https://www.redhat.com/en/about/brand/standards/typography

For UI/product design, also see [PatternFly](https://www.patternfly.org/) and [ux.redhat.com](https://ux.redhat.com/).

## Color

**Red Hat red (red-50):** `#ee0000` — include a pop of it in everything; do not flood large areas with it.

### Core palette (use these first)

| Token              | Hex       |
| :---               | :---      |
| red-05             | `#fef0f0` |
| red-10             | `#fce3e3` |
| red-20             | `#fbc5c5` |
| red-30             | `#f9a8a8` |
| red-40             | `#f56e6e` |
| red-50 (brand red) | `#ee0000` |
| red-60             | `#a60000` |
| red-70             | `#5f0000` |
| red-80             | `#3f0000` |
| white              | `#ffffff` |
| gray-10            | `#f2f2f2` |
| gray-20            | `#e0e0e0` |
| gray-30            | `#c7c7c7` |
| gray-40            | `#a3a3a3` |
| gray-45            | `#8c8c8c` |
| gray-50            | `#707070` |
| gray-60            | `#4d4d4d` |
| gray-70            | `#383838` |
| gray-80            | `#292929` |
| gray-90            | `#1f1f1f` |
| gray-95 (ux black) | `#151515` |
| black              | `#000000` |

### Secondary (with core only; 1–2 per composition)

| Family | Key hex     |
| :---   | :---        |
| orange | `#f5921b` (orange-40) |
| yellow | `#ffcc17` (yellow-30) |
| teal   | `#37a3a3` (teal-50)   |
| purple | `#5e40be` (purple-50) |

### Information (status/UI only — not decorative)

| Token             | Hex       | Use                          |
| :---              | :---      | :---                         |
| success-green-50  | `#63993d` | Success / increase           |
| danger-orange-50  | `#f0561d` | Negative / destructive       |
| interaction-blue-50 | `#0066cc` | Links / interaction        |
| teal              | (teal family) | Neutral / no severity    |
| purple            | (purple family) | Info / tip               |

Do **not** use red for negative/error meaning — red is the brand color. Use danger-orange for destructive states.

### Color principles

- Stick to core colors for anything that must look unmistakably Red Hat.
- Use red as intentional pops, not full-bleed fills.
- Fill large areas with lightest tints, darkest shades, or subtle brand gradients only.
- Never invent colors or gradients outside the standards.
- Meet WCAG AA contrast: ≥4.5:1 for small text, ≥3:1 for large text/icons.
- Do not rely on color alone; add labels or icons for diagrams and status.

## Typography

### Fonts

| Font              | When to use                                      |
| :---              | :---                                             |
| Red Hat Display   | Default; large or bold headlines                 |
| Red Hat Text      | Body copy, paragraphs, small UI text             |
| Red Hat Mono      | Code only (or technical stylized headlines)      |
| Noto Sans         | Non-Latin scripts                                |

Fonts are open source (SIL). Prefer them over system defaults.

### Type rules

- Sentence case everywhere, including headlines. No all caps. Avoid title case except proper nouns/product names.
- Flush left by default; center only when aligning with other centered elements. Never justify body text.
- Body copy: black or white. Red is fine for large headlines/pull quotes, not long paragraphs.
- Emphasize with bold **or** color — one treatment at a time. No italics, underline, or all caps for emphasis.
- Hyperlinks: interaction blue (`#0066cc` on light / blue-30 on dark) with underline.
- Line height: 1.2–1.5× for most text; ~1.1× for very large display type. Never outside 1.1–1.5×.
- Line length: about 20–100 characters. Prefer white space over cramming.
- Do not change tracking; use the fonts' built-in spacing.
- Prefer “and”; use `&` only when space is tight. Never `+` for “and”.

## Docs sites (Material for MkDocs)

Only when the user asks to Red Hat–brand an MkDocs site: follow the `mkdocs` skill for layout/config, then overlay brand fonts + CSS:

**`mkdocs.yaml` theme fonts:**

```yaml
theme:
  font:
    text: Red Hat Text
    code: Red Hat Mono
  palette:
    - scheme: default
      primary: custom
      accent: custom
    - scheme: slate
      primary: custom
      accent: custom
```

**`docs/assets/stylesheets/extra.css`:**

```css
:root {
  --md-primary-fg-color: #ee0000;
  --md-primary-fg-color--light: #f56e6e;
  --md-primary-fg-color--dark: #a60000;
  --md-accent-fg-color: #ee0000;
}

[data-md-color-scheme="slate"] {
  --md-primary-fg-color: #ee0000;
  --md-primary-fg-color--light: #f56e6e;
  --md-primary-fg-color--dark: #a60000;
  --md-accent-fg-color: #ee0000;
}
```

## Angular apps

Only when the user asks to Red Hat–brand an Angular app: follow the `angular` skill for project structure and ng-bootstrap, then overlay brand fonts and colors in global SCSS. Do not switch the component library unless asked.

Load Red Hat Text, Display, and Mono (Google Fonts or `@fontsource`). In `src/styles.scss`:

```scss
:root {
  --rh-red-50: #ee0000;
  --rh-red-40: #f56e6e;
  --rh-red-60: #a60000;
  --rh-gray-95: #151515;
  --rh-interaction-blue-50: #0066cc;
  --bs-primary: var(--rh-red-50);
  --bs-link-color: var(--rh-interaction-blue-50);
}

html {
  font-family: "Red Hat Text", sans-serif;
}

h1, h2, h3, h4, h5, h6 {
  font-family: "Red Hat Display", sans-serif;
}

code, pre, kbd, samp {
  font-family: "Red Hat Mono", monospace;
}
```
