# Module 2: Semantic DOM Flattening

Removes wrapper divs that do nothing and replaces generic containers with landmarks and a clean heading tree. The visual output must stay identical. This module can break layout, so it needs screenshots and a build and test pass on both sides of the change. Run it after [Module 1](./module_1_builder_artifact_cleanup.md); [Module 3](./module_3_utility_class_consolidation.md) runs next.

## 2.1 When to run

- After Module 1 has passed its checks.
- Preconditions (in analysis-only mode, see the [dispatcher](./00_dispatcher.md) Modes, skip preconditions 1 to 3: run the 2.2 checks, read the markup and list proposed changes with their risk, without editing or rendering):
  1. The build and test commands are known and their current result is recorded; the Module 1 result matches its recorded baseline. A failure that exists before the edit is recorded as the baseline and is not fixed here.
  2. The user approved checkpoint commits at Module 1 step 9, or makes each one by hand. If they declined, do not run this module; modules 3 and 4 run on the Module 1 result. The Module 1 result is committed as its own local checkpoint before the first component, with the checkpoint steps in [Module 1](./module_1_builder_artifact_cleanup.md) section 1.4 step 9: exact paths staged with `git add -- <path>`, `git status --short` checked, `git commit -m "m1: builder artifact cleanup"`, short sha recorded. In a symptom-routed run (dispatcher, Routing by symptom) or a zero-hit run the user insisted on, Module 1 made no edits: ask for checkpoint approval before the first component, with the same three choices as Module 1 step 9, and the clean `<baseline-sha>` commit stands in for the `m1` checkpoint.
  3. You can render the pages. The dispatcher's Step 0 baseline (untouched code) already exists; Module 4 compares against it. This module takes a second, local set: full-page "before" screenshots of the Module 1 result at 375, 768 and 1440 px for every route you will touch, with the same capture settings as Step 0. Save them as `${TMPDIR:-/tmp}/kirby-shots/m2/<page>-<width>-before.png` and `-after.png` (create the folder with `mkdir -p`). The top-level `${TMPDIR:-/tmp}/kirby-shots/*-before.png` files belong to Step 0 and are read-only for all later modules. If you cannot render the pages, ask the user to supply the before screenshots; do not run this module blind.
  4. You have a list of the routes and components in scope. Work one component at a time.

## 2.2 Breakage risks and their checks

| Risk | How it breaks | Check before the edit |
|---|---|---|
| Child selectors | `.card > div`, `> div:first-child`, `:nth-child` or `:has(+ .x)` stop matching once a wrapper goes | `grep -rIn --exclude-dir=node_modules -e '> *div' -e ':nth-child' -e ':first-child' -e ':last-child' -e ':has(' --include='*.css' --include='*.scss' --include='*.js' --include='*.mjs' --include='*.ts' --include='*.tsx' --include='*.jsx' --include='*.html' --include='*.vue' --include='*.svelte' --include='*.astro' --exclude-dir=.git --exclude-dir=dist --exclude-dir=build --exclude-dir=.next .` |
| Heading anchors | Links to `#pricing` break if an `id` moves or a heading is renamed | `grep -rIn --exclude-dir=node_modules -e 'href="#' -e "href='#" -e 'scrollIntoView' -e 'getElementById' .` and list every `id` on headings and sections |
| Flex and grid context | Removing a wrapper moves its child into a different flex or grid parent, so gaps, alignment or column spans change | For each wrapper, read the computed `display` of the parent and the wrapper in dev tools or from its classes |
| Descendant and bare tag selectors | `.nav ul`, `header a` or `section h2` start or stop matching when a tag changes (a div becomes `<nav>` with a `<ul>`) | Grep A below; read each hit |
| JS and test queries | `querySelector('div.x')`, `closest('div')` or test queries by tag stop matching when a div becomes `<section>`; tests that regex or string-match raw HTML (`<section class="..." id="...">`) break on any tag or attribute change | `grep -rIn --exclude-dir=node_modules -e 'querySelector' -e 'closest(' -e "getByRole" .`, then grep B below for raw-HTML matches in tests |
| Child-structure utilities and sibling selectors | `space-y-*`, `divide-*`, arbitrary variants such as `[&>div]:` and `*:`, and `+` or `~` rules target direct children or siblings, so removing a wrapper changes spacing | `grep -rIn --exclude-dir=node_modules -e 'space-[xy]-' -e 'divide-[xy]' -e '\[&>' -e '\*:' --include='*.tsx' --include='*.jsx' --include='*.html' --include='*.css' .` and read every `+` or `~` rule in the CSS hits |
| Invalid nesting | A `<div>` moved into a `<p>` (or a block element into an inline one) is invalid HTML and breaks hydration | Check the parent tag before every merge |

```bash
# Grep A: descendant selectors and bare tag rules
grep -rInE --exclude-dir=node_modules --include='*.css' --include='*.scss' --include='*.js' --include='*.ts' \
  --include='*.tsx' --include='*.jsx' --include='*.html' --include='*.vue' --include='*.svelte' --include='*.astro' \
  -e '[.#][A-Za-z0-9_-]+ +(div|section|ul|ol|li|nav|header|footer|main|article|h[1-6])\b' \
  -e '^ *(header|footer|nav|main|section|article|ul|li)\b[^-]' .
# Grep B: tests that match raw HTML
grep -rInE --exclude-dir={node_modules,.git} --include='*.test.*' --include='*.spec.*' --include='*_test.*' \
  -e '<(div|section|main|nav|header|footer|article|h[1-6])[ >]' .
```

A hit means the element stays, or the selector is updated in the same diff with the reason written in the report.

Add `--exclude-dir` for archive folders (`archive`, `old`, `backup`) and build caches (`.wrangler`, `.vercel`, `.netlify`, `.astro`, `.svelte-kit`, `dist`, `build`, `.next`) to every grep above. Where shared chrome (header, nav, footer) is injected from partials, edit the source partial and regenerate it as in the dispatcher's "Generated regions and asset pipelines" section.

## 2.3 Rules

Wrappers:
- Remove a div only when it has one child and no classes, style, ref, event handler, `id`, `key` that matters, or attribute that any selector or script reads.
- Merge nested flex or grid containers only when one parent produces the same layout. Move the classes onto the surviving element.

Landmarks (minimum):
- Exactly one `<main>` per page.
- One page-level `<header>` and `<footer>`. A `<header>` inside an `<article>` or `<section>` is valid and stays.
- `<nav>` for navigation blocks, with links in `<ul>` and `<li>`.
- `<section>` only when it has an accessible name (`aria-labelledby` pointing at its heading). Otherwise keep the div.
- An existing `<section>` with no accessible name: add `aria-labelledby` pointing at its heading, adding a new `id` to the heading if it has none. Never rename an existing `id` and never downgrade the section to a div. A section with no heading stays as it is; list it.
- `<article>` for self-contained items such as a post card or a review.
- All content sits inside a landmark. Check for blocks outside them (for example content after `</main>` and before `<footer>`); move a block in only if the screenshots match, otherwise list it.

Lists:
- Fake lists (items split by `<br>` with tick characters, div grids of items, bare rows of links) become `<ul>` or `<ol>` with `<li>`, plus a reset of `list-style`, `margin` and `padding` (in the project's stylesheet or as utilities on the list) so browser bullets and indents do not appear. This reset is the one styling change allowed here, because it keeps the render identical. Check the screenshots.
- Do not put `aria-label` on a generic `<div>` or `<span>`; it has no role to name. List it for the [Module 5](./module_5_handoffs.md) hand-off to `better-accessibility`.

Headings (minimum):
- One `<h1>` per page.
- No skipped levels (`h2` to `h4`).
- No empty headings.
- Heading tags used only for font size: do not retag them here, because the fix needs a new class and a visual check. List each one with file and line in the report for Module 3 or the user.

For the full WCAG 2.2 AA landmark and heading rules, hand off through [Module 5](./module_5_handoffs.md).

## 2.4 Procedure

1. Take the "before" screenshots (2.1).
2. Run every check in 2.2 and save the hits.
3. Pick one component. List its wrapper divs and mark each as remove, merge or keep with a reason.
4. Apply the landmark and heading rules. Keep every existing class, `id` and attribute on the new element.
5. Using the checkpoint approval recorded at Module 1 step 9 (or in precondition 2 for a routed run), run the build and tests, and capture the routes this component touches. When the build and tests match the baseline and the captures show no visible shift, make a checkpoint without asking again (if the user chose to commit by hand, ask them to make it and record its sha) with the Module 1 step 9 steps, message format included: stage exactly the paths this component touched, check `git status --short`, `git commit -m "m2: <component> (<n>)"` where `<n>` numbers each attempt at that component, and record the short sha. If any check fails, roll back the current component: list every path it touched and run `git status --short`. If it shows a modified path this component did not touch, leave that path alone and tell the user. Restore each touched tracked path with `git restore --source=HEAD -- <path>`, and delete the new files this component created (list them first; never delete a file it did not create). If a touched path was untracked before this step and is not new, stop and tell the user. Then record the component under "Skipped changes". HEAD holds the Module 1 commit for the first component and the previous component's checkpoint after that, so the Module 1 edits survive. Checkpoint commits are local; the user may squash them. Never run `git checkout -- .` or `git reset --hard`.
6. Repeat steps 3 to 5 for each component.
7. Take the "after" screenshots at the same widths and routes. Compare them as in 2.7. A visible shift means you undo that component's checkpoint. Revert in reverse checkpoint order, newest first, from the last checkpoint back to that component's: `git revert --no-edit <sha>^..<last-sha>`, using the shas recorded at each checkpoint. The checkpoint approval from Module 1 step 9 covers these revert commits; if the user chose to commit by hand, show them the command and let them run it. Then rerun the build and tests and recapture the affected routes at 375, 768 and 1440 px. Only then record the component as skipped; redo any later component the revert also undid through steps 3 to 5, or record it as skipped too. If the revert conflicts, stop and ask the user. Never run `git checkout -- .` or `git reset --hard`.
8. Hand off to Module 3. Full checks, including the accessibility scan, run in [Module 4](./module_4_verification.md).

## 2.5 Prompt block

```text
Refactor this markup to semantic HTML (WCAG 2.2 AA). The rendered output must look identical.

- Remove single-child wrapper divs that have no layout, style or script purpose.
- Merge nested flex or grid containers when one parent gives the same layout.
- Landmarks: one <main>, a page <header> and <footer>, <nav> with <ul>/<li> for link lists, <section> only with an accessible name (name an existing unnamed section with aria-labelledby; never rename an id or downgrade it), <article> for self-contained items. No content outside landmarks.
- Fake lists become <ul>/<li> with list-style, margin and padding reset.
- Headings: one <h1>, no skipped levels, no empty headings. List heading tags used only for size; do not retag them.
- Move existing classes, ids and attributes onto the new elements.
- Do not change any element that a CSS selector, script or test depends on (child selectors, :nth-child, #anchors, querySelector, closest).

If a change could alter layout or behaviour, skip it and list it.

Output: a unified diff, then a report listing divs removed, landmarks added, heading changes, and skipped changes with reasons.
```

## 2.6 Do and do not

Do:
- Work one component at a time and test after each.
- Keep every `id` that a link, script or test points at.
- Revert and record any change the screenshots show.

Do not:
- Run this module without before and after screenshots.
- Rename or renumber a heading's `id`.
- Add `role` attributes where a native element does the job.
- Change text, add or remove classes, or change styling. Moving an element's existing classes onto the surviving element (2.3) is allowed, and so is the fake-list reset. Class consolidation is Module 3.

## 2.7 Verification

- Before and after screenshots show no visible shift at 375, 768 and 1440 px on every touched route, reviewed as in [Module 4](./module_4_verification.md) section 2. Small antialiasing differences after hand review are not failures.
- Build and tests pass, same result as the baseline.
- `grep -c '<main'` per page layout (per HTML file on sites without layouts) returns 1; no content sits outside a landmark; one `<h1>` per rendered page.
- Every `#anchor` found in 2.2 still resolves to an element with that `id`.
- Every child selector hit in 2.2 still matches, or its update is in the diff.

## 2.8 Output template

````markdown
### Module 2: Semantic DOM flattening

**Diff**
```diff
<unified diff of every changed file>
```

**Divs removed**: <n> (<file>: <count>, ...)
**Landmarks added**: <element>, <file:line>
**Heading changes**: <old> -> <new>, <file:line>

**Skipped changes** (not applied)
- <file:line>, <planned change>, <reason: selector hit | flex/grid context | anchor | screenshot diff | test failure>

**Screenshots**: before <paths>, after <paths>, result <match | differences listed>
**Build / tests**: <command> -> <pass|fail>, baseline <pass|fail>
````
