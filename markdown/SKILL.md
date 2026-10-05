---
name: markdown
description: Markdown writing conventions. Use when creating or editing Markdown files, README files, tables, or generated documentation content.
---

## Structure

When generating documentation, put a short summary first, then the instructions and commands, then details below the instructions.

When a README or other documentation already exists, update it when the project changes.

## Tables

- Use spaces so columns line up in the raw file. Align the vertical bars.
- Use only three dashes (`---`) in separator cells.
- Always use the alignment colon (`:`) to indicate column alignment. For example, `:---` for left-aligned, `:---:` for centered, and `---:` for right-aligned.

```markdown
| Field | Values        | IntValue         |
| :---  | :---          | ---:             |
| env   | preprod, prod | $1,234,567,89.10 |
```
