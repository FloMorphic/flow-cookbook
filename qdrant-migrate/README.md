# Qdrant Migrate

**Walk a whole dataset in an external store, transform every record, write it out.** That shape is behind migrations, backfills, cleanups, re-taggings, format conversions — and this recipe makes it concrete with one instance: re-embedding a Qdrant knowledge base from an old model into a new one. The flow creates a fresh collection, then walks the source **page by page** — re-embedding each point's text with a *new* model (here `gemini-embedding-2`, 3072-dim), cleaning its payload tags across — until the source is exhausted. The point isn't the re-embed; it's the capabilities you'd reuse in any of those cases, shown working together: a **topology loop** (a backward edge that turns the page cursor) sweeping the collection, and **`scope` cardinality** (one upsert node, run once per point) fanning out *within* each page.

![The flow: Start → Create collection → Scroll → Upsert → Page Cursor → Rule, with the Rule's `continue` port looping back to Scroll](retriever-with-qdrant.png)

> Full write-up: [Cardinality Is the Loop You Didn't Write](https://inflowenger.com/blog/cardinality-is-the-loop-you-didnt-write)

**Teaches:** a **topology loop** — a backward edge over the durable Context, terminated by a `rule` node reading a cursor · `scope` cardinality (one node, once per element) · `$this` — the current element inside a cardinal node · payload fields **bound from runtime values** (`{{$this.payload.text}}`, `{{$this.payload.component}}`) · `input` vs `_get` (a counter that accumulates in its own scope while reading the page out of another) · a plugin reaching an external store the runtime was never compiled against.

---

## The plugin is installed, not compiled in

This recipe needs the **Qdrant plugin** (`qdrant-vec`). It isn't built into the FloMorphic or Inflowenger runtime — it's a [plugin](https://inflowenger.com/blog/extend-like-an-extension) you install, and it can run *anywhere it can reach your Qdrant*: on the same box, on your network, next to a database that has held vectors since long before this flow existed. It registers the actions this flow uses — `qdrant.collection.create`, `qdrant.points.scroll`, `qdrant.points.upsert` — plus a `qdrant.meta.collections` picker the config forms call to list collections. Install it and point it at your Qdrant before importing.

---

## Topology

```
Start
  ↓
Create collection   plugin · qdrant.collection.create → selected_docs (3072-dim, Cosine)
  ↓
Scroll source       plugin · qdrant.points.scroll  ←──────────────────────────┐  one page of chat_kb_demo
  ↓                    offset: {{$.source_stage.result.next_page_offset}}       │
Upsert points        plugin · qdrant.points.upsert         ← the cardinal node  │
  scope: $.source_stage.result.points[*]     runs ONCE PER point in the page    │
  text: {{$this.payload.text}}   ·   tag component ← {{$this.payload.component}} │
  ↓                                                                             │
Page Cursor          js   · counter += this page's point count                  │
  ↓                                                                             │
Rule                 rule · reads the cursor from the last scroll               │
  ├─ continue ─────────────────────────────────────────────────────────────────┘  cursor present → loop
  └─ done  → end                                                                    cursor empty → finished
```

The **backward edge** is `Rule --continue--> Scroll source`. That single edge is the loop; everything else is a straight chain.

---

## How it works

- **Create collection** (`plugin`, key `new_coll`) calls `qdrant.collection.create` once, making `selected_docs` with `size: 3072` (the dimensionality of `gemini-embedding-2`) and `distance: Cosine`. Vector size is a property of the *target* model — the one number that has to change when you migrate between embedding models, and the reason re-embedding is a real operation and not a copy.

- **Scroll source collection** (`plugin`, key `source_stage`) calls `qdrant.points.scroll` on `chat_kb_demo` for one page (`limit: 40`). Its offset is **its own last cursor** — `{{$.source_stage.result.next_page_offset}}` — so the first pass (offset absent) reads page one, and every later pass reads from where the previous page stopped. The response lands at `$.source_stage.result` as `{ points: [...], next_page_offset: ... }`.

- **Upsert points** (`plugin`, key `transfer`, **`scope: $.source_stage.result.points[*]`**) is the cardinal node. Its scope is a *many-valued* JSONPath — every point in the page — so the runtime runs it **once per element**, each pass seeing its own point as **`$this`**:
  - `text: {{$this.payload.text}}` — the point's stored text, handed to the new embedding model. The plugin embeds it and stores the vector, keeping `payload.text` so results still carry their source.
  - `tags: component ← {{$this.payload.component}}` — a payload tag **initialised at runtime from the element itself**. Each written point is shaped by the document it came from; nothing is hardcoded. A node's fields are templates, and inside a cardinal node those templates resolve against the *current* element.
  - `collection: selected_docs`, `wait: true` — write into the collection the first node made, and return only once applied.

- **Page Cursor** (`js`, scope `$.migrate_state`) keeps a running count of migrated points. Scoped to `$.migrate_state`, its slice arrives as `input`; it reaches the page *out of* its scope with `_get("$.source_stage.result")` and does `input.counter += stage.points.length`. The last expression (`input`) is the node's output — a clean illustration of **`input` vs `_get`**: accumulate in your own scope, read across into another.

- **Rule** (`rule`, scope `$.source_stage`) is the loop's **termination condition** — and its handlers are its output ports. It has two, `done` and `continue`, and its logic reads the cursor from the last scroll (`input.result?.next_page_offset`): a cursor present routes **`continue`**, an empty cursor routes **`done`**. In FloMorphic a rule firing a tag *is* the branch the flow takes — so this node *is* the "are there more pages?" decision, drawn on the canvas rather than buried in a condition.

The loop turns — scroll a page, fan the upsert across its points, count, check the cursor — until the scroll returns no `next_page_offset`, the Rule routes `done`, and the flow ends with the whole source collection re-embedded into `selected_docs`.

### Cardinality vs. the topology loop — both, here

This recipe is a good place to see the distinction the write-up makes. The **topology loop** (the `Rule --continue--> Scroll` backward edge) is the runtime's first-class way to iterate: it carries state across passes (the cursor, the counter), branches, and stops on a condition. **Cardinality** (`scope: …points[*]`) is the shortcut for the narrower case — *apply the same transform to every element of a set, independently*, with no shared state or stop condition. Paging across the collection needs the topology loop; fanning across the points *in* a page is just a scope. Reach for the loop when the passes depend on each other; reach for cardinality when they don't.

---

## Run it

1. Install the **Qdrant plugin** and point it at your Qdrant instance.
2. Import `qdrant-migrate.json` into the FloMorphic canvas.
3. On each plugin node, select your **plugin settings profile** (this export ships with none — no credentials on the graph) and confirm the collection names: source in **Scroll source collection** (`chat_kb_demo`), target in **Create collection** / **Upsert points** (`selected_docs`).
4. Confirm the target **vector size** matches your new embedding model (`3072` for `gemini-embedding-2`) and that the plugin's configured embedding provider *is* that new model — the re-embed happens inside the upsert.
5. Hit **Run**. `Create collection` fires once; then the page loop turns — each `Upsert points` fires once per point in the page, `Page Cursor` counts, and `Rule` routes `continue` until the cursor runs out and it routes `done`. Every pass traces onto the diagram.

To migrate *your* data, change only the two collection names, the target vector `size`, and the embedding provider — the topology is the same for any collection.

---

## Notes on this export

- The `settingsId` on every plugin node (an install-specific credential reference) was **removed**, so no credential travels with the flow — select your own plugin settings profile after import (step 3).
- The upsert node's `scope` was **corrected** from the raw canvas export's `"$.source_stage.result[*] "` (a stray trailing space, and one level too shallow) to `"$.source_stage.result.points[*]"`. The scroll response lands at `$.source_stage.result` as `{ points, next_page_offset }` — the same shape the Rule and Page Cursor already read — so the points to fan across are `result.points`, not `result` itself. **Verify after import:** run it and confirm `selected_docs` ends with the same point count the source held (the Page Cursor's total). If your plugin build returns a different scroll shape, adjust this one scope to match.
- The flow is otherwise kept as exported — node ids, titles, and positions — so the canvas matches the screenshots in this folder.
- Vector size `3072` and the `gemini-embedding-2` framing are the demo's target model; change both together when you migrate to a different one.
