# Project documentation

This document records **decisions and incremental development** for outlet matching work in this repository. It complements `README.md` (service and batch CLI) and `docs/phase1-mvp.md` (deployable MVP).

---

## Two tracks (do not conflate)

| Track | Location | Problem |
|--------|-----------|---------|
| **Service / batch MVP** | `src/faiss_matcher/`, `artifacts/` from `build_index` | Match **distributor customer names** to **iNova master** names (single text field; FAISS index over master). |
| **HK VEEVA ↔ distributor** | `outlet_matching_cust_name_mapping.ipynb` (experiments); **`faiss_matcher.pipeline.hk_veeva_snowflake_match`** + `function_app.py` (Snowflake / hosted) | Link **VEEVA CRM HK accounts** to **HK distributor ship-to** rows (name **and** address features; assign / align with `MASTER_INOVA_ID`). |

The sections below focus on the **HK** track (notebook history, Snowflake CLI parity, embedding cache, and **deployment checkpoint** for hosted Snowflake read/write). The master-outlet FAISS service track above is separate unless we explicitly share code later.

---

## HK matching: milestone 1 (first version)

**Goal:** Establish an end-to-end baseline—load sales and VEEVA data, build a deduplicated distributor outlet table, and propose a best ship-to match per VEEVA row using semantic similarity.

**Approach:**

- Build comparable text on both sides by **concatenating name and full address** into one string (distributor: `SHIP TO NAME` + `ship_to_full_address`; VEEVA: `INO_ACCOUNT_NAME__C` + `ship_to_full_address_veeva`).
- **Embed** with `sentence-transformers/all-MiniLM-L6-v2`, **L2-normalize** vectors so inner product equals **cosine similarity**.
- **Retrieve** the single best candidate (initially **FAISS** `IndexFlatIP` over the distributor corpus; equivalent to exhaustive cosine search at this corpus size).

**Rationale:** One embedding per row is simple, fast to implement, and good for a first pass. It does **not** expose separate interpretability for “name vs address” and can let one field dominate the vector in ways that are hard to tune.

**Outputs:** Matched distributor fields (`matched_SHIP_TO_NAME`, `matched_ship_to_full_address`, `matched_MASTER_INOVA_ID`) and a single **match score** (`faiss_cosine_match_pct` as cosine × 100).

---

## HK matching: milestone 2 (weighted name + address)

**Goal:** Make scoring **explicitly two-signal**, align with business intuition (legal / account name vs address), and handle **missing names** without inventing a fake name similarity.

**Approach:**

- Embed **four** lists separately: distributor name, distributor address, VEEVA account name, VEEVA address string.
- For every VEEVA row and every distributor row, compute cosine similarity for **name–name** and **address–address** (`sim_n`, `sim_a`).
- **Decision rule (product requirement):**
  - If **both** names are non-empty (after trim): **combined = 0.5 × sim_n + 0.5 × sim_a**.
  - If **either** name is missing: **combined = sim_a** (100% weight on address for that pair).
- Choose the distributor row that **maximizes `combined`** (full matrix over ~3.7k distributors is acceptable; **FAISS is not required** for this step at current scale).

**Rationale:**

- A **weighted sum** (here 50/50) is easier to explain and threshold than a single concatenated embedding.
- Using **address-only** when a name is absent avoids blending in meaningless “empty string” name embeddings.
- We deliberately **did not** use `max(sim_n, sim_a)` as the primary ranker: that tends to **over-accept** when one field accidentally looks strong while the other is wrong.

**Outputs:** `faiss_name_match_pct`, `faiss_address_match_pct`, `faiss_weighted_match_pct`; `faiss_cosine_match_pct` is kept as an alias of the weighted score for backward compatibility.

---

## Embedding cache (after milestone 2)

**Problem:** Four encoding passes over the full HK sets are slow on repeated notebook runs.

**Decision:** Persist float32 matrices under `artifacts/hk_veeva_dist_emb_cache/` with `meta.json` storing **model id**, **MD5 fingerprint** of all input strings (four lists in fixed order), and **row counts**. On fingerprint or length mismatch, cache is ignored and regenerated.

**Repo hygiene:** That directory is listed in `.gitignore` so large `.npy` files are not committed by default.

---

## Deployment checkpoint: HK pipeline, Snowflake in/out (hosted)

**Status:** We treat this as a **checkpoint**: Snowflake **read** (HK sales + VEEVA via `sql/*.sql`) and **write** (auto / manual result tables via `write_pandas`) are verified in the deployment target, and the primary operating mode is **no local CSV inputs**—the same logic that worked on a laptop now runs where IT hosts it (Azure Function App in our case).

**What we wanted to avoid**

- **Two pipelines:** Maintaining different code for “local script” vs “cloud” invites drift. The match stays in `faiss_matcher.pipeline.hk_veeva_snowflake_match.run_pipeline()`; the Function App is only an HTTP shell that parses optional JSON and calls that function (see repo-root `function_app.py`).
- **Mystery paths in the cloud:** Locally, relative paths like `sql/hk_sales_zuellig.sql` work because the shell’s cwd is the repo root. Hosted workers may not start with that cwd. **`HK_VEEVA_REPO_ROOT`** (set automatically in `function_app.py` to the deployment root, overridable via App Settings) resolves SQL files and the embedding cache directory against a known root so behavior matches the CLI without hard-coding absolute paths in code.
- **Secrets in files:** `.env` is fine for development; in Azure, the same **names** should appear under **Application Settings** (and secret material via Key Vault references or the platform’s secret store), not in the deployed zip.

**Thought process (why keep SQL files in the repo?)**

- Queries stay **reviewable in Git**: changing filters or joins is a PR, not a one-off edit in a portal.
- **Fully qualified** table names inside SQL keep reads working when sales and VEEVA live in different databases/schemas—no extra indirection layer required for the first milestone.

**Thought process (local CSV flags)**

- `--local-hk-sales-csv` / `--local-veeva-csv` remain a **debug escape hatch** (e.g. reproducing a row without querying prod). **Production** runs omit them; all source data comes from Snowflake.

**Thought process (compute and cache)**

- **Model load + encoding** dominate runtime and memory. A small Consumption plan may hit **timeout or OOM**; if that happens, the fix is infrastructure (larger worker, Premium, dedicated plan, or eventually a container/job pattern)—not silent truncation of the match logic.
- Deploy artifacts often **exclude** `artifacts/hk_veeva_dist_emb_cache/` (see `.funcignore`) to keep packages small. **First run** on a fresh instance may rebuild the embedding cache (slower). If cold-start cost becomes a problem, next steps are persistent storage for that directory or baking a cache into the deployment with eyes open on size and freshness.

**Operational pointer**

- HTTP trigger: `function_app.py`, route `hk-veeva-snowflake-match`. POST body can include `create_tables`, `no_write`, `quiet`, and optional `database` / `schema` / `warehouse` overrides—equivalent to the CLI flags for first-time table creation and testing.

---

## Snowflake CLI (same HK match)

The notebook pipeline is also exposed as `python -m faiss_matcher.pipeline.hk_veeva_snowflake_match`: load HK sales and VEEVA rows via SQL files (defaults: `sql/hk_sales_zuellig.sql` → `DEV_IN_RAW.ZUELLIG.HK_SALES_ZUELLIG`, `sql/veeva_address_hk.sql` → `PROD_RAW.VEEVA.VEEVA_ADDRESS` with HK filter), run `hk_veeva_dist_match` (weighted embeddings + cache), split rows with `Thresholds` / `decide_action` into **auto** vs **manual/review**, and `write_pandas` to **`DEV_IN_RAW.TEST.FAISSMATCH_HK_AUTO`** and **`DEV_IN_RAW.TEST.FAISSMATCH_HK_MANUAL`** by default (override with flags or `SNOWFLAKE_DATABASE` / `SNOWFLAKE_SCHEMA` / `--auto-table` / `--manual-table`). Reads use fully qualified table names so sales and VEEVA can live in different databases. Progress logs use prefixes `[hk_veeva_sf]` and `[hk_veeva_match]`; pass **`--quiet`** to reduce console output.

**Authentication** is centralized in `faiss_matcher.snowflake_util.connect_from_env`: **`SNOWFLAKE_AUTHENTICATOR=snowflake_jwt`** with either **`SNOWFLAKE_PRIVATE_KEY_KV_SECRET_ID`** (Azure Key Vault **Secret** URL — PEM private key in the secret value), or **`AZURE_KEY_VAULT_URL`** + **`SNOWFLAKE_PRIVATE_KEY_KV_SECRET_NAME`**, or **`SNOWFLAKE_PRIVATE_KEY_PATH`** (local PEM). PAT/password/SSO paths unchanged. A **PAT is not** key-pair auth. Non-exportable Key Vault **Keys** cannot be used for Snowflake’s JWT login; store the PKCS#8 PEM as a **Secret**.

---

## Operational notes (notebook / local runs)

- **`TOKENIZERS_PARALLELISM=false`** and conservative **`batch_size`** during `encode` reduced flaky behavior and segfaults on **macOS** in practice; full-matrix scoring is cheap relative to encoding.
- **Stability of `MASTER_INOVA_ID`:** IDs are assigned from the **current row order** of the deduplicated distributor dataframe. If that ordering changes, IDs are not stable across runs unless we later pin ordering (e.g. sort by `SHIP TO CODE`).

---

## Likely next steps (not implemented here)

- **Thresholds and review queues:** auto-accept only above tuned cutoffs; export ambiguous rows (as in `faiss_matcher.pipeline.batch_match` outputs).
- **Blocking / hybrid retrieval:** if the distributor corpus grows, replace full `argmax` over all pairs with FAISS (or blocks keyed on district / prefix) and **rerank** with the same weighted rule.
- **Timer trigger or orchestration:** if HK match should run on a schedule, add a time-triggered function (or move long runs to Durable Functions / container job) once HTTP-only validation is stable.
- **Persistent embedding cache** in the hosted environment if cold-start latency is unacceptable.

---

*Last updated: HK notebook milestones (baseline → weighted name/address → embedding cache); Snowflake CLI parity; deployment checkpoint with Snowflake read/write on hosted worker and Azure Functions entry (`function_app.py`, `HK_VEEVA_REPO_ROOT`).*
