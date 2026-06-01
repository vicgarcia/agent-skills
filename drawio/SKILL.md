---
name: drawio
description: Create and edit draw.io diagram files (.drawio) by writing XML directly. Use when the user asks to create, modify, or update any diagram — flowcharts, architecture diagrams, process flows, decision trees, org charts, ER diagrams, UML, network diagrams, or any other visual diagram.
compatibility: No dependencies required. Produces .drawio XML files readable by draw.io desktop, the diagrams.net browser editor, and the VS Code draw.io extension.
---

# draw.io Diagram Files

Create and edit `.drawio` files by generating XML directly. No CLI, no external tools — just write the file.

---

## File Format

Use the simplified `<mxGraphModel>` format (no outer `<mxfile>` wrapper needed). draw.io accepts both; this format is easier to generate and less prone to errors.

```xml
<mxGraphModel>
  <root>
    <mxCell id="0" />
    <mxCell id="1" parent="0" />
    <!-- diagram elements here, all with parent="1" -->
  </root>
</mxGraphModel>
```

The two structural cells (`id="0"` and `id="1"`) are **mandatory**. Every diagram element is a child with `parent="1"` (or the ID of a container).

Use the full `<mxfile>` wrapper only when the diagram needs multiple pages:

```xml
<mxfile>
  <diagram id="page-1" name="Page 1">
    <mxGraphModel> ... </mxGraphModel>
  </diagram>
  <diagram id="page-2" name="Page 2">
    <mxGraphModel> ... </mxGraphModel>
  </diagram>
</mxfile>
```

**Always generate uncompressed, plain XML.** Never generate Base64 or compressed content.

---

## Critical Rules

1. IDs must be unique within the diagram — use descriptive strings (`"start"`, `"decision-1"`) or sequential numbers
2. Vertices require `vertex="1"`, edges require `edge="1"` — never both on the same cell
3. Every edge cell must contain `<mxGeometry relative="1" as="geometry" />` as a child — self-closing edge cells are invalid
4. Edge `source` and `target` must reference existing vertex IDs
5. Children of containers use coordinates **relative to the container**, not the canvas
6. HTML in `value` attributes must be XML-escaped: `<` → `&lt;`, `>` → `&gt;`, `&` → `&amp;`, `"` → `&quot;`
7. Never include XML comments (`<!-- -->`) in generated output
8. Coordinates: `(0,0)` is top-left, x increases right, y increases down

---

## Shape Reference

### Vertices

Every shape is an `mxCell` with `vertex="1"` and an `<mxGeometry>` child.

**Rectangle (process / step)**
```xml
<mxCell id="proc1" value="Process" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="140" height="60" as="geometry" />
</mxCell>
```

**Diamond (decision)**
```xml
<mxCell id="dec1" value="Condition?" style="rhombus;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="200" width="140" height="80" as="geometry" />
</mxCell>
```

**Ellipse (start / end terminal)**
```xml
<mxCell id="start" value="Start" style="ellipse;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="40" width="100" height="60" as="geometry" />
</mxCell>
```

**Parallelogram (input / output)**
```xml
<mxCell id="io1" value="Input Data" style="shape=parallelogram;perimeter=parallelogramPerimeter;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="140" height="60" as="geometry" />
</mxCell>
```

**Cylinder (database / storage)**
```xml
<mxCell id="db1" value="Database" style="shape=cylinder3;whiteSpace=wrap;html=1;boundedLbl=1;backgroundOutline=1;size=15;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="100" height="70" as="geometry" />
</mxCell>
```

**Document**
```xml
<mxCell id="doc1" value="Report" style="shape=document;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="120" height="80" as="geometry" />
</mxCell>
```

**Hexagon**
```xml
<mxCell id="hex1" value="Step" style="shape=hexagon;perimeter=hexagonPerimeter2;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="120" height="70" as="geometry" />
</mxCell>
```

**Actor (person)**
```xml
<mxCell id="user1" value="User" style="shape=umlActor;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="40" height="70" as="geometry" />
</mxCell>
```

**Cloud**
```xml
<mxCell id="cloud1" value="Internet" style="shape=cloud;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="140" height="80" as="geometry" />
</mxCell>
```

**Note / annotation**
```xml
<mxCell id="note1" value="See spec v2" style="shape=note;whiteSpace=wrap;html=1;size=15;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="120" height="60" as="geometry" />
</mxCell>
```

**Text label (no border)**
```xml
<mxCell id="lbl1" value="Section Title" style="text;html=1;align=center;fontStyle=1;fontSize=14;" vertex="1" parent="1">
  <mxGeometry x="100" y="40" width="200" height="30" as="geometry" />
</mxCell>
```

### Standard sizes

| Shape | Width | Height |
|-------|-------|--------|
| Rectangle | 140 | 60 |
| Diamond | 140 | 80 |
| Ellipse (terminal) | 100 | 60 |
| Cylinder | 100 | 70 |
| Document | 120 | 80 |
| Parallelogram | 140 | 60 |

---

## Connector Reference

Every edge must include `<mxGeometry relative="1" as="geometry" />` — never self-close an edge cell.

**Orthogonal (right-angle) — use for flowcharts and architecture**
```xml
<mxCell id="e1" value="" style="edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;" edge="1" source="proc1" target="dec1" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

**Labeled edge**
```xml
<mxCell id="e2" value="Yes" style="edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;" edge="1" source="dec1" target="proc2" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

**Always add a label background** on any labeled edge where the line may cross other lines — prevents the label from being unreadable against the line stroke:
```
edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;labelBackgroundColor=#ffffff;labelBorderColor=none;
```

**Straight line (no routing) — use for UML class diagrams**
```xml
<mxCell id="e3" value="" style="html=1;" edge="1" source="class1" target="class2" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

**Dashed edge (optional / weak relationship)**
```xml
<mxCell id="e4" value="" style="edgeStyle=orthogonalEdgeStyle;dashed=1;html=1;" edge="1" source="a" target="b" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

**Bidirectional arrow**
```xml
<mxCell id="e5" value="" style="edgeStyle=orthogonalEdgeStyle;startArrow=classic;startFill=1;endArrow=classic;endFill=1;html=1;" edge="1" source="a" target="b" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

**No arrowhead (plain line)**
```xml
<mxCell id="e6" value="" style="edgeStyle=orthogonalEdgeStyle;endArrow=none;html=1;" edge="1" source="a" target="b" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

**Entity-relation style — use for ERDs**
```xml
<mxCell id="e7" value="" style="edgeStyle=entityRelationEdgeStyle;html=1;" edge="1" source="entity1" target="entity2" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

### Edge style reference

| Style | Use for |
|-------|---------|
| `edgeStyle=orthogonalEdgeStyle` | Flowcharts, architecture, process flows |
| `edgeStyle=entityRelationEdgeStyle` | ER diagrams |
| (no edgeStyle) | UML class, direct connections |
| `curved=1` | Mind maps, informal flows |

Use one consistent edge style throughout a diagram.

### Arrow markers

| `endArrow=` value | Appearance |
|-------------------|------------|
| `classic` | Filled triangle (default) |
| `open` | Open/hollow triangle |
| `block` | Filled block |
| `oval` | Circle |
| `diamond` | Filled diamond |
| `none` | No arrowhead |

Use `endFill=0` for hollow variants (e.g. UML aggregation: `endArrow=diamond;endFill=0`).

### Waypoints (explicit edge routing)

When the auto-router produces crossings or bad paths, force an edge through specific canvas coordinates using `<Array as="points">` inside `<mxGeometry>`. Each `<mxPoint>` is an absolute canvas coordinate the edge must pass through.

```xml
<mxCell id="e1" value="" style="edgeStyle=orthogonalEdgeStyle;html=1;" edge="1" source="svc-a" target="svc-b" parent="1">
  <mxGeometry relative="1" as="geometry">
    <Array as="points">
      <mxPoint x="120" y="400" />
      <mxPoint x="120" y="520" />
    </Array>
  </mxGeometry>
</mxCell>
```

Use waypoints to route an edge **left or right of a column** rather than through it — the most common fix for same-column congestion. Two points at the same x forms a vertical detour; two at the same y forms a horizontal detour.

### Exit / entry constraints

Control which face of a node an edge connects to using `exitX`, `exitY`, `entryX`, `entryY` style params. Values are 0–1 representing normalized position on the bounding box edge.

| Style params | Meaning |
|---|---|
| `exitX=0.5;exitY=1;exitDx=0;exitDy=0;` | Exit from bottom center |
| `exitX=1;exitY=0.5;exitDx=0;exitDy=0;` | Exit from right middle |
| `exitX=0;exitY=0.5;exitDx=0;exitDy=0;` | Exit from left middle |
| `exitX=0.5;exitY=0;exitDx=0;exitDy=0;` | Exit from top center |
| `entryX=0.5;entryY=0;entryDx=0;entryDy=0;` | Enter from top center |
| `entryX=0;entryY=0.5;entryDx=0;entryDy=0;` | Enter from left middle |

Always pair `exitX/Y` with `exitDx=0;exitDy=0` and `entryX/Y` with `entryDx=0;entryDy=0` — the Dx/Dy offsets are required, even at zero.

Example — force a same-column edge to exit left and enter left, routing it around the column:
```xml
<mxCell id="e1" value="" style="edgeStyle=orthogonalEdgeStyle;html=1;exitX=0;exitY=0.5;exitDx=0;exitDy=0;entryX=0;entryY=0.5;entryDx=0;entryDy=0;" edge="1" source="svc-a" target="svc-b" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

Combine exit/entry constraints with waypoints for full control over a problem edge.

---

## Layout Guidelines

### Grid

Use this rigid grid for placement. Do arithmetic mentally and write coordinates directly into XML — do not narrate coordinate calculations.

- Column x = `col_index × 180 + 40` (col 0 = 40, col 1 = 220, col 2 = 400, …)
- Row y = `row_index × 120 + 40` (row 0 = 40, row 1 = 160, row 2 = 280, …)

For wider or taller shapes, add extra spacing (e.g. add 60 to y-spacing when using tall diamonds).

### Flow orientation

- **Top-down**: steps flow down the page, decisions branch left/right — best for sequential processes, flowcharts
- **Left-right**: steps flow across the page — best for pipelines, timelines, data flows

Pick one and commit. Do not mix orientations within a diagram.

### Minimum spacing

- 40px between sibling nodes
- 20px clearance from container walls to child nodes

### Routing corridors

For diagrams with many edges — especially architecture and service diagrams — reserve intentional empty columns or rows as routing space. A corridor is simply a column or row with no nodes; the auto-router uses that space to route edges without crossing nodes.

- **Top-down layout**: leave a gap row between service groups where horizontal cross-group edges will travel
- **Left-right layout**: leave a gap column between stages where vertical routing will occur
- No special XML is needed — just skip that col/row index when placing nodes

**Same-column and same-row edge conflicts**: when two nodes are stacked in the same column (top-down flow) and need to connect to each other, the auto-router routes the edge through adjacent cells and frequently produces crossings. Resolution in order of preference:
1. Place the target in an adjacent column so the edge travels through empty routing space
2. Use explicit waypoints to route the edge left or right around the column
3. Use exit/entry constraints to force the edge out a specific side, then use waypoints to clear the column

---

## Containers and Groups

### Swimlane (titled container)

Use for grouping by actor (who), system boundary, or phase.

```xml
<mxCell id="lane1" value="Customer" style="swimlane;startSize=30;fillColor=#f5f5f5;strokeColor=#666666;html=1;" vertex="1" parent="1">
  <mxGeometry x="0" y="0" width="800" height="150" as="geometry" />
</mxCell>
<!-- Child nodes use coordinates relative to the lane -->
<mxCell id="n1" value="Place Order" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="lane1">
  <mxGeometry x="120" y="45" width="140" height="60" as="geometry" />
</mxCell>
```

Stack lanes vertically (same x=0, increasing y). Cross-lane edges must have `parent="1"`, not a lane ID.

**Swimlane defaults:**
- `startSize=30` — header height
- `horizontal=0` — header on the left side instead of top (vertical lane label)
- Child nodes: `y=45` (centered in 150px-tall lane), `x=120` (clears the 110px label area when `horizontal=0`)

### Invisible group container

```xml
<mxCell id="grp1" value="" style="group;" vertex="1" parent="1">
  <mxGeometry x="100" y="100" width="320" height="200" as="geometry" />
</mxCell>
<mxCell id="c1" value="Service A" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="grp1">
  <mxGeometry x="20" y="20" width="120" height="60" as="geometry" />
</mxCell>
```

### Zone / region background (non-container)

Use a plain rectangle placed **before** foreground nodes in the XML to delineate areas visually — without the parent-child coordinate complexity of swimlanes. Since XML order = z-order, the zone renders behind anything declared after it.

```xml
<!-- Zone background — declared first, renders under all sibling nodes -->
<mxCell id="zone-api" value="API Layer" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#f0f8ff;strokeColor=#6c8ebf;strokeWidth=1;fontSize=11;fontStyle=1;verticalAlign=top;opacity=60;" vertex="1" parent="1">
  <mxGeometry x="20" y="20" width="520" height="180" as="geometry" />
</mxCell>
<!-- Foreground nodes — same parent="1", absolute coordinates that fall inside the zone -->
<mxCell id="svc-a" value="Service A" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;" vertex="1" parent="1">
  <mxGeometry x="60" y="80" width="140" height="60" as="geometry" />
</mxCell>
```

Zone nodes have `parent="1"` and use absolute canvas coordinates — they are **siblings**, not parents, of the nodes inside them. This keeps edge routing simple: all edges also have `parent="1"` and no container scoping applies.

Recommended zone style:
```
rounded=1;whiteSpace=wrap;html=1;fillColor=#f5f5f5;strokeColor=#999999;opacity=60;verticalAlign=top;fontSize=11;fontStyle=1;
```

Use zones instead of swimlanes when: the grouping is purely visual, edges cross zone boundaries freely, or nested container coordinates would be inconvenient.

### Nested architecture containers

Use nested swimlanes for hierarchical architecture (region → zone → service):

```xml
<mxCell id="vpc" value="VPC" style="swimlane;startSize=24;fillColor=#dae8fc;strokeColor=#6c8ebf;html=1;" vertex="1" parent="1">
  <mxGeometry x="0" y="0" width="600" height="300" as="geometry" />
</mxCell>
<mxCell id="az" value="Availability Zone" style="swimlane;startSize=24;fillColor=#fff2cc;strokeColor=#d6b656;html=1;" vertex="1" parent="vpc">
  <mxGeometry x="20" y="36" width="260" height="240" as="geometry" />
</mxCell>
<mxCell id="svc" value="API Server" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="az">
  <mxGeometry x="60" y="60" width="140" height="60" as="geometry" />
</mxCell>
<!-- Cross-container edges go at parent="1" -->
<mxCell id="e1" edge="1" parent="1" source="svc" target="db1" style="edgeStyle=orthogonalEdgeStyle;html=1;">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

---

## Style Palettes

Apply colors by adding `fillColor` and `strokeColor` to any shape's style string. All palettes work on any shape type.

### neutral

Clean, professional, appropriate for any context.

| Role | fillColor | strokeColor | fontColor |
|------|-----------|-------------|-----------|
| Start/end | `#f5f5f5` | `#666666` | `#333333` |
| Process | `#ffffff` | `#666666` | `#333333` |
| Decision | `#f5f5f5` | `#666666` | `#333333` |
| I/O | `#ffffff` | `#999999` | `#333333` |
| Data/storage | `#f0f0f0` | `#666666` | `#333333` |

### blueprint

Blue-toned, technical, good for software/system diagrams.

| Role | fillColor | strokeColor | fontColor |
|------|-----------|-------------|-----------|
| Start/end | `#d5e8d4` | `#82b366` | `#000000` |
| Process | `#dae8fc` | `#6c8ebf` | `#000000` |
| Decision | `#fff2cc` | `#d6b656` | `#000000` |
| I/O | `#e1d5e7` | `#9673a6` | `#000000` |
| Data/storage | `#f8cecc` | `#b85450` | `#000000` |

This palette maps semantic roles to color — use it consistently:
- **Green** = start, success, complete
- **Blue** = process, action, system
- **Yellow** = decision, condition, warning
- **Purple** = input, output, interface
- **Red** = end, error, terminal

### minimal

No fill, borders only — clean and printable.

| Role | fillColor | strokeColor | fontColor |
|------|-----------|-------------|-----------|
| All shapes | `none` | `#000000` | `#000000` |
| Emphasis | `none` | `#000000` | `#000000` (use `fontStyle=1` for bold) |

### sketch

Hand-drawn look via rough.js. Add `sketch=1` to any shape's style.

```
rounded=1;whiteSpace=wrap;html=1;sketch=1;fillColor=#fff2cc;strokeColor=#d6b656;
```

Combine sketch with any palette colors above. Keep it consistent — apply `sketch=1` to all shapes in the diagram.

### Applying a palette

When the user names a palette (e.g. "use blueprint"), apply the color pairs from that palette to all shapes based on their semantic role. When the user specifies colors or describes a style, derive a consistent palette from their description and apply it uniformly.

**Example — blueprint process step:**
```
rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;
```

**Example — blueprint decision:**
```
rhombus;whiteSpace=wrap;html=1;fillColor=#fff2cc;strokeColor=#d6b656;
```

---

## Visual Emphasis

### Stroke width

`strokeWidth` controls border thickness. Default is `2` for most shapes. Use it to draw the eye to critical nodes.

```
strokeWidth=3;   <!-- subtle emphasis -->
strokeWidth=4;   <!-- strong emphasis — use sparingly, one or two nodes max -->
```

Example — API gateway as the focal point of an architecture diagram:
```
rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;strokeWidth=4;
```

### Font style bitmask

`fontStyle` is a bitmask — add values to combine effects:

| Value | Effect |
|-------|--------|
| `0` | Normal (default) |
| `1` | Bold |
| `2` | Italic |
| `4` | Underline |
| `3` | Bold + Italic |
| `5` | Bold + Underline |
| `6` | Italic + Underline |

```
fontStyle=1;   <!-- bold — use for section titles and key nodes -->
fontStyle=2;   <!-- italic — use for annotations and notes -->
```

---

## Common Element Patterns

These are composable building blocks, not locked workflows.

### Decision branch

One decision diamond with Yes/No edges leading to two paths that reconverge:

```xml
<mxCell id="d1" value="Approved?" style="rhombus;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="220" y="160" width="140" height="80" as="geometry" />
</mxCell>
<mxCell id="yes" value="Approve" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="40" y="300" width="140" height="60" as="geometry" />
</mxCell>
<mxCell id="no" value="Reject" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="1">
  <mxGeometry x="400" y="300" width="140" height="60" as="geometry" />
</mxCell>
<mxCell id="e1" value="Yes" style="edgeStyle=orthogonalEdgeStyle;html=1;" edge="1" source="d1" target="yes" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
<mxCell id="e2" value="No" style="edgeStyle=orthogonalEdgeStyle;html=1;" edge="1" source="d1" target="no" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

### Loop back

An edge from a later step back to an earlier step. Use a labeled edge with `edgeStyle=orthogonalEdgeStyle` — the router handles the path automatically.

```xml
<mxCell id="retry" value="Retry" style="edgeStyle=orthogonalEdgeStyle;html=1;" edge="1" source="fail" target="start" parent="1">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

### Parallel tracks (fork/join)

Model with horizontal spread — fork from one node to several siblings, join them back to a single node below:

```
         [Fork]
        /   |   \
    [A]   [B]   [C]
        \   |   /
         [Join]
```

Use standard spacing; edges route automatically.

### Hierarchical parent-child

Use nested containers or a tree layout with edges flowing downward. For org charts and trees, place the parent at `row 0` and children at `row 1` spread across columns.

### Swimlane flow (actor-based)

Stack horizontal swimlanes; place process nodes inside the appropriate lane; draw cross-lane edges at `parent="1"`.

---

## Editing Existing Files

When the user asks to modify an existing `.drawio` file:

1. **Read the file as XML** — identify existing cells by their `id`, `value`, and shape style
2. **Understand the structure** — note containers, layers, and parent-child relationships before making changes
3. **Targeted edits**:
   - Add a node: create a new `mxCell` with a unique ID not already in the file
   - Remove a node: delete its `mxCell` and any edges that reference its ID as source or target
   - Modify a node: update `value`, `style`, or `mxGeometry` attributes in place
   - Reroute an edge: update its `source` or `target` attributes
   - Rename a label: update the `value` attribute
4. **Preserve existing IDs** — never reassign an ID that already exists
5. **Preserve existing layout** — when adding nodes, position them consistently with the existing grid and spacing; don't move nodes that aren't being changed

When the change is substantial (restructuring, new sections, major layout shift), a full redraw may be cleaner than targeted edits — use judgment based on scope.

---

## Agent Instructions

### Interpret intent first

Before writing any XML, identify:
- **Create or edit?** — Does a file exist to modify, or is this a new diagram?
- **What elements** — List all nodes and connections described by the user
- **What structure** — Is grouping (swimlanes, containers, nested layers) appropriate?
- **What palette** — Has the user named one? Described a style? Default to `blueprint` if unspecified.
- **What orientation** — Top-down or left-right?

### Pre-write step

Before emitting XML:
- Enumerate all nodes with their semantic role (start, process, decision, I/O, data, end)
- Assign an ID to each (descriptive strings preferred)
- Assign a `(col, row)` grid position to each node
- **Identify routing corridors**: for diagrams with 6+ nodes or cross-group edges, note which columns or rows will be left empty as routing space — especially when same-column/row nodes need to connect laterally
- **Flag problematic edges**: any edge between nodes in the same column (top-down) or same row (left-right) will conflict with the auto-router; plan to use explicit waypoints or exit/entry constraints for these before writing
- List all edges with source ID → target ID and label (if any); mark which edges need label backgrounds, waypoints, or constraints

Then write XML directly from this plan — do not narrate coordinate calculations in prose.

### Write the file

Write the complete XML to a `.drawio` file. Use the simplified `<mxGraphModel>` format unless multiple pages are needed.

Name files descriptively: `order-flow.drawio`, `system-architecture.drawio`, `user-registration.drawio`.

### Confirm with the user

After writing, briefly describe what was created or changed so the user can verify it matches their intent before opening it.

### Dense diagrams: export-verify loop

Routing problems — crossings, labels overlapping lines, edges clipping nodes — are only visible in the rendered output. For diagrams with 10+ nodes or many cross-column edges, suggest the user export to PNG after the first draft and check layout before adding remaining edges or labels. A single export pass early catches structural routing problems when they are cheap to fix — targeted waypoints and constraints on a half-populated diagram are far easier than cleanup after it is fully built.

---

## Exporting with the draw.io CLI

The draw.io desktop app includes a CLI for headless export of `.drawio` files to PNG, SVG, and PDF. This is optional — the `.drawio` file is the primary output of this skill. Use the CLI when the user asks for an image or PDF export.

### Check if installed

```bash
drawio --version
```

If the command is not found, inform the user and provide install instructions for their platform (see below).

### Export commands

```bash
# PNG (default: 1x scale)
drawio --export --format png --output diagram.png diagram.drawio

# PNG at 2x (high-res / retina)
drawio --export --format png --scale 2 --output diagram@2x.png diagram.drawio

# SVG
drawio --export --format svg --output diagram.svg diagram.drawio

# PDF
drawio --export --format pdf --output diagram.pdf diagram.drawio

# Specific page from a multi-page file (0-indexed)
drawio --export --format png --page-index 0 --output page1.png diagram.drawio

# All pages as separate PNGs
drawio --export --format png --all-pages --output diagram.png diagram.drawio
```

### Key flags

| Flag | Description |
|------|-------------|
| `--export` | Enable export mode |
| `--format` | Output format: `png`, `svg`, `pdf`, `jpg`, `xml` |
| `--output` | Output file path |
| `--scale` | Scale factor (default: 1). Use `2` for high-res |
| `--page-index` | Export a specific page (0-indexed) |
| `--all-pages` | Export all pages |
| `--width` | Fix output width in pixels (scales to fit) |
| `--height` | Fix output height in pixels (scales to fit) |
| `--border` | Border padding in pixels around the diagram |
| `--transparent` | Transparent background (PNG only) |
| `--quality` | JPEG quality 1–100 (JPG only) |

### Linux headless requirement

On Linux servers and containers with no display, the app requires a virtual framebuffer. Install and run with `xvfb-run`:

```bash
sudo apt install xvfb
xvfb-run drawio --export --format png --output diagram.png diagram.drawio
```

---

### Installing the draw.io desktop app

If `drawio --version` fails, inform the user. Provide the instructions for their platform:

**macOS**
```bash
brew install --cask drawio
```

**Linux — Debian/Ubuntu (amd64)**
```bash
# Get the latest version tag from GitHub
VERSION=$(curl -s https://api.github.com/repos/jgraph/drawio-desktop/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
curl -L -o drawio.deb "https://github.com/jgraph/drawio-desktop/releases/download/v${VERSION}/drawio-amd64-${VERSION}.deb"
sudo apt install ./drawio.deb

# Also install Xvfb for headless export
sudo apt install xvfb
```

**Linux — Debian/Ubuntu (arm64)**
```bash
VERSION=$(curl -s https://api.github.com/repos/jgraph/drawio-desktop/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
curl -L -o drawio.deb "https://github.com/jgraph/drawio-desktop/releases/download/v${VERSION}/drawio-arm64-${VERSION}.deb"
sudo apt install ./drawio.deb
sudo apt install xvfb
```

**Linux — RHEL/Fedora (x86_64)**
```bash
VERSION=$(curl -s https://api.github.com/repos/jgraph/drawio-desktop/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
sudo rpm -i "https://github.com/jgraph/drawio-desktop/releases/download/v${VERSION}/drawio-x86_64-${VERSION}.rpm"
```

**Linux — AppImage (any distro, no install needed)**
```bash
VERSION=$(curl -s https://api.github.com/repos/jgraph/drawio-desktop/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
curl -L -o drawio.AppImage "https://github.com/jgraph/drawio-desktop/releases/download/v${VERSION}/drawio-x86_64-${VERSION}.AppImage"
chmod +x drawio.AppImage
# Use ./drawio.AppImage in place of drawio in all export commands
```

**Windows**

Download the installer from the [latest release](https://github.com/jgraph/drawio-desktop/releases/latest) — look for `draw.io-*-windows-installer.exe`. After install, `drawio` is available in the terminal.

---

## Validation Checklist

Before writing the file, verify:

- [ ] `id="0"` and `id="1" parent="0"` structural cells are present
- [ ] All cell IDs are unique
- [ ] Every edge has `<mxGeometry relative="1" as="geometry" />` as a child (not self-closing)
- [ ] Every edge `source` and `target` references an existing vertex ID
- [ ] Children of containers have `parent="<containerId>"` and use relative coordinates
- [ ] HTML in `value` is XML-escaped (`&lt;` not `<`)
- [ ] No XML comments in the output
- [ ] Non-rectangular shapes have matching `perimeter=` in their style (diamond → `rhombusPerimeter`, ellipse → `ellipsePerimeter`, parallelogram → `parallelogramPerimeter`)
- [ ] All vertices have `x`, `y`, `width`, `height` in their `mxGeometry`
- [ ] Edges between same-column nodes (top-down) or same-row nodes (left-right) use explicit waypoints or exit/entry constraints — not left to the auto-router
- [ ] Labeled edges that may cross other lines include `labelBackgroundColor=#ffffff;labelBorderColor=none;` in their style
- [ ] Zone background cells appear **before** the nodes they cover in the XML (z-order = declaration order)
