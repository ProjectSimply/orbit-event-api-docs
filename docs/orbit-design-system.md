# Orbit design-system implementation

Source: [B2C + B2B websites — Figma](https://www.figma.com/design/5oN02SUp3cWDasZmfGRnnG/B2C---B2B-%E2%80%A8websites?node-id=0-1&m=dev)

The Figma file has a subscribed **Design System 2026** library and a component-overview page for desktop and mobile. The implementation treats the rendered Orbit components and the `Orbit Variables` library as authoritative; the file also contains an older local `PS Template` collection with some conflicting same-named values.

## CSS architecture

The canonical CSS custom properties live in `src/scss/sitewide/tokens.scss`. They are ordered from primitives to semantic roles:

1. Colour primitives such as `--color-brand-lilac`.
2. Semantic roles such as `--color-action-primary` and `--color-surface-page`.
3. Responsive type and spacing values, with mobile as the default mode and desktop overrides at 1024px.
4. Layout, shape, effect, and component roles.
5. Deprecated starter-template aliases, retained so unconverted modules continue to render during migration.

New component styles should consume semantic roles. A component may expose a local override, for example `--button-primary-background`, but should not redefine a global primitive.

## Foundations currently mapped

### Colour

| Role | Token | Value |
| --- | --- | --- |
| Page surface | `--color-surface-page` | Dark lilac `#1a1935` |
| Section surface | `--color-surface-section` | Lilac `#9282de` |
| Raised/light surface | `--color-surface-raised` | Light cream `#fbfaf4` |
| CTA surface | `--color-surface-cta` | Dark pink `#3c112d` |
| Primary text | `--color-text-primary` | Cream `#eeeadb` |
| Text on light surfaces | `--color-text-inverse` | UI black `#161616` |
| Primary action | `--color-action-primary` | Lilac `#9282de` |
| Accent action | `--color-action-accent` | Pink `#fa34ae` |
| Focus | `--color-focus` | Vibrant blue `#68f7f5` |

The B2C developer-handover page is the authority for theme composition. Every desktop handover frame uses dark lilac as its root fill. The homepage then alternates dark lilac with a lilac featured-event section and a dark-pink CTA section; cream is the default text colour on those dark and saturated surfaces.

### Typography

| Role | Mobile | Desktop | Family |
| --- | --- | --- | --- |
| Display | 42 / 0.96 | 72 / 0.96 | F37 Analog Bold |
| H1 | 30 / 1 | 64 / 1 | F37 Analog Bold |
| H2 | 24 / 1 | 32 / 1 | F37 Analog Bold |
| H3 | 18 / 1 | 24 / 1 | F37 Analog Bold |
| H4 | 15 / 1 | 24 / 1 | F37 Analog Bold |
| Body large | 15 / 1.3 | 24 / 1.3 | Manrope Regular |
| Body | 12 / 1.3 | 16 / 1.3 | Manrope Regular |
| Body small | 11 / 1.3 | 14 / 1.3 | Manrope Regular |
| Label large | 11 / 1.1 | 13 / 1.1 | Fragment Mono Regular |
| Label | 9 / 1.1 | 11 / 1.1 | Fragment Mono Regular |

The repository does not currently contain licensed F37 Analog or Fragment Mono webfont files. CSS uses the intended family names first and falls back safely. Replace the existing Adobe kit or add approved self-hosted `@font-face` assets when the client supplies font licensing and files. Do not substitute a lookalike into the canonical tokens.

### Spacing and layout

- Mobile gutter: 20px.
- Desktop gutter: 64px.
- Desktop canvas maximum: 1512px, giving a 1384px content area at the maximum width.
- Responsive spacing values are exposed as `--space-1` through `--space-9`.
- Section spacing is 64px in both current Figma modes.
- The historical `--sp-*` names remain compatibility aliases only.

### Shape and effects

- Small radius: 8px.
- Card radius: 16px.
- Pill radius: 999px.
- Large shadow: `0 20px 20px 4px rgba(0, 0, 0, 0.2)`.
- Standard transition: 300ms using the starter's existing expressive easing curve.

## Component implementation sequence

Build components in dependency order so later layouts compose established primitives:

1. Foundations: final font files, icons, link treatments, colour themes, and responsive grid helpers.
2. Controls: buttons, form fields, checkboxes/radios, tags, filter controls, and carousel controls.
3. Global shell: B2C/B2B navigation, notification/cookie UI, and both footer variants.
4. Content atoms: media card, event row/card, article/resource card, logo item, stat, quote, and accordion item.
5. Section patterns: 50/50, logo grid, stats list/tabs, statements, testimonials, news/resources, contact, free text, full media, case studies, timeline, forms, and FAQs.
6. Heroes: homepage, text, video, 50/50, and featured-event variants.
7. Page composition: map approved section patterns onto ACF flexible-content layouts, then add WordPress editor previews and template-level regression fixtures.

For every component, verify the desktop and mobile overview examples, keyboard focus, reduced-motion behaviour where relevant, long content, missing optional content, and both light/inverse colour contexts.
