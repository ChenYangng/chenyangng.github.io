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
