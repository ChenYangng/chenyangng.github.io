# Notes User Site Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal GitHub Pages user site for `ChenYangng` that publishes public Agent notes from the repository `ChenYangng.github.io`.

**Architecture:** Use a content-only repository with plain Markdown files rendered by GitHub Pages from the root of the `main` branch. Keep the site intentionally small: a landing page, one `Agent` section page, three initial note pages, and a repository `README.md`.

**Tech Stack:** Git, GitHub Pages, Markdown, default GitHub Pages/Jekyll rendering

---

## File Structure

### New files

- `/.gitignore`
  Ignore local junk such as `.DS_Store`.
- `/README.md`
  Repository overview for GitHub visitors with links into the published site structure.
- `/index.md`
  Public homepage for the GitHub Pages site.
- `/agent/index.md`
  Landing page for the Agent section.
- `/agent/getting-started.md`
  Introductory note for readers who are new to Agent systems.
- `/agent/concepts.md`
  Core concepts note covering key Agent mental models.
- `/agent/tools.md`
  Tools and workflows note covering frameworks, CLIs, and usage patterns.
- `/assets/.gitkeep`
  Keeps the `assets/` directory in git until real assets are added.

### Existing files used for planning context

- `/docs/superpowers/specs/2026-04-09-notes-user-site-design.md`
  Approved design document that defines scope and structure.
- `/docs/superpowers/plans/2026-04-09-notes-user-site.md`
  This execution plan.

## Task 1: Initialize The Local Repository

**Files:**
- Create: `/.gitignore`

- [ ] **Step 1: Create a minimal `.gitignore`**

```gitignore
.DS_Store
```

- [ ] **Step 2: Verify `.gitignore` content**

Run: `sed -n '1,20p' .gitignore`
Expected:

```text
.DS_Store
```

- [ ] **Step 3: Initialize git on `main`**

Run: `git init -b main`
Expected: output includes `Initialized empty Git repository`

- [ ] **Step 4: Verify branch state**

Run: `git status --short --branch`
Expected:

```text
## No commits yet on main
?? .gitignore
?? docs/
```

- [ ] **Step 5: Commit the repository bootstrap**

Run:

```bash
git add .gitignore docs/
git commit -m "chore: initialize notes site workspace"
```

Expected: one root commit is created on `main`

## Task 2: Scaffold The Site Entry Points

**Files:**
- Create: `/README.md`
- Create: `/index.md`
- Create: `/agent/index.md`
- Create: `/assets/.gitkeep`

- [ ] **Step 1: Create the content directories**

Run: `mkdir -p agent assets && touch assets/.gitkeep`
Expected: no output

- [ ] **Step 2: Create the repository README**

```md
# Notes

Public notes site for Agent research, practice, and working references.

## Structure

- [Site Home](./index.md)
- [Agent Notes](./agent/index.md)

## Publishing

This repository is intended to be published as the GitHub Pages user site for `ChenYangng`.
Once published, the site URL will be:

`https://chenyangng.github.io/`
```

- [ ] **Step 3: Create the site homepage**

```md
# Notes

Research, practice, and working notes about Agent systems.

## Start Here

- [Enter Agent Notes](./agent/index.md)

## About

This site is a lightweight public knowledge base. It starts with Agent notes and is designed to expand over time without adding heavy tooling.

## Recent Notes

- [Getting Started With Agents](./agent/getting-started.md)
- [Agent Concepts](./agent/concepts.md)
- [Agent Tools](./agent/tools.md)
```

- [ ] **Step 4: Create the Agent section landing page**

```md
# Agent

Notes about Agent systems, from first principles to tools and workflows.

## Reading Path

1. [Getting Started](./getting-started.md)
2. [Concepts](./concepts.md)
3. [Tools](./tools.md)

## Notes In This Section

- [Getting Started](./getting-started.md) - A practical entry point for understanding what Agent systems are and why they matter.
- [Concepts](./concepts.md) - Core ideas such as loops, planning, tools, and memory.
- [Tools](./tools.md) - A compact survey of frameworks, CLIs, and working styles.
```

- [ ] **Step 5: Verify the scaffolded files exist**

Run: `find . -maxdepth 2 -type f | sort`
Expected:

```text
./.gitignore
./README.md
./agent/index.md
./assets/.gitkeep
./docs/superpowers/plans/2026-04-09-notes-user-site.md
./docs/superpowers/specs/2026-04-09-notes-user-site-design.md
./index.md
```

- [ ] **Step 6: Commit the site scaffold**

Run:

```bash
git add README.md index.md agent/index.md assets/.gitkeep
git commit -m "feat: scaffold notes site structure"
```

Expected: a commit is created with the landing pages

## Task 3: Write The Initial Agent Notes

**Files:**
- Create: `/agent/getting-started.md`
- Create: `/agent/concepts.md`
- Create: `/agent/tools.md`

- [ ] **Step 1: Create `agent/getting-started.md`**

```md
# Getting Started With Agents

This page introduces what Agent systems are, why they matter, and how to begin studying them.

## Why it matters

Agent systems turn models from passive responders into systems that can plan, use tools, and complete multi-step work.

## Key points

- An Agent is usually a loop, not just a single model call.
- Tool use is what lets an Agent act on the world outside the prompt.
- Good Agent design starts with a narrow task and clear success criteria.
- Observability matters early because multi-step behavior is harder to debug than a single response.

## Notes

When learning Agent systems, it helps to separate three layers:

1. The model
2. The control loop
3. The tools and environment

That separation makes it easier to compare frameworks without confusing the framework with the underlying Agent pattern.

## Related

- [Agent Home](./index.md)
- [Agent Concepts](./concepts.md)
- [Agent Tools](./tools.md)
```

- [ ] **Step 2: Create `agent/concepts.md`**

```md
# Agent Concepts

This page captures the core concepts that make Agent systems understandable and debuggable.

## Why it matters

Without a clear mental model, Agent systems can look more magical than they really are, which makes them harder to improve.

## Key points

- Planning decides what to do next.
- Tool use connects reasoning to action.
- Memory can be short-term, long-term, or externalized into files and systems.
- Evaluation should look at both final output and intermediate behavior.
- Constraints are part of the design, not an afterthought.

## Notes

Some useful questions to ask when reading or building an Agent:

- What is the unit of work?
- What state survives between steps?
- When does the loop stop?
- Which tools are available, and how are failures handled?

These questions often reveal more than the model choice alone.

## Related

- [Agent Home](./index.md)
- [Getting Started With Agents](./getting-started.md)
- [Agent Tools](./tools.md)
```

- [ ] **Step 3: Create `agent/tools.md`**

```md
# Agent Tools

This page surveys the tools and workflows commonly used when building or operating Agent systems.

## Why it matters

Most real Agent systems become useful only when model reasoning is paired with tools, execution environments, and clear operator workflows.

## Key points

- Tooling choices affect reliability as much as model choices do.
- Frameworks are useful, but some tasks only need a simple handwritten loop.
- CLIs and local sandboxes are often the fastest way to prototype Agent behavior.
- The best tool surface is small, explicit, and easy to observe.

## Notes

Useful categories to compare:

- Model SDKs
- Agent frameworks
- Retrieval and memory layers
- Execution environments
- Evaluation and tracing tools

It is usually better to start with the minimum tool surface that solves the task and add abstractions only when they reduce repeated work.

## Related

- [Agent Home](./index.md)
- [Getting Started With Agents](./getting-started.md)
- [Agent Concepts](./concepts.md)
```

- [ ] **Step 4: Verify the note files**

Run: `find agent -maxdepth 1 -type f | sort`
Expected:

```text
agent/concepts.md
agent/getting-started.md
agent/index.md
agent/tools.md
```

- [ ] **Step 5: Sanity-check the content rendering order**

Run: `sed -n '1,80p' agent/getting-started.md && sed -n '1,80p' agent/concepts.md && sed -n '1,80p' agent/tools.md`
Expected: each file begins with a level-1 heading, a one-sentence summary, and the sections `Why it matters`, `Key points`, `Notes`, and `Related`

- [ ] **Step 6: Commit the initial notes**

Run:

```bash
git add agent/getting-started.md agent/concepts.md agent/tools.md
git commit -m "feat: add initial agent note set"
```

Expected: a commit is created with the three note pages

## Task 4: Prepare Publishing Metadata And Local Verification

**Files:**
- Modify: `/README.md`
- Modify: `/index.md`

- [ ] **Step 1: Add explicit publishing instructions to `README.md`**

Append this section if it is not already present:

```md
## GitHub Pages Setup

After pushing this repository to GitHub as `ChenYangng.github.io`:

1. Open repository settings.
2. Open the `Pages` section.
3. Set the source to `Deploy from a branch`.
4. Select the `main` branch and `/ (root)` folder.
5. Save and wait for the site to publish.
```

- [ ] **Step 2: Add a publishing note to the homepage**

Append this section to `index.md`:

```md
## Publish

This site is intended for GitHub Pages deployment from the root of the `main` branch in the `ChenYangng.github.io` repository.
```

- [ ] **Step 3: Verify there are no whitespace or patching issues**

Run: `git diff --check`
Expected: no output

- [ ] **Step 4: Preview the final top-level files**

Run: `sed -n '1,200p' README.md && sed -n '1,200p' index.md`
Expected: both files include the Notes overview and GitHub Pages publishing guidance

- [ ] **Step 5: Commit the publishing guidance**

Run:

```bash
git add README.md index.md
git commit -m "docs: add publishing instructions"
```

Expected: a commit is created with publishing guidance

## Task 5: Create The Remote Repository And Publish

**Files:**
- Modify: repository settings on GitHub

- [ ] **Step 1: Create the GitHub repository**

In the browser, create a new public repository named `ChenYangng.github.io` under the `ChenYangng` account.

Expected: the repository exists at `https://github.com/ChenYangng/ChenYangng.github.io`

- [ ] **Step 2: Add the GitHub remote locally**

Run:

```bash
git remote add origin https://github.com/ChenYangng/ChenYangng.github.io.git
```

Expected: no output

- [ ] **Step 3: Push the `main` branch**

Run:

```bash
git push -u origin main
```

Expected: branch `main` is pushed and set to track `origin/main`

- [ ] **Step 4: Enable GitHub Pages from the repository root**

In the browser, open `Settings` -> `Pages`, choose `Deploy from a branch`, then select:

- Branch: `main`
- Folder: `/ (root)`

Expected: GitHub shows that the site is being built from the selected source

- [ ] **Step 5: Verify the published site**

Open: `https://chenyangng.github.io/`

Expected:

- The homepage loads
- The `Agent` page opens correctly
- The three note pages load from the published site

- [ ] **Step 6: Capture the final repository state**

Run: `git status --short --branch`
Expected:

```text
## main...origin/main
```

## Self-Review

### Spec coverage

- Repository name and publishing model are covered in Tasks 1 and 5.
- Required file structure is covered in Tasks 2 and 3.
- Homepage information architecture is covered in Task 2 and Task 4.
- Agent landing page and initial three pages are covered in Tasks 2 and 3.
- README and publishing guidance are covered in Tasks 2 and 4.

### Placeholder scan

The plan uses exact file paths, exact file contents, exact commands, and explicit expected outcomes. No `TODO`, `TBD`, or "implement later" placeholders remain.

### Type consistency

The repo name `ChenYangng.github.io`, branch `main`, root publish folder `/ (root)`, and final URL `https://chenyangng.github.io/` are used consistently throughout the plan.
