# Flow Cookbook

Worked [FloMorphic](https://inflowenger.com/flomorphic) flows — import, run, and learn the pattern.

Each recipe is a single workflow export plus a short write-up of the *idea* it teaches. They aren't a library you depend on; they're examples you read, run, and adapt. If you're new to FloMorphic, start with [Getting Started](https://github.com/FloMorphic/getting-started), then come here to see how a real pattern is wired.

---

## How to use a recipe

Every recipe folder holds a `*.json` workflow export and a `README.md`.

1. **Import the export.** In the FloMorphic canvas, import the recipe's `.json` (or paste it into the AI-import dialog). The nodes land wired and laid out.
2. **Pick a provider profile.** The exports carry **no credentials** — LLM nodes reference a Node Settings profile *by id*, and that id is intentionally left blank. Open each model node's drawer and select your own provider/model profile before running. (This is why you can commit and share a flow without leaking a key.)
3. **Read the review list.** On import, the planner reports anything that still needs a hand — a store to select, a settings profile to choose. Read it before running; it names each seam.
4. **Run it.** Most recipes seed their own demo context in the first node, so you can hit **Run** with no external input and watch the events trace onto the diagram.

Anything a recipe can't know about your install (a settings profile, a placeholder tool with no backing implementation) is called out in that recipe's README.

---

## Recipes

| Recipe | Teaches | Level |
| --- | --- | --- |
| `eval-loop/` | An LLM node that **evaluates another LLM node**: an observer/LLM-judge routing by bound function, a self-refining loop drawn as a backward edge, and `clear_history` (why a judge re-reads fresh each pass while the worker it judges must not). | Intro |
| `agent-under-review/` | The same idea, scaled up: a tool-using **worker agent** judged by a **panel** of three evaluators in parallel, a deterministic aggregator that routes on the combined score, **human escalation** (park/resume) on non-convergence, and a refinement **budget**. | Intermediate |

More land over time. Each folder is self-contained — read its README for the topology, the mechanic it turns on, and what to fill in after import.

---

## The ideas these recipes keep reusing

A few FloMorphic primitives show up across most recipes; the write-ups assume them:

- **Context** — one JSON document every node reads and writes. Templates (`{{$.path}}`) resolve into any *text* field; a node's `key`/`scope` decide where its output lands.
- **`scope` cardinality** — a node's `scope` is a JSONPath; a single-valued scope runs it once, a many-valued scope (`$.items[*]`) runs it once per element. There is no loop node.
- **Loops are edges** — a loop is ordinary nodes wired into a cycle with a backward edge and a counter, over the durable Context.
- **Functions and handlers are ports** — an LLM node's bound function, or a `rule` node's handler, is an output port. The model calling a function (or the rule firing a tag) *is* the branch the flow takes.
- **`input` vs `_get`** — in code (`js`/`rule` nodes), the scoped slice arrives as `input`; reach data *outside* the node's scope with `_get("$.path")`.
- **`clear_history`** — an LLM node's seeded messages resolve their variables once and persist across a loop; `clear_history: true` re-seeds them every pass, so a node that must re-read the current Context each cycle (a judge) does, while one that should accumulate a conversation (a worker) doesn't.

---

## Contributing a recipe

One pattern per folder:

```
your-recipe/
  your-recipe.json     # the workflow export (no credentials — leave settings profiles blank)
  README.md            # the goal, the topology, the mechanic it teaches, what to fill in after import
```

Keep the export runnable from a self-seeding first node where possible, and name in the README anything an importer must select or supply. The point of a recipe is that someone can import it, run it, and come away understanding *one* thing they can now build themselves.

---

**Docs & concepts:** [inflowenger.com/flomorphic](https://inflowenger.com/flomorphic) · **Install:** [github.com/FloMorphic/getting-started](https://github.com/FloMorphic/getting-started)
