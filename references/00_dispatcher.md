# AI Builder Hygiene: Dispatcher

Cleans HTML, JSX and TSX exported from AI visual builders (Lovable, v0, Bolt, Cursor scaffolds). The goal is code hygiene, accessibility and maintainability: fewer tool artifacts, a real landmark and heading structure, and utility classes tied to design tokens. Rendering stays the same. Not for hand-built sites: if the detection grep (run order step D) finds nothing, the skill does not apply.

Out of scope: this skill does not hide AI authorship, does no scraper evasion or de-fingerprinting, does not randomize values, and does not remove analytics, tag manager or consent scripts.

## Locked decisions

These are fixed. Do not relax them.

1. Never randomize spacing or radius values. Consolidate repeated utility clusters into semantic classes tied to design tokens. Flag inconsistent values for the same role and propose one value per role.
2. Never remove analytics, tag manager or consent scripts. Remove only builder dev and debug tooling, for example the `lovable-tagger` dev plugin, builder preview or debug scripts and builder CDN debug hooks. List uncertain items instead of deleting them.
3. Flag builder badges and attribution ("Made with Lovable"). Never strip them silently: the builder's plan terms may require them.
4. `data-id`, `data-component-*`, `__v0*` and `__ai*` may be read by tests, CSS or JS. Grep the repo for each one before deleting it.
5. Module 2 can break layout (`> div` selectors, flex and grid context, heading anchors). It needs before and after screenshots and a build and test run that matches the baseline.
6. The output is a diff plus a verification report. Never hand back "cleaned code only". In analysis-only mode (see Modes) nothing is edited, so the output is the proposed-change list instead.
7. Run detection (step D) first, then Step 0, then modules 1, 2, 3, 4 in that order. Module 2 runs only with checkpoint commits (run order item 2). Module 5 routes design and copy work to other skills and documents the optional builder-hygiene project rule. It performs no cleanup.
8. Project rules versus this skill. A project's own rules (AGENTS.md, CLAUDE.md, CONTRIBUTING or similar) may require auto-commit, auto-deploy, a commit-message format, a review gate or a task file. Where they conflict with this skill, this skill's gates win: commit only with the approvals described here, and never push or deploy. Follow the project's commit-message format for checkpoint commits (keep the `m1` or `m2` tag in the message), and follow its other rules where they do not conflict. The report states that deploy is a separate step for the owner. Project-locked wording and design docs override the generic checks in [Module 5](module_5_handoffs.md).

## Modes

- **Full run (default):** step D, Step 0, then modules 1 to 4.
- **Analysis only (audit):** when the user asks for an audit, a review or a dry run, or does not approve edits. No baseline captures, no commits, no edits, no installs, and no script that writes files (`build`, `predeploy`, `deploy`) unless the user agrees. Run step D (it stops on zero hits as below), then every detection, inventory grep and check in modules 1 to 3, the generated-regions check below, and Module 4 section 1 toolchain detection (list the scripts only; no baseline worktree). The git preconditions (clean tree, checkpoint approval) do not apply, because nothing is written. Output: a list of proposed changes, each with file, line, the module rule behind it and a risk (low: no layout or behaviour effect; medium: a selector, anchor, test or generator hit; high: layout or behaviour may change), plus any Module 5 hand-offs.

## Inputs

- The repo or the files exported from the builder (HTML, JSX, TSX, CSS, Tailwind config).
- The project's build and test commands. If no build exists, use a stand-in from [Module 4](module_4_verification.md) section 1 (`check`, `validate` or a documented gate command). If there is neither, write `[needs: build command]` and ask before Module 1 starts.
- A way to take screenshots of the key pages (Playwright, a browser tool, or the user's own screenshots).
- In-scope routes: the routes the user names, or every user-facing route in the router when the user names none. Above 10 routes (30 captures per set), ask the user to name a subset, for example one page per template, and record it.

## Generated regions and asset pipelines

Before the first edit (in audit mode, list them only), find what the project generates. Read the `scripts` in `package.json` and any `scripts/` or `tools/` folder for names with `partial`, `inject`, `include`, `hash`, `fingerprint`, `sitemap`, `lastmod` or `--check`, and grep for generated-region markers:

```bash
grep -rIliE --exclude-dir={node_modules,.git,dist,build,.next,out,coverage,.wrangler,.vercel,.netlify} \
  -e '@generated' -e 'do not edit' -e 'BEGIN[ _-]?(GENERATED|PARTIAL|INCLUDE)' .
```

When they exist: edit the source partial, never an injected copy, then run the generator and its `--check`. After a CSS or JS edit, run the hash or fingerprint script and stage the rewritten references with the same checkpoint, by explicit path. Follow the project's sitemap `lastmod` rule. Add each `--check` script to the Module 4 checks.

## Preconditions

| Before | Required |
|---|---|
| Step 0 | Step D found builder artifacts, or the user insisted. A known build command or stand-in, a screenshot route (Module 4 section 2, routes A to C), and a Lighthouse route (or the accessibility baseline recorded as `not run`). |
| Module 1 | The Step 0 baseline recorded. `git rev-parse --verify HEAD` succeeds and the tree is clean (scoped to the package root, see Module 1 precondition 1). With no repo, no commit or a dirty tree, stop before any edit and ask the user; with their approval, follow Module 1 precondition 1 (an agreed `.gitignore` and a `baseline: untouched export` commit, or their changes committed separately as `wip: changes before builder cleanup`). In every case, before the first edit, record the pre-cleanup baseline sha (`git rev-parse --short HEAD`) as `<baseline-sha>`; Module 4 uses it. |
| Module 2 | Build and test commands known and their current result recorded; the Module 1 result matches its recorded baseline. A failure that exists before the edit is recorded as the baseline and not fixed here. Record `[no test script]` where the project defines no tests. The user approved checkpoint commits at Module 1 step 9, or makes each one by hand, and the reviewed Module 1 diff is committed as a local checkpoint. If they declined, skip Module 2. |
| Module 3 | Module 2 verified (build and tests match the baseline, screenshots match), or Module 2 skipped because checkpoint commits were declined, or a symptom-routed run (see Routing by symptom). |
| Module 4 | Modules 1 to 3 applied (Module 3 may have reported not applicable), or 1 and 3 when Module 2 was skipped, or the one symptom-routed module. |

If a precondition fails, stop and tell the user what is missing. Do not skip ahead.

## Run order

D. Detection, before anything else. Run the [Module 1](module_1_builder_artifact_cleanup.md) section 1.1 detection grep from the repo root. If it prints nothing, report `not applicable: no builder artifacts found` with the command you ran, and stop: no Step 0, no edits. Continue only if the user insists. Then Module 1 preconditions 1 and 2 still apply (clean tree, `<baseline-sha>` recorded, build and test baseline), but Module 1 has nothing to remove: its steps 6 and 7 run as report-only (step 6 sorts the scripts into keep or uncertain and removes nothing; step 7 records any badge it finds), and modules 2 and 3 run only on the user's request, after Step 0. Module 2 then asks for checkpoint approval itself (its precondition 2).
0. Step 0: baseline. Run the types and lint checks ([Module 4](module_4_verification.md) section 1) on the untouched code and record their results, because Module 4's baseline worktree has no `node_modules`. On the untouched code, capture full-page screenshots at 375, 768 and 1440 px for every in-scope route and run the Lighthouse accessibility baseline. Use the capture protocol in [Module 4](module_4_verification.md) section 2 (network idle, `document.fonts.ready`, reduced motion and `animation: none`, carousels and dates frozen, full page). Save the files outside the source tree and record the paths. Module 4 compares against this set. If no capture route works, ask the user for the 375, 768 and 1440 px captures of the untouched code and record them as user-supplied. If Lighthouse cannot run, record the accessibility baseline as `not run` and compare screenshots only.
1. [Module 1: Builder artifact cleanup](module_1_builder_artifact_cleanup.md). Remove builder attributes, generator comments and meta tags, and builder dev tooling. Once the user has reviewed the Module 1 diff, ask once whether they approve local checkpoint commits for the rest of the run. With a yes, commit the diff as its own local checkpoint (stage exactly the touched paths, `git commit -m "m1: builder artifact cleanup"`, record the short sha; the full steps are in Module 1 step 9) at the end of Module 1 or the start of Module 2, before the first Module 2 component.
2. [Module 2: Semantic DOM flattening](module_2_semantic_dom_flattening.md). Remove wrapper divs, add landmarks, fix the heading tree. Skipped if the user declines checkpoint commits; modules 3 and 4 then run on the Module 1 result.
3. [Module 3: Utility class consolidation](module_3_utility_class_consolidation.md). Extract repeated utility strings into semantic classes, flag inconsistent values.
4. [Module 4: Verification](module_4_verification.md). Build, tests, before and after screenshots, axe or Lighthouse accessibility check, diff and report.

## Routing by symptom

Use this table when the user names one problem instead of asking for the full run. Step D, Step 0 and the Module 1 git preconditions (clean tree, `<baseline-sha>`) still apply; the module order does not, so the routed module runs alone. A routed Module 2 still needs checkpoint approval: ask for it before its first component, as in Module 1 step 9. Module 4 then verifies the module that ran.

| Symptom | Module |
|---|---|
| `data-lovable-*`, `data-v0-*`, `data-bolt-*`, `data-source-line`, `__v0` or `__ai` attributes; "Generated by" comments that name a builder; generator `<meta>`; `lovable-tagger` in the Vite config | [module_1_builder_artifact_cleanup.md](module_1_builder_artifact_cleanup.md) |
| Div soup, single-child wrappers, no `<main>` or `<nav>`, several `<h1>` tags, skipped heading levels | [module_2_semantic_dom_flattening.md](module_2_semantic_dom_flattening.md) |
| Long shadcn or Tailwind class strings repeated across components, mixed radius or spacing values for the same role, repeated inline styles | [module_3_utility_class_consolidation.md](module_3_utility_class_consolidation.md) |
| "Did the cleanup break anything?", accessibility score, report needed | [module_4_verification.md](module_4_verification.md) |
| Generic look, weak contrast, stacked pills above the h1, template-sounding copy | [module_5_handoffs.md](module_5_handoffs.md) |

## Outputs

- A unified diff of every change.
- A verification report from module 4: build and test results, before and after screenshots, accessibility check results, removed categories, items left in place and why, inconsistent values with the proposed fix, and a line saying deploy is a separate step for the owner.

## Hand-off rule

This skill fixes code structure. It does not redesign or rewrite. Send visual design issues and copy issues to [module_5_handoffs.md](module_5_handoffs.md), which routes them to other skills. Do not fix them inside modules 1 to 3.

## Related

- Contrast checks: [kirby-accessible-contrast](../../kirby-accessible-contrast/SKILL.md).
