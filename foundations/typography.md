# Typography

> Reference only. See repo [README](../README.md).

Machine-readable version: [`tokens/typography.json`](../tokens/typography.json).

## Font families

| Token | Family | Role |
|---|---|---|
| `inter` | Inter | Primary UI font — nearly everything (labels, body, nav, cards) |
| `playfairDisplay` | Playfair Display | Editorial/placeholder text — italic serif accents where the app wants a "journal" feel |
| `permanentMarker` | PermanentMarker_400Regular | Handwritten accent (used sparingly — journaling/personal touches) |
| `caveat` | Caveat_400Regular | Handwritten accent, alternate style |
| `mono` | Courier New | Monospace (utility) |

Pattern: **one clean sans (Inter) carries the interface; two handwritten fonts (PermanentMarker, Caveat) and one serif (Playfair) are reserved for moments that should feel personal/journal-like rather than "app chrome."** Worth preserving as a *strategy* — a UI voice + a "handwritten journal" voice — even if the specific fonts change in v2.0.

## Type scale

| Token | Size (pt) |
|---|---|
| `xs` | 8 |
| `sm` | 12 |
| `base` | 16 |
| `md` | 18 |
| `lg` | 20 |
| `xl` | 24 |
| `2xl` | 32 |
| `3xl` | 36 |
| `4xl` | 40 |

## Weights

| Token | Value |
|---|---|
| `regular` | 400 |
| `semibold` | 600 |
| `bold` | 700 |
| `black` | 900 |

## Named text styles (Figma-matched)

These are the predefined composite styles (`TEXT_STYLES`) the app used to keep text consistent across screens rather than hand-setting size/weight/color every time:

| Style | Family | Size | Weight | Notes |
|---|---|---|---|---|
| `headerGreeting` | Inter | 24 (xl) | 600 | White |
| `navTab` | Inter | 24 (xl) | 600 | Active: white / Inactive: `#2E2E2E` |
| `cardTitle` | Inter | 16 (base) | 600 | White |
| `entryLabel` | Inter | 16 (base) | 700 | Colored `lime800` — used for field labels |
| `entryInput` | Inter | 24 (xl) | 400 | White, large — journaling input feels roomy, not cramped |
| `mapControlButton` | Inter | 8 (xs) | 700 | Black — tiny, high-contrast map chrome |
| `mapLocationLabel` | Inter | 16 (base) | 600 | White |
| `placeholderText` | Playfair Display | 18 (md) | 600, *italic* | Black — this is where the serif/editorial voice shows up, e.g. empty-state or placeholder copy |

## Accessibility note

Text using `GradientText` (see effects doc) explicitly caps `maxFontSizeMultiplier={1.5}` since gradient-masked text can't reflow as gracefully as plain text under large Dynamic Type. Feed card overlay text (title/date over photos) caps at `1.2`. Worth carrying the *practice* forward: cap font scaling deliberately on decorative/overlay text, not globally.
