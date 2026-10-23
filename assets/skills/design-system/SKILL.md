---
name: design-system
description: Create or extend a product's design system for its desktop app — DESIGN.md at the project root, the source of truth for how every screen looks, moves and reads on macOS, Windows and Linux, in the DESIGN.md format. Use once per product before the first screens are designed, after reading the product docs and researching the best desktop apps in its niche, and again when a later phase needs a foundation the system does not cover.
---

# Design System

You are a senior product designer and design engineer writing the design system for this product's desktop app. The result is `DESIGN.md` at the project root: one document that makes every screen, built by anyone in any session, look like it belongs to the same product, and makes that product look like itself rather than like every other AI-generated app.

It follows the DESIGN.md format (`github.com/google-labs-code/design.md`): design tokens in YAML front matter, and markdown sections that explain how to apply them. `/desktop-build` reads it before every screen is designed, design tools find it at the root, and its tokens become the CSS variables the code uses. So it is precise where precision matters (values, names, states) and plain everywhere else.

## Inputs

Read these first; ask for anything missing.

| Input | What it gives you |
| --- | --- |
| Product brief and PRD, `docs/product-docs/` | Who the product is for, the problem, the positioning, the tone it should carry |
| `docs/feature-areas.md` | The surfaces: how many areas, which are data-dense, which are transactional, which are consumer-facing |
| Business rules and app flows, `docs/business-rules/`, `docs/app-flows/` | The states screens need (pending, failed, expired), the data types that recur (money, dates, statuses), the roles and how different their worlds are |
| `CLAUDE.md` | The audience this repo serves, whether a second audience shares the brand, the platforms, and which OS capabilities (tray, file types, multiple windows) the system must give a pattern to |
| Brand assets, if any | The logo system from the Ideas Blueprint, brand colors, a name, a tagline. The logo's color and geometry are inputs to the palette and shape language, not the other way round |
| Another client's `DESIGN.md` for the same product, if one exists (at the sibling client repo path in `CLAUDE.md`) | The brand decisions — palette, type, voice, shape — are taken from it so the product looks like itself on every client; the desktop components, windows, menus and motion are this system's own |
| An existing `DESIGN.md` | In a later phase: extend it, never regenerate it |

If a frontend-design skill is installed in this Claude Code, read it as well; it sharpens the aesthetic judgement this skill asks for.

## Step 1 — Understand the product

Before any color is picked, write down, in a few lines each:

- **What kind of app this is on the desk**, because it sets density, pace and tone. Most desktop apps are the first of these; say which applies where.
  - *A work tool open all day* (operations, creative tools, communication, data-heavy apps): users live in it, so the system leans compact and keyboard-first. Dense lists and tables, panels the user sizes, a command palette, shortcuts on every primary action, restrained color, and chrome that recedes.
  - *A focused utility used in short sessions* (a sync client, a recorder, a launcher): one job, few screens, often a tray presence. Clear state, few decisions per screen, and an exit.
  - *A consumer product used for enjoyment* (media, reading, creation): the content is the interface, so imagery, motion and personality carry more weight.
- **Who is looking at it, and in what state.** Experts or first-timers; alone or presenting on a shared screen; on a 13-inch laptop or a 32-inch display at 200% scaling. This decides density, type size, and how much the interface explains itself.
- **What the brand should feel like**, in three or four plain adjectives taken from the brief, and what it must not feel like.
- **The recurring content**: the data types that appear everywhere (amounts, dates, statuses, names, avatars, long text), because the system has to make those consistent first.
- **The constraints**: accessibility level (WCAG AA is the floor, with VoiceOver, Narrator and Orca), dark mode (yes, following the OS, unless the brief says otherwise), right-to-left or a second language, brand colors that are fixed, and **platform conventions**: the system follows each OS where they differ in chrome (title bar, menu bar, dialog button order, shortcut modifiers, context menus) and is the product's own everywhere else. Say which conventions are kept per platform and which are unified.

## Step 2 — Research the references

The goal is to learn what the best products in this space do well and why, then do it our own way. References are inspiration, never a source of truth and never something to copy.

**On Mobbin** (through its MCP, `platform: web` — desktop apps are catalogued with the web — plus the macOS and Windows design guidelines for the chrome; follow the Mobbin search skill if it is installed):

1. Search flows and screens for **three to five direct competitors or neighbours** named in the brief, and **two or three category leaders** whose interface is widely admired, in the product's category or a close one (a dense tool learns from the best dense tools, whatever their domain).
2. For each, look at the actual screens — the images, not the metadata — of the surfaces this product will have: the main workspace, the main list and detail, a form, the sidebar and navigation, the command palette, a settings window, an empty state, the update and offline states.
3. Record, per reference, what makes it work: how it builds hierarchy, how much color it uses and where, its density and spacing rhythm, its type choices, how it handles states and feedback, what its components have in common, what its copy sounds like, and what it does that is distinctive. Also what it does badly, so we avoid it.

**Outside Mobbin**, where it helps: the products' own marketing sites. Where the direction draws on a product documented in the corpus at `github.com/VoltAgent/awesome-design-md`, that product's DESIGN.md shows its exact decisions; learn from it, never copy. Cite everything you looked at.

**Then extract**: a short list of the principles the strong references share, and the two or three moves that would make this product distinctive rather than generic. That list is what the system is built from.

## Step 3 — Propose the direction → **wait for my approval**

Present one direction, with the reasoning from Steps 1 and 2, and one alternative where the choice is genuinely open. Include:

- **Principles** — four to six, each one sentence, each with a consequence a builder could act on ("Density over decoration: tables default to compact rows, and whitespace is spent on grouping, not on margins").
- **Palette** — the brand color and why, the neutral scale and its temperature, the semantic colors, the accent policy (how much color a screen may carry), with hex values and the contrast ratios of text on surface, light and dark.
- **Typography** — the families, the scale, the weights in use, and the reasoning. Free fonts only, two families at most: Google Fonts by default, or another open-source font (SIL OFL or similar). Never a font that must be bought, or one licensed for personal use only.
- **Shape and depth** — the radius scale and the elevation approach (borders, shadows, or surface steps).
- **Density and layout** — the base unit, the minimum window size, the panels (which the user can size, which collapse, at what widths), and the shell's form (sidebar, title area, status bar, and why). The shell itself is designed and built in `/desktop-build`'s Phase 0 from this decision.
- **Keyboard stance** — which actions carry shortcuts, how shortcuts are shown, what the command palette covers.
- **Motion stance** — how much, how fast, and what never moves.
- **Imagery and icon stance** — illustration, photography, neither; the icon set; the tone of imagery.
- **Voice** — how the product talks, with three example sentences.

Show it: a style tile rendered on the `/design` canvas where available, otherwise an HTML page in `docs/design/style-tile.html`, at a desktop window's width, with the palette, the type scale, three buttons in their states, an input, a list row, a table row, a menu and the title area, light and dark. Judge it with your own eyes before showing it: if it looks like a template, change it before I see it.

**What generic looks like, so you can avoid it:** a blue-purple gradient on a dark background; Inter at every size with no tracking decisions; cards inside cards with the same shadow; pastel badges in six colors; the same 8px radius on everything; glassmorphism; a web page's margins inside a desktop window; a sidebar that is a list of icons with no labels; icons used as decoration. Distinctive comes from a few deliberate decisions carried everywhere: one accent used with discipline, a type pairing with a point of view, a radius or border rule that is recognisably the product's, spacing that creates rhythm, motion that has one personality.

## Step 4 — Write `DESIGN.md`

Tokens go in the front matter, using only the format's keys. Values the format has no key for — shadows, motion, window sizes, shortcuts — are written as tables in their sections. Color tokens use shadcn's variable names, so `DESIGN.md`, `globals.css` and the components share one vocabulary; each color's light value has a `-dark` twin.

```yaml
---
version: alpha
name: "{Product} Design System"
description: One paragraph that captures the whole system in words.
colors:
  background: "#…"
  background-dark: "#…"
  foreground: "#…"
  foreground-dark: "#…"
  # …the same pair for primary, primary-foreground, secondary, secondary-foreground, muted, muted-foreground,
  # accent, accent-foreground, destructive, success, warning, info, border, input, ring, card, card-foreground,
  # popover, popover-foreground, chart-1 to chart-5, sidebar, sidebar-foreground, sidebar-primary,
  # sidebar-primary-foreground, sidebar-accent, sidebar-accent-foreground, sidebar-border, sidebar-ring,
  # and the product's own tokens
typography:
  display: { fontFamily: …, fontSize: …, fontWeight: …, lineHeight: …, letterSpacing: … }
  # …h1, h2, h3, h4, body-lg, body-md, body-sm, caption, label-md, button, mono, numeric (fontFeature: '"tnum"')
rounded: { none: 0px, sm: …, md: …, lg: …, xl: …, full: 9999px }
spacing: { base: 4px, … }            # the scale, matching Tailwind's steps
components:
  button-primary: { backgroundColor: "{colors.primary}", textColor: "{colors.primary-foreground}", typography: "{typography.button}", rounded: "{rounded.md}", padding: … }
  button-primary-hover: { backgroundColor: … }
  button-primary-dark: { backgroundColor: "{colors.primary-dark}", textColor: "{colors.primary-foreground-dark}" }
  # …one entry per component and state, with a -dark entry carrying its dark colors
---
```

Component entries use only the format's properties: `backgroundColor`, `textColor`, `typography`, `rounded`, `padding`, `size`, `height`, `width`. Borders, shadows and focus rings are described in the component's section.

Then the sections, with exactly these `##` headings in this order:

```markdown
## Overview            — the system in a paragraph; the principles from Step 3; then Key Characteristics: the five to eight visual decisions that make it this product; which platform conventions are kept and which are unified
## Colors              — grouped by role (brand and accent, surface, text, semantic, charts), each with its use; the accent policy; hover, active and disabled; light and dark; a contrast table
## Typography          — the families, why, and where each loads from; a hierarchy table (token, size, weight, line height, tracking, use); measure and tracking rules; tabular figures for numbers in data
## Layout              — the base unit and the spacing scale with use; the shell (title area, sidebar, panels, status bar); the minimum window size and the content width; which panels the user sizes and which collapse; screen anatomy (header, filters, content, actions)
## Elevation & Depth   — how hierarchy is shown (shadows, borders or surface steps), as a table of level, treatment, shadow value and use
## Shapes              — the radius scale as a table of token and use; borders; focus rings
## Iconography         — the set, sizes, stroke, when an icon stands alone and when it needs a label; the app icon per platform (macOS, Windows, Linux) and the tray icon's template form
## Imagery             — photography or illustration, tone, treatment, aspect ratios, avatars, empty-state art, what never appears
## Motion              — the personality; a table of durations and easings by name; window and panel transitions; what animates and what never does; reduced motion
## Components          — one entry per component in use: anatomy, variants, sizes, every state (default, hover, focus, active, disabled, loading, error, selected), and when to use it. Buttons, inputs and fields, selects, checkboxes and radios, switches, forms, cards, tables and lists (virtualised), tree views, tabs, the sidebar, the title area, panels and splitters, toolbars, the status bar, the command palette, dialogs and sheets, menus and context menus (native, so only their trigger is styled), tooltips with their shortcut, toasts and inline alerts, badges and status, avatars, skeletons, empty states, filters, date pickers, the update banner; then the signature components only this product has
## Keyboard            — the shortcut table (action, macOS, Windows and Linux), how shortcuts are shown in menus and tooltips, focus order and focus rings, what the command palette covers
## Patterns            — screen patterns (workspace, list, detail, create/edit form, settings window, wizard, dashboard) and state patterns (loading, empty, error, permission denied, offline, update available, force update); which flows open a secondary window and which stay in the main one
## Content and Voice   — tone, person and tense, capitalisation, punctuation; formats for numbers, dates and money; menu and button labels; error, empty-state, confirmation and update copy; words we use and words we never use
## Window Sizes        — a table of the window sizes the system is designed for, from the minimum to ultrawide; how the shell, panels, tables, forms and dialogs adapt; 100% and 200% OS scaling
## Accessibility       — contrast, focus, keyboard, screen readers (VoiceOver, Narrator, Orca), motion, labels, and the checks every screen passes
## Do's and Don'ts     — the ten rules a builder is most likely to break
## Iteration Guide     — how to apply the system (reference components and tokens by name, one component at a time, the default body style, how scarce the accent is) and how a later phase extends it
```

**Rules for the content:**

- **Every value is a decision, with a use.** A token nobody is told when to use will be used wrongly.
- **Tokens are the source; prose explains them.** A value in prose always matches its token, and names it (`{colors.primary}`).
- **States are not optional.** A component entry without its disabled, loading and error states is incomplete.
- **Plain where it can be, precise where it must be.** The Overview and Content and Voice sections read like a person wrote them; the token and component sections are exact.
- **Contrast is computed, not assumed.** Every text-on-surface pair in the colors table shows its ratio and passes AA (4.5:1 body, 3:1 large text and UI). Fix the palette rather than the rule.
- **Platform differences are stated, not left to habit.** Where macOS, Windows and Linux differ (the title bar, the menu bar, dialog button order, the shortcut modifier, context menus), the component says what each gets; where the product is the same on all three, it says so.
- **Nothing that belongs elsewhere.** No code, no component props, no business rules, no screen inventories. The system says how things look, move and read; the screen specs say what they contain.
- **No stub sections.** A section with nothing to say for this product is left out.
- **Length follows the product**: a dense tool with many components runs long; a focused consumer product stays short. Cut anything that repeats.

## Step 5 — Verify and apply

1. **Lint it:** `npx @google/design.md lint DESIGN.md`. No errors. The linter checks every component pair against 4.5:1, so a contrast warning is fixed in the palette unless the pair is large text or a non-text element that the contrast table shows at 3:1 or more. Orphaned-token warnings are expected for colors no component token can reference (borders, rings, charts); any other warning is resolved.
2. **Read it back as a builder.** Pick two screens from the app flows (one list, one form) and describe each using only `DESIGN.md`, on each platform, with its shortcuts. Anything you had to invent is a gap; fill it.
3. **Check the tokens** cover every shadcn variable with its `-dark` twin, and every pair in the contrast table passes.
4. **Apply the tokens** if the foundation exists: run `/desktop-setup`'s design tokens step so `globals.css` and the shadcn components carry the system, then render the component set on a dev-only route (`#/design-check`, created here if it does not exist, excluded from production builds) and look at it, light and dark, at the minimum window size and maximised on a large display, at 100% and 200% scaling, on each platform you can run. Fix what looks wrong in the system, not in the screen.
5. **Record** in the `Design system` row of `CLAUDE.md`'s project configuration that `DESIGN.md` exists and the phase it was last extended in, and report: the direction, the references consulted, what makes this product distinctive, and any decision left open.

## Later phases

A later phase extends the system when it introduces something the system does not cover: a new component, a new data visualisation, a new surface. Add the tokens and the section text in place, lint again, and compare with the committed version (`git show HEAD:DESIGN.md > /tmp/DESIGN.prev.md`, then `npx @google/design.md diff /tmp/DESIGN.prev.md DESIGN.md`); the token changes it lists go in the report. Update the `Design system` row in `CLAUDE.md` with the phase. Never change an existing token's value in a later phase without a screen-by-screen check of what uses it; a palette change is a project, not an extension.
