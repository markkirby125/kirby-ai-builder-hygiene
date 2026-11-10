# Module 3: Utility Class Consolidation

Replaces long utility strings that repeat across a builder export with a small set of semantic classes tied to the project's design tokens. Rendering stays identical. Runs after [Module 2](./module_2_semantic_dom_flattening.md) and before [Module 4](./module_4_verification.md).

## Purpose

Builder exports paste the same 6 to 10 Tailwind utilities onto every card, caption and button. A later design change then means editing 40 copies by hand, and the copies drift (`rounded-lg` here, `rounded-xl` there). This module names each repeated cluster once, by its role, so one edit changes every instance.

## Preconditions

In analysis-only mode (dispatcher, Modes), skip the module-order and Step 0 preconditions: run the utility-system check and the step 1 counts, and list proposed classes and inconsistent values without editing.

- A utility system exists. Check for a `tailwindcss` or `unocss` dependency in `package.json`, a `tailwind.config.*` or `uno.config.*` file, `@tailwind` or `@import "tailwindcss"` in the CSS, a Tailwind CDN `<script>` tag, or `@apply` in the CSS. If none exists, the utility steps do not apply: report `Module 3: not applicable (no utility system)` and stop, unless the user asks for the plain-CSS fallback below.
- In a full run, modules 1 and 2 are done and their diffs are reviewed. If Module 2 was skipped because checkpoint commits were declined, work on the Module 1 result. In a symptom-routed run (dispatcher, Routing by symptom), this module runs alone after step D and Step 0, with the Module 1 git preconditions; modules 1 and 2 are not required.
- The Step 0 baseline from the [dispatcher](./00_dispatcher.md) exists (screenshots and Lighthouse accessibility results taken on the untouched code). [Module 4](./module_4_verification.md) compares against it.
- You know where the project defines tokens: `tailwind.config.*` (`theme.extend`), or CSS variables in the main CSS file (`globals.css`, `index.css`, `app.css`, or a Tailwind v4 `@theme` block).
- You know the project's component layer: an existing `@layer components` block, or existing React components such as shadcn's `Card` and `Button` in `components/ui/`. Plain-CSS fallback: on a site with a hand-written stylesheet, the component layer is that stylesheet, and the project's own custom properties take the place of token utilities. Run the procedure below unchanged: count repeated class lists and inline styles (step 1, same 3-use threshold), group by role, and move a repeated inline-style cluster into one role-named class in that stylesheet.
- You know whether the stylesheet is generated, injected or fingerprinted (dispatcher, "Generated regions and asset pipelines"). After editing a fingerprinted or versioned stylesheet (a hash in the file name or a `?v=` query), run the project's hash script and stage the rewritten references with the same checkpoint.

## Procedure

1. **Count repeated strings.** Run from the repo root:
   ```bash
   grep -rhoE --include='*.tsx' --include='*.jsx' --include='*.html' \
     -e 'className="[^"]+"' -e 'class="[^"]+"' --exclude-dir={node_modules,.git,dist,build,.next,out,coverage,.wrangler,.vercel,.netlify} . \
     | sort | uniq -c | sort -rn | awk '$1 >= 3'
   ```
   This misses strings inside `cn(...)`, `clsx(...)` and template literals. Grep those calls by hand and add them to the count. Count repeated inline styles the same way:
   ```bash
   grep -rhoE --include='*.tsx' --include='*.jsx' --include='*.html' \
     -e 'style="[^"]+"' -e 'style=\{\{[^}]+\}\}' --exclude-dir={node_modules,.git,dist,build,.next,out,coverage,.wrangler,.vercel,.netlify} . \
     | sort | uniq -c | sort -rn | awk '$1 >= 3'
   ```
   The second pattern catches one-line JSX style objects (`style={{ padding: 16 }}`); style objects that span lines or hold nested braces, and style constants passed by name, need a hand grep. Save both outputs as the "before" count.
2. **Group by role.** For each string used 3 or more times, write down what the element does: a content card, a meta caption, a primary button. Two strings with the same role belong to one group even if their values differ.
3. **Prefer what exists.** If the project already has a component for the role (for example shadcn `<Card>`), replace the repeated markup with that component. Create a CSS class only when no component fits. Never swap markup without a screenshot check of that component: render one instance before and after and compare. If the component adds layout classes (`flex`, `flex-col`, `gap`, `py`), pass the project's overrides (`!p-0` in Tailwind v3, `p-0!` in v4 where the leading `!` form is deprecated, or `gap-0` when `cn` uses tailwind-merge) or create the semantic class instead.
4. **Extract.** Add one class per role in the component layer, named by role, not by look: `.app-card`, `.meta-caption`, `.btn-primary`. Never `.rounded-box` or `.grey-text`.
5. **Use tokens.** Inside each class, use the project's token utilities (`bg-card`, `text-muted-foreground`, `p-6` from the spacing scale). Do not add arbitrary values such as `p-[23px]` or raw hex colors. If the project has no spacing scale or tokens, record `tokens: none` and ask the user; do not invent a scale.
6. **Flag inconsistent values.** Where one role uses different values (`rounded-lg` and `rounded-xl` on cards), list each value with its count and file locations. Propose ONE value per role, normally the most frequent. Apply it only after the user approves, because it changes how the minority renders. Until then, keep the minority value as an extra utility next to the semantic class.
7. **Remove no-op utilities.** Delete exact duplicates (`p-4 ... p-4`). For conflicts where two utilities set the same property (`flex` and `grid`, `p-4` and `p-6`), the winner is decided by the generated stylesheet order, not by the order in the class string. Read the element's computed value (dev tools, or a build) and delete only the utility that does not apply. Treat a default utility (`block` on a `div`, `flex-row` on `flex`) as removable only when nothing else in the cascade sets that property on the element. If you cannot confirm it, keep both and list it under "left in place". `w-full` on a block element is usually a no-op, but check its parent first: in a flex or grid parent it is not. Leave anything you are unsure about and list it.
8. **Recount.** Run the step 1 commands again and save them as the "after" count.

## Prompt block

```text
Consolidate repeated utility classes in the files below. Rendering must not change.
1. List every class string used 3 or more times, with its count.
2. Group strings by the role of the element (card, caption, button), not by look.
3. Where the project already has a component for a role, use it, but screenshot one
   instance before and after. If the component adds layout classes (flex, gap, py),
   pass overrides or use a semantic class instead. Otherwise add one
   semantic class per role in the project's component layer (@layer components with
   @apply), named by role: .app-card, .meta-caption, .btn-primary.
4. Use only the project's spacing, radius and color tokens. No arbitrary values.
5. Where one role uses different values, list each value with its count and propose
   ONE value. Do not apply it; keep the minority value as an extra utility for now.
6. Remove duplicate utilities and the losing side of a conflict, but only after the
   computed value confirms which one applies. Never assume the last class in the
   string wins. Remove a default-only utility only when nothing else sets that
   property. List any you are unsure of.
7. Never randomize, vary or jitter any value.
Output: a diff, the extracted CSS, before/after counts, and the inconsistent-value table.
```

## Worked example

Before: the shadcn card string appears 14 times across 6 files, in two variants.

```tsx
<div className="bg-card text-card-foreground rounded-xl border shadow-sm p-6">  {/* 11x */}
<div className="bg-card text-card-foreground rounded-lg border shadow-sm p-6 w-full">  {/* 3x, block parent */}
```

After: the project has `components/ui/card.tsx`, so markup uses it. Its `Card` adds `flex flex-col gap-6 py-6`, so the swap passes `gap-0` (this assumes the project's `cn` helper uses tailwind-merge, which drops `gap-6` and `py-6` in favour of `gap-0` and `p-6`; without it, class-string order decides nothing, so read the computed values) and one card was screenshotted before and after. Pages without React use the class.

```tsx
<Card className="gap-0 p-6">                 {/* 11x */}
<Card className="gap-0 p-6 rounded-lg">      {/* 3x, kept until the user approves rounded-xl */}
```

```css
@layer components {
  .app-card { @apply bg-card text-card-foreground rounded-xl border shadow-sm p-6; }
}
```

Flag: card radius uses `rounded-xl` (11) and `rounded-lg` (3). Proposed: `rounded-xl` for every card. Removed: `w-full` (3x, block element in a block parent).

## Do

- Name classes by role and keep the count of new classes small.
- Reuse existing components and tokens before adding anything.
- Keep the inconsistent-value table for the user to decide.

## Do not

- Never randomize, vary or jitter spacing, radius or any other value. Consistency is the goal.
- Do not change a value for a role without the user's approval.
- Do not extract strings used fewer than 3 times.
- Do not rename or remove classes that tests, CSS or JS select on. Grep for each one first.
- Do not add a new CSS framework, plugin or dependency.

## Output template

````markdown
### Module 3: Utility class consolidation
**Repeated strings (3+ uses):** before <n> strings / <n> total uses; after <n> / <n>
**Extracted:**
| Class or component | Role | Replaced uses | Files |
|---|---|---|---|
| .app-card / <Card> | content card | 14 | 6 |

**Inconsistent values (proposal, not applied):**
| Role | Values found (count) | Proposed |
|---|---|---|
| card radius | rounded-xl (11), rounded-lg (3) | rounded-xl |

**No-op utilities removed:** <utility, count, reason>
**Left in place (unsure):** <item, reason>
**Diff:** <path or inline diff>
````
