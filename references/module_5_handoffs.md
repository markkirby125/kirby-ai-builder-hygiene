# Module 5: Hand-offs

This skill cleans code. It does not judge visual design or rewrite copy. When [Module 4](./module_4_verification.md) or the user raises those issues, route them to the skills below and pass the list of findings. Do not do that work here. Run on its own (routed by symptom), this module edits nothing, so it skips Module 4 verification and just reports the hand-offs.

Project docs come first. Before routing, read the project's own design, brand and product docs and any locked wording (a style guide, `DESIGN.md`, a copy deck, rules in `AGENTS.md`). They override the generic checks in sections A and B: if the project allows em dashes or fixes a phrase, keep it. Check those docs against the live code for drift first; report any mismatch and do not use a stale doc as the brief.

## A. Visual design review

Route to the `impeccable` skill for layout, hierarchy and CTA review, to the `better-colors` skill for palette and token work, and to the `kirby-accessible-contrast` skill for contrast fixes (optional, from the Kirby skills collection; if it is not installed, list the contrast findings for the user).

Route accessibility findings that need more than this skill's landmark and heading minimums (for example `aria-label` on a generic element, full WCAG 2.2 AA landmark and heading review) to the `better-accessibility` skill.

Name these checks in the hand-off:
- A palette grounded in the brand, with WCAG AA contrast for text and controls.
- The value proposition leads the hero. No stack of pills or badges above the `<h1>`.
- Layouts chosen to fit the content (comparison table, step sequence, two-column contrast, spec list) instead of a default 3-card icon grid.
- Solid CTA buttons with clear hover and focus states. No glow shadows.
- One primary CTA above the fold.
- Reduced motion respected and focus always visible.
- Heading tags used only for font size (listed by Module 2): retag each one with a class only after a visual check.

## B. Copy edit

Route to the `no-ai-slop` skill to cut AI patterns from existing text, and to the `copy-editing` skill for a full marketing copy pass.

Name these checks in the hand-off:
- Cut filler vocabulary.
- Rewrite "it's not just X, it's Y" lines as one plain claim.
- Delete throat-clearing openers and generic closing headings.
- No em dashes and no engagement bait.
- Answer the reader's question in the first two sentences.
- Active subject-verb-object sentences, with varied length.
- Invent no facts. Mark each unsourced claim `[verify]`.

## C. Project rule for builder-export repos

A repo qualifies only if the detection grep in [Module 1](./module_1_builder_artifact_cleanup.md) (section 1.1), run from its root, prints at least one file. Module 1 holds the copy of record; do not keep a second version here.

Rule template (3 lines):

```markdown
1. No builder attributes, generator comments or generator meta tags in committed code.
2. No purpose-free single-child wrapper divs (see kirby-ai-builder-hygiene Module 2); strict heading tree (one h1, no skipped levels).
3. See the kirby-ai-builder-hygiene skill for the cleanup workflow.
```

Where it goes: only in a repo where the grep hits, as a file in that repo's own `.claude/rules/` folder (for example `.claude/rules/builder-hygiene.md`). Never install it globally. Never write it into any repo without the user's go-ahead: show the grep result and the file path, and wait for a yes. Keep the copy and visual lists out of the rule; they live in sections A and B.
