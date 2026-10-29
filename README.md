# How Reliable Are LLM Judges on Financial QA?

Evaluating LLM judges on financial question answering using FinanceBench, with a focus on numerical accuracy, judge biases, and hybrid verification.

## Project objective

An LLM judge evaluates another model's answer. This project tests whether those judgments agree with human correctness labels, especially when an answer contains a subtle financial or numerical error.

The project compares prompt-based judging with a hybrid approach that combines language-model interpretation and numerical checks in code. It is a study of evaluation reliability, rather than a project to train a new financial answering model.

## Research questions

- How often do judge verdicts agree with FinanceBench's human labels?
- Can judges detect changes to numerical values, units, signs, and reporting periods?
- Do judges accept valid equivalents, such as $1.577 billion and $1,577 million?
- Do answer order and irrelevant verbosity affect judgments?
- Does hybrid verification improve numerical evaluation at a reasonable cost?

## Expectations and scope

- Verify the available data, label meanings, and file fields before running experiments.
- Begin with a small, inspectable baseline before adding judge families or bias probes.
- Preserve original data and human labels. Record any audited corrections separately with their reasons.
- Define correctness, refusal handling, numerical tolerances, and ambiguous-case handling before final evaluation.
- Separate development and final evaluation by question ID. All answers and probe variants associated with one question must stay in the same split.
- Keep human correctness labels out of judge prompts. Use them only for evaluation afterward.
- Treat numerical matching as one source of evidence: a matching value can still refer to the wrong metric or year.
- Record prompts, model identifiers, settings, timestamps, raw responses, parsing failures, usage, and available cost information.
- Report uncertainty and limitations. Answers to the same question are related observations, not independent questions.
- Treat improvements as hypotheses. A well-supported negative result is a valid outcome.

## Data organization

Use the original [FinanceBench repository](https://github.com/patronus-ai/financebench) as the data source. Confirm its current usage and redistribution terms before committing downloaded data or derived records containing source content.

The evaluation uses two layers:

1. **Question records:** questions, reference answers, supporting evidence, and justifications.
2. **Response records:** answers produced by model configurations and their human labels.

Join these records using `financebench_id`. A question can have several model responses, so each evaluation item also needs an answer-source identifier.

Important source files include:

| Source | Purpose |
| --- | --- |
| `data/financebench_open_source.jsonl` | Questions, reference answers, evidence, and justifications |
| `data/financebench_document_information.jsonl` | Document metadata and source links |
| `results/*.jsonl` | Model responses and human evaluation labels; inspect actual fields before implementing the join |

Validate the actual schema rather than relying only on descriptions. For example, evidence records may use `doc_name`; document types may be lowercase; question IDs are strings such as `financebench_id_03029`. Evidence page numbers are zero-indexed and may differ from printed page numbers.

## Planned file arrangement

This is the intended layout. Only this README has been created so far; scripts and commands below describe planned implementation.

```text
Financial QA/
├── README.md
├── requirements.txt              # Dependencies used by the implementation
├── .env.example                  # Environment variable names; no credentials
├── .gitignore                    # Exclude secrets, local data, caches, and outputs
│
├── config/
│   └── experiment.json           # Models, splits, tolerances, and run settings
│
├── data/
│   ├── raw/
│   │   └── financebench/         # Original source checkout or downloaded files
│   └── processed/
│       ├── evaluation.jsonl      # Joined and standardized evaluation items
│       ├── splits.json           # Development/evaluation question IDs
│       └── probes.jsonl          # Controlled answer variants and expected labels
│
├── prompts/
│   ├── plain_judge.txt
│   ├── pairwise_judge.txt
│   ├── rubric_judge.txt
│   └── numeric_extraction.txt
│
├── src/
│   ├── __init__.py
│   ├── prepare_data.py
│   ├── inspect_example.py
│   ├── run_judge.py
│   ├── numeric_checks.py
│   ├── make_probes.py
│   └── analyze.py
│
├── tests/                        # Focused data-integrity and numerical checks
│   ├── test_prepare_data.py
│   └── test_numeric_checks.py
│
├── docs/
│   ├── grading_rubric.md         # Correctness, refusals, and ambiguity policy
│   ├── literature_review.md      # Related work and limits of novelty claims
│   └── annotation_audit.md       # Independent label review and disagreements
│
└── outputs/
    ├── judgments/               # Separate run directories and raw responses
    ├── tables/
    ├── figures/
    └── report.md
```

### Script responsibilities

| File | Responsibility |
| --- | --- |
| `prepare_data.py` | Load source files, validate joins, preserve labels, and produce standardized records and question-level splits |
| `inspect_example.py` | Display a question, reference, evidence, candidate responses, and human labels in readable form |
| `run_judge.py` | Run selected judge setups, cache completed requests, and save predictions with reproducibility metadata |
| `numeric_checks.py` | Normalize values and units, apply documented tolerances, and flag extraction failures or ambiguous cases |
| `make_probes.py` | Create controlled variants, preserve their original item IDs, and record changes and expected outcomes |
| `analyze.py` | Compare predictions with labels and produce metrics, uncertainty estimates, cost summaries, and error breakdowns |

Keep original data in `data/raw/`, standardized inputs in `data/processed/`, and judge predictions in `outputs/`. Do not overwrite inputs with predictions.

### Standard evaluation record

The following is a proposed internal format. Candidate answers and labels shown as placeholders must be populated from inspected source response records.

```json
{
  "item_id": "financebench_id_03029__configuration_name",
  "question_id": "financebench_id_03029",
  "answer_source": "configuration_name",
  "company": "3M",
  "doc_name": "3M_2018_10K",
  "question": "What is the FY2018 capital expenditure amount (in USD millions) for 3M?",
  "reference_answer": "$1577.00",
  "justification": "Capital expenditure corresponds to purchases of property, plant and equipment.",
  "evidence": [],
  "candidate_answer": "<source model response>",
  "human_label": "<source human label>"
}
```

Preserve full evidence in real records. Save predictions separately against `item_id`, with a run identifier and prompt version. Probe records also need a unique probe ID, parent item ID, perturbation type, and reviewed expected label.

## Experimental workflow

1. **Inspect five questions.** Include numerical lookups, calculations, and qualitative reasoning. Trace each answer to its evidence.
2. **Prepare the data.** Join source responses, inspect unmatched records, check label counts, and verify unique item IDs.
3. **Define the rubric and splits.** Audit a shared sample independently before adjudicating disagreements. Finalize policies on the development subset.
4. **Run the plain judge.** Start with a small pilot, inspect its raw responses, and resolve parsing or data issues.
5. **Add comparison setups.** Implement pairwise judging and rubric-based debiasing. Define tie and order-disagreement handling in advance.
6. **Create reviewed probes.** Include both incorrect changes and valid equivalent expressions. Each probe should isolate an intended factor.
7. **Add hybrid verification.** Evaluate numerical extraction separately from comparison. Define fallbacks and measure their costs.
8. **Freeze the experiment.** Evaluate the final configurations on held-out questions without further prompt or tolerance tuning.
9. **Analyze and report.** Compare methods on the same eligible items and explain disagreements and failure categories.

Planned command interface, to implement in the scripts:

```bash
python -m src.prepare_data
python -m src.inspect_example --question-id financebench_id_03029
python -m src.run_judge --config config/experiment.json
python -m src.make_probes --config config/experiment.json
python -m src.analyze --config config/experiment.json
```

## Judge setups

| Setup | Approach |
| --- | --- |
| J1: Plain | Classify a single candidate answer using the question and reference |
| J2: Pairwise | Select the correct answer from a controlled pair |
| J3: Debiased | Apply a written rubric and specified reasoning instructions, with pairwise position swaps where applicable |
| J4: Hybrid | Combine semantic assessment with structured numerical extraction and deterministic comparison |

Report pairwise selection accuracy separately from single-answer correctness classification. Do not assume that selecting the better answer proves either answer correct. A combined debiasing prompt is a candidate baseline, not an established best method for finance.

## Evaluation and deliverables

Report:

- Accuracy, Cohen's kappa, and confusion matrices against the defined human-label policy.
- False acceptance of incorrect answers and false rejection of correct answers, with explicit denominators.
- Numerical perturbation detection and acceptance of valid numerical equivalents.
- Position and verbosity sensitivity, distinguishing order effects from repeated-run variability.
- Extraction failures, abstentions, and coverage; do not report retained-answer accuracy without coverage.
- Cost per 1,000 judgments, including extra calls, order swaps, and fallbacks.
- Confidence intervals using question-level resampling where appropriate.

Expected deliverables are reproducible preparation and evaluation code, versioned prompts and settings, a documented label audit, comparison tables and figures, and a report explaining results and limitations. Published data and outputs must respect source terms.

## Related work to read first

- [FinLFQA](https://aclanthology.org/2025.findings-emnlp.908/): financial long-form answers, attribution, numerical evaluation, and human comparison.
- [When the Judge Is Wrong](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6482162): financial judge reliability against graph-verified answers.
- [financebench-verification](https://github.com/fabio-rovai/financebench-verification): deterministic checks evaluated against released FinanceBench human labels; reproduce relevant audit findings before relying on them.
- [JudgeBench](https://arxiv.org/abs/2410.12784): evaluation of judges on objectively checkable response pairs.

These works overlap with the proposed study. Frame the contribution around controlled financial error types, valid numerical equivalents, transparent comparisons, and reproducible analysis rather than claiming the first financial judge evaluation.

## First milestone

Implement `prepare_data.py` and `inspect_example.py`. Display the available responses to `financebench_id_03029` and explain why their human labels are reasonable before making paid judge calls.
