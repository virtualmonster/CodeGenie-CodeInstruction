# Code Guide

This document is used two ways in DevOps Loop:

1. As **coding instructions** handed to the coding agent (Copilot, IBM Bob, etc.) when it
   picks up a work item.
2. As **review criteria** handed to the reviewing agent when it looks over the resulting
   pull request.

Write and review code with both uses in mind. If you're the coding agent, hold yourself to
this before you open the PR — don't rely on the reviewer to catch it.

---

## 1. Core Engineering Principles

* **Default to modifying what exists — don't build a parallel implementation.** Before
  writing a new component, file, or function, search the repo for something that already
  does most of the job (an existing component, a stub, a placeholder, a partial
  implementation). A work item phrased as "make X do Y," "give X a default response," "the
  button doesn't work yet," or any description of *current* behavior that needs to change
  is a strong signal to edit that existing file in place — not scaffold a new one next to
  it. A second, competing implementation of the same feature (e.g. a brand-new chatbot
  component when one is already built and already mounted in the UI) leaves dead code
  behind and creates ambiguity about which one is actually live. Only create a new file
  when the work item asks for functionality that genuinely has no existing home in the
  codebase.
* **Match the size of the change to the size of the ask.** A one-line color fix doesn't need
  a refactor of the component around it. Don't rename things, restructure folders, or
  "clean up while you're in there" unless that was the actual task — it makes the diff
  harder to review and increases the chance of breaking something unrelated.
* **Don't add abstractions for a single use case.** A helper function used once isn't
  reusable, it's indirection. Three similar lines of code are usually better than a
  premature shared function that has to flex to cover all three.
* **Don't add speculative flexibility.** No config options, feature flags, or extra
  parameters for behavior nobody asked for yet. Build what's needed now.
* **Centralize values that are duplicated across files.** If a color, API base URL, or
  other constant shows up in more than one component, it belongs in one shared module that
  everything imports — not copy-pasted as a literal in each place. (This repo already does
  this correctly in one spot — `src/utils/headerTheme.js` holds the color constants that
  `Header.jsx` imports — but other components duplicate the same hex values as raw string
  literals instead of importing from it. When you touch one of those, migrate it to the
  shared source instead of adding another copy.)
* **Remove dead code you replace.** If you rewrite a component to use a different
  approach (inline styles instead of a stylesheet, a new component instead of an old one),
  delete the files the old approach used. Don't leave both in the repo — a stray
  `chatbot.css` and `chatbot-example.html` that nothing imports is exactly the kind of
  clutter that later gets test coverage written against it, for code that isn't even live.
* **Never create two files whose names differ only by case** (e.g. `Header.css` and
  `header.css`). Case-insensitive filesystems (Windows, macOS) can only hold one of them —
  checking the repo out there silently corrupts whichever file "loses." This isn't
  theoretical; it has actually happened in this repo's history.
* Is the code DRY? Are functions and variables named semantically, and scoped as tightly
  as possible? Is formatting/indentation consistent with the rest of the file?
* Do not add a framework or library that isn't already in the solution (e.g. adding React
  to a plain HTML page) unless the task explicitly calls for it.

## 2. Working as a Coding Agent

* **Don't fabricate.** Don't invent an API, a library method, a config option, or a
  behavior you haven't confirmed exists. If you're not sure a package exports something,
  check its actual source/types or docs rather than guessing from a plausible-sounding
  pattern.
* **Verify before declaring done.** Run the code (or the tests, or both) before finishing.
  "It should work" is not verification. If the change touches the UI, actually render it.
  If it touches a function, actually call it. A green test suite you watched pass is real
  evidence; a diff that merely looks correct is not.
* **A red test suite at the start of a task is a signal, not background noise.** If tests
  are already failing when you start, and your task touches the same area, fix them
  forward as part of your change rather than working around them or leaving them red.
* **State assumptions.** If the work item is ambiguous about something material (which
  component to touch, what a color name maps to, whether an old file should be deleted),
  make a reasonable call, and say what you assumed and why in the PR description — don't
  silently guess and hope it was right.
* **Stay inside the lane of the actual runtime you'll ship on.** Code that works on your
  authoring environment but not on the CI/build agent isn't done — see the Node
  compatibility note under Dependencies below. This is the single most common way an
  agent's local success fails to reproduce in the pipeline.

## 3. Project Structure & Conventions

* Respect the existing folder layout. For a typical frontend: application code under
  `src/` (organized by `components/`, `pages/`, `utils/`, etc.), tests under a top-level
  `tests/` folder (see Testing below) — don't invent a third convention.
* Use the project's existing path alias (e.g. `@/components/Header`) for cross-folder
  imports instead of long relative paths (`../../../components/Header`). Check
  `vite.config.js` / `tsconfig.json` / `jsconfig.json` for what's already configured before
  assuming one doesn't exist.
* If a project has an established pattern for something (a theme/constants module, a
  particular state-management approach, a particular way of structuring API calls), follow
  it. Don't introduce a second, competing pattern for the same problem.

## 4. Dependency Management

* **Check runtime/engine compatibility before adding or upgrading any dependency**,
  test tooling included. CI build agents often run an older LTS runtime than a coding
  agent's authoring environment. For this project's DevOps Loop Build agents specifically,
  that runtime is **Node 16.20.2** — noticeably older than what a modern dev sandbox
  usually runs. A dependency that installs and runs fine locally can fail outright in CI
  with a startup `SyntaxError` if its actual minimum Node version is higher. Before adding
  or bumping a package, check its declared `engines.node` range:
  ```
  npm view <package>@<version> engines
  ```
  and confirm the CI runtime satisfies it — not just your local one. When a whole
  dependency chain (test framework + its plugins + jsdom, say) needs to move together to
  stay compatible, downgrade/pin the whole set consistently rather than leaving one package
  out of step with the rest.
* **Keep the lockfile in sync and committed.** If you change a dependency version in
  `package.json`, regenerate `package-lock.json` (`npm install`, not a hand edit) and
  commit both together. A drifted lockfile makes `npm ci` fail outright — which usually
  means it fails in CI, not locally, since local dev often uses `npm install` instead.
* Don't add a dependency to solve something the standard library or an already-installed
  package already does.
* Run `npm audit` (or equivalent) mentally before adding something with lots of transitive
  dependencies; prefer a smaller, well-maintained library over a heavy one for a small job.

## 5. Unit Testing

* **Tests live in the same repository as the code they test, in a top-level `tests/`
  folder that mirrors the structure of `src/`.** Never colocate tests next to source files,
  and never put tests in a separate repository from the application code — both make tests
  easy to lose track of and disconnected from the changes that should have updated them.
  For a component at `src/components/Header.jsx`, its test belongs at
  `tests/components/Header.test.jsx`.
* Import the module under test using the project's path alias (`@/components/Header`), not
  a relative path — it keeps the import correct regardless of how deep the test file sits
  in the `tests/` tree, and matches convention with the rest of the codebase.
* **Use whatever test framework is already configured** (check `package.json` and the test
  config in `vite.config.js`/`vitest.config.js`/etc.). Don't introduce a second, competing
  test framework into the same project.
* **What needs a test**: new components, new utility functions, and new business logic
  should get one. Bug fixes should get a test that would have caught the bug. Trivial
  passthroughs (a one-line getter, a component that's pure JSX with no logic) don't need
  dedicated tests just to hit a coverage number.
* **Never assert an exact color or style value — no exceptions, including theme/style
  utilities themselves.** Prefer checking visible text, ARIA roles/labels, element
  presence, and the outcome of user interactions over checking colors (hex/rgb), fonts, or
  pixel dimensions. This project's palette gets changed often and deliberately (rebrands,
  demos, A/B looks) — a suite that breaks on every palette change stops being a useful
  signal and starts being noise the next change has to wade through. An earlier version of
  this guide carved out an exception for "a test whose entire purpose is verifying a theme
  utility" — that exception was a mistake: it was read as license to write comprehensive
  "must be pink, must not be blue" test suites, which broke on the very next color change
  and had to be deleted twice. If you need to test a theme/style function's *behavior*
  (that it accepts overrides, that it returns the right shape of object), assert on
  structure — `Object.keys(result)`, `typeof result`, that an override is threaded through
  relative to the default — never on what the actual color values are.
* **Never write a "should NOT be `<old value>`" assertion.** A negative check against a
  specific value you just changed away from (a color, a class name, a string) encodes a
  point-in-time fact as if it were a permanent invariant. It will not catch a regression
  back to the old value in any way a normal assertion on the new value wouldn't already
  catch, and it actively breaks the moment that value is chosen again for an unrelated
  reason. If you're tempted to write one, you almost certainly don't need the test at all.
* **Don't create a new test file to verify something an existing test file already
  covers.** If a module or component already has a test file, extend or update it — don't
  add a second, differently-named file that tests the same thing (`colorTheme.test.js`
  next to an existing `headerTheme.test.js` covering the same module, say). Two files
  asserting overlapping things under different names is how a value gets updated in one
  place and silently missed in the other, which is exactly the kind of self-inflicted CI
  failure this section is trying to prevent.
* **Tests must be deterministic.** No dependency on real wall-clock time, real network
  calls, unseeded randomness, or execution order between test files. Mock external
  services and timers; don't mock the thing you're actually testing.
* **Keep test names descriptive of behavior**, not implementation (`"disables Add to Cart
  when out of stock"`, not `"test 4"`).
* **Use the test framework's built-in reporters** (e.g. Vitest's built-in `junit` reporter
  via `--reporter=junit --outputFile=test-results.xml`) instead of hand-rolling a custom
  results-conversion script. Less bespoke tooling for the next person (or agent) to
  maintain, fewer places for the CI pipeline's expectations and the actual test output to
  drift apart.
* Don't commit generated test artifacts (`test-results.xml`, `test-results.json`,
  `coverage/`) — make sure they're in `.gitignore`.

## 6. Security Basics

* Never commit secrets, API keys, or credentials — use environment variables or the
  project's existing secret-management approach, and use placeholders in examples/tests.
* Validate and sanitize anything crossing a trust boundary (user input, query params,
  external API responses) — don't assume upstream data is well-formed.
* Don't introduce common web vulnerabilities: unescaped user content rendered as HTML
  (XSS), string-concatenated queries (injection), `eval`/`Function` on untrusted input.
* Keep dependencies reasonably current for known CVEs, but don't do an unrelated mass
  dependency bump as a side effect of an unrelated task — that's a separate, deliberate
  change.

## 7. Version Control & Git Hygiene

* **Work on a feature branch and open a pull request. Never push directly to `main`** (or
  whatever the default branch is) — assume it's protected even if you haven't explicitly
  checked, and let the PR process run its course, including any required reviews.
* Keep commits scoped to the task — don't bundle an unrelated fix or cleanup into the same
  commit/PR as the requested change, even a good one. Flag it separately instead.
* Write commit messages that explain **why**, not just what — the diff already shows what
  changed; the message should carry the context a future reader (human or agent) won't get
  from the code alone.
* Don't force-push over a branch's history, skip commit hooks, or bypass required status
  checks to get a PR to go green.

## 8. Documentation & Comments

* Default to no comments. Add one only when the *why* isn't obvious from the code itself —
  a non-obvious constraint, a workaround for a specific bug, behavior that would surprise a
  reader. Don't narrate what the code does when good naming already does that.
* Keep comments and docstrings in sync with the code. A stale comment describing behavior
  that no longer exists (e.g. a test file's header comment still describing a color scheme
  that was changed three commits ago) is worse than no comment — it actively misleads the
  next reader.
* Don't reference the current task, ticket number, or "the fix for X" in code comments —
  that context belongs in the commit message/PR description and rots as the codebase
  evolves past it.

## 9. Framework/Language-Specific Notes

### React / Vite / Vitest (this project's stack)

* Follow the existing component conventions (function components, hooks, the existing
  props/styling approach) rather than introducing a different pattern.
* Reuse existing shared utilities (theme constants, API client helpers) rather than
  re-implementing the same thing locally in a new component.

### HTML

* Is the document valid? (`<!DOCTYPE html>`, `<html>`, `<head>`, `<title>`, `<body>`)
* Are CSS files linked in the `<head>`?
* Are JS files linked near the end of `<body>` (or `<head>` with `defer`/`async` if
  appropriate)?
* If the page is responsive, is the viewport meta tag included?
* Do form inputs have associated labels?

### CSS

* Are selectors clear and semantic? Are IDs used sparingly (prefer classes)?

### Vanilla JS / jQuery

Make sure code that depends on the DOM is wrapped appropriately:

```js
// vanilla JS
document.addEventListener('DOMContentLoaded', function () {
  // code here
});

// or... jQuery
$(document).ready(function () {
  // code here
});
```

---

## 10. Pull Request Review Checklist

A condensed pass for the reviewing agent — if any of these are unclear from the diff, ask
rather than assume:

- [ ] Does the change do only what the work item asked, with no unrelated refactors bundled in?
- [ ] Are there tests for new/changed behavior, and do they live in `tests/` mirroring `src/`?
- [ ] Do the tests assert behavior rather than exact color/style values (including in a theme/style utility's own tests)?
- [ ] Any "should NOT be `<old value>`" negative assertions that should just be deleted?
- [ ] Any new test file that duplicates coverage an existing test file already has for the same module?
- [ ] Did the PR run the tests and confirm they pass — not just that they exist?
- [ ] Are any new/bumped dependencies checked against the CI runtime's Node version?
- [ ] Is `package-lock.json` (or equivalent) updated and consistent with `package.json`?
- [ ] Any dead code left behind from an approach the change replaced?
- [ ] Any new files that collide in name with an existing one only by case?
- [ ] Any secrets, credentials, or hardcoded environment-specific values introduced?
- [ ] Does the commit/PR message explain *why*, and is that context missing from the diff itself?
