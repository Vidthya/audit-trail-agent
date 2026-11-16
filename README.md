# audit-trail-agent

A small, honest demonstration of an agentic document workflow with a control and evidence layer: read invoices, check them against bank lines, send anything doubtful to a named human, and keep a tamper-evident record of every step.

It runs on synthetic data only, which are invoices with known correct answers, so every claim it makes can be measured.

## What it does

1. Extract supplier, invoice number, date and total from invoice text. Every field must come with a **quote** from the document.
2. Ground-check each field: the quote must really appear in the document and must support the value. An invented or misread value is blocked, not accepted.
3. Match the invoice to bank lines with plain deterministic rules: `ok`, `amount_mismatch`, `unpaid`, `duplicate_payment`.
4. Route every exception or failed extraction to a named reviewer (round-robin). Only the assigned reviewer can record a decision.
5. Record every step in an append-only, hash-chained evidence trail. `verify()` detects edited, deleted or reordered entries. Inputs are stored as hashes, with the model name and prompt hash.
6. Evaluate against the known truth: field accuracy, exception precision/recall, and how many items went to a human.

The language model only does step 1. Steps 2-6 are ordinary code, so the control layer does not depend on the model behaving.

## Run it

```bash
python -m venv .venv && .venv\Scripts\activate      # Windows  (Mac/Linux: source .venv/bin/activate)
pip install -r requirements-dev.txt
pytest
python -m audit_agent.run --n 200 --extractor rules fake
```

No Codespaces needed. On Google Colab, use `audit_trail_agent_colab.py` or the notebook (single file, same code).

## Example output (200 synthetic invoices, seed 7)

| Extractor | All fields correct | Sent to review | Exception precision | Exception recall |
|---|---|---|---|---|
| rules-baseline | 83.5% | 50 | 34.0% | 100% |
| fake-llm | 100% | 17 | 100% | 100% |

The baseline fails on one deliberately unusual invoice layout (about 15% of documents). Those failures are routed to a human as `extraction_failed` rather than guessed, which is why its precision is low but its recall stays at 100%.

## Limitations (read these)

- **The data is synthetic.** Real invoices are messier. These numbers show the mechanism works, not that it will work on your documents.
- **`fake-llm` is a scripted test stand-in, not a language model.** Its 100% says nothing about any real model. It exists so tests and CI need no API key, and so the grounding check can be tested against deliberate hallucinations.
- **`AnthropicLLM` has not been run against the live API in this repo.** To try it: `pip install -r requirements-llm.txt`, set `ANTHROPIC_API_KEY`, then `python -m audit_agent.run --extractor rules anthropic`. Only that result tells you anything about a real model; report it next to the baseline.
- Matching rules are simple (60-day window, supplier name and amount). They are a control example, not a production reconciliation engine.
- The hash chain shows tampering inside the file; it does not stop someone rewriting the whole file. A real system would anchor the latest hash somewhere separate.

## Layout

```
audit_agent/   synth, rules, llm, extract, matching, trail, review, pipeline, evaluate, run
tests/         41 tests
audit_trail_agent_colab.py   single-file version (made by make_colab.py)
```
