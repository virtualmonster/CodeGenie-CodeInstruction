# Code Guide

This document is used two ways in DevOps Loop:

1. As **coding instructions** handed to the coding agent (Copilot, IBM Bob, etc.) when it
   picks up a work item.
2. As **review criteria** handed to the reviewing agent when it looks over the resulting
   pull request.

Write and review code with both uses in mind. If you're the coding agent, hold yourself to
this before you open the PR — don't rely on the reviewer to catch it.

---

## 0. Task Interpretation — Modify Before Create

Before producing a plan, requirements, implementation approach, or code, determine whether
the work item describes a change to an existing application or genuinely requests new
functionality.

**The default assumption is that the repository contains an existing application that must
be modified. Do not assume a greenfield implementation.**

### Determine the change mode

Treat the task as an **EXISTING IMPLEMENTATION / MODIFY** task when the work item:

- refers to an existing screen, page, component, workflow, API, service, button, form,
  process, behaviour, style, or feature;
- uses verbs such as improve, change, update, fix, replace, enhance, restyle, correct,
  refactor, extend, enable, disable, or make;
- describes current behaviour and asks for different behaviour;
- asks for a visual or UX change to something that already exists;
- refers to functionality in the definite form, such as "the checkout", "the header",
  "the chatbot", "the API", or "the login page".

Only treat a task as **NEW IMPLEMENTATION / CREATE** when:

- the work item explicitly asks for a new feature, page, component, service, application,
  or capability; or
- repository inspection confirms that no suitable implementation currently exists.

**Do not infer CREATE merely because the ticket describes the desired end state.**

For example, a work item saying:

> Improve the graphical look and feel of the checkout process. Change colour to black and
> yellow theme.

means:

> Modify the existing checkout implementation and its existing styling so that it uses a
> black and yellow visual theme.

It does **not** mean:

> Build a new checkout application containing cart, shipping, payment and confirmation
> stages.

Do not invent functionality that was not requested.

### Repository inspection comes before solution design

For MODIFY tasks, inspect the repository before deciding what needs to be implemented.

Before writing code:

1. Locate the existing implementation of the feature described by the work item.
2. Identify how that implementation is reached by the running application.
3. Identify the smallest set of existing files/components that need to change.
4. Identify existing tests covering the affected area.
5. Identify existing shared utilities, styles, configuration, services and conventions
   relevant to the change.
6. Make the requested change within that existing implementation wherever practical.

Do not design a replacement implementation before examining what already exists.

Do not create a parallel page, component, service, route, workflow or application simply
because creating one would be easier than understanding the existing implementation.

### Existing runtime code is authoritative

When multiple candidate implementations exist, determine which one is actually used by the
running application.

Prefer changing live, referenced code over:

- examples;
- prototypes;
- unused components;
- old implementations;
- demo files;
- abandoned stylesheets;
- similarly named but unreferenced modules.

Do not make a change in a new or unused component and assume the application will use it.

### Preserve the intent and scope of the work item

When expanding a work item into requirements or an implementation plan, preserve its scope.

Do not silently turn:

- "change" into "create";
- "improve" into "replace";
- "fix" into "redesign";
- "restyle" into "reimplement";
- "add X to Y" into "build a new Y";
- a cosmetic change into a functional redesign.

Do not invent new workflow steps, screens, fields, APIs, services, dependencies, documentation,
configuration, tests, or features unless they are necessary to satisfy the work item.

If the ticket is ambiguous, prefer the smallest change consistent with the stated intent.

### New files require justification

For a MODIFY task, creating new files is the exception rather than the default.

Before creating a new:

- component;
- page;
- stylesheet;
- utility;
- service;
- configuration file;
- test file;
- README;
- documentation file;
- example;
- application scaffold;

confirm that an appropriate existing artifact does not already exist and that the new file
is genuinely necessary to fulfil the work item.

Do not create alternative versions of existing functionality.

Do not scaffold a parallel application or feature.

If the repository already contains the requested feature, **modify it**.

### Do not invent deliverables

Do not automatically add standard software-project deliverables that the work item did not
request.

In particular, do not assume every task requires:

- a README;
- documentation;
- inline comments;
- a new test file;
- a new test framework;
- new components;
- new configuration;
- new dependencies;
- sample/example code;
- an application scaffold.

Only create or modify artifacts needed for the requested change.

### Scope check before implementation

Before making changes, be able to answer:

- What existing functionality is the work item referring to?
- Where is that functionality implemented?
- Is this MODIFY or CREATE?
- What is the smallest change that satisfies the request?
- Am I about to create something that already exists?
- Am I adding behaviour or deliverables the ticket did not request?

If the answer to the last two questions is yes, reconsider the implementation.

---

## 1. Core Engineering Principles

* **Default to modifying what exists — don't build a parallel implementation.**
  Existing runtime code is authoritative.

  Before writing a new component, file, or function, search the repository for something
  that already does most or all of the job: an existing component, stub, placeholder,
  partial implementation or currently mounted feature.

  A work item phrased as "make X do Y", "give X a default response", "the button doesn't
  work yet", "improve X", "change X", or any description of current behaviour that needs
  to change is a strong signal to edit the existing implementation in place.

  A second, competing implementation of the same feature — for example a brand-new chatbot
  component when one is already built and mounted in the UI — leaves dead code behind and
  creates ambiguity about which implementation is actually live.

  Only create something new when the work item asks for functionality that genuinely has
  no existing home in the codebase.

* **Match the size of the change to the size of the ask.**
  A one-line colour fix doesn't need a refactor of the component around it.

  Don't rename things, restructure folders, redesign neighbouring functionality, or
  "clean up while you're in there" unless that is part of the actual task.

  Larger diffs are harder to review and increase the chance of breaking something unrelated.

* **Preserve existing behaviour unless the work item asks to change it.**
  If a task concerns styling, do not alter application behaviour.
  If it concerns one workflow step, do not redesign the entire workflow.
  If it concerns one API response, do not redesign the API.

* **Don't add abstractions for a single use case.**
  A helper function used once isn't reusable, it's indirection.

  Three similar lines of code are usually better than a premature shared function that has
  to flex to cover all three.

* **Don't add speculative flexibility.**
  No config options, feature flags, extra parameters, extension points or abstractions for
  behaviour nobody asked for yet.

  Build what's needed now.

* **Centralize values that are duplicated across files.**
  If a colour, API base URL, or other constant shows up in more than one component, it
  belongs in one shared module that everything imports — not copy-pasted as a literal in
  each place.

  This repository already does this correctly in one spot:
  `src/utils/headerTheme.js` holds the colour constants that `Header.jsx` imports.

  Other components duplicate the same hex values as raw string literals instead of
  importing from it. When touching one of those, migrate it to the shared source rather
  than adding another copy.

* **Remove dead code you replace.**
  If you rewrite an implementation using a different approach, remove artifacts from the
  old approach that are no longer used.

  Don't leave both implementations in the repository.

  A stray `chatbot.css` or `chatbot-example.html` that nothing imports is exactly the kind
  of clutter that later gets test coverage written against it, even though it isn't live.

* **Never create two files whose names differ only by case**
  (for example `Header.css` and `header.css`).

  Case-insensitive filesystems such as Windows and macOS can only hold one of them.
  Checking the repository out there can silently corrupt whichever file loses.

* Is the code DRY?
* Are functions and variables named semantically?
* Are variables scoped as tightly as possible?
* Is formatting and indentation consistent with the surrounding code?

* **Do not add a framework or library that isn't already in the solution**
  unless the task explicitly requires it.

  For example, don't add React to a plain HTML page merely because the coding agent prefers
  working in React.

---

## 2. Working as a Coding Agent

### Inspect first, change second

Do not start implementation by scaffolding the desired result.

First understand:

- what application already exists;
- where the relevant behaviour lives;
- how the application currently implements it;
- which files are actually used;
- how the requested change fits into the current architecture.

The implementation plan should be based on repository evidence, not assumptions about what
a typical application might contain.

### Don't fabricate

Don't invent an API, library method, config option, component, endpoint, property or behaviour
you haven't confirmed exists.

If you're not sure a package exports something, check its actual source, types or documentation
rather than guessing from a plausible-sounding pattern.

### Verify before declaring done

Run the code, tests, or both before finishing.

"It should work" is not verification.

If the change touches the UI, actually render it where practical.

If it touches a function, actually exercise it.

A green test suite you watched pass is real evidence; a diff that merely looks correct is not.

### Understand the baseline

Where practical, run relevant existing tests before making the change.

This establishes whether failures already exist and helps distinguish regressions caused by
your change from unrelated pre-existing problems.

A failing test suite at the start is a signal, but it is **not permission to broaden the
scope of the task indefinitely**.

If failures are:

- directly related to the area being changed, investigate them;
- caused by the requested change, fix the implementation or legitimately update the affected
  test;
- clearly unrelated and pre-existing, report them rather than fixing unrelated code unless
  the work item requests it.

### State material assumptions

If the work item is genuinely ambiguous about something material — for example which
component to touch, what a business term means, or whether old functionality is expected to
remain — make a reasonable call and state the assumption in the PR description.

Do not silently invent additional scope.

### Stay inside the actual runtime you'll ship on

Code that works in the authoring environment but not on the CI/build agent isn't done.

See the Node compatibility guidance under Dependency Management.

This is a common way an agent's local success fails to reproduce in the pipeline.

---

## 3. Project Structure & Conventions

* Respect the existing folder layout.

  For a typical frontend:

  - application code under `src/`;
  - organized using existing folders such as `components/`, `pages/`, `utils/`, etc.;
  - tests under the existing project test structure.

  Do not invent a third convention.

* Use the project's existing path alias, for example:

  ```text
  @/components/Header
