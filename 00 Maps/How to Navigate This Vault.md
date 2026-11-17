---
tags: [map, guide]
---
# How to Navigate This Vault

## 1. Open it
Obsidian → **Open folder as vault** → choose `~/Documents/NWIS Learning Vault`. Start at [[Home]].

## 2. Structure
| Folder | What's in it |
|---|---|
| `00 Maps` | Entry points: [[Home]], the MOCs (maps of content), [[Learning Path]] |
| `01 System Overview` | The whole system in one page: [[System Overview]], [[Integration Landscape]] |
| `02 Codebases` | One folder per repo, starting at its repo-map note |
| `03 Concepts` | Foundations: HTTP, transactions, DI, state... |
| `04 Design Patterns` | Named patterns and where they appear |
| `05 Best Practices` | Conventions worth copying |
| `06 Legacy to Modern` | What changed in the rewrite, and why |

## 3. Three ways to read
- **Guided:** follow [[Learning Path]] top to bottom.
- **By repo:** open a repo map from [[Codebases MOC]], then follow its links.
- **By idea:** pick a topic from [[Concepts MOC]] or [[Design Patterns MOC]]; each note lists where the codebase uses it.

Tip: every codebase note ends with `## Related`; every concept note has `In this codebase`. Click both ways. A codebase note is an **example**, a concept note is the **theory**.

## 4. The graph
Open it with **Cmd+G** (global graph) or the graph icon in the sidebar.

**Colours** (already configured): gold = Maps, red = System Overview, blue = Codebases, green = Concepts, purple = Patterns, orange = Practices, grey = Legacy.

**Local graph** is the most useful view: open any note → right-click its tab → **Open local graph**. Set **Depth** to 2 to see its neighbourhood.

**Controls in the graph panel (gear icon):**
- **Filters → Search:** type `path:"02 Codebases/tgmdp_backend"` to show one repo. Try `tag:#concept`, or `-path:"02 Codebases/Legacy"` to hide legacy.
- **Groups:** edit colours or add your own queries.
- **Display:** *Arrows* shows link direction; *Text fade threshold* shows labels at lower zoom; *Node size* and *Link thickness* adjust readability.
- **Forces:** *Repel force* up spreads clusters out; *Link distance* up lengthens edges; *Center force* up pulls everything in.
- **Animate:** replays how the vault grew, in order of creation.
- **Orphans** toggle (Filters): turn off to hide unlinked notes.

**Reading the shape:** big hubs (many links) are the ideas the code leans on most, for example [[Layered Architecture]] or [[Dependency Injection]]. Clusters are repos; the bridges between clusters are shared concepts. Dense concept nodes sitting between *Backend* and *Frontend* clusters show what the two sides have in common.

## 5. Other handy features
- **Cmd+O** quick-switcher, **Cmd+Shift+F** search across everything.
- **Backlinks panel** (right sidebar): for a concept, lists every note that uses it.
- **Hover** a link while holding Cmd to preview it without leaving the note.
- Mermaid diagrams render in reading view; switch with **Cmd+E**.
- **Bookmarks:** star [[Learning Path]] and [[Home]].

## 6. Make it yours
Add your own notes beside these and link them by typing two opening square brackets and a note title. They join the graph automatically. A good habit: after reading a codebase note, write a 3-line summary in your own words in a new note and link to it.

Back to [[Home]].
