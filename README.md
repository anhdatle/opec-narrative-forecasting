# OPEC narrative forecasting: research code

Code accompanying *The Predictive Content of Official OPEC Narratives for Production Policy and Oil-Price Direction*.

Download and extract `opec-narrative-forecasting-code.zip` to obtain the complete source tree, then run the commands below from the extracted directory. The archive preserves the relative directories used by the scripts. This initial GitHub upload distributes the source as a ZIP, not as individually browsable source files.

The framework compares historical baselines, structured OPEC Narrative Index (ONI) features, and frozen retrieval-augmented LLM classification. Targets are next-month WTI direction and announced net-new policy action at retained decision dates, not realised production.

## Code-only release

This repository contains acquisition and analysis code, prompts and JSON schemas. It contains **no raw data, document manifests, extracted labels, saved predictions, numerical result files, model weights, credentials, or manuscript**. A schema is a specification, not a dataset. Data exclusion is the author's distribution choice; obtaining information through an API does not itself establish a redistribution prohibition.

This is a source-code release, not a frozen replication archive. Fresh source retrieval and LLM inference may change sample membership, labels and numerical results. The original price snapshot was retrieved on 24 July 2026. Live EIA and OPEC sources do not guarantee historical vintage reconstruction. Independent human criterion validation is not included.

## Environment

Use Python 3.11 or later in an isolated environment:

```sh
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m playwright install chromium
```

Dependency ranges are not a frozen original environment. Record installed versions when running a new study. LLM stages require a separately installed local Ollama server and suitable hardware/model licences; the primary model tag is `llama3.3:70b`. Do not run 70B inference without checking memory and runtime requirements.

## 1. Retrieve WTI from EIA API v2

Register for your own free key at [EIA registration](https://www.eia.gov/opendata/register.php). See [official API documentation](https://www.eia.gov/opendata/documentation.php).

Set `EIA_API_KEY` privately in your shell environment. Do not commit it or put it in command histories, screenshots or shared logs. Then run:

```sh
python scripts/fetch_wti.py
```

The downloader requests `https://api.eia.gov/v2/petroleum/pri/spt/data/`, `frequency=daily`, `data[0]=value`, `facets[series][]=RWTC`, ascending period order, and 5,000-row pages with increasing offsets. It saves period, series and value to a uniquely time-stamped CSV under `03_data/raw/eia/`. It avoids printing request URLs containing credentials.

Point the analysis at that actual new file, without renaming it to pretend it is the paper's historical snapshot:

```sh
export OPEC_WTI_CSV='/absolute/path/to/the/newly/downloaded.csv'
```

Monthly targets use arithmetic averages of available daily observations, then log ratios of monthly averages with a fixed neutral band of +/-0.02. Non-reporting days are not filled. The study's price horizon ends in June 2026. The legacy `03_data/pull/pull_eia.py` also fetches auxiliary Brent/futures series; these are not needed for the reported classification tasks.

## 2. Retrieve OPEC documents from official websites (not an API)

Sources are [OPEC press releases](https://www.opec.org/press-releases.html) and [MOMR publications](https://publications.opec.org/momr). The browser-based collectors save cleaned text locally and build `03_data/manifest/document_index.jsonl`:

```sh
python 03_data/scrape/scrape_opec_press_releases.py --sleep 2
python 03_data/scrape/scrape_opec_momr.py --sleep 2
```

Respect provider terms, access restrictions and request limits. Stop if access is denied; do not bypass authentication or challenges. Website structure changes may require maintenance. Press releases are deduplicated by release identifier; MOMR chapters by report/chapter identifiers. Reporting month is not independently verified first-publication time. New retrievals can include documents beyond the paper's corpus; reconstruct and audit the intended coverage locally before comparison.

## 3. Construct measurements locally

With Ollama available at `http://localhost:11434` (or your trusted `OLLAMA_URL`):

```sh
python 04_experiments/annotation/run_annotation.py --model llama3.3:70b
python 04_experiments/annotation/run_annotation.py --model llama3.3:70b --schema 02_method/prompts/delta_q_extraction_schema_v1.json --system-prompt-file 02_method/prompts/delta_q_system_prompt_v1.txt --out-tag llama3.3_70b_deltaq
python 04_experiments/annotation/build_event_calendar.py
python 04_experiments/annotation/build_oni_pca.py 03_data/derived/annotations/llama3.3_70b_annotations.jsonl
```

The event builder consumes extracted production decisions. Review its conflicts and date resolution before scoring. Machine adjudication records from the paper are intentionally not distributed as data; absence of those records can change labels. Re-running temperature-zero inference is not a guarantee of identical historical outputs. The corpus-level PCA script generates a compatibility/sensitivity representation; **origin-specific PCA is the paper's primary representation**, generated below.

## 4. Forecast and compare

Run from the repository root after all required inputs have been created:

```sh
python 04_experiments/models/run_trend_classification.py
python 04_experiments/models/run_rag_llm_classifier.py --task price --model llama3.3:70b
python 04_experiments/models/run_rag_llm_classifier.py --task production --model llama3.3:70b
mkdir -p 05_results
python 04_experiments/models/evaluate_rag_llm_classifier.py --input 04_experiments/models/rag_llm_price_llama3.3_70b_tfidf_v2.jsonl --expected 50 --output 05_results/rag_llm_price_summary.json
python 04_experiments/models/evaluate_rag_llm_classifier.py --input 04_experiments/models/rag_llm_production_llama3.3_70b_tfidf_v2.jsonl --expected 61 --output 05_results/rag_llm_production_summary.json
python 04_experiments/models/run_nested_rag_fusion.py
python 04_experiments/models/run_paired_forecast_tests.py
python 04_experiments/models/run_dependence_aware_tests.py
python 04_experiments/models/run_expanding_oni_pca_robustness.py
python 04_experiments/models/run_log_loss_inference.py
python 04_experiments/models/run_matched_policy_ablation.py
```

Inspect script input filenames before running; several analysis scripts consume saved outputs from preceding steps. Additional scripts expose threshold, seed and retriever sensitivity analyses. A missing artifact is not evidence that a result has been reproduced. The complete end-to-end pipeline has not been rerun from live sources for this release; syntax and local import checks are separate from numerical replication.

Chronological fitting and recorded-date retrieval restrictions apply to supplied observations, not pretrained model knowledge. Policy targets are machine-coded announced actions; maintain includes extensions/reaffirmations with no net-new adjustment. Treat probability scores separately from realised economic utility.

## Security and distribution

Keep data, outputs and credentials local; `.gitignore` excludes the main generated-data locations. Inspect staged files before every upload. No software licence is assigned in this initial release; contact the author before reuse beyond applicable platform rights. Model and source-provider terms remain separate.
