# dbx-sdp-meta

Demo of [sdp-meta](https://docs.databricks.com/aws/en/ldp/developer/sdp-meta) (Databricks Labs metadata-driven framework for Lakeflow Spark Declarative Pipelines) on the `retail-org` sample dataset.

Everything is serverless and deployed with Declarative Automation Bundles. sdp-meta is pinned to `databricks-labs-sdp-meta==0.1.0`.

## Architecture

```
samples retail-org files
        |  00_setup_landing / 01_simulate_increments
        v
/Volumes/<catalog>/sdp_meta_landing/files/{retail_reference,retail_sales}/<entity>/
        |  Auto Loader (cloudFiles) - driven by dataflowspecs
        v
sdp_meta_bronze  (+ <table>_quarantine)      <- sdp-meta bronze pipelines x2
        |  select_exp / where_clause, AUTO CDC (SCD1 / SCD2, multi-source)
        v
sdp_meta_silver                              <- sdp-meta silver pipelines x2
        |  hand-written SQL
        v
sdp_meta_gold  (3 materialized views + 3 metric views)  <- gold_retail pipeline
        |
        v
AI/BI dashboard "SDP Meta Retail Demo"
```

Control plane: `sdp_meta_control.bronze_dataflowspec` / `silver_dataflowspec`, written by the onboarding job from `conf/onboarding.yml`.

| Data flow group | Entities | Pipelines |
| --- | --- | --- |
| `retail_reference` | products (CSV `;`), customers (CSV, SCD2), loyalty_segments, promotions (CSV multiLine), active_promotions (Parquet) | `sdp_meta_bronze_retail_reference`, `sdp_meta_silver_retail_reference` |
| `retail_sales` | sales_orders (JSON), sales_stream (JSON) -> silver `orders` (multi-source CDC) | `sdp_meta_bronze_retail_sales`, `sdp_meta_silver_retail_sales` |

## sdp-meta features demonstrated

* Auto Loader bronze ingestion, multiple formats and reader options, all from YAML
* Data quality expectations: `expect`, `expect_or_drop`, `expect_or_quarantine`, `expect_or_fail`
* Quarantine tables (`bronze_quarantine_table`)
* Silver transformations (`select_exp`, `where_clause`)
* SCD Type 2 customers (`silver_cdc_apply_changes`, with deletes)
* Multi-source CDC into one table (`silver_cdc_apply_changes_flows`)
* Incremental loading (new files only)
* Metadata-only onboarding of a new table (`suppliers` - uncomment in `conf/onboarding.yml`)
* Table properties, clustering, comments
* `data_flow_group` - several pipelines from one spec table
* Dataflowspec versioning / audit (`version`, `updateDate`, Delta history)

## Repo layout

| Path | Purpose |
| --- | --- |
| `databricks.yml` | Bundle definition (`sdp_meta_retail`, targets `dev` / `prod`) |
| `resources/variables.yml` | Catalog, schemas, groups, sdp-meta version, serverless |
| `resources/sdp_meta_onboarding_job.yml` | Onboarding job: stage conf to volume, write dataflowspecs |
| `resources/sdp_meta_pipelines.yml` | 4 sdp-meta pipelines + gold pipeline + `run pipelines` job |
| `resources/sdp_meta_retail_demo_job.yml` | Main job `sdp_meta_retail_demo` (onboard -> bronze -> silver -> gold -> metric views) |
| `conf/onboarding.yml` | All data flows (YAML) |
| `conf/dqe/<entity>/bronze_expectations.yml` | Data quality rules |
| `conf/silver_transformations/*.yml` | Silver select / where per group |
| `gold/gold_retail.sql` | Gold materialized views |
| `gold/metric_views/*.yml` | Metric view definitions |
| `notebooks/init_sdp_meta_pipeline` | The single generic pipeline notebook |
| `notebooks/00_setup_landing` | Copy retail-org samples into the landing volume |
| `notebooks/01_simulate_increments` | Batches 1-4: new sales, customer changes/deletes, bad rows, expect_or_fail |
| `notebooks/create_metric_views` | Creates metric views from `gold/metric_views/` |
| `notebooks/02_demo_walkthrough` | Presenter notebook (read-only) |
| `notebooks/99_reset` | Drops demo tables/checkpoints for a clean rerun (destructive) |

## Run it

1. Deploy: `databricks bundle deploy -t dev` (or the Deploy button in the bundle UI).
2. Run `notebooks/00_setup_landing` once to populate the landing volume.
3. Run job `sdp_meta_retail_demo`.
4. Optionally run batches in `notebooks/01_simulate_increments` and rerun the job.
5. Open the dashboard **SDP Meta Retail Demo** and `notebooks/02_demo_walkthrough`.

## Design decisions

1. Catalog `mg_aws_serverless_workspace_catalog`, schema-per-layer with prefix `sdp_meta_` (landing, control, bronze, silver, gold).
2. Dataset: `retail-org` samples only.
3. Source files copied into volume `sdp_meta_landing.files`.
4. All features above in scope.
5. Separate bronze + silver pipeline per data_flow_group (4 serverless triggered pipelines).
6. Declarative Automation Bundles in this repo.
7. YAML metadata.
8. Pin sdp-meta `0.1.0`; serverless everywhere.
9. Gold: SQL pipeline with `fact_order_lines`, `dim_customer_current`, `customer_history_changes`; metric views `retail_sales_metrics`, `customer_metrics`, `customer_change_metrics`; 3-page dashboard.
10. Repo structure as above.

## Gotchas (sdp-meta 0.1.0)

* `bronze_quarantine_table` is required whenever a bronze DQE file is set.
* `expect_or_quarantine` rules describe the *bad* rows; pair with an inverse `expect_or_drop`.
* Onboarding paths are literal - bundle variables are not substituted inside `conf/` files.
* Never delete a landing file Auto Loader has already seen; rewrite it instead.
* Quote metric view YAML expressions containing `: `.

## Not yet done

* File-arrival job for `retail_sales/` (bronze + silver sales -> gold).
* Publish the dashboard into the bundle (`resources/dashboard.yml`).
