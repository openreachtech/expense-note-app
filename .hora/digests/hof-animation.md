# hof-animation
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-animation/ -->

**Read the source above whenever this leaves a question open.**

Motion serves the interface, never decorates it. Decide whether the element should animate at all
and why; then pick the technique. Every animation is built from design tokens and `.unit-*` scoped
styles (the `hof-css` rules apply), and **animates only `transform` and `opacity`** so the
compositor does the work.

## Deciding whether to animate at all

Every animation must answer "why does this animate?" **If the only answer is "it looks cool" and
the user will see it often, don't animate.**

Valid purposes — the complete list:

- **Spatial consistency** — an element enters and exits from the same direction.
- **State indication** — a control morphs to show its state changed (loading → done, collapsed → expanded).
- **Feedback** — a button scales down on press, confirming the interface heard the tap.
- **Preventing jarring changes** — content appearing/disappearing without a transition feels broken; a short fade smooths it.
- **Explanation** — a marketing or onboarding animation that demonstrates how a feature works.

**Match motion to frequency.** The more often a user sees an animation, the less it should draw
attention; a repeated action feels *slower* when animated, because the delay compounds.

| How often the user sees it | Decision |
| --- | --- |
| Keyboard-initiated / many times a day (menu toggles, shortcuts) | No animation |
| Frequent (hover, list navigation) | Remove or drastically reduce |
| Occasional (modals, drawers, toasts) | Standard animation |
| Rare / first-run (onboarding, celebrations) | Can add delight |

**Never animate keyboard-initiated actions** — the motion detaches the result from the keystroke
that caused it.

## Easing tokens

Always transition through an easing token — **never the bare CSS keywords (`ease`, `linear`, …)
and never a hard-coded curve.** The tokens are:

```css
--transition-timing-ease-in: cubic-bezier(0.4, 0, 1, 1);
--transition-timing-ease-out: cubic-bezier(0, 0, 0.2, 1);
--transition-timing-ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
```

| The element is… | Use |
| --- | --- |
| Entering or exiting (appearing/disappearing) | `--transition-timing-ease-out` |
| Moving or morphing on screen (already visible) | `--transition-timing-ease-in-out` |
| Changing color on hover/focus | `--transition-timing-ease-out` (short) |
| In constant motion (progress bar, marquee) | `linear` |

**Never use `ease-in` for entrances or exits** — it starts slow, which makes the interface feel
sluggish at the moment the user is watching most closely. Reserve `--transition-timing-ease-in`
for elements leaving toward an off-screen destination, if at all.

If a standard curve feels too weak, **add a stronger named token to `variables.css`** rather than
inlining a `cubic-bezier` in the component.

> **These three tokens do not exist yet in this project.** The skill sources them from
> `assets/css/variables.css`, which here is an empty `:root {}`. `@openreachtech/furo-vue` does
> **not** supply them either — its `lib/assets/css/furo/0035.variables-semantic-dimension.css`
> declares only `--transition-timer: 0.2s` (a duration, under a `/* Motion */` comment). Declaring
> a `--transition-timing-*` token is therefore a change to the shared `variables.css`, not a local
> one. Verified 2026-09-11.

## Durations

**The skill states no duration rule and permits no specific set of values.** These are the
durations its own examples use, and the only guidance available:

| Where | Value |
| --- | --- |
| Tooltip bubble | `0.125s` |
| Popover/menu unfold; button press feedback | `0.16s` |
| Entry scale; dropdown panel | `0.18s` |
| Crossfade with blur masking | `0.2s` |
| Subsequent tooltip in a group ("instant" mode) | `transition-duration: 0s` |

`0.3s` appears only in a **Bad** example (alongside `all` and `ease-in`). Furo's own
`--transition-timer: 0.2s` is the library's duration token; the skill never mentions it.

## Techniques

| Technique | Rule |
| --- | --- |
| Entry scale | **Never animate from `scale(0)`** — start from `scale(0.95)` (or higher) combined with `opacity`, so the entrance has a visible shape the whole time. |
| Popover origin | A popover/dropdown/select panel/menu scales **from its trigger**: set `transform-origin` to the corner nearest the trigger, e.g. `top left`. Prefer a positioner's exposed variable — `transform-origin: var(--popper-transform-origin, top left)` — so the origin stays correct when the panel flips. **Exception — modals** and full-screen overlays keep `transform-origin: center`. |
| Tooltip instancing | First tooltip pays a delay + fade; while any tooltip in the group is open, neighbours open **instantly, no delay and no animation** (`.unit-tooltip.is-instant > .bubble { transition-duration: 0s }`). The initial delay is timing, not a transition, so it lives in the controller (a `*Clerk`/`*Detector` module or the component context), not the CSS. |
| Blur masking | When a crossfade looks off no matter the easing or duration, add a subtle `filter: blur()` during the transition so the eye perceives one transformation instead of two objects swapping. **Keep the blur under `20px`, typically just `1–2px`** — heavy blur is expensive to paint, especially in Safari. |

Smallest complete form (entry scale — the shape all four share):

```css
.unit-card > .badge {
  transform: scale(0.95);
  opacity: 0;
  transition:
    transform 0.18s var(--transition-timing-ease-out),
    opacity 0.18s var(--transition-timing-ease-out);
}

.unit-card > .badge.is-shown {
  transform: scale(1);
  opacity: 1;
}
```

Never `transition: all` (Bad example, verbatim: `transition: all 0.3s ease-in;` — `all` + ease-in +
bare keyword). If a `transform` scale causes a 1px sub-pixel jump, promote the element with
`will-change: transform`.

## Reduced motion — the skill is SILENT

**`prefers-reduced-motion` appears nowhere in this skill** — not in `SKILL.md`, not in any of the
six references. Grep-verified across all 239 lines. The skill offers no reduced-motion guidance, no
media-query form, and no token.

This is a gap, not a permission. The project has committed to **WCAG 2.2 AA**
(`expense-note-frontend-staff/ai/contexts/uiux-context.md`), and checkpoint 18 audits against it.
Any motion this skill leads you to add is unguarded by it. Source the reduced-motion requirement
from the WCAG commitment and the uiux context, not from here.

## Applicability to #sign-in (checkpoint 15)

**On this skill's own guidance, this screen should author no animation CSS.**

- The one animated moment — the submit button's pending spinner while `signIn` is in flight — is
  **FuroButton's own**, already implemented in the library: `FuroButton.vue` declares
  `animation: furo-button-spin var(--transition-timer) linear infinite` plus
  `transition-property: color, background-color, border-color, box-shadow` with
  `transition-duration: var(--transition-timer)` for its state colours. Nothing to add; a competing
  transition would fight it.
- A sign-in form is a **frequent, keyboard-initiated** interaction (submit on Enter). Both of the
  skill's own rules — the frequency table's "No animation" row and "Never animate keyboard-initiated
  actions" — point at not animating.
- The entry-scale, popover-origin, tooltip-instancing and blur-masking techniques address surfaces
  this screen does not have.

If a later moment genuinely needs motion, it is most likely "preventing jarring changes" (a short
fade on an error message appearing) — that is the one valid purpose this screen can plausibly claim,
and it still needs a reduced-motion guard the skill does not supply.

## Conflicts with `D:\ORT\rules\05-frontend.md` (always-on — the rule WINS)

Named, not resolved.

| Conflict | Skill | Rule 05-frontend |
| --- | --- | --- |
| **`transition:` shorthand** | Every example uses the shorthand (`transition: transform 0.18s var(--transition-timing-ease-out), opacity …`) | §6-4 bans CSS shorthand outright, citing the implicit reset of values you did not write. The individual-property form is required: `transition-property`, `transition-duration`, `transition-timing-function`. **Precedent:** FuroButton itself writes the individual form. |
| **`px` units** | `filter: blur(2px)`; "under `20px`, typically just `1–2px`"; "1px sub-pixel jump" | §6-6 requires `rem`, never `px`. Note §6-6 permits decimals for dimensions smaller than font-size (border-radius, padding, icons), which is where a blur radius sits. |
| **Colour literals** | Not violated — the skill animates only `transform`/`opacity` | §6-2 — no conflict; recorded so the implementer need not re-check. |
| **`.unit-*` roots and child combinators** | Skill uses `.unit-button > .content`, `.unit-menu > .popper > .content` | §6-7 / §6-8 — **agrees**. No conflict. |
