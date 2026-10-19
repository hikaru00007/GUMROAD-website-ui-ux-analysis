# Gumroad Design Language

Reverse-engineered from the production build of `gumroad.com`. Everything here
is lifted from the shipped CSS (`design-CpolmKbM.css`), the Inertia page props,
and the compiled React bundles — not from screenshots.

**Researched:** `/discover` · `/blog` · `/pricing` · `/features` · `/about` · `/gumclaw`
**Date:** 2026-09-28

---

## The five rules

If you read nothing else, this is the whole system.

1. **Every solid thing has a 1px black border.** `border: solid 1px`. Always.
   The border is the load-bearing part of the aesthetic, not a finishing touch.
2. **Shadows are hard, never blurred.** `4px 4px 0` — zero blur radius. A blurred
   shadow instantly reads as generic; a zero-blur shadow reads as *printed*.
3. **Shadow offset always equals the element's travel distance.** Move 4px on
   hover → reveal a 4px shadow. Move 1px → reveal a 2px shadow. This single
   invariant is why the interactions feel mechanical and intentional.
4. **Color is flat and unmodulated.** Six saturated brights at full value, no
   gradients, no tints, no 50-shade ramps in the marketing surface. Saturation
   is the personality, so they refuse to dial it down.
5. **Pink is reserved for exactly one meaning: interactivity or emphasis.**
   `#ff90e8`. Link hover, focus ring, newsletter button, active states. Nothing
   else. One accent, used relentlessly, is why it reads as confident.

---

## 1. Stack

| Layer | Choice |
|---|---|
| Server | Ruby on Rails |
| Bridge | **Inertia.js** — every element carries `inertia="..."` attributes |
| Framework | React 18 (`forwardRef`, `createContext`, `useSyncExternalStore`) |
| Build | **Vite** — `assets.gumroad.com/vite/assets/entrypoints/*.js` |
| CSS | **Tailwind CSS v4** — `oklch()` palette, `@theme`, `@starting-style`, `@variant` |
| Components | `cva` (class-variance-authority) + `tailwind-merge` |
| Fonts | Self-hosted woff2 on `assets.gumroad.com/fonts/` |
| Editor | ProseMirror (note the `prosemirror-gapcursor-blink` keyframe) |
| Payments | Stripe (`stripe:pk` meta), PayPal, Discord, Apple, Kindle, Zoom… |
| Edge | Cloudflare (rocket-loader, insights beacon) |

Two details worth stealing:

- **`.scoped-tailwind-preflight`** — they scope the Tailwind reset so it can't
  destroy non-Tailwind page styles. Essential when Tailwind is one layer of a
  larger app.
- **A single CSS entry point** (`entrypoints/design-*.css`) is loaded on every
  page. The design language is a build artifact, not a set of per-page sheets.

---

## 2. The signature mechanic

This is the part you're in love with, and it's more disciplined than it looks.

### The button

Extracted verbatim from the compiled bundle:

```js
cva(
  "inline-flex items-center justify-center gap-2 cursor-pointer " +
  "border border-border rounded text-current font-[inherit] no-underline " +
  "transition-transform " +
  "hover:-translate-1 " +
  "hover:shadow-[0.25rem_0.25rem_0_var(--color-black)] " +
  "active:translate-0 active:shadow-none",
  { variants: { size: {
      default: "px-4 py-3 text-base leading-snug",
      sm:      "p-2 text-sm leading-[1.3]",
      icon:    "size-12 box-border",
  } } }
)
```

Read it as three states:

| State | Transform | Shadow |
|---|---|---|
| Rest | `translate(0, 0)` | none |
| Hover | `translate(-4px, -4px)` | `4px 4px 0 #000` |
| Active | `translate(0, 0)` | none |

**The shadow is not added — it's revealed.** The button moves up-and-left into
the space the shadow vacates. Nothing fades, nothing grows. The element looks
*pressed into the page like a letterpress plate*.

Three consequences that make it feel expensive:

- `transition-transform` — **not** `transition-all`. Only `transform` is
  animated. The shadow is a hard swap, which is what keeps it crisp. Animating
  shadows would blur the whole effect.
- `font-[inherit]` — the button inherits the page's typeface. Buttons never
  look "system-UI pasted on."
- `text-current` — text color follows whatever the parent set. This is what
  makes the same component work as a black button, a pink button, and a ghost
  button without variant sprawl.

### The same rule at a different scale

The 40px circular arrow on blog cards:

```
group-hover:-translate-x-px group-hover:-translate-y-px
group-hover:shadow-[2px_2px_0_0_#000]
```

1px travel → 2px shadow. Not 1px. The rule scales: **shadow offset ≈ 1× travel
for large elements, 2× for small ones** (small elements need a visible shadow
to register at all). The point is the relationship, not the number.

### The nav

The nav link is one class string doing double duty:

```
border border-transparent            /* border always present → no layout shift */
lg:bg-transparent lg:text-black      /* desktop: invisible pill */
bg-black text-white p-4 text-lg      /* mobile: solid black block */
lg:rounded-full lg:py-2 lg:px-4
hover:border-black
lg:dark:hover:border-white/35
```

The 1px border is **transparent at rest and black on hover**. Since the border
box never changes size, the outline appears without a single pixel of layout
shift. This is the cheapest possible "bold outline" and they use it everywhere.

---

## 3. The offset-print component

The most distinctive object on the site. Used for the primary CTA, the
newsletter submit, and every Discover product card. Three stacked layers:

```html
<div class="relative inline-block group">
  <!-- layer 2: yellow, static -->
  <div class="absolute inset-0 rounded-sm bg-yellow border border-black
              transition-transform duration-150 z-2"></div>
  <!-- layer 1: red, slides out on hover -->
  <div class="absolute inset-0 rounded-sm bg-red border border-black
              transition-transform duration-150
              group-hover:translate-x-2 group-hover:translate-y-2 z-1"></div>
  <!-- layer 3: the actual control -->
  <a class="relative inline-flex rounded-sm border border-black
            transition-all duration-150
            group-hover:-translate-x-2 group-hover:-translate-y-2 z-3
            h-14 px-8 text-xl bg-black text-white dark:bg-pink dark:text-black">
    Start selling
  </a>
</div>
```

It's a **misregistration effect** — the visual language of cheap two-color
offset printing or a misaligned screen print. On hover the control slides up-left
while the red ghost slides down-right, opening a two-tone chromatic gap. Both
ghosts carry their own black border, so the gap is framed, not just colored.

Implementation notes:

- The hover state lives on the **parent** (`group`), not the `<a>`. The parent
  moves nothing; only the two ghosts and the child move.
- `duration-150` — Gumroad's hover transitions run at **150ms**, not the 200–300ms
  you'd normally reach for. Instant, mechanical, unapologetic.
- `rounded-sm` (4px), *not* `rounded-full`. Sharp corners on the color stack;
  the pills are reserved for nav and tags.
- Sizing is deliberately chunky: `h-14 px-8 text-xl`, going to `h-16 px-10` at
  `lg`. 56–64px tall. These are buttons you can see from across the room.

**Dark mode flip:** the primary CTA becomes `bg-pink text-black`. Black-on-white
inverts to pink-on-black. The single loudest move in the entire system, and it's
one class pair.

---

## 4. Color

### The brand palette

Every value below is a real token from the shipped stylesheet.

| Token | Value | Role |
|---|---|---|
| `--pink` | `#ff90e8` | **Accent.** Links on hover, focus, CTA in dark, newsletter |
| `--purple` | `#90a8ed` | Direct-sales panel (Pricing) |
| `--green` | `#23a094` | Discover-channel panel (Pricing) |
| `--orange` | `#ffc900` | Warm highlight |
| `--red` | `#dc341e` | The offset ghost layer, danger, discounts |
| `--yellow` | `#f1f333` | The offset ghost layer, announcement bar |
| `--gray` | `#f4f4f0` | **Page background** |
| `--dark-gray` | `#242423` | Dark-mode background |

### Two details most people miss

**The neutrals are warm, not gray.** `#f4f4f0` has a yellow cast; `#242423` is
olive-black, not neutral. Pure `#f5f5f5` / `#1a1a1a` would look clinical next to
brilliant pink. The warm base is what lets the brights sit on it without
vibrating.

**Pink is the only color that never changes between themes.** In the dark-mode
media query, ten variables flip — `--body-bg`, `--primary`, `--color`,
`--filled`, `--border-alpha`, and their contrast partners. `--accent: 255 144 232`
is defined in `:root` and **never overridden**. The accent is theme-invariant by
design, which is why it detonates against `#242423`.

### The RGB-triplet trick

Colors aren't stored as hex. They're stored as space-separated channels and
consumed with a slash:

```css
--accent: 255 144 232;                    /* raw channels */
--color-accent: rgb(var(--accent));       /* semantic token */
--color-muted: rgb(var(--color)/var(--gray-3));   /* same token, 50% alpha */
--color-active-bg: rgb(var(--color)/var(--gray-1)); /* 10% alpha */
```

This is why the whole theme is ~10 variables. Because the alpha is applied at
the *consumption* site, "muted" and "subtle" are free derivatives rather than
hand-picked hex values. Nothing to keep in sync. The CSS defines exactly three
gray steps — `.1`, `.2`, `.5` — and derives every neutral from them.

**Borrow this.** It's the single highest-leverage trick in their system.

### No gradients on the marketing surface

The Tailwind v4 gradient utilities exist in the bundle but are essentially unused
on these pages. There is exactly one: a `pointer-events-none` black fade at the
bottom of a scroll container
(`bg-gradient-to-t from-black`, `h-8`) so text doesn't collide with the edge.
Flatness is the rule; a gradient is a bug.

---

## 5. Type

### The typeface

```css
--font-family: ABC Favorit, Avenir, Montserrat, Corbel, URW Gothic,
               source-sans-pro, sans-serif;
```

**ABC Favorit** is self-hosted from `assets.gumroad.com/fonts/ABCFavorit-Regular.woff2`.
It's a geometric sans with a slightly retro, friendly character — wide apertures,
low stroke contrast. It reads "small-business optimistic" rather than "tech
startup."

The important architectural detail: **the font is a runtime, server-driven
setting, not a CSS constant.** The SSR shell ships:

```html
<div id="design-settings" data-settings='{"font":{"name":"ABC Favorit","url":"..."}}'></div>
```

Inertia passes the same object into a React `DesignSettingsProvider` context, and
components read it via `useDesignSettings().font`. **The typeface is themable
per-request.** That's how Gumroad supports custom storefront fonts on creator
pages without a separate design system. The fallbacks (`Avenir → Montserrat →
Corbel → URW Gothic`) are all geometric sans, so a font load failure degrades
gracefully instead of collapsing to Arial.

### The scale

`xs .75 · sm .875 · base 1 · lg 1.125 · xl 1.25 · 2xl 1.5 · 3xl 1.875 ·
4xl 2.25 · 5xl 3 · 6xl 3.75 · 7xl 4.5 · 8xl 6 · 9xl 8` (rem)

Line height is baked into the token and **tightens hard as size grows** — this is
what makes the display type look set rather than scaled:

| Size | Line height | What it's used for |
|---|---|---|
| `base` | 1.5 | body |
| `3xl` | 1.2 | sub-display |
| `5xl`–`9xl` | **1** | hero, big numbers |

The About hero:

```html
<h1 class="leading-none text-6xl md:text-7xl lg:text-8xl">Go from<br class="sm:hidden"> <span class="whitespace-nowrap">0 to $1</span></h1>
```

60px → 72px → 96px. Note the two typographic controls: an explicit `<br>` that's
**hidden on mobile**, and `whitespace-nowrap` to stop "0 to $1" breaking across
lines. They treat the line break as a design element and control it at every
breakpoint.

### Weight

`400 / 500 / 600 / 700`, but the interesting part is what's *missing*:

- Hero `h1` — no weight class at all (inherits 400). **Display type is light.**
- Nav, pricing numbers, FAQ questions — `font-medium` (500)
- Blog post titles — `text-[2.5rem] leading-[1.2] font-normal` (400)
- Card titles — `font-medium`

Huge type at weight 400 is the single biggest reason Gumroad doesn't feel
aggressive. The **color** and the **border** provide the punch; the type stays
quiet. If you set your hero to `font-bold` you've lost the plot — you'd get a
generic neobrutalist look instead of theirs.

### One nonstandard utility

`font-regular` appears in the bundle. It's a custom class, not a Tailwind
default — presumably an alias pinning weight 400. Worth noting because it shows
the team hits for a class name out of muscle memory rather than reasoning about
the value.

---

## 6. Space and shape

### Grid

Tailwind v4 with `--spacing: .25rem` (4px). Alongside it sits a legacy scale,
`--spacer-1` through `--spacer-8` = `4 / 8 / 12 / 16 / 24 / 32 / 48 / 64px`.
The overlap is deliberate migration debt; both work.

**Spacing steps up with viewport**, often in `vw`:

```
px-4 py-16 lg:ps-16 lg:pe-16      section padding
px-4 py-16 md:p-24 xl:p-32        pricing panels
py-6 sm:py-12 md:py-16           FAQ rows
lg:px-[8vw] md:py-56             About hero
```

`gap-[4vw] md:gap-[12vw]` on FAQ rows, with the answer indented to
`pl-[calc(3rem+4vw)] md:pl-[calc(3rem+12vw)]` — the `calc()` keeps the body text
aligned to the question while the icon-to-question gap stays fluid. That's
careful work.

Section rhythm is large and consistent: `gap-16`, `py-24`, `my-32`. The site
breathes.

### Radius

```
xs .125 · sm .25 · md .375 · lg .5 · xl .75 · 2xl 1 · 3xl 1.5rem
--border-radius-3: 10rem   /* pill */
```

The language is **binary**, and this matters more than the numbers:

- **`rounded-sm` (4px)** — cards, buttons, inputs, color-stack ghosts. Nearly
  everything.
- **`rounded-full`** — nav pills, avatars, tags. Only where the shape is
  semantically a pill or a dot.

They do not use `rounded-2xl` for cards on the marketing surface. Cards are
*almost* square, which keeps the black border reading as a printed edge. Soft
8–16px corners would turn the same system into Material.

### Border

`--border-width: .0625rem` = **1px**, set once on the universal selector:

```css
* {
  --border: solid .0625rem rgb(var(--parent-color)/var(--border-alpha));
  --outline: .125rem solid rgb(var(--accent));   /* 2px pink, focus */
}
```

`--parent-color` defaults to `var(--color)` but accepts `inherit`, which is how
a component inverts its own children. One variable, universal reach.

**Empty states use `border-2 border-dashed`.** Dashed is the only border-style
in the system and it's reserved exclusively for "nothing here" — a nice piece of
semantic encoding.

### Iconography

Two conventions, both at `currentColor`:

- **UI icons** — 24px grid, `stroke-width 2`, `fill="none"`, `stroke-linecap:
  round`, `stroke-linejoin: round`
- **Brand icons** — official SVGs at their real viewBox (`0 0 98 96` GitHub,
  `0 0 1200 1227` X), `fill="currentColor"`

Because everything inherits color, the logo inverts for free in dark mode — no
second logo asset, no `picture` element, no filter. The footer wordmark even has
a `dark:invert` utility defined for the rare raster case.

Sizing: `size-[1lh]` for inline icons (scales to the current line height — so
icons never need per-breakpoint sizes), plus a `--big-icon-size: 1.25em` for
larger contexts. `size-[1lh]` is a genuinely smart trick worth stealing.

---

## 7. Dark mode

`prefers-color-scheme` only. **There is no manual toggle.** The entire dark theme
is ten variable overrides inside one media query:

```css
@media (prefers-color-scheme: dark) {
  :root, :host {
    --body-bg: #242423;        /* was #f4f4f0 */
    --primary: 221 221 221;    /* was 0 0 0     */
    --contrast-primary: 0 0 0;
    --color: 221 221 221;      /* was 0 0 0     */
    --filled: 0 0 0;           /* was 255 255 255 */
    --neutral-color: 221 221 221;
    --neutral-filled: 0 0 0;
    --neutral-border-alpha: .35;
    --contrast-filled: 221 221 221;
    --contrast-black: 221 221 221;
    --border-alpha: .35;       /* was 1 */
  }
}
```

Everything else is untouched. Four things to notice:

1. **Borders get weaker, not different.** `--border-alpha: .35` instead of
   switching to a lighter color. One variable handles every border in the app —
   and a translucent white border over dark gray looks *better* than a solid
   mid-gray would.
2. **The value inverts, the hue doesn't.** `#f4f4f0 → #242423` is roughly
   luminance inversion with the warm cast preserved.
3. **`--accent` is not in the list.** Pink is theme-invariant by design.
4. **Flat color panels keep their fill and flip only the text** — Pricing's
   purple and green panels get `dark:text-black` and stay fully saturated.

---

## 8. Motion

Fast, transform-only, always reduced-motion aware.

| Property | Value |
|---|---|
| `--transition-duration` | `.14s` |
| `--default-transition-duration` | `.15s` |
| Hover | `duration-150` |
| Card group | `duration-150` |
| Nav | `duration-200` |
| Accordion | `duration-300` |
| Easing | `cubic-bezier(0,0,.2,1)` out · `cubic-bezier(.4,0,.2,1)` in-out |

Nothing exceeds 300ms. Anything that moves spatially uses `transform` only, so
it stays on the compositor.

**Accessibility is handled properly**, which is rarer than it should be:

```css
motion-safe:animate-[marquee-scroll_80s_linear_infinite]
motion-reduce:animate-none
hover:[animation-play-state:paused]
```

The tag marquee is an 80-second linear loop that pauses on hover and dies
completely under `prefers-reduced-motion`. Same discipline on the accordion's
`+`→`×` rotation (`transition-transform duration-300` on a single `<line>`), and
`forced-colors:` variants exist throughout (dashed borders for disabled controls,
which is exactly the right treatment for forced-colors mode).

The parallax coins on About/Pricing are a custom system —
`data-hero-parallax-container` on the parent, `data-parallax-intensity` and
`data-scroll-intensity` per element (values like `0.03` / `-0.20`). Pointer
speed and scroll speed are separate axes. Wrapped in `pointer-events-none` so
decoration can never eat a click.

---

## 9. Layout patterns, page by page

### Discover — the card grid

```
grid grid-cols-1 gap-8 md:grid-cols-2 lg:grid-cols-3 lg:gap-8
```

Each card is the offset-print stack (§3) with `bg-red` + `bg-yellow` ghosts,
`md:w-20` / `lg:w-auto` for the title block, `lg:border-l` dividers, and a
`rounded-full` avatar. A horizontally scrolling "collections" rail uses
`auto-cols-[min(20rem,60vw)] lg:auto-cols-[40rem]` with the scrollbar hidden via
`[scrollbar-width:none]` + `[&::-webkit-scrollbar]:hidden` — the thumb-size-on-
mobile / chunky-on-desktop trick.

The black hero band: `relative flex min-h-[72vh] w-full flex-col items-center
justify-center bg-black`. Full-bleed black, vertically centered, 72vh.

Announcement bar: `h-14 w-full overflow-hidden border-b border-black bg-yellow-400`
— yellow, hard black bottom border, 56px tall.

### Pricing — color as layout

No cards. Two full-bleed fields split the viewport, divided by a hard rule:

```html
<div class="flex flex-col border-b border-black dark:border-white/35
            overflow-hidden lg:flex-row">
  <div class="bg-purple ... lg:w-1/2 px-4 py-16 md:p-24 xl:p-32
              border-b lg:border-b-0 lg:border-r">
    <h3 class="font-medium text-5xl lg:text-6xl">10% + $0.50</h3>
  </div>
  <div class="bg-green ... lg:w-1/2 px-4 py-16 md:p-16 xl:p-32">30%</div>
</div>
```

The numbers are 48–60px at `font-medium`. The comparison is made by **hue and
position, not by UI chrome**. Note `space-y-12 md:space-y-8` — internal spacing
*decreases* as the panel grows, because a bigger box needs less help.

FAQ rows are `border-t border-black` (top rule only, no bottom) with `py-6
sm:py-12 md:py-16`, a 48px plus icon whose vertical line rotates to form an `×`,
and a fluid `gap-[4vw] md:gap-[12vw]`.

### Features — labeled sections

Three sections, each a color field + prose + a testimonial. Eyebrow labels are
"Customizable Options", "Payment Integrations", "Comprehensive Platform" —
category, not benefit. Headings are two words: "Your store, your way", "Money,
incoming", "From start to finesse".

Testimonials pair a monospace handle (`motionbynick`, `stephsmithio`,
`dvassallo`, `boringbotlab`) with the quote in quotes and an attribution line
that names the person *and what they sell*. Proof beats adjectives.

### About — proof as hero-adjacent

Hero, then a `0 → $1 → $10k+ → procreate brushes` progression, then four
creator stories, then a wall of ~40 keyword pills in `--color-accent`, then the
closing CTA. A live stat sits in the nav: **`9.7K`** with a star icon, linking
to GitHub. Real-time social proof in the header.

The keyword wall is the discovery strategy made visual — the actual tag cloud
from their index (`vrchat`, `notion template`, `procreate`, `3d model`…), so the
About page doubles as a sales page for the marketplace.

### Gumclaw — the shortest page on the site

```
Meet Gumclaw
Gumroad is run by an agent.
[one sentence]
How it works
  Runs support        / Handles operations
  Ships code
```

48 words of body copy. No illustration carousel, no feature grid, no
testimonials. Three `h3`s, three paragraphs. When the claim is strong enough,
the design gets out of the way — the starkest page is the most confident one,
and it's the newest.

---

## 10. Voice

Consistent across all six pages, and it's doing real work.

**Chapter-title openers.** "Go from 0 to $1." / "Built for new beginnings." /
"Money, incoming." / "Runs support." Short, declarative, present tense.

**The running manifesto.** *"Share your work. Someone out there needs it."*
appears on About and Features as the closing CTA. It's the brand thesis, and
they repeat it verbatim rather than paraphrasing.

**Second person, imperatives.** "Sell anything." "Make your own road." "Place
small bets." "Start selling." Never "you can" — just the verb.

**Rhetorical reversal as a device.** "Instead of building a company... start
selling a side project!" / "Don't take risks. That's scary!" / "Instead of
selling a book... start by selling a blog post!" The About page stacks two of
these. It reframes the reader's objection in their own framing, then answers it
in five words.

**Numbers do the persuading.** "$1,967,258 — the amount of income earned by
Gumroad digital entrepreneurs last week." Full precision, no rounding, no
abbreviation. A round number reads as marketing; `1,967,258` reads as a database
query. Paired with the nav's live `9.7K`, it establishes that the numbers on the
site are generated, not written.

**Plain, specific, occasionally profane.** "I like to move fast and break shit"
(Nick Greenawalt). "But I received... exactly $0" (trendsvc). The testimonials
are allowed to be people.

**Merchant-of-record explained without jargon.** "Since January 1, 2025, Gumroad
handles ALL your tax obligations. Yes, you read that right" — the "yes, you read
that right" is doing the work of a footnote.

**Scarcity is playful, not anxious.** The About page builds a "Gumroad Way"
manifesto — Start Small, Get Better Together, Learn Quickly — then: "We want you
to try them, lots of them, and find out what works. You don't have to be a tech
expert."

---

## 11. Where they break their own rules

Useful, because the inconsistencies reveal which rules are load-bearing and
which are accidents.

| Inconsistency | Detail |
|---|---|
| **Container padding inverts** | `px-8 … sm:px-6 lg:px-8` — mobile has *more* padding than tablet. Looks like an unnoticed regression. |
| **Two different pinks on hover** | Marketing pages use the brand `#ff90e8`; the blog title link uses `hover:text-pink-600` (`oklch(59.2% .249 .584`) — a much darker magenta. Different token, different feel. |
| **Display weight is inconsistent** | Hero/FAQ/pricing = `font-medium`; blog post titles = `font-normal`; blog index heading = `text-gray-600` (a *muted* heading). |
| **Two color systems coexist** | A full Tailwind v4 `@theme` in `oklch()` and a legacy `:root` block of raw hex + RGB triplets. The bridge is `rgb(var(--accent))`. Migration is incomplete but functional. |
| **One breakpoint in v4** | `@theme` declares only `--breakpoint-lg: 64rem`; the legacy block keeps `--breakpoint-sm: 640px` and `--breakpoint-lg: 1024px`. Same value, two homes. |
| **`--dynamic-grid` is a hack** | `min(100%, calc((31.25rem - 100%)*1000))` — collapses to 0 above 500px. Cute, unreadable. |
| **Spacer overlap** | Tailwind's `--spacing` and the legacy `--spacer-1…8` both in use. |

The through-line: **the aesthetic rules are enforced, the token plumbing is
migrating.** Nobody has to think about the visual language. Everybody occasionally
has to think about which pink.

---

## 12. Steal this — the minimum viable Gumroad kit

Everything above, reduced to what you actually need to get the look.

### 1. Tokens

```css
:root {
  color-scheme: light dark;

  /* raw channels — alpha is applied at consumption */
  --accent: 255 144 232;          /* #ff90e8 */
  --success: 35 160 148;
  --warning: 255 201 0;
  --danger: 220 52 30;
  --info: 144 168 237;
  --primary: 0 0 0;
  --filled: 255 255 255;
  --color: 0 0 0;

  /* semantic */
  --color-accent: rgb(var(--accent));
  --color-foreground: rgb(var(--color));
  --color-background: rgb(var(--filled));
  --color-muted: rgb(var(--color) / .5);
  --color-active-bg: rgb(var(--color) / .1);
  --color-border: rgb(var(--color) / var(--border-alpha, 1));

  /* shape */
  --border-width: .0625rem;
  --border: solid .0625rem rgb(var(--color) / var(--border-alpha, 1));
  --outline: .125rem solid rgb(var(--accent));
  --radius-sm: .25rem;

  /* shadows — zero blur, always */
  --shadow-1: .25rem .25rem 0 rgb(var(--color));
  --shadow-2: .5rem .5rem 0 rgb(var(--color));

  /* motion */
  --transition-duration: .14s;
  --ease-out: cubic-bezier(0, 0, .2, 1);
}

@media (prefers-color-scheme: dark) {
  :root {
    --primary: 221 221 221;
    --color: 221 221 221;
    --filled: 0 0 0;
    --body-bg: #242423;
    --border-alpha: .35;
  }
}

* {
  --border: solid .0625rem rgb(var(--color) / var(--border-alpha, 1));
  --outline: .125rem solid rgb(var(--accent));
  min-width: 0;
  max-width: 100%;
}

body {
  font-family: "ABC Favorit", Avenir, Montserrat, Corbel, sans-serif;
  font-size: 1rem;
  line-height: 1.375;
  background: #f4f4f0;
  color: rgb(var(--color));
}
```

### 2. The button

```html
<a class="btn">Start selling</a>
```

```css
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: .5rem;
  padding: .875rem 1rem;
  font: inherit;
  font-size: 1rem;
  line-height: 1.375;
  color: currentColor;
  text-decoration: none;
  cursor: pointer;
  background: transparent;
  border: var(--border);
  border-radius: .25rem;
  transition: transform .15s var(--ease-out);
}
.btn:hover  { transform: translate(-4px, -4px); box-shadow: 4px 4px 0 #000; }
.btn:active { transform: translate(0, 0);     box-shadow: none; }

.btn--primary { height: 3.5rem; padding-inline: 2rem; font-size: 1.25rem;
                background: #000; color: #fff; }
@media (prefers-color-scheme: dark) {
  .btn--primary { background: #ff90e8; color: #000; }
}
```

### 3. The offset-print CTA

```html
<div class="offset group">
  <span class="offset__ghost offset__ghost--b" aria-hidden="true"></span>
  <span class="offset__ghost offset__ghost--a" aria-hidden="true"></span>
  <a class="offset__face btn btn--primary" href="/signup">Start selling</a>
</div>
```

```css
.offset { position: relative; display: inline-block; }
.offset__ghost {
  position: absolute; inset: 0;
  border: 1px solid #000;
  border-radius: .25rem;
  transition: transform .15s var(--ease-out);
}
.offset__ghost--b { background: #f1f333; z-index: 2; }
.offset__ghost--a {
  background: #dc341e; z-index: 1;
  transition: transform .15s var(--ease-out);
}
.offset__face { position: relative; z-index: 3; }

.offset:hover .offset__ghost--a { transform: translate(8px, 8px); }
.offset:hover .offset__face     { transform: translate(-8px, -8px); }
```

### 4. Type

```html
<h1 class="display">Go from 0 to $1</h1>
```

```css
.display {
  font-weight: 400;            /* NOT bold — this is the whole trick */
  font-size: clamp(3rem, 8vw, 6rem);
  line-height: 1;
  letter-spacing: -.025em;
}
```

### 5. Rules to keep

- 1px black border on every solid element
- Zero blur on every shadow
- Shadow offset tied to hover travel
- `font-weight: 400` on display type
- Pink for interaction only
- Warm neutrals, not gray
- Flat fills, no gradients
- `rounded-sm` for cards, `rounded-full` only for pills
- `duration-150`, `transform` only
- `prefers-reduced-motion` on every animation

---

## Files

| File | Contents |
|---|---|
| `assets/gumroad-tokens.css` | Copy-paste token layer from this document |
| `assets/gumroad-components.css` | Button, offset-print, nav, card, accordion, field |
| `_research/design.css` | Full 152 KB stylesheet as served in production |
| `_research/*.html` | Raw SSR markup for all six pages |
| `_research/*.txt` | Extracted plain text |
| `_research/tokens-raw.txt` | All 288 CSS custom properties, every value |
| `_research/components.txt` | 64 extracted component class strings |
| `_research/inertia.js` | Compiled React base bundle (component source) |
| `_research/Index-DHcCfXIR.js` | Compiled Discover page |
| `_research/pages.txt` | All 112 Inertia page → chunk mappings |
