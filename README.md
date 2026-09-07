# Fuzzy Entity Matching

[![CI](https://github.com/ItsMrZxD/fuzzy-entity-matching/actions/workflows/ci.yml/badge.svg)](https://github.com/ItsMrZxD/fuzzy-entity-matching/actions/workflows/ci.yml)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**Fuzzy entity resolution and record linkage between two CSV datasets — matches
company names that are spelled differently across sources, scoring every match
0–100. Built on RapidFuzz and pandas.**

The same company shows up as `Apple Inc.` in one system, `Apple` in another,
and `Aple Inc` in a spreadsheet someone typed by hand. This tool links those
records: every row in dataset A is matched to its closest row in dataset B,
with a confidence score you can threshold on. Typical uses are deduplication,
CRM and vendor-list reconciliation, and joining datasets that share no common
ID — the classic record-linkage problem.

Results are split into high- and low-confidence files around a configurable
threshold, so the confident matches can be used directly while the borderline
ones go to a human for review.

## Features

- **Fuzzy string matching** on company names with three selectable similarity
  metrics, benchmarked against each other on the included sample data
- **Name normalisation** that strips trailing legal suffixes and punctuation
  before comparison, while preserving original spellings in the output
- **Confidence scoring** (0–100) on every match, with a tunable high/low
  threshold
- **Three CSV outputs** — all matches, high confidence, and low confidence
- **Self-seeding sample data**, so the tool runs end to end on a fresh clone
  with no data of your own
- Deterministic, stable-sorted output that does not vary between runs
- Progress bar for large datasets, and a `--compare` mode that benchmarks all
  three metrics side by side

## Requirements

Python 3.11+. Dependencies are pinned in `requirements.txt`:

| Package | Purpose |
|---|---|
| `pandas>=2.0` | CSV loading and the results table |
| `rapidfuzz>=3.0` | the fuzzy string similarity metrics |
| `tqdm>=4.66` | progress bar (optional — matching works without it) |

## Installation

```bash
git clone https://github.com/ItsMrZxD/fuzzy-entity-matching
cd fuzzy-entity-matching
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## Usage

```bash
python main.py
```

On the first run, if `data/dataset_a.csv` and `data/dataset_b.csv` do not
exist, small sample datasets with intentionally messy company names are
generated automatically. Drop in your own CSVs — one column named `name` — to
match real data. Existing files are never overwritten.

Common variations:

```bash
python main.py --scorer token_sort      # pick a different similarity metric
python main.py --threshold 90           # stricter high-confidence cutoff
python main.py --compare                # benchmark all three metrics
python main.py --only-above-threshold   # export confident matches only
python main.py --data-dir ./mydata --output-dir ./results
```

### Command-line options

| Flag | Default | Description |
|------|---------|-------------|
| `--data-dir PATH` | `data/` | directory containing `dataset_a.csv` and `dataset_b.csv` |
| `--output-dir PATH` | `output/` | directory where result CSVs are written |
| `--threshold N` | `85` | confidence cutoff between high and low matches (0–100) |
| `--scorer {token_sort,token_set,wratio}` | `wratio` | similarity metric to use |
| `--only-above-threshold` | off | export only matches scoring at or above the threshold |
| `--compare` | off | benchmark all three metrics side by side **instead of** exporting matches |
| `--no-progress` | off | disable the tqdm progress bar |

### Output files

A default run writes three files to `output/`:

| File | Contents |
|---|---|
| `matches.csv` | every A record with its best B match |
| `matches_high_confidence.csv` | matches scoring at or above the threshold |
| `matches_low_confidence.csv` | matches below the threshold — the review queue |

Each has the columns `original_name_A`, `matched_name_B`, `confidence_score`.

## How the fuzzy matching works

1. **Cleaning** — every name is lowercased, punctuation is replaced with
   spaces, whitespace is collapsed, and trailing legal suffixes (`Co`,
   `Company`, `Corp`, `Corporation`, `Inc`, `Incorporated`, `LLC`, `LLP`,
   `Ltd`, `Limited`, `PLC`) are stripped. `"Coca-Cola Company, Inc."` and
   `"coca cola"` clean to the same string.
2. **Matching** — for each cleaned name in A, RapidFuzz's `process.extractOne`
   scans *every* cleaned name in B and returns the single best match with a
   0–100 similarity score.
3. **Reporting** — results keep the *original* spellings from both files, are
   sorted by confidence descending with a stable sort (so output is
   deterministic across runs and platforms), and are split into high/low
   confidence groups around the threshold.

Comparison runs on the cleaned names so that suffixes and punctuation cannot
dominate the score, but nothing you see in the output has been rewritten.

### Choosing a similarity metric

| Metric | How it scores | Best for |
|--------|--------------|----------|
| `token_sort_ratio` | Sorts the words in both names, then compares. Fixes word-order differences, but every word still has to be present. | Names with shuffled word order and few extra words. |
| `token_set_ratio` | Compares the *intersection* of words to each name. Extra words on one side barely hurt the score. | Names where one side has extra words (`"Tesla Motors"` vs `"Tesla"` → 100). Riskier: a short name inside a longer unrelated one also scores 100. |
| `WRatio` | RapidFuzz's weighted combination of several strategies with length penalties. | General-purpose default; avoids token_set's false-positive risk. |

Benchmarked on the sample data with `python main.py --compare`:

```
scorer        avg confidence  high (>= thr)  low (< thr)
token_sort              81.9              6            6
token_set               94.1             11            1
wratio                  92.2             11            1
```

`token_sort_ratio` punishes name pairs where one side has extra words, so it
misses obvious matches like *Tesla Motors → Tesla Inc*. `token_set_ratio` and
`WRatio` both recover all 11 true matches; **`WRatio` is the default** because
it avoids `token_set_ratio`'s known failure mode of scoring any substring-name
pair at 100. Switch metrics any time with `--scorer`.

## Changing the threshold

Two ways:

- Per run: `python main.py --threshold 90`
- Permanently: edit the `THRESHOLD` constant near the top of `main.py`

Raising it trades recall for precision. On the sample data, raising the
threshold from 85 to 95 moves borderline-but-true matches like
*Proctor & Gamble → Procter & Gamble Co* (92.9) into the low-confidence file —
exactly the group a human should review before trusting.

## Example output

`output/matches.csv` after a default run on the sample data:

```
original_name_A,matched_name_B,confidence_score
Apple Inc.,Apple,100.0
Microsoft Corporation,Microsoft Corp,100.0
Alphabet Inc.,Alphabet,100.0
Meta Platforms,Meta Platforms Incorporated,100.0
Berkshire Hathaway Inc.,Berkshire-Hathaway,100.0
Johnson & Johnson,Jonson and Johnson,95.0
The Coca-Cola Company,Coca Cola Co.,95.0
Proctor & Gamble,Procter & Gamble Co,92.9
Tesla Motors,Tesla Inc,90.0
"Amazon.com, Inc.",Amazon,90.0
Nvidia Corporaton,NVIDIA Corporation,90.0
International Business Machines,Tesla Inc,54.0
```

Console summary:

```
Summary
  Total records matched : 12
  Average confidence    : 92.2
  Above threshold (85) : 11
  Below threshold (85) : 1
```

## Tests

```bash
python -m unittest
```

Linting (the same check CI runs, configured in `pyproject.toml`):

```bash
pip install ruff
ruff check .
```

CI runs both against Python 3.11 and 3.14 on every push.

## Assumptions and known limitations

- Each input CSV has one column named `name`; rows with missing or blank names
  are dropped with a console note instead of crashing.
- Legal suffixes are only stripped from the **end** of a name, so a word like
  *Limited* in the middle of a name survives cleaning.
- Every A record gets its best B match even when nothing is truly similar —
  that is what the confidence score and threshold are for. The sample's
  *Oracle Corporation* (only in B) is never claimed by a good match.
- **Character-based fuzzy matching cannot resolve acronyms.** *International
  Business Machines* vs *IBM* shares almost no characters, so it lands at 54.0
  in the low-confidence file. Solving that needs an alias dictionary or
  embedding-based matching, both out of scope here.
- **Matching is O(len(A) × len(B)).** Fine for thousands of records; for
  millions you would want blocking or indexing first.
- Matching is one-directional and many-to-one: several A records can claim the
  same B record, and B records with no A counterpart are never reported.

## License

MIT — see [LICENSE](LICENSE).
