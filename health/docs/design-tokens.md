# TDC Health — Design Tokens

Referenced by `CLAUDE.md` ("Tokens in docs/design-tokens.md") and
`008-navigation-app-shell`. These are the values the Flutter theme
(`ThemeData`) is generated from — **code reads this file's tokens, not ad hoc
hexes.** Semantic naming, not raw hex references.

Design stance (locked in `CLAUDE.md`): warm/calm, site-blue family, large
type + tap targets for elderly users, every screen readable at arm's length.

## Color

### Brand
| Token | Value | Use |
|---|---|---|
| `primary` | `#4B83F2` | Site blue — actions, active states, links |
| `primary-pressed` | `#3A6BD1` | Pressed/active variant |
| `primary-soft` | `#EAF1FE` | Selected chips, featured-action card bg |
| `primary-ink` | `#2A4E9E` | Text on `primary-soft` |

### Neutrals (warm-biased, never pure grey)
| Token | Value | Use |
|---|---|---|
| `bg` | `#FAF9F7` | App background (warm off-white) |
| `surface` | `#FFFFFF` | Cards, sheets |
| `border` | `#E7E2DC` | Hairlines, dividers |
| `ink` | `#26221E` | Primary text (warm near-black) |
| `ink-dim` | `#6B645C` | Secondary text |
| `ink-faint` | `#9C948A` | Placeholders, timestamps |

### Semantic (status ≠ brand; used only for state, never decoration)
| Token | Value | Use |
|---|---|---|
| `alert` | `#C2410C` | Low stock, missed dose, allergies text |
| `alert-soft` | `#FDEEE4` | Alerts-strip chip bg |
| `ok` | `#15803D` | Setup-complete, dose taken |
| `ok-soft` | `#E9F6EE` | Calm/complete card state |
| `warn` | `#A16207` | Pending states (OTP sent, sync pending) |
| `warn-soft` | `#FBF3DE` | |

Rule: cipher-free, claim-free color — semantic colors mark *state*, they
never dramatize ("danger red" walls are banned; the responder page's
allergy red is `alert` on white, not white on red).

## Type

Font: system default stack (SF/Roboto) — no custom font
(boring/already-paid-for). Elderly-first scale — **base is 18, not 16.**

| Token | Size/line | Weight | Use |
|---|---|---|---|
| `display` | 32/38 | 700 | Blood group on preview; big numbers ("4 days left") |
| `title` | 24/30 | 700 | Screen titles |
| `heading` | 20/26 | 600 | Card titles, section heads |
| `body` | 18/26 | 400 | Default text |
| `body-strong` | 18/26 | 600 | Emphasis within body |
| `caption` | 15/20 | 400 | Timestamps, helper text — **minimum size in the app**; nothing smaller ships |
| `overline` | 13/16 | 600, +0.08em, uppercase | Chip labels, section eyebrows only |

## Spacing & shape

4-pt base grid: `1=4` `2=8` `3=12` `4=16` `5=20` `6=24` `8=32` `10=40`.

| Token | Value | Use |
|---|---|---|
| `radius-card` | 16 | Profile cards, sheets |
| `radius-control` | 12 | Buttons, inputs |
| `radius-chip` | 999 | Chips, status pills |
| `screen-pad` | 20 | Horizontal screen padding |
| `card-gap` | 12 | Between stacked cards |

## Touch & accessibility (hard rules)

- Minimum tap target **48×48dp**; primary actions **56dp** tall.
- One primary button per screen; it sits in the thumb zone (bottom third).
- Text contrast ≥ 4.5:1 on its background (all token pairs above comply).
- Checklist checkboxes (`006`): 32dp visual, 48dp hit area.
- Respect OS font scaling up to 1.3× without layout breakage; test at 1.3×.

## Motion

Calm = minimal. Durations: 150ms (state), 250ms (navigation). Standard
easing only. No celebratory motion except the test-scan celebration
screen (P1) — that one moment gets confetti; nothing else does.
