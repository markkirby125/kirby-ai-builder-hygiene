# AI Builder Hygiene: Dispatcher

Cleans HTML, JSX and TSX exported from AI visual builders (Lovable, v0, Bolt, Cursor scaffolds). The goal is code hygiene, accessibility and maintainability: fewer tool artifacts, a real landmark and heading structure, and utility classes tied to design tokens. Rendering stays the same.

Out of scope: this skill does not hide AI authorship, does no scraper evasion or de-fingerprinting, does not randomize values, and does not remove analytics, tag manager or consent scripts.

## Locked decisions

These are fixed. Do not relax them.

1. Never randomize spacing or radius values. Consolidate repeated utility clusters into semantic classes tied to design tokens. Flag inconsistent values for the same role and propose one value per role.
2. Never remove analytics, tag manager or consent scripts. Remove only builder dev and debug tooling, for example the `lovable-tagger` dev plugin, builder preview or debug scripts and builder CDN debug hooks. List uncertain items instead of deleting them.
3. Flag builder badges and attribution ("Made with Lovable"). Never strip them silently: the builder's plan terms may require them.
4. `data-id` and `data-component-*` may be read by tests, CSS or JS. Grep the repo for each one before deleting it.
5. Module 2 can break layout (`> div` selectors, flex and grid context, heading anchors). It needs before and after screenshots and a build and test run that matches the baseline.
6. The output is a diff plus a verification report. Never hand back "cleaned code only".
7. Run Step 0, then modules 1, 2, 3, 4 in that order. Module 2 runs only with checkpoint commits (run order item 2). Module 5 routes design and copy work to other skills and documents the optional builder-hygiene project rule. It performs no cleanup.

## Inputs

- The repo or the files exported from the builder (HTML, JSX, TSX, CSS, Tailwind config).
- The project's build and test commands. If none exist, write `[needs: build command]` and ask before Module 1 starts.
- A way to take screenshots of the key pages (Playwright, a browser tool, or the user's own screenshots).
- In-scope routes: the routes the user names, or every user-facing route in the router when the user names none.

## Preconditions

| Before | Required |
|---|---|
| Step 0 | A known build command, a screenshot route (Module 4 section 2, routes A to C), and a Lighthouse route. |
| Module 1 | The Step 0 baseline recorded. `git rev-parse --verify HEAD` succeeds and the tree is clean. With no repo, no commit or a dirty tree, stop before any edit and ask the user; with their approval, follow Module 1 precondition 1 (an agreed `.gitignore` and a `baseline: untouched export` commit, or their changes committed separately as `wip: changes before builder cleanup`). In every case, before the first edit, record the pre-cleanup baseline sha (`git rev-parse --short HEAD`) as `<baseline-sha>`; Module 4 uses it. |
| Module 2 | Build and test commands known and their current result recorded; the Module 1 result matches its recorded baseline. A failure that exists before the edit is recorded as the baseline and not fixed here. Record `[no test script]` where the project defines no tests. The user approved checkpoint commits at Module 1 step 9, or makes each one by hand, and the reviewed Module 1 diff is committed as a local checkpoint. If they declined, skip Module 2. |
| Module 3 | Module 2 verified (build and tests match the baseline, screenshots match), or Module 2 skipped because checkpoint commits were declined. |
| Module 4 | Modules 1 to 3 applied, or 1 and 3 when Module 2 was skipped. |

If a precondition fails, stop and tell the user what is missing. Do not skip ahead.

## Run order

0. Step 0: baseline. On the untouched code, capture full-page screenshots at 375, 768 and 1440 px for every in-scope route and run the Lighthouse accessibility baseline. Use the capture protocol in [Module 4](module_4_verification.md) section 2 (network idle, `document.fonts.ready`, reduced motion and `animation: none`, carousels and dates frozen, full page). Save the files outside the source tree and record the paths. Module 4 compares against this set. If no capture route works, ask the user for the 375, 768 and 1440 px captures of the untouched code and record them as user-supplied. If Lighthouse cannot run, record the accessibility baseline as `not run` and compare screenshots only.
1. [Module 1: Builder artifact cleanup](module_1_builder_artifact_cleanup.md). Remove builder attributes, generator comments and meta tags, and builder dev tooling. Once the user has reviewed the Module 1 diff, ask once whether they approve local checkpoint commits for the rest of the run. With a yes, commit the diff as its own local checkpoint (stage exactly the touched paths, `git commit -m "m1: builder artifact cleanup"`, record the short sha; the full steps are in Module 1 step 9) at the end of Module 1 or the start of Module 2, before the first Module 2 component.
2. [Module 2: Semantic DOM flattening](module_2_semantic_dom_flattening.md). Remove wrapper divs, add landmarks, fix the heading tree. Skipped if the user declines checkpoint commits; modules 3 and 4 then run on the Module 1 result.
3. [Module 3: Utility class consolidation](module_3_utility_class_consolidation.md). Extract repeated utility strings into semantic classes, flag inconsistent values.
4. [Module 4: Verification](module_4_verification.md). Build, tests, before and after screenshots, axe or Lighthouse accessibility check, diff and report.

## Routing by symptom

Use this table when the user names one problem instead of asking for the full run. The preconditions above still apply.

| Symptom | Module |
|---|---|
| `data-lovable-*`, `data-v0-*`, `data-source-line`, `__v0` attributes; "Generated by" comments; generator `<meta>`; `lovable-tagger` in the Vite config | [module_1_builder_artifact_cleanup.md](module_1_builder_artifact_cleanup.md) |
| Div soup, single-child wrappers, no `<main>` or `<nav>`, several `<h1>` tags, skipped heading levels | [module_2_semantic_dom_flattening.md](module_2_semantic_dom_flattening.md) |
| Long shadcn or Tailwind class strings repeated across components, mixed radius or spacing values for the same role | [module_3_utility_class_consolidation.md](module_3_utility_class_consolidation.md) |
| "Did the cleanup break anything?", accessibility score, report needed | [module_4_verification.md](module_4_verification.md) |
| Generic look, weak contrast, stacked pills above the h1, template-sounding copy | [module_5_handoffs.md](module_5_handoffs.md) |

## Outputs

- A unified diff of every change.
- A verification report from module 4: build and test results, before and after screenshots, accessibility check results, removed categories, items left in place and why, inconsistent values with the proposed fix.

## Hand-off rule

This skill fixes code structure. It does not redesign or rewrite. Send visual design issues and copy issues to [module_5_handoffs.md](module_5_handoffs.md), which routes them to other skills. Do not fix them inside modules 1 to 3.

## Related

- Contrast checks: [kirby-accessible-contrast](../../kirby-accessible-contrast/SKILL.md).
