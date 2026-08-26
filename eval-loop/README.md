# Eval Loop

**One LLM node evaluates another.** A worker model analyses a product catalogue; a senior "evaluator" model reads the worker's whole conversation and decides, each cycle, whether the goal is met — routing back for another round or out to a final result. An observer, an LLM-judge, and a self-refining loop, built from nodes and a backward edge. No runtime feature knows the word "evaluation."

> Full write-up: [Your Evaluator Is a Node. Your Loop Is an Edge.](https://inflowenger.com/blog/your-evaluator-is-a-node)

**Teaches:** LLM-as-judge as routing (a bound function *is* a port) · a loop as a backward edge + counter · `clear_history` (a judge re-reads fresh each pass; the worker it judges accumulates) · `input` vs `_get` for in-scope vs out-of-scope reads.

---

## Topology

```
Start
  ↓
Seed Demo Product        js   · lays down the whole demo context ($)
  ↓
User Analyser            llm  ←──────────────────────────┐  the worker (in practice: an agent)
  ↓                                                       │
Senior Evaluator         llm  · reads the whole convo,    │  the observer / judge
  │                             calls continue | answered  │
  ├─ answered ─→ Finalize Result   js  · done             │
  └─ continue ─→ Append Teacher Question  js  ────────────┤  writes the judge's question
                       ↓                                   │  into the worker's history
                 Loop Counter       js  · count + 1 ───────┘
```

---

## How it works

- **Seed Demo Product** (`js`, scope `$`, no key) lays down the entire starting context — the catalogue, the project goal, an empty `user_request_analyse` and `evaluator`, and `loop.count = 0` — so you can run without an external Context.
- **User Analyser** (`llm`, key `user_request_analyse`) is the node under evaluation. **No `clear_history`** — its conversation must *grow* each cycle.
- **Senior Evaluator** (`llm`, key `evaluator`) reads the goal and the worker's full message history, then calls one of two bound functions. **The function it calls is the branch:** `continue` loops, `answered` exits. Its prompt names those functions explicitly and reads `{{$.loop.count}}` to know which cycle it is on — so `clear_history: true` re-seeds it every pass, keeping the count and history current.
- **Append Teacher Question** (`js`, scope `$.user_request_analyse`) is scoped to the worker's slice, so it reaches the evaluator with `_get("$.evaluator")`, lifts the question out of the function call, and appends it as the worker's next turn — the senior model steering the worker.
- **Loop Counter** (`js`, scope `$.loop`, key `count`) increments the count the evaluator reads.
- **Finalize Result** (`js`, root scope) reads `input.evaluator` **directly — no `_get`** (contrast the teacher node, which is scoped down and must reach out), and packages the worker's answer with the judge's closing summary on the `answered` branch.

The single edge that makes it a loop: **Loop Counter → User Analyser**.

---

## Run it

1. Import `eval-loop.json` into the FloMorphic canvas.
2. Select a **provider profile** on **User Analyser** and **Senior Evaluator** (this export ships with none — no credentials on the graph).
3. Hit **Run**. The judge probes on cycle 0 (its prompt requires it), the question lands in the worker's history, the counter ticks, and the loop turns until the judge calls `answered`. Every turn traces onto the diagram.

---

## Notes on this export

- **Finalize Result** was repaired from the raw canvas export (the original had an unbalanced `JSON.parse(...)` and referenced an undefined variable). It now reads the evaluator's closing tool call the same way `Append Teacher Question` does.
- The `settingsId` from the source canvas was removed, so no install-specific credential reference travels with the flow — select your own provider profile after import (step 2 above).
- The `Append Teacher Question` fallback question (used only if the judge routes `continue` without a `question`) is a goal-directed follow-up, matching the article.
- Node ids, titles, and positions are kept as exported — including the descriptive `Loop Counter Flag Used By Senior LLm` title — so the canvas layout matches the write-up.
