# Notes User Site Design

## Summary

Build a minimal GitHub Pages user site for a public knowledge base focused on Agent notes. The repository will be named `<user>.github.io`, published from the `main` branch root, and implemented with plain Markdown so the first version can go live quickly with minimal maintenance overhead.

## Goals

- Launch a public notes site as quickly as possible.
- Keep the content structure simple enough to maintain directly in GitHub.
- Start with a single top-level category, `Agent`.
- Preserve a directory structure that can grow into a broader knowledge base later.

## Non-Goals

- Building a full personal portfolio homepage.
- Introducing a static site framework such as VitePress, Docusaurus, or MkDocs in v1.
- Adding multiple content categories in the first release.
- Adding search, tagging, or automatic content indexing in the first release.

## Chosen Approach

Use a GitHub Pages user site repository named `<user>.github.io` and publish directly from the root of the `main` branch. Content will be authored as plain Markdown files. The homepage will act as a lightweight landing page for the knowledge base instead of a traditional personal homepage.

This approach is preferred because it minimizes setup time, avoids build tooling, keeps the repo readable on GitHub itself, and leaves room for future migration to a more capable documentation framework if the content grows substantially.

## Alternatives Considered

### 1. Plain Markdown + GitHub Pages default publishing

Recommended for v1.

Pros:
- Fastest path to launch
- No build tooling or Actions required
- Works well for a small, curated set of notes

Cons:
- Limited navigation and search
- Less polished documentation UX

### 2. Jekyll with light configuration

Not chosen for v1.

Pros:
- Still compatible with GitHub Pages defaults
- More theme and layout flexibility

Cons:
- Adds configuration and templating complexity early
- More structure than needed for the first release

### 3. VitePress, Docusaurus, or MkDocs

Not chosen for v1.

Pros:
- Best documentation experience
- Strong navigation and future scalability

Cons:
- Slower setup
- Requires a build and deploy workflow
- Higher long-term maintenance burden from day one

## Repository And Site Structure

The initial structure will be:

```text
<user>.github.io/
  README.md
  index.md
  agent/
    index.md
    getting-started.md
    concepts.md
    tools.md
  assets/
```

### File responsibilities

- `README.md`
  Repository-facing overview for visitors on GitHub, with a short description and a link to the published site.
- `index.md`
  Published site homepage.
- `agent/index.md`
  Category landing page for Agent content.
- `agent/getting-started.md`
  Introductory page for readers new to Agent topics.
- `agent/concepts.md`
  Core concepts and mental models.
- `agent/tools.md`
  Notes on tools, frameworks, CLIs, and workflows.
- `assets/`
  Images and other static assets for future content.

## Information Architecture

### Homepage

The homepage should behave as a notes landing page, not a resume-style personal homepage.

It should contain:

- Site title: `Notes`
- A one-line description such as "Research, practice, and working notes about Agent systems."
- A primary entry point linking to the `Agent` category
- A short `About` section describing the purpose of the site
- A lightweight `Recent Notes` section listing a few curated links manually

### Agent category page

The `Agent` landing page should include:

- A short description of how Agent topics are organized
- A suggested reading path
- A list of available notes with one-line descriptions

The initial reading order should be:

1. `getting-started.md`
2. `concepts.md`
3. `tools.md`

## Content Model

Each note should use the same lightweight structure:

```md
# Title

One sentence explaining what this page covers.

## Why it matters
Why the topic is important.

## Key points
3-5 core ideas.

## Notes
Examples, working observations, references, or commentary.

## Related
Links to adjacent pages.
```

### Writing rules

- One page should cover one topic.
- Each page should begin with a one-sentence summary.
- Filenames should use short, lowercase, hyphenated English names.
- Pages should link to related notes to gradually build a connected knowledge base.

## Publishing Flow

The site will be deployed with GitHub Pages using:

- Repository type: GitHub Pages user site
- Repository name: `<user>.github.io`
- Visibility: `Public`
- Source branch: `main`
- Source folder: `/ (root)`

Expected publication URL:

`https://<user>.github.io/`

## First Release Scope

The first release is complete when:

- The repository exists as `<user>.github.io`
- GitHub Pages is enabled from the root of `main`
- The homepage is published
- The `Agent` category landing page is published
- The three initial Agent note pages are published
- The repository `README.md` clearly explains the project

## Future Evolution

Possible later upgrades, not part of v1:

- Add more top-level categories such as `LLM`, `Tools`, or `Notes`
- Introduce a static documentation framework
- Add stronger navigation, search, and tagging
- Split the homepage into a lightweight root landing page plus deeper section indexes

## Risks And Mitigations

### Risk: the site feels too bare

Mitigation:
Use a clean homepage structure and a clear category landing page so the site still feels intentional even without a framework.

### Risk: content structure becomes inconsistent over time

Mitigation:
Use a fixed note template and naming convention from the start.

### Risk: future migration becomes painful

Mitigation:
Keep content in plain Markdown with a predictable directory layout so it can be migrated later with minimal restructuring.

## Implementation Notes

When implementation begins, the work should prioritize:

1. Repository initialization
2. Base file scaffolding
3. Homepage content
4. Agent section content
5. GitHub Pages configuration

No framework, theme system, or deployment automation should be introduced unless the scope changes.
