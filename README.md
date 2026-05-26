# Project Ideas

A personal notepad for project ideas. Each project lives in its own directory.

## Conventions

- **One directory per project** — all files related to a project live under its own top-level directory (e.g., `./ltvt/`).
- **Notes** — project-specific notes always go in a `notes/` subdirectory inside the project directory (e.g., `./ltvt/notes/`). Notes must be Markdown files and must include a metadata block at the top:
  ```markdown
  ---
  summary: Brief description of this note
  last_updated: YYYY-MM-DD
  tags: [tag1, tag2]
  ---
  ```
- **Scratch files** — `.scratch/` is for local temporary files and is never checked in to version control.

## Projects

| Project Name | Last Updated | Project Overview |
|---|---|---|
| [ltvt](./ltvt/) | 2026-05-26 | A modernization of the Lunar Terminator Visualization Tool — a Moon-surface rendering app originally written in Delphi 6, using JPL ephemeris data and lunar texture/elevation maps. |
