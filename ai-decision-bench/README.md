# AI Decision bench — nimble (local Ollama)

**Run the same decision 24 times and find out whether you can trust it.** A built-in **AI Decision** node triages support messages into four teams, and the flow around it is a bench: a queue of labelled cases × *N* repetitions, a loop that feeds them through one at a time, and a scorer that reports accuracy, per-case stability across repetitions, the confidence band each case landed in, and how often the flow routed somewhere other than where the answer pointed.

The decider is a **local Ollama** serving the `nimble` decision model, reached over the **System One** wire protocol — so the whole bench runs on your own machine, with no API spend, at roughly 2–3 seconds per decision.

**Teaches:** the **`ai-decision`** built-in — typed questions (`choice` / `score` / `noul`), a **calibrated probability distribution** over options you declare instead of free text, and **every option of a routed question as an output port** · `route: true` vs `route: false` (one question branches the canvas, two ride along as data) · the `_exception` port and what fires it · a **bench loop** built from ordinary nodes — queue, guard, lift, advance, back edge — used to measure *variance* rather than to process a list · **port-marker nodes** that turn "which edge fired" into data you can score · reading a finished run's Context as the experiment's result sheet.

---

## Topology

```
Start
  ↓
Reset state & build queue              js    key: —   (resets i / results / report / queue at $)
  ↓
More cases?                            rule  key: loop_state        handlers: next | done
  │
  ├─ done ──────────────────────────────────────────▶ Score the run     js   key: report
  │                                                        ▲
  └─ next                                                  │
       ↓                                                   │
     Lift current case                 js    key: current  │
       ↓                                                   │
     Have a case?                      rule  key: have_case_state     handlers: yes | no
       │                                                   │
       ├─ no ──────────────────────────────────────────────┘
       │
       └─ yes
            ↓
          Triage decision (nimble)     ai-decision   key: decision
            ├─ category.billing        ─▶ Port: billing         ┐
            ├─ category.technical      ─▶ Port: technical       │  five js nodes,
            ├─ category.account_access ─▶ Port: account_access  ├─ all key: fired_port
            ├─ category.other          ─▶ Port: other           │
            └─ _exception              ─▶ Port: _exception      ┘
                                            ↓ ↓ ↓ ↓ ↓   (all five)
                                        Record result   js   key: results
                                            ↓
                                        Next index      js   key: i
                                            └──── back edge ──▶ More cases?
```

Nineteen edges, fourteen nodes, one of which is the decision. Everything else exists to ask it the same question repeatedly and write down what came back.

---

## How it works

### 1. Reset state & build queue (`js`, no `key`)

The only node with **no `key`**. A key-less `js` node commits *at* its scope, so an object result is merged into the Context at the top level — four slots in one node:

```js
let fresh = { i: 0, results: [], report: null, queue: queue }
fresh
```

That is the reset: every run starts from a clean counter, an empty result list and no report, whatever the Context held before. The queue it builds is the **cross product of cases and repetitions** — `reps` copies of every case, each tagged with a unique `id` (`t1_billing_clear#2`) and the `base` id it came from:

```js
queue.push({ id: c.id + '#' + r, base: c.id, expect: c.expect, acceptable: c.acceptable, text: c.text })
```

`id` keeps the rows distinct; `base` is what the scorer groups by to compare repetitions of the same case. With the shipped seed (8 cases, `reps: 3`) the queue is 24 entries long.

### 2. The loop (`rule` → `js` → `rule` → … → `js` → back edge)

There is no loop node in FloMorphic; a loop is nodes wired into a cycle over the durable Context. This is the canonical four-part shape:

| Part | Node | What it does |
| --- | --- | --- |
| **guard** | `More cases?` (`rule`, handlers `next` \| `done`) | `input.i < queue.length ? 'next' : 'done'` — the `done` port is the only exit |
| **lift** | `Lift current case` (`js`, key `current`) | `queue[input.i]` copied to a fixed address, because a template cannot index by a variable (`{{$.queue[$.i]}}` does not resolve — `{{$.current.text}}` does) |
| **body** | `Have a case?` → `Triage decision` → ports → `Record result` | one case, one decision |
| **advance** | `Next index` (`js`, key `i`) | `input.i + 1`, then a **back edge to the guard** — never to the body |

`Have a case?` is a second guard, immediately after the lift: `c && c.id ? 'yes' : 'no'`. It costs one node and it is what stops a malformed or short queue from handing the decider a `null` subject. Its `no` port exits to the scorer, same as `done`.

### 3. Triage decision (`ai-decision`, key `decision`)

The node the recipe is about. Its `body.state` is the subject template — one token, so the lifted case's text is sent as the thing to evaluate:

```json
"body": { "state": "{{$.current.text}}" }
```

Then three **typed questions**. Each declares the answers it may return, and the model answers with a probability over *every* declared option in a single call. It never writes prose, and it cannot answer outside the options.

| Question | `type` | `route` | Options |
| --- | --- | --- | --- |
| `category` | `choice` | **`true`** | `billing` · `technical` · `account_access` · `other` |
| `urgency` | `score` | `false` | `low` → `normal` → `high` → `critical` (ordered, lowest first) |
| `needs_human` | `noul` | `false` | `yes` / `no` |

Three types, three jobs:

- **`choice`** — pick one of N unordered options. This is the classifier.
- **`score`** — the options are an *ordered ladder*, so the answer carries a `score` as well as a label. `urgency` came back with `score: 1.475` on a case torn between `normal` (0.43) and `high` (0.52) — the expected position on the ladder, which a plain choice would have thrown away.
- **`noul`** — a yes/no predicate. The answer carries `p`, the probability of `yes`.

**`route` is what puts ports on the canvas.** `category` has `route: true`, so each of its four options becomes an output port tagged `category.<option>`, and the top answer's port fires. The other two questions ride along as data under `$.decision.answers` — same call, same cost, no edges. That split is the design lever: *route the question the flow must branch on; keep the rest as data a later node can read.*

A routed question also gives the node an **`_exception` port**, which fires on a provider error, a missing answer, or a confidence below `min_confidence` (`0` here, i.e. off). The bench wires it like any other port and counts it, so a provider hiccup shows up as a number in the report instead of a silent gap.

The full distribution lands under `key`:

```json
"category": {
  "answer": "technical", "confidence": 0.920, "routed": true, "tag": "category.technical",
  "probabilities": { "technical": 0.980, "billing": 0.016, "other": 0.004, "account_access": 0.0005 }
}
```

`probabilities` is the whole point. An LLM that replies `technical` tells you nothing about how close the call was; this tells you it was not close at all — and, on the ambiguous case, that it very much was.

### 4. Five port markers → one `Record result`

Each of the five ports goes to a one-line `js` node that writes its own name:

```js
let firedPort = 'category.billing'
firedPort
```

They all share `key: fired_port`, and all five converge on `Record result`. The ports are mutually exclusive, so exactly one marker runs per pass — this is how **the edge the flow actually took becomes data in the Context**, next to the answer that caused it. Without the markers you could score the decision but not the routing.

`Record result` then flattens one row per case — ids, the expectation, the fired port, and `{answer, confidence}` for each of the three questions — and appends it to `$.results`, guarding against a double-append by id:

```js
let seen = prior.filter(function (r) { return r.id === row.id })
let results = seen.length ? prior : prior.concat([row])
```

### 5. Score the run (`js`, key `report`)

Reached from `done` or from `have_case → no`. It reads `$.results` and computes:

- **`category_accuracy_all_reps`** — over every row *except* those labelled `expect: "ambiguous"`. A case may instead carry `acceptable: ["billing", "account_access"]`, and then any of those counts as correct.
- **`stability[]`**, one entry per `base` case: `reps`, the **distinct** category and urgency answers seen across them, `category_stable` / `urgency_stable` booleans, the `conf_min`/`conf_max` band, and `all_reps_correct`.
- **`unstable_category`** / **`unstable_urgency`** — the case ids that answered differently between repetitions. This is the number a single run can never give you.
- **`routing_mismatches`** — rows where `fired_port !== 'category.' + answer`.
- **`exceptions`** — rows that left through `_exception`.

---

## What the shipped run actually found

`sample-run.json` is the Context left behind by run #674: 8 cases × 3 reps = **24 decisions in 65 seconds** against `nimble` on a local Ollama.

```
category_accuracy_all_reps : 1.0      (21 scorable rows; the 3 ambiguous rows are excluded)
distinct_cases             : 8
unstable_category          : []
unstable_urgency           : [t8_misleading_vocab]
exceptions                 : 0
routing_mismatches         : 1
needs_human_distinct_answers: [yes]
```

Four things in there are worth more than the accuracy number:

**Confidence tracked difficulty.** The seven cases with a single right answer landed at **0.87–0.99** confidence. `t5_ambiguous` — *"My card expired so the payment failed, and now my subscription is suspended and I cannot get into my account at all"*, genuinely both billing and access — landed at **0.42–0.50**, every repetition. The model was not merely right or wrong; it was *uncertain in the right place*. That is the argument for a decision model over a chat model asked to output a label, and you can only see it because the whole distribution is in the Context.

**The adversarial cases held.** `t7_angry_but_routine` (furious tone, trivial invoice request) stayed `billing` / `normal` — tone did not inflate urgency, which is exactly what the `urgency` instructions ask for. `t8_misleading_vocab` ("account balance chart is rendering wrong… display bug") stayed `technical` across all three reps despite the billing vocabulary.

**One case was unstable, on the question that should be.** `t8` moved between `normal` and `high` urgency between repetitions — `score: 1.475`, sitting between two levels, so repetition tips it. Nothing is broken; the bench is telling you that if this flow had an SLA branch on `urgency`, *that* case would flap. A `rule` node reading `$.decision.answers.urgency.score` with an explicit threshold would be the fix, and `unstable_urgency` is how you knew to go look.

**`needs_human` is a badly written question, and the bench says so.** All 24 rows answered `yes` — including the job application, which a help-centre link would obviously not resolve but a human recruiter should read. A predicate that never says `no` carries no information. The finding is about the *question*, not the model: `needs_human_distinct_answers` is in the report precisely to catch a question that has collapsed to a constant. Rewriting it (`"Can a templated reply or a help-centre article fully resolve this, with no account change and no back-and-forth?"`) and re-running the bench is the next edit.

### The one routing mismatch

Row `t3_access_clear#3` answered `account_access` at 0.979 confidence, but the recorded `fired_port` was `category.technical` — the value from the *previous* pass (`t2_technical_clear#3`). The decision and the edge were both right; the **`$.fired_port` read by `Record result` lagged one iteration**, because the marker's write and the next node's read of the shared Context raced inside a tight loop. 1 row in 24.

Two takeaways, in order of usefulness:

1. **Prefer the answer's own tag.** `$.decision.answers.category.tag` and `$.decision.routed` already carry which port fired, written by the same node in the same commit as the answer — no second node, no race. The marker nodes are here because they make the mechanic *visible* on the canvas, and because disagreement between the two is itself a measurement; a production flow should read the tag.
2. **Score the thing you suspect.** `routing_mismatches` exists in the report because routing is worth cross-checking. Had the flow only recorded answers, this would have been invisible — and a flow that acts on the port while logging the answer would have had a 4% silent disagreement between what it did and what it told you.

---

## Run it

### 1. Serve a decision model locally

Ollama, serving the `nimble` decision model:

```bash
ollama pull nimble
ollama serve        # http://localhost:11434
```

### 2. Create a Node Settings profile

The export carries **no profile** — `settingsId` is deliberately blank, so nothing about your endpoint or key is in this repo. Create one under **Node Settings** bound to node type `ai-decision`:

```json
{
  "title":  "Decision-Ai-SystemOne-nimble-Local",
  "provider": "systemone",
  "url":      "http://localhost:11434",
  "model":    "nimble",
  "timeout_seconds": 30,
  "access_token": "<any value — a local Ollama does not check it>"
}
```

`provider: "systemone"` selects the System One wire protocol and posts to `/v1/systemone` on the `url`. A profile that names its own `url` must also name the `model` that endpoint serves — hence `nimble` here.

### 3. Import and seed

1. Import `ai-decision-bench.flow.json` into the canvas. It compiles clean, with one warning — see [Notes](#notes-on-this-export).
2. Open **Triage decision (nimble)** and select the profile from step 2. It is the only node that needs anything.
3. Create a Context document and paste **`seed-context.json`** into it — the 8 labelled cases and `reps: 3`. The flow reads `$.cases` and `$.reps` and writes everything else itself.
4. Hit **Run** against that Context.

Watch the loop trace: the decision node lights a different `category.*` port per pass, and the trace thickens on whichever port the corpus leans toward. When `More cases?` finally fires `done`, read `$.report`.

### 4. Make it your bench

The point of a bench is the corpus, and `seed-context.json` is a template to replace:

```json
{ "id": "t5_ambiguous",
  "expect": "ambiguous",
  "acceptable": ["billing", "account_access"],
  "text": "My card expired so the payment failed, and now my subscription is suspended…" }
```

- `expect` is the single right answer. Use `"ambiguous"` plus `acceptable` for a case with more than one defensible answer — `expect: "ambiguous"` drops the row out of the accuracy denominator while still scoring it under `all_reps_correct`.
- `reps` is the variance budget. `1` scores accuracy only; `3` is enough to catch a flapping case; `10` on a local model costs nothing but wall clock.
- Changing the taxonomy means editing **two** places that must stay in step: the `category` options on the decision node (each one is a port), and the five port-marker nodes plus their edges. Add a fifth team and you add an option, a marker and an edge.
- Every option `description` and every `instructions` string is a template — `{{$.path}}` vars resolve before the call — so a question can be parameterised from the Context (your current policy text, a threshold) without touching the canvas.

---

## Notes on this export

- **One import warning, and it is benign here.** The planner reports *"2 branches arrive at 'Score the run', so it runs 2 times — inbound edges do not merge."* It is right in general and wrong in this case: the two inbound edges are `More cases? → done` and `Have a case? → no`, ports of two different guards that cannot both fire in a run, so the scorer runs once. The static check cannot know they are exclusive. Do not "fix" it with a `promissall` — a `promissall` waits for *every* inbound branch, and here one of them never arrives, so the run would stall until it timed out. This is the shape where you overrule the warning; if the two edges could ever both fire, you would not.
- **`settingsId` is blanked.** No endpoint, model or token travels in this file. Reselect a profile after import (step 2 above) — it is the one thing the flow cannot know about your install.
- **`access_token` in the profile.** The System One protocol sends one; a local Ollama ignores it. Point the same flow at hosted Jev (`api.typesafe.ai`) or at OpenAI's Decisions API (`provider: "decisions"`, `gpt-6-luna`) and it becomes a real credential. Nothing in the flow changes when you re-point it — the questions, the options and the port tags are identical across both protocols, which is also what makes this bench useful for *comparing* deciders: same corpus, same scorer, swap the profile.
- **Ceilings on the decision node**, enforced before the call: 64 questions, 64 evidence rows, 255 choice options, and 10 score levels on hosted Jev / the Decisions API (**up to 26 against a local Ollama decider**). Hitting one is almost always a bad `{{$.path}}` or a loop upstream rather than intent.
- **No `evidence` rows in this flow.** The decision rests on `body.state` alone. `evidence` is the RAG side of an AI Decision — `[{ "source": "policy.pdf", "text": "{{$.kb.clause}}" }]` — and a question can then cite `evidence[0].text` by backticked path. Adding a retrieval node before the decider and feeding its hits in as evidence is the natural extension of this recipe; see [`naive-rag/`](../naive-rag/) for the retrieval half.
- **Not a reasoner.** No tools, no memory, no chain of thought — one fast scored call over options you declared. When a decision needs multi-step judgment or the documents themselves have to be searched, that is an `llm` node with bound functions, not this one. See [`eval-loop/`](../eval-loop/).
- Node refs, titles, keys and positions are kept as exported.
