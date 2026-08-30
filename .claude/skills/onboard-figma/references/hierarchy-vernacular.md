# Reference: Figma hierarchy vernacular

The words designers use for the parts of a Figma file, and the rules for what nests inside what.

## Scope

The containers — the things that hold other things. Everything in a Figma file lives inside one of
these, so knowing the eight of them and their nesting rules is enough to read any file's structure
and to talk about it the way the designer who built it does.

Out of scope on purpose: shapes and text, FigJam objects, component properties, and the
non-container vocabulary (auto layout, constraints, styles, variables). Those are all *things
inside* the containers, not the hierarchy.

Neighbours own adjacent ground, so this file doesn't repeat them:

- Calling the API and what it costs → `references/rest-api.md`
- Turning a Figma URL into a file key and a layer id → `references/design-url-decoding.md`

Verified against Figma's docs on 2026-08-30.

## The tree

```
File
└── Page                         a tab in the left sidebar; each page has its own canvas
    ├── Section                  a labelled region of the canvas
    │   └── Frame                a screen
    ├── Frame                    a top-level frame — a screen, an artboard
    │   ├── Frame                a nested frame — a card, a nav bar
    │   │   ├── Group
    │   │   └── Instance         a copy of a component…
    │   │       └── …            …with its own contents inside
    │   └── Group
    └── Component set            the variants of one component, grouped
        ├── Component            Size=Large, State=Default
        └── Component            Size=Small, State=Default
```

**A file's first two levels are fixed** — a file holds pages, and a page holds everything else.
Below that, nesting is arbitrary and often ten or more deep.

Figma nests child objects inside their parent frame or group, which is what lets you collapse and
expand them in the layers panel. Order in the panel is z-order: **the topmost row is the frontmost
layer.**

## The eight containers

| Container | What it is | Holds | Lives in |
| --- | --- | --- | --- |
| **File** | The whole document, one Figma URL | Pages | A project |
| **Page** | A tab in the left sidebar. "Each page is its own canvas" | Anything | A file |
| **Section** | A labelled region for grouping related work and guiding collaborators | Anything, including other sections | **A page only** — never a frame or group |
| **Frame** | A container with its own size and properties. The workhorse | Anything | Anywhere |
| **Group** | Layers combined so they move as one | Anything | Anywhere |
| **Component** | The main component — "defines the properties of the component" | Anything | Anywhere |
| **Component set** | The variants of one component kept together — its states, sizes, or colours | Components, each one a variant (`Size=Large, State=Hover`) | Anywhere |
| **Instance** | "A copy of the component you can reuse in your designs" | Its own contents, inherited from the component | Anywhere |

Two nesting rules are worth committing to memory, because they're the only real constraints:

- **Sections are page-level.** A section can contain frames, groups, even other sections — but a
  section can never sit inside a frame or a group. If something looks like a section nested in a
  frame, it's a frame.
- **Everything else nests freely.** A frame in a group in a component in a frame is legal, common,
  and usually accidental.

## Frame vs group vs section

These three look alike in the layers panel and behave nothing alike. This is the distinction that
causes the most confusion in a handoff conversation.

| | Frame | Group | Section |
| --- | :-: | :-: | :-: |
| Sets its own size | ✓ | – (takes its children's size) | ✓ |
| Has its own fills, corners, shadows | ✓ | – | fills only |
| Auto layout, constraints, layout grids | ✓ | – | – |
| Clips content that overflows | ✓ | – | – |
| Can be marked ready for development | ✓ | – | ✓ |
| Can sit inside a frame or group | ✓ | ✓ | **✗** |

Figma's own framing: groups "are collections of layers and not distinct elements, so they don't
have dimensions or properties of their own", and their bounds adjust to fit whatever is inside
them. Frames, by contrast, "can have dimensions and properties of their own—like fills, rounded
corners, and shadows", plus "auto layout, constraints, and layout grids, that allow you to control
or influence the layers inside them."

**In practice: a group is bookkeeping, a frame is design.** If a designer says "wrap it in a
frame," they want layout behaviour. If they say "group it," they just want the layers panel tidier.

## Frames, screens, and artboards

**A top-level frame is a screen.** Top-level frames "sit directly on the canvas" rather than inside
another object; Figma bolds them in the layers panel and shows their name on the canvas. These are
the things a developer usually means by "a screen" and a designer might call an artboard — Figma
has no separate artboard concept, it's just a frame at the top level.

A **nested frame** is a frame placed inside another frame or object, making it "both a parent and a
child at the same time" — a card, a nav bar, a list row.

So "frame" alone is ambiguous in conversation and worth disambiguating: a whole screen, or a
component-sized box inside one. Ask which.

## Words that mean two things

| Word | Sense one | Sense two |
| --- | --- | --- |
| **canvas** | "The file's main workspace where you can create and manipulate designs" — the infinite surface | The surface belonging to *one page*. Every page has its own |
| **layer** | Any row in the layers panel — frames and groups included | Casually, a *leaf*: a shape or text, as opposed to a container |
| **page** | A tab in the left sidebar | Never a screen. A screen is a top-level frame |
| **component** | The main component — what a designer usually means | The instance sitting in a design — what a developer usually means |

> **"On the canvas" means top level.** When a designer says something is on the canvas, they mean
> it isn't nested inside a frame or group — not that it's on the drawing surface generally, since
> everything is.

## Source of truth

- [Explore design files](https://help.figma.com/hc/en-us/articles/15297425105303-Explore-design-files) — the canvas, pages, and the shape of a file
- [Frames in Figma Design](https://help.figma.com/hc/en-us/articles/360041539473-Frames-in-Figma-Design) — top-level vs nested frames
- [The difference between frames and groups](https://help.figma.com/hc/en-us/articles/360039832054-The-difference-between-frames-and-groups)
- [Organize your canvas with sections](https://help.figma.com/hc/en-us/articles/9771500257687-Organize-your-canvas-with-sections)
- [View layers and pages in the left sidebar](https://help.figma.com/hc/en-us/articles/360039831974-View-layers-and-assets-in-the-Layers-Panel) — nesting and the layers panel
- [Guide to components in Figma](https://help.figma.com/hc/en-us/articles/360038662654-Guide-to-components-in-Figma) · [Create and use variants](https://help.figma.com/hc/en-us/articles/360056440594-Create-and-use-variants)
