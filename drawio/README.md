# drawio

Create and edit draw.io diagram files (`.drawio`) by writing the XML directly — flowcharts, architecture diagrams, process flows, decision trees, org charts, ER diagrams, UML, and network diagrams.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

## What it covers

- **File format** and the critical rules for valid, editable `.drawio` XML.
- **Shape and connector references** — the vocabulary for nodes and edges.
- **Layout guidelines**, containers/groups, and style palettes.
- **Visual emphasis** and common element patterns.
- **Cloud icons** — tested patterns for AWS, Azure, and Google Cloud service icons, plus how wrong names fail silently and how to catch them.
- **Auto-layout** — write nodes without coordinates and let draw.io place and route them (`--layout`), and repair generated files with `--normalize`.
- **Editing existing files** and a validation checklist.
- **Exporting** to PNG/SVG/PDF/HTML with the draw.io desktop CLI, including multi-page export, embedding editable source, and Windows specifics (the installer doesn't add it to `PATH`).

## Requirements

None to author `.drawio` files — they are plain XML. Exporting to image formats, auto-layout, and checking icons by eye all use [draw.io desktop](https://github.com/jgraph/drawio-desktop/releases/latest) (`winget install JGraph.Draw` on Windows). Files open in draw.io desktop, the diagrams.net browser editor, and the VS Code draw.io extension.

On Windows the installer does not add `drawio` to `PATH`; call `C:\Program Files\draw.io\draw.io.exe` directly.

## Canonical documentation

- [diagrams.net documentation](https://www.drawio.com/doc/)
- [draw.io desktop releases](https://github.com/jgraph/drawio-desktop/releases/latest)
