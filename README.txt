CLASSICAL AI REASONING AGENT
============================

Natural-language knowledge base -> symbolic representation -> classical reasoning -> True / False

Files
-----
reasoning_agent.ipynb  Complete implementation, evaluation on the public datasets (accuracy and
                       inference time are printed), reasoning traces, writes submission.csv and
                       regenerates report.pdf.
report.pdf             Representation, reasoning algorithms, experiments, reasoning traces.
submission.csv         Predictions: columns  id,flag  (flag is True or False).
README.txt             This file.

Requirements
------------
Python 3.9 or newer.
    pip install datasets pandas reportlab
The reasoning engine itself uses only the Python standard library. `datasets` downloads the
public data from Hugging Face; `reportlab` is only needed to rebuild report.pdf (last cell).

yale-nlp/FOLIO is a gated dataset: accept its terms on its Hugging Face page and run
    huggingface-cli login
once. Alternatively, put FOLIO_train.jsonl and FOLIO_validation.jsonl next to the notebook.

Results on the public datasets (end-to-end from English, from report.pdf)
-------------------------------------------------------------------------
Dataset                         Queries   Accuracy   Majority baseline
RuleTaker (jise/ruletaker)         1625     98.46 %       54.22 %
CounterBench                       1200     75.00 %       50.42 %
FOLIO (yale-nlp/FOLIO)             1204     62.21 %       61.79 %
All public queries                 4029     80.64 %
Total inference time: 1.67 s (0.41 ms per query)
FOLIO with its gold first-order formulas (engine check, not in the total): 81.06 %

Testing in Google Colab (recommended)
-------------------------------------
1. Go to https://colab.research.google.com -> File -> Upload notebook -> reasoning_agent.ipynb
2. FOLIO is gated. Once: open https://huggingface.co/datasets/yale-nlp/FOLIO and accept the terms,
   then create a "Read" token at https://huggingface.co/settings/tokens.
3. In Colab click the key icon (Secrets) on the left, add a secret named HF_TOKEN with the token
   as value, and switch "Notebook access" on.
   (No token? Drag FOLIO_train.jsonl and FOLIO_validation.jsonl into Colab's Files pane instead.)
4. Runtime -> Run all. The data loading cell should print, for each dataset, a line such as
       RuleTaker: ... queries from jise/ruletaker
   If it prints "fallback" instead, that dataset was not downloaded: check the internet / token.
5. Read the evaluation table: FINAL ACCURACY and TOTAL INFERENCE TIME are printed below it, and
   the next cells show accuracy per split/depth and a few wrongly answered examples.
6. Download the results from the Files pane: report.pdf and submission.csv (rebuilt by the run),
   and the notebook with its outputs via File -> Download -> Download .ipynb.
7. To check the private-test path, upload a test.csv (columns id, context, statement) to the
   Files pane and run the notebook again: submission.csv will then hold its predictions.

How to run locally
------------------
1. Open reasoning_agent.ipynb in Jupyter / VS Code / Colab.
2. Kernel -> Restart & Run All.
3. The evaluation cell prints, per dataset and overall:
       FINAL ACCURACY (...)  and  TOTAL INFERENCE TIME (...)
4. submission.csv is written next to the notebook.

Private test set
----------------
Put the test file next to the notebook as test.csv (csv, json or jsonl are accepted; change
TEST_PATH in the first code cell otherwise). Columns are detected automatically:
    id        : id / example_id / question_id      (row index if absent)
    context   : context / given_info / premises / theory
    statement : statement / question / query / conclusion
    optional  : premises-FOL, conclusion-FOL (used by the resolution prover when present)
Running the notebook then writes submission.csv with one prediction per test row.
If no test file exists, submission.csv contains predictions for the public evaluation queries.
The prediction column name (default: flag) can be changed with LABEL_COLUMN in the first code cell.

Datasets used for development
-----------------------------
RuleTaker     jise/ruletaker            (splits train, test, ood_test; columns context, statement, flag)
CounterBench  CounterBench/CounterBench (data_balanced_alpha_V1.json, data_balanced_backdoor_V2.json)
FOLIO         yale-nlp/FOLIO            (splits train, validation)
If a download fails, the notebook says so and falls back to a few built-in RuleTaker rows,
built-in CounterBench template items, and local FOLIO files.

Reported accuracy is end-to-end from English (context + statement only). FOLIO also ships
first-order formulas; a separate table row shows the prover on those as an engine check, and
it is not included in the final accuracy.

System design (short)
---------------------
* Parsing (rule-based, no machine learning, no generative AI):
    - controlled English -> signed literals and Horn rules with variables
    - ground rules with fixed connectives ("Assuming A, we can say B") -> propositional rules
    - causal sentences   -> Boolean structural equations, priors, evidence, interventions
    - FOL strings        -> AST -> NNF -> Skolem normal form -> clauses
    - free English       -> shallow NL->FOL patterns (fallback)
* Reasoning (classical):
    - forward chaining with unification and stratified negation as failure (closed world),
      or classical negation (open world); backward chaining as a cross-check
    - counterfactuals: DFS enumeration of possible worlds, abduction -> intervention -> forward
      propagation, prior-weighted vote
    - resolution refutation: set of support, lightest-clause-first (uniform-cost) given-clause
      loop, factoring, subsumption, per-query budget
* Output: True if the statement is entailed by the knowledge base, else False.
