# LLM SQL Eval

A benchmark framework for evaluating large language models (LLMs) on text-to-SQL generation tasks. This repo measures how well various LLMs can generate correct SQL queries from natural language questions against the IPL cricket database (2021–2024 seasons).

## What This Does

**LLM SQL Eval** tests LLMs on their ability to convert natural language questions into executable SQL queries. It:

1. Takes questions in plain English + the database schema
2. Calls an LLM (via OpenRouter) to generate SQL
3. Runs the generated SQL against a real SQLite database (IPL 2021–2024)
4. Compares the results to hand-verified "gold standard" queries
5. Scores each model by execution accuracy across test sets

This is a **practical evaluation** of SQL generation—the model's answer only counts if the query runs without error *and* returns the correct rows.

## Stack

- **Language:** Python 3
- **Framework / Runtime:** LangChain + OpenRouter API (multi-model endpoint)
- **Key Libraries:**
  - `langchain-openrouter` — unified LLM interface
  - `pandas` — data manipulation and result comparison
  - `sqlite3` — database execution
  - `python-dotenv` — environment config

## Repository Structure

```
.
├── main.py                      Main eval orchestrator; iterates models, runs queries, logs scores
├── evaluator.py                 Comparison logic; "execution accuracy" with flexible column matching
├── golden_dataset_generator.py  20 hard/brutal test questions with gold SQL
├── golden_dataset.csv           Questions, gold queries, and expected results
├── schema_extractor.py          Utility to extract schema from database
├── db.py                        Builds SQLite DB from Kaggle IPL CSV files
├── schema.sql                   Database schema (2 tables: matches, deliveries)
├── model_openrouter_slug.py     Lists the 5 models under test
├── ipl_2021_2024.db             SQLite database (~7.4 MB); IPL match & delivery data
├── eval_results.csv             Results from latest run; one row per (model, question)
├── golden_dataset.csv           Golden (test) dataset; 30 curated questions
└── data/                        Input CSVs for building the DB (not tracked; download separately)
```

### How It Fits Together

1. **Setup phase**: `db.py` loads IPL match and delivery data from Kaggle CSVs into `ipl_2021_2024.db`.
2. **Question phase**: Golden dataset (`golden_dataset.csv`) contains 30 test questions, each with a gold SQL query and expected result.
3. **Generation phase**: For each model (via OpenRouter), `main.py` sends the schema + question to the LLM.
4. **Execution phase**: The model's generated SQL runs against the database; if it errors, the result is `None`.
5. **Evaluation phase**: `evaluator.py` compares the generated result to the gold result using **execution accuracy**—values match if they're semantically equivalent (row order, column count, etc. can vary).
6. **Reporting phase**: Per-model scores written to `eval_results.csv`; console shows detailed pass/fail for each question.

## Evaluation Methodology

### Execution Accuracy (Path A)

The evaluator compares **data values only**, not schema or order:

- **Row count must match** exactly.
- **Values compared semantically**: `24395` == `24395.0` (numerically), NULL == None, strings trimmed and compared.
- **Column count flexible**: If the gold result has 2 columns but the model returned 3 (e.g., added a COUNT column), the evaluator checks if the smaller set's values appear as a subset in the larger set, row by row.
- **Row order ignored** by default (`order_sensitive=False`); set `order_sensitive=True` for queries where order matters (e.g., rankings).

**Why flexible column count?** Many cricket questions allow multiple valid answers. For example, "Who scored the most?" can correctly return just the name, or name + run total. The evaluator tolerates both.

### Test Set: Easy (30 questions) + Hard (20 questions)

- **Easy** (`golden_dataset.csv`): 30 foundational questions covering CTEs, aggregates, joins, NULL filtering.
- **Hard** (`golden_dataset_generator.py`): 20 advanced questions with 1–5 traps each: ratios, HAVING clauses, wicket attribution, self-referential logic.

## How to Run It

### Prerequisites

1. **Python 3.8+** and pip
2. **OpenRouter API key** (free tier available; get one at https://openrouter.ai)
3. **IPL Database** (already included: `ipl_2021_2024.db`)

### Setup

```bash
# Clone repo
git clone https://github.com/narsingh2231/llm-sql-eval.git
cd llm-sql-eval

# Install dependencies
pip install langchain-openrouter python-dotenv pandas

# Create .env file with your OpenRouter key
echo "OPENROUTER_API_KEY=sk-or-..." > .env
```

### Run the Evaluation

```bash
# Evaluate all 5 models (GPT-5.6 Terra, Kimi K3, Grok 4.5, Claude Sonnet 5, MiniMax M3)
python main.py
```

**Output:**
- Console logs each model's progress: per-question pass/fail + per-model score.
- `eval_results.csv` — detailed results (model, question ID, difficulty, correct/incorrect, reason, generated SQL).

### (Optional) Rebuild the Database

If you have the Kaggle IPL dataset files (`data/matches.csv`, `data/deliveries.csv`):

```bash
python db.py
```

This rebuilds `ipl_2021_2024.db` and validates referential integrity.

### Validate Golden Queries

Check that all 20 hard queries execute without error:

```bash
python golden_dataset_generator.py
```

## Key Files

| File | Purpose |
|------|---------|
| `main.py` | Orchestrates the full eval loop: load models, generate SQL, run on DB, evaluate, score. |
| `evaluator.py` | Core comparison logic; `evaluate_one()` returns `{"correct": bool, "reason": str}`. |
| `golden_dataset_generator.py` | Hard test set (20 questions); each with embedded SQL and explanation of traps. |
| `model_openrouter_slug.py` | Maps friendly model names to OpenRouter API slugs. |
| `db.py` | Loads Kaggle IPL CSVs, filters to 2021–2024, builds indexed SQLite DB. |
| `schema.sql` | DDL for `matches` and `deliveries` tables; included in prompts to LLMs. |
| `ipl_2021_2024.db` | Pre-built SQLite database; ready to use. |
| `eval_results.csv` | Latest run results; one row per (model, question). |

## Example Output

```
============================================================
MODEL: GPT-5.6 Terra  (openai/gpt-5.6-terra-pro)
============================================================
  #1  [easy  ] OK   match
  #2  [easy  ] OK   match
  #3  [hard  ] XX   mismatch
  ...
  SCORE: 24/30 = 80.0%

============================================================
FINAL SCOREBOARD (execution accuracy)
============================================================
  GPT-5.6 Terra     24/30  =  80.0%
  Kimi K3           22/30  =  73.3%
  Grok 4.5          26/30  =  86.7%
  Claude Sonnet 5   28/30  =  93.3%
  MiniMax M3        20/30  =  66.7%

Detailed results saved -> eval_results.csv
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Missing OPENROUTER_API_KEY` | Add `OPENROUTER_API_KEY=sk-or-...` to `.env` file. |
| `ModuleNotFoundError: langchain_openrouter` | Run `pip install langchain-openrouter`. |
| SQL generation timeout | OpenRouter may rate-limit; wait a moment and re-run. |
| `ipl_2021_2024.db` not found | Database is in the repo; if missing, regenerate with `python db.py`. |

## Citation & Data

- **IPL Dataset**: From [Kaggle](https://www.kaggle.com/datasets/amogh2004/ipl-20-complete-dataset-20082024).
- **Model Providers**: [OpenRouter](https://openrouter.ai) aggregates access to GPT, Claude, Grok, and others.

## License

MIT (or as specified in the repository).

## Contributing

To add new test questions, edit `golden_dataset_generator.py`, add a new dict to the `GOLDEN` list, validate with `python golden_dataset_generator.py`, and re-run `main.py`.

---

**Last updated:** September 2026  
**Status:** Active evaluation framework
