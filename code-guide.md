# Code Guide

Guidelines for coding agents (Copilot, IBM Bob, etc.) generating or reviewing code in this
project, and for whoever reviews their pull requests.

### General

* Is the code DRY? What areas could benefit from modularization or functions?
* Is the code formatted correctly? Indentation and spacing matter.
* Are functions and variables named appropriately? Semantic naming matters.
* Are variables and functions scoped as tightly as possible?
* Can variables be organized better (objects, arrays, etc.)?
* Do not add frameworks or libraries that aren't already in the solution (e.g. adding React
  to a plain HTML application) without being asked to.
* Do not leave dead code behind. If you replace an approach (e.g. rewriting a component to
  use inline styles instead of a separate stylesheet), remove the files the old approach
  used (the stylesheet, example HTML, etc.) rather than leaving both in the repo.
* Never create two files whose names differ only by case (e.g. `Header.css` and
  `header.css`). Case-insensitive filesystems (Windows, macOS) can only hold one of them,
  and checking the repo out there silently corrupts whichever file "loses" — this is a real
  failure mode, not a theoretical one.

### Unit Tests

* **Tests live in the same repository as the code they test.** Never put unit tests in a
  separate repo from the source — it makes them a pain to run, easy to forget about, and
  disconnected from the code changes that should have updated them.
* **Location**: put tests in a top-level `tests/` folder that mirrors the structure of the
  source folder, not colocated next to the source files. For a component at
  `src/components/Header.jsx`, its test belongs at `tests/components/Header.test.jsx`.
* Import the module under test using the project's existing path alias (e.g. `@/components/Header`)
  instead of a relative path — it keeps imports correct regardless of how deep the test file
  sits in the `tests/` tree.
* Use whatever test framework and libraries are **already configured** in the project
  (check `package.json`). Don't introduce a second, competing test framework.
* **Check Node/runtime engine compatibility before adding or upgrading a test dependency.**
  CI build agents often run an older LTS runtime than a local dev machine — for this
  project's DevOps Loop Build agents, that's **Node 16.20.2**. A dependency that works
  locally on a newer Node can fail outright in CI with a cryptic `SyntaxError` at startup.
  Before adding or bumping a package like `vitest`, `jsdom`, or `@testing-library/*`, check
  its declared `engines.node` range (`npm view <package>@<version> engines`) and confirm it's
  satisfied by the CI runtime, not just your local one.
* **Prefer robust, behavior-focused assertions over brittle implementation-detail ones.**
  Assert on visible text, ARIA roles, element presence, and user-facing interaction
  outcomes. Avoid hardcoding exact computed style values (hex/rgb colors, pixel values)
  wherever a behavioral assertion would do — colors and styling get tweaked often during
  active development, and a test suite that breaks on every palette change stops being
  useful signal. Where a test's whole point *is* to check a visual theme (a theme-utility
  function, say), that's fine — just keep it to the minimum needed to catch a real
  regression, not an assertion for every style property.
* If application behavior changes, **update the tests that behavior touches in the same
  change** — don't leave known-stale tests for someone else to discover as a CI failure
  later. A red test suite at the start of a task is a signal to fix forward, not to ignore.
* Use the test framework's built-in reporters (e.g. Vitest's built-in `junit` reporter)
  instead of hand-rolling a custom results-conversion script — less bespoke tooling to
  maintain.
* Keep the lockfile (`package-lock.json`, `yarn.lock`, etc.) committed and in sync with the
  manifest. If you change a dependency version, regenerate the lockfile — don't hand-edit
  either file, and don't let them drift (a stale lockfile breaks `npm ci` outright).

### Version Control

* Work on a feature branch and open a pull request. Never push directly to `main` (or
  whatever the default/protected branch is) — assume it's protected even if you haven't
  checked, and let the PR process run.
* Write commit messages that explain *why*, not just *what* — the diff already shows what
  changed.

### HTML

* Is the HTML document valid? (`<!DOCTYPE html>`, `<html>`, `<head>`, `<title>`, `<body>`)
* Are CSS files linked in the `<head>`?
* Are JS files linked near the end of the `<body>` (or `<head>` if appropriate, e.g. `defer`/`async`)?
* If the page is responsive, is the viewport meta tag included?
* Do form inputs have labels?

### CSS

* Are selectors clear to read and semantic?
* Are IDs used sparingly?

### Vanilla JS / jQuery

Make sure code that depends on the DOM is wrapped appropriately:

```js
// vanilla JS
document.addEventListener('DOMContentLoaded', function() {
  // code here
});

// or... jQuery
$(document).ready(function() {
  // code here
});
```
