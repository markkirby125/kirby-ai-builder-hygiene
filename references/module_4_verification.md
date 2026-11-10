# Module 4: Verification

Proves modules 1 to 3 changed the code and not the page. Runs last. The skill's output is the diff plus the report below, never cleaned code on its own. The before captures for steps 2 and 3 are the dispatcher's Step 0 baseline, taken on untouched code before Module 1. This module takes the after set and compares it to that baseline.

## 1. Detect the toolchain

In analysis-only mode (dispatcher, Modes), run only the detection and script listing below: no scripts, no baseline worktree, no sections 2 to 4.

Pick the package manager from the lockfile: `pnpm-lock.yaml` means pnpm, `bun.lock` or `bun.lockb` means bun, `yarn.lock` means yarn, `package-lock.json` means npm. List the scripts:

```bash
npm pkg get scripts    # works in any Node project, whatever the manager
```

Run only the scripts that exist and that the user confirmed in the dispatcher's Inputs step (ask before any script not on that list), with the detected manager (`pnpm run build`, `bun run build`, `npm run build`):

| Check | Script names to look for | If missing |
|---|---|---|
| Build | `build` | Use a stand-in: `check`, `validate` or a gate command the project documents. If there is none, stop and ask the user how the project builds. Never use `predeploy`, `deploy` or any script that publishes or writes deploy output as the build |
| Types | `typecheck`, `type-check`, `tsc` | Only if `tsconfig.json` exists, run the local binary with any package manager: `./node_modules/.bin/tsc --noEmit`. Never use a package-manager exec, `npx` or `bun x`, which can fetch from npm. If it fails to start, report "types: not run (no local typescript)" |
| Lint | `lint` | Skip and note it |
| Tests | `test`, `test:unit`, `test:e2e` | Skip and note it. Say the change has no test cover |
| Generator checks | each `--check` script found in the dispatcher's "Generated regions and asset pipelines" step | Skip; none exist |

Watch-mode test runners never exit in a terminal. Run them once: pass `--run` (Vitest, for example `npm test -- --run`) or set `CI=true` (Jest).

Judge every check against the baseline recorded before Module 1: build and tests from Module 1 precondition 2, screenshots and Lighthouse from the dispatcher's Step 0. Types and lint baselines come from Step 0. Only if Step 0 did not record them, run them on the baseline commit in a separate worktree. Tell the user first. Run `git cat-file -e <baseline-sha>^{commit}`; if the sha does not resolve (the history was squashed or rewritten), record `before: not recorded` and skip the worktree. Otherwise create it with `git worktree add --detach "${TMPDIR:-/tmp}/kirby-baseline-<baseline-sha>" <baseline-sha>`, which refuses a folder that already exists and is not empty (if it does, stop and tell the user). Run the checks there (the worktree has no `node_modules`; if a check cannot start, record `before: not recorded`), then remove it with `git worktree remove "${TMPDIR:-/tmp}/kirby-baseline-<baseline-sha>"` (no `--force`). Write the full path in every command: shell variables do not persist between commands. If the removal fails, run `git worktree prune` and tell the user which folder is left over; do not delete it yourself. Here `<baseline-sha>` is the pre-cleanup baseline recorded in Module 1 precondition 1. It is not the `m1` or an `m2` checkpoint, which already contain cleanup edits. Never revert or stash the working tree to get one. If that is not possible, record `before: not recorded`. A check that already failed in the baseline is a baseline failure: record it, do not fix it here, and judge the after run against it.

## 2. Before and after screenshots

Serve the production build with the project's `preview` script; use `dev` only if no preview exists. For a Worker with static assets (a `wrangler.toml` or `wrangler.json`/`wrangler.jsonc` with an `assets` setting), or any site whose routes need its server, use the project's `dev` script, not a plain static file server. Capture the same pages at 375, 768 and 1440 px before and after. Before each capture, wait for network idle and `document.fonts.ready`, force reduced motion and `animation: none`, and freeze carousels and date output.

Save screenshots outside the repo. Create the folder once with `mkdir -p "${TMPDIR:-/tmp}/kirby-shots"`, then use `${TMPDIR:-/tmp}/kirby-shots/<page>-<width>-before.png` and `-after.png`. The top-level `-before` files belong to Step 0 and are read-only for all later modules. Module 2 keeps its own local set in `${TMPDIR:-/tmp}/kirby-shots/m2/`. No screenshots go in the repo or in the diff.

- **Route A (default): chrome-devtools MCP.** `new_page` or `navigate_page` to each URL, `resize_page` to 375, 768 and 1440 px wide, then `take_screenshot` with full page on.
- **Route B: Playwright, only if the project already lists `@playwright/test`, `playwright` or `playwright-core` in `package.json`.** Run the local binary only: `npx --no-install playwright screenshot --full-page --viewport-size=1440,900 <url> "${TMPDIR:-/tmp}/kirby-shots/<page>-1440-after.png"` (Step 0 writes `-before.png`; later modules never do) (or `pnpm exec playwright ...`; with only `playwright-core`, the binary is `playwright-core`), and the same for each width. If the browser binary is missing, do not install it: print the failure and use Route C. The bare CLI cannot wait for fonts, force reduced motion or freeze carousels and dates. If the project has its own Playwright script or config that applies the capture protocol, use it. Otherwise record "Route B: capture protocol not applied" in the report, capture only the after set with it (never rewrite the Step 0 `-before` files; if Step 0 used another route, record the route mismatch), and judge each diff by hand before reverting: a difference from animation, a carousel or a date is not a shift.
- **Route C: the user's own screenshots.** Ask the user for screenshots at the three widths and record where they came from.

Do not install a browser tool to run this module. If no route works, say so in the report and mark the screenshot row as not run.

Compare each pair. If ImageMagick is installed, `compare -metric AE "${TMPDIR:-/tmp}/kirby-shots/<page>-<width>-before.png" "${TMPDIR:-/tmp}/kirby-shots/<page>-<width>-after.png" "${TMPDIR:-/tmp}/kirby-shots/<page>-<width>-diff.png"` prints the count of changed pixels. A small nonzero count is normal from antialiasing and image decoding: open the diff image and review it by hand. Without ImageMagick, view each pair side by side. Any visible shift in layout, spacing or content is a failure: find the change that caused it and revert that change as in "Reverting a change" below.

Also check by hand what a screenshot cannot show: tab through the page once, and open any menus, dialogs and accordions.

## 3. Accessibility check

- **Lighthouse (default):** run the chrome-devtools `lighthouse_audit` tool on each page and read the accessibility section of the report. The before results come from Step 0.
- **axe:** only if the project already has `@axe-core/playwright`, `jest-axe` or `vitest-axe`. Run its existing test.

Reading results: compare the list of failing audit IDs (for example `color-contrast`, `heading-order`, `landmark-one-main`), not only the score. Pass means no new failing ID and a score no lower than before. A fixed ID (often `heading-order` or `landmark-one-main` after Module 2) is a gain; list it. A new failure is a regression: revert the change that caused it as in "Reverting a change" below.

**Reverting a change.** If a checkpoint holds only that change (an `m2` checkpoint for one component), run `git revert --no-edit <sha>`; the checkpoint approval from Module 1 step 9 covers it, or the user runs it by hand. If no checkpoint isolates it (checkpoints declined, or it shares a checkpoint with approved work, as every Module 3 change does in `m3`), undo only the lines that change touched, by hand, and show the user the diff. Never restore whole files or the tree, never run `git checkout -- .` or `git reset --hard`: that also discards the Module 1 cleanup. If a revert conflicts, stop and ask the user. After any revert, rerun the build and tests and recapture the affected routes at 375, 768 and 1440 px, and report the post-revert results, not the earlier ones. Contrast failures that existed before go to the hand-off in [Module 5](./module_5_handoffs.md).

## 4. Pass/fail checklist

Every line needs PASS, FAIL or N/A with a reason.

**Module 1: builder artifact cleanup**
- [ ] No builder attributes remain: `grep -rIn -e 'data-lovable-' -e 'data-v0-' -e 'data-bolt-' -e 'lovable-tagger' -e 'data-source-line' -e 'data-source-file' -e '__v0' -e 'v0-container' -ie 'generated by \(lovable\|v0\|bolt\)' --exclude-dir={node_modules,.git,dist,build,.next,out,coverage,.wrangler,.vercel,.netlify} .` prints nothing (the same command as the Module 1 inventory; the brace expansion needs bash or zsh); any remaining hit is listed as uncertain or left in place, with its reason
- [ ] No builder `<meta name="generator">` remains: `grep -rIliE --exclude-dir={node_modules,.git,dist,build,.next,out,coverage,.wrangler,.vercel,.netlify} -e 'name="generator"[^>]*content="[^"]*(lovable|v0|bolt|cursor)' -e 'content="[^"]*(lovable|v0|bolt|cursor)[^"]*"[^>]*name="generator"' .` prints nothing
- [ ] Each removed `data-id` and `data-component-*` was grepped first and nothing read it
- [ ] Analytics, tag manager and consent scripts, CAPTCHA or bot protection, and vendored widgets are all still present (count before equals count after)
- [ ] Builder badges or attribution were flagged to the user, not removed
- [ ] Uncertain items are listed, not deleted

**Module 2: semantic DOM flattening**
- [ ] One `<main>`; `<nav>` uses `<ul>/<li>`; every `<section>` has an accessible name or is listed with a reason (Module 2: a section with no heading stays as it is)
- [ ] One `<h1>`, no skipped heading levels, no empty headings
- [ ] No CSS selector (`> div`, `:nth-child`) or JS query that pointed at a removed wrapper
- [ ] Heading anchor links (`#id`) still resolve

**Module 3: utility class consolidation**
- [ ] Before/after counts of repeated strings are in the report
- [ ] Inconsistent values are proposed, not applied without approval
- [ ] No randomized or new arbitrary values

**All modules**
- [ ] Build passes (or matches its baseline failure)
- [ ] Each generator `--check` script passes (N/A if none exist)
- [ ] Types, lint and tests match or beat the baseline
- [ ] Screenshots at 375, 768 and 1440 px show no visible shift (diff images reviewed by hand)
- [ ] No new accessibility failures

## 5. Report template

````markdown
## Builder hygiene: verification report
**Repo / branch:** <name> / <branch>   **Package manager:** <npm|pnpm|bun|yarn>
**Diff:** <path to patch or PR link>

| Check | Before | After | Result |
|---|---|---|---|
| Build | pass | pass | PASS |
| Types | 2 errors | 2 errors (same) | PASS (baseline) |
| Tests | 41 pass | 41 pass | PASS |
| Screenshots (375/768/1440) | captured | 312 px changed, diff reviewed, no visible shift | PASS |
| Lighthouse a11y | 88, fails: heading-order | 94, fails: none | PASS |

**Changes by module:** M1 <n> attributes, <n> comments, <n> scripts removed; M2 <n> wrappers removed, landmarks added: <list>; M3 <n> classes extracted, repeated strings <before> to <after>
**Flagged for the user:** <badges, inconsistent values, uncertain items>
**Not run:** <check, reason>
**Deploy:** not done; a separate step for the owner
**Checklist:** <n> PASS, <n> FAIL, <n> N/A (full list below)
````
