# The function call *is* the routing decision

### One prompt in Claude Code, one MCP server, and a prescription router that knows when to say "I don't know"

---

A prescription is almost never just drugs.

Here is a real-shaped one from an outpatient clinic — twelve lines, one visit, one patient with three complaints at once:

```
 1. Amoxicillin 500 mg capsule — 1 capsule PO three times daily for 7 days (#21)
 2. USG Abdomen atas + bawah, puasa 6 jam sebelum pemeriksaan
 3. Paracetamol 500 mg tablet — 1 tab PO every 6 hours as needed for fever (#20)
 4. Thorax X-Ray PA view, standing
 5. Complete blood count with differential + CRP quantitative
 6. Nebulizer salbutamol 2.5 mg, twice daily, administered at the clinic treatment room
 7. MRI Lumbal spine with gadolinium contrast — rule out disc herniation L4-L5
 8. Omeprazole 20 mg — 1 tablet 30 minutes before breakfast (#14)
 9. Physiotherapy, lumbar traction, 3 sessions over 2 weeks
10. Urinalysis dipstick + sediment microscopy
11. Echo 2D
12. Lanjutkan obat rutin dari dokter sebelumnya
```

Three of those lines are drugs. Four are imaging. Two are lab specimens. Two are things done to the patient in a chair. And one is a sentence that cannot be actioned by anybody.

Today a human at the front desk reads that list and splits it five ways. It is clerical work, it is boring, and it is also unforgiving: send line 6 to the pharmacy and someone hands a patient a vial of salbutamol they were supposed to inhale in the treatment room.

I wanted to see how much of that judgment could be moved into a workflow — and, more to the point, how fast I could *build* the workflow by describing it rather than drawing it. So I opened Claude Code, pointed it at a FloMorphic install over MCP, and typed a prompt.

What came back was a fifteen-node flow that routes all twelve lines, extracts typed fields on the way, and refuses to guess on line 12. It took one prompt, a handful of corrections, and about two minutes per run.

This is the interesting part of how it works, and the prompt that produced it.

---

## The idea: stop parsing the classifier's answer

The default way to build this is a classifier. Ask a model to label each line — `medication`, `imaging`, `lab`, `procedure` — get a string back, then write a `switch` on the string. It works until the model returns `"Medication"` with a capital M, or `"imaging/radiology"`, or a helpful sentence explaining its choice. Now you own a string parser, and the parser is where the bugs live.

There is a better move, and it falls out of function calling.

Give the model five **bound functions**, one per department:

| Function | Department |
| --- | --- |
| `dispense_medication` | Pharmacy |
| `schedule_imaging` | Radiology / Imaging |
| `order_lab_test` | Laboratory |
| `schedule_procedure` | Procedure / Therapy |
| `flag_unclear` | Needs clarification |

In FloMorphic, **every bound function on an LLM node is an output port on the canvas.** So when the model calls `schedule_imaging`, the edge out of that port is the edge the flow takes. There is no string to parse, because there is no string. The routing decision and the control flow are the same event.

```
Route line (LLM)
  ├─ dispense_medication  ─▶ Collect → Pharmacy
  ├─ schedule_imaging     ─▶ Collect → Radiology
  ├─ order_lab_test       ─▶ Collect → Laboratory
  ├─ schedule_procedure   ─▶ Collect → Procedure
  ├─ flag_unclear         ─▶ Collect → Needs clarification
  └─ _exception           ─▶ Collect → Router error
```

And then the second thing happens, which is why this pattern is worth more than the parser it deletes.

**Functions have parameters.** So the call that routes the line also *extracts* it:

- `dispense_medication(drug_name, strength, dosage_form, dose_instruction, quantity, duration)`
- `schedule_imaging(modality, body_region, view_or_protocol, contrast, preparation, clinical_question)`
- `order_lab_test(test_name, specimen, panel_items, fasting_required, urgency)`
- `schedule_procedure(procedure_name, agent_or_equipment, sessions, location, notes)`
- `flag_unclear(reason, candidate_departments, suggested_followup)`

Classification and structured extraction, one call. And notice what the parameter lists actually are: **each one is the intake form of the receiving department.** Pharmacy needs a dose instruction. Radiology needs a body region and a prep note. The lab needs a specimen type. The schema isn't a description of the text — it's a description of what the destination will do with it.

That reframing is the whole design. You are not asking a model to categorise text. You are asking it to **fill in the right form**, and which form it reaches for is the routing.

---

## The three lines that make it hard

Any approach looks good on line 1. The flow is worth building because of these:

**Line 6 — the drug that is not a dispensable.** `Nebulizer salbutamol 2.5 mg` contains a drug name, a dose, and a unit. Every keyword matcher on earth sends it to pharmacy. It belongs to Procedure, because nothing is handed over — the clinic nebulizes the patient. What fixes it isn't a keyword rule, it's the *description* on the function:

> `schedule_procedure` — "The line is something performed on the patient at the facility **rather than handed over or analysed**: nebulization, wound care, injection administered in clinic, physiotherapy, traction, dialysis, minor surgery."

One clause — *handed over or analysed* — carves Procedure cleanly away from Pharmacy and Laboratory in a way no list of words does. The model got it right and demoted salbutamol to `agent_or_equipment`, which is exactly where the treatment room wants it.

**Line 2 — two languages in one breath.** `USG Abdomen atas + bawah, puasa 6 jam sebelum pemeriksaan`. The system prompt carries a four-item glossary (`USG` = ultrasonography, `Thorax` = chest, `puasa` = fasting, `Echo 2D` = echocardiography, `R/` opens a drug) because teaching the shorthand is cheaper than hoping it's inferred. It came back as `modality: ultrasound`, `body_region: abdomen`, `preparation: "fasting 6 hours before examination"`.

And `contrast: "unknown"` — not `"no"`. The line doesn't say. That distinction is enforced by the field descriptions, which read like a lawyer wrote them: *"e.g. 500 mg; **empty if not stated**"*, *"yes, no, or unknown — **only what the line states**"*. Every optional field is optional on purpose; a `required` strength field is a standing invitation to invent one.

The sharpest version of this is `clinical_question` on imaging orders. The patient's working diagnosis — *dyspepsia + suspected lumbar radiculopathy, r/o community-acquired pneumonia* — is in the prompt, because it genuinely disambiguates a chest X-ray. And the system prompt then explicitly fences it:

> The working diagnosis is background only: do not copy it into a field unless the line itself states it.

**Both halves are necessary.** Give the model the context, then forbid it from laundering that context into extracted data. It held: line 2's `clinical_question` came back empty, and line 7's came back `"rule out disc herniation L4-L5"` — because line 7 is the one line that states its own.

**Line 12 — the one that must not be answered.** *"Lanjutkan obat rutin dari dokter sebelumnya"* — continue the routine medication from the previous doctor. Which medication? The line doesn't say, and no amount of context makes it say.

A four-department router with no escape hatch will put *something* in pharmacy here. That is how a system that is 92% right becomes a system nobody trusts: not because it's wrong often, but because you can't tell which 8% to check.

So `flag_unclear` is a **first-class destination with its own schema** — `reason`, `candidate_departments`, `suggested_followup` — and the system prompt tells the model to reach for it whenever the line isn't actionable. Here's what came back:

> **reason:** The line "Lanjutkan obat rutin dari dokter sebelumnya" (Continue routine medications from the previous doctor) does not specify any particular medication, dosage, or quantity to dispense. It is a general instruction to continue prior medications without listing actionable drug details.
>
> **suggested_followup:** Ask the prescriber or patient to provide the specific list of routine medications (drug names, doses, and quantities) that should be continued.

It translated the Indonesian, explained why it can't route, and wrote the question to ask. That is a more useful output than any department assignment would have been — and it only exists because declining was given a schema instead of being treated as a failure.

---

## Two details that do the real work

**One line at a time, with no memory.** The router sees exactly one line per call. It never sees the list. That's not a performance choice — a model shown twelve lines starts summarising them, and a model shown its own eleven previous answers starts being consistent with them instead of correct about the current one.

In FloMorphic, an LLM node's seeded messages resolve their templates *once* and persist across loop passes. For a per-line router that's fatal, so the node sets **`clear_history: true`**: the messages re-seed every pass, and every line is routed by a model with no memory of the others. Twelve independent judgments, flat token cost per line.

**The model picks the port; the graph names the destination.** Behind each port is a small JS node that parses the function arguments and appends a row. The `department` and `category` on that row are **hard-coded in the node**, not taken from model output — the node knows which port it sits behind. The model chose the door; it doesn't also get to write the sign above it.

Those nodes upsert rather than append (keyed on line number), so re-routing line 7 replaces line 7's row. Small thing; it's what makes the loop re-runnable when you're debugging one stubborn line.

**And the sixth port.** Every function-calling LLM node gets an `_exception` port — provider error, prose instead of a call, unusable response. It's wired to a collect node like any other department, so a router failure becomes **a row in the output** rather than a gap between twelve inputs and eleven outputs. It fired zero times in the shipped run. That's the point of wiring it anyway: the alternative to a visible zero is an invisible hole.

---

## What one run looks like

Twelve lines, thirteen LLM calls (twelve routings plus one summary), **125 seconds** end to end against a hosted model at `temperature: 0.3`.

The node-visit counters from the run tell the whole story without reading a single output:

| Node | Visits |
| --- | --- |
| Loop guard | **13** — twelve `next`, one `done` |
| Route line | **12** |
| → Radiology | **4** (lines 2, 4, 7, 11) |
| → Pharmacy | **3** (lines 1, 3, 8) |
| → Laboratory | **2** (lines 5, 10) |
| → Procedure | **2** (lines 6, 9) |
| → Needs clarification | **1** (line 12) |
| → Router error | **0** |

4 + 3 + 2 + 2 + 1 = 12. Every line routed exactly once. `Echo 2D` — two words — came back as a cardiac 2-D echocardiogram. The twelve routed rows then feed a final summary call that writes the note the front desk actually reads, under a deliberately humble instruction: *"Use only the routed data you are given. Do not re-classify anything and do not add clinical advice."*

Worth being clear about the boundary: this flow routes *who owns a line* and extracts *what the line says*. It does no clinical reasoning — no interaction checking, no dose validation, no allergy cross-reference. The seed patient's sulfa allergy is sitting right there in the context and the flow never looks at it, because checking it is a different system with different stakes and a different approval process. Intake routing is safe to automate precisely because it's a clerical decision with a visible audit trail. Dose checking isn't, and `flag_unclear` is where that line gets drawn.

---

## How it got built: one prompt, over MCP

None of this was drawn on a canvas.

FloMorphic ships an MCP server, so a coding agent can hold the whole install as tools: read the workflow design guide, compile a candidate graph, save it, launch a run, read the context the run left behind. Point Claude Code at it and the build loop becomes conversational:

```
flo_get_design_guide   → the node catalog, scope/branch rules, wiring conventions
flo_plan_patch         → compile a candidate graph, no save — catch design problems first
flo_apply_patch        → save it as a real workflow
flo_upsert_context     → seed the test prescription
flo_start_process      → run it
flo_get_context        → read what the run actually produced
flo_export_workflow    → the portable document (what ships in the cookbook)
```

The agent reads the guide *first*, drafts the graph as a compact patch, plans it to see the compile errors and design warnings, fixes them, saves, runs, reads the output, and iterates — without a human clicking anything. The design-guide step matters more than it sounds: it's the same instruction set the product's own build-with-AI dialog uses, so the agent is working from the platform's actual grammar rather than guessing at it.

### The prompt

Here's a prompt shaped to produce this flow. The structure is the reusable part — **domain, destinations, the hard cases, the non-negotiables, the test data** — and the specifics are swappable:

> Use the flomorphic MCP server on this install. Read `flo_get_design_guide` first, then build me a workflow and run it.
>
> **The job.** An outpatient clinic in Indonesia receives prescriptions whose lines are a mix of things: drugs to dispense, imaging orders, lab tests, and procedures done at the clinic. I want each line routed to the department that owns it, with the details extracted into that department's own fields.
>
> **Build it as a per-line router, not a classifier.** One `llm` node with one bound function per department, so the function the model calls is the port the flow takes — I don't want a label string parsed downstream. The departments are Pharmacy, Radiology/Imaging, Laboratory, Procedure/Therapy, and a fifth function for lines that can't be routed. Wire the `_exception` port too, as a sixth collector, so a router failure shows up as a row and not a gap.
>
> **Design each function's parameters as the receiving department's intake form** — pharmacy wants drug name, strength, form, dose instruction, quantity, duration; imaging wants modality, body region, view/protocol, contrast, preparation, clinical question; the lab wants test name, specimen, panel items, fasting, urgency; procedures want the procedure name, the agent or equipment used, sessions, location. Make exactly one field required per function — the identity of the thing — and leave everything else optional, with field descriptions that say explicitly to leave it empty if the line doesn't state it. I would rather have a blank field than an invented dose.
>
> **Hard cases it must get right.** Lines come in Indonesian and English with clinic shorthand — put a glossary in the system prompt (USG = ultrasonography, Thorax = chest, puasa = fasting, Echo 2D = echocardiography, R/ introduces a drug). A nebulizer line names a drug but is a procedure, not a dispensable — make the function descriptions carve that boundary by what *happens* to the thing (handed over vs. analysed vs. performed on the patient), not by keywords. Pass the patient's working diagnosis in as background because it disambiguates, then forbid the model in the system prompt from copying it into any extracted field unless the line itself states it.
>
> **Non-negotiables.** The router sees exactly ONE line per call and must never see the list — set `clear_history: true` so it has no memory of previous lines. Loop over the lines with a cursor (guard rule → lift the current line to a fixed key → route → advance → back edge); derive the next index from the row just processed so the loop can't fail to advance. Behind each port, a `js` node that parses the function arguments (handle `arguments` arriving as either a JSON string or an object), stamps the department as a constant in the node rather than trusting model output, and upserts the row by line number so a re-route replaces instead of duplicating. All collectors write the same context key. Reset that key at the start of the run so a reused Context can't leave stale rows behind.
>
> **Ending.** When the loop is done, tally by department and have a second `llm` node write the intake note the front desk reads — told to use only the routed data, not to re-classify, and not to add clinical advice.
>
> **Then test it.** Seed a context with a realistic 12-line prescription that includes all five cases: three drugs with different formats (one missing its dosage form), four imaging orders (one in Indonesian with a fasting prep, one just "Echo 2D", one stating its own clinical question), two labs, a nebulizer, a physiotherapy course, and one line that simply says to continue the previous doctor's routine medication. Run it, then read the context back and show me every routed row — I want to check line by line whether the extraction invented anything.

Two things about that prompt are doing most of the work.

The **non-negotiables** paragraph is where the engineering lives. `clear_history`, the cursor shape, upsert-by-key, stamping the department in the node — those are the decisions that separate a flow that demos from a flow that survives a re-run, and an agent won't reliably infer them from a description of the business problem.

And the last paragraph is the real instruction: **"run it, then show me every routed row — I want to check whether the extraction invented anything."** Asking for a built flow gets you a built flow. Asking for a built flow *plus the evidence it worked on the cases you said were hard* gets you a flow that was debugged before you saw it.

### The scars

It did not work first time, and the failures are worth knowing about because they're the two failure modes of this pattern.

**Nine runs died on `code 22: node process counter met limit threshold`.** That's a runaway loop hitting the 500-node-visit ceiling — the signature of a cursor whose guard never reaches `done`. If the advance step reads a counter that hasn't committed yet, it writes back the same index, and the loop spins forever. The fix is in the shipped flow: the advance derives the next index from the *row it just processed* rather than from the counter it's replacing. It's the first place to look if you adapt this and it hangs.

**And one bug survived into the saved run**, which is why the cookbook ships it unfixed. The accumulator key was never reset at the start of a run. The same Context document had been reused across a long afternoon of iterations, including an earlier, longer prescription — so three rows from that older run (lines 13, 14, 15, all empty) were still sitting in the list. No collector touched them, because collectors upsert by line number and those numbers no longer existed.

The tally gave it away immediately: `total_lines: 12, routed: 15, unrouted: -3`. A negative count is a nice property for a bug to have.

Then it got worse in the most instructive way available. The summary node did its job perfectly on the data it was handed, and wrote this into the note for the front desk:

```
13. [empty line] — Ask prescriber to specify the intended instruction.
14. [empty line] — Ask prescriber to specify the intended instruction.
15. [empty line] — Ask prescriber to specify the intended instruction.
```

Three phantom clarifications, confidently formatted, in a document a human was going to act on. Nothing hallucinated — the model was faithful to its input. The input was stale.

That's the lesson I'd keep from the whole exercise: in a workflow over a durable shared context, **state you don't explicitly reset is state you've decided to inherit**. The `_exception` port, the `flag_unclear` schema, the field-level "leave it empty" instructions — all of that careful work to keep the model from inventing anything, and the one piece of fiction that reached the output came from the plumbing.

---

## Try it

The flow, the seed prescription and the trimmed run output are in the Flow Cookbook under [`prescription-intake-router/`](.) — fifteen nodes, twenty edges, no credentials. The export's model profiles are deliberately blank: point the two LLM nodes at any provider that supports function calling, paste in the seed context, and hit Run.

The thing to watch on the canvas is the trace out of the router. Twelve passes, and each one lights exactly one of six ports. The thickness of each port's trace *is* the department breakdown — the classification is visible as topology, which is the clearest argument for the pattern that I know of.

**Full technical write-up:** [`README.md`](README.md) — node by node, including the parts I skipped here.
