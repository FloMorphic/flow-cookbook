# Qdrant Migrate

**Re-embed a knowledge base from one model into another — with one upsert node.** A source Qdrant collection was vectored long ago under some embedding model. This flow reads its points, re-embeds each one's text with a *new* model (here `gemini-embedding-2`, 3072-dim), copies the payload tags across, and writes the result into a fresh collection. The whole per-document transfer is **one node** — the runtime runs it once per element because its `scope` is a many-valued JSONPath. No loop node. No map step. Cardinality is a property of the node, not a construct you build.

> Full write-up: [Cardinality Is the Loop You Didn't Write](https://inflowenger.com/blog/cardinality-is-the-loop-you-didnt-write)

**Teaches:** `scope` cardinality (a many-valued scope runs a node once per element) · `$this` — the current element inside a cardinal node · payload fields **bound from runtime values** (`{{$this.payload.text}}`, `{{$this.payload.component}}`) so each written point is shaped by the document it came from · a plugin node reaching an external store the runtime was never compiled against · (expansion) a scroll **cursor** + a backward edge that sweeps an entire collection.

---

## The plugin is installed, not compiled in

This recipe needs the **Qdrant plugin**. It isn't built into the FloMorphic or Inflowenger runtime — it's a [plugin](https://inflowenger.com/blog/extend-like-an-extension) you install, and it can run *anywhere it can reach your Qdrant*: on the same box, on your network, next to a database that has held vectors since long before this flow existed. It registers three actions this flow uses — `qdrant.collection.create`, `qdrant.points.upsert`, `qdrant.points.scroll` — plus a `qdrant.meta.collections` picker the config forms call to list collections. Install it and point it at your Qdrant before importing.

---

## Topology

```
Start
  ↓
Create collection      plugin · qdrant.collection.create → selected_docs (3072-dim, Cosine)
  ↓
Upsert points          plugin · qdrant.points.upsert   ← the cardinal node
  scope: $.result.result[*]        runs ONCE PER source point
  text: {{$this.payload.text}}     re-embedded with the new model
  tags: component ← {{$this.payload.component}}
  ↓
Scroll points          plugin · qdrant.points.scroll   ← the expansion (see below)
  offset: {{$.scrolled.result.next_page_offset}}
```

The spine that runs and does the work is **Create collection → Upsert points**. `Scroll points` is wired in to show where the pattern goes next — turn it into a loop and it sweeps the whole source collection (see [The expansion](#the-expansion-a-cursor-and-a-backward-edge)).

---

## How it works

- **Create collection** (`plugin`, key `new_coll`) calls `qdrant.collection.create` once, making `selected_docs` with `size: 3072` (the dimensionality of `gemini-embedding-2`) and `distance: Cosine`. Vector size is a property of the *target* model — this is the number that has to change when you migrate between embedding models, and it's why re-embedding is a real operation and not a copy.

- **Upsert points** (`plugin`, key `transfer`, **`scope: $.result.result[*]`**) is the whole point of the recipe. Its scope is a *many-valued* JSONPath — every element of the source array `$.result.result[*]`. A single-valued scope would run the node once; a many-valued one runs it **once per element**, each run seeing its own element as **`$this`**. So:
  - `text: {{$this.payload.text}}` — the current point's stored text, handed to the new embedding model. The plugin embeds it and stores the vector, keeping `payload.text` so results still carry their source.
  - `tags: component ← {{$this.payload.component}}` — a payload tag **initialised at runtime from the element itself**. Each written point is shaped by the document it came from; nothing is hardcoded. This is the mechanic to take away: a node's fields are templates, and inside a cardinal node those templates resolve against the *current* element.
  - `collection: selected_docs`, `wait: true` — write into the collection the previous node made, and return only once applied.

- **`$.result.result[*]` is the input contract.** The cardinal node reads its source array from `$.result.result[*]` — the points to migrate. In this export that array is supplied to the flow as its starting Context (e.g. the result of a prior scroll/search you paste in or pipe from an upstream node). Give the flow a Context whose `$.result.result` is an array of `{ payload: { text, component, ... } }` objects and each becomes one written point.

### The expansion: a cursor and a backward edge

The interesting property of `qdrant.points.scroll` is that it hands back a **`next_page_offset`** — a cursor into the collection. **Scroll points** feeds that cursor straight back into its own request:

```
offset: {{$.scrolled.result.next_page_offset}}
```

Read one page, and the *next* run of the same node reads from where the last one stopped. Wire a backward edge so the flow returns to the scroll after each page is transferred, and the two-node transfer becomes a **pump that walks the entire collection** — page, re-embed every point, page again — until `next_page_offset` comes back empty. A loop, in FloMorphic, [is an edge you draw](https://inflowenger.com/blog/no-prompt-can-invent-a-third-edge), not a node you configure. This recipe ships the transfer and the cursor; closing the loop is left as the one line you add to turn a sample into a migration.

---

## Run it

1. Install the **Qdrant plugin** and point it at your Qdrant instance.
2. Import `qdrant-migrate.json` into the FloMorphic canvas.
3. On each plugin node, select your **plugin settings profile** (this export ships with none — no credentials on the graph) and confirm the collection names: source in **Scroll points** (`chat_kb_demo`), target in **Create collection** / **Upsert points** (`selected_docs`).
4. Confirm the target **vector size** matches your new embedding model (`3072` for `gemini-embedding-2`) and that the plugin's configured embedding provider *is* that new model — the re-embed happens inside the upsert.
5. Provide a starting Context whose `$.result.result` is an array of source points (each with `payload.text` and `payload.component`). Hit **Run** — `Create collection` fires once, `Upsert points` fires once per element, and each write traces onto the diagram.

---

## Notes on this export

- The `settingsId` on every node (an install-specific plugin/credential reference) was **removed**, so no credential travels with the flow — select your own plugin settings profile after import (step 3).
- A stray trailing space in the upsert node's `scope` (`"$.result.result[*] "` as raw-exported) was **trimmed** to `"$.result.result[*]"` so the JSONPath resolves cleanly.
- The flow is kept as exported otherwise — node ids, titles, positions, and the `Scroll points` node — so the canvas matches the screenshot and the "expansion" section has something real to point at. The shipped edges (`start → create → upsert → scroll`) do **not** close the scroll loop; that backward edge is the one line you add to go from sample to full-collection migration.
- Vector size `3072` and the `gemini-embedding-2` framing are the demo's target model; change both together when you migrate to a different one.
