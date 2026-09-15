# ATI bronze-to-silver ingestion failure — handoff

**Incident:** 2026-09-03 (and earlier runs on 2026-08-20), partitions `20260505`–`20260707`.
**Platform:** Azure Synapse. Storage account `saedmsprdizadls01`, containers `caas-edms-bronze` / `caas-edms-silver`.
**Pipelines:** `00_pl_edms_bronze_to_silver_AT` (split + header cleanup) → `21_pl_edms_bronze_to_silver-excel_fileloop` (copy).
**Config:** `config.ETL_Lakehouse_Config`, `datalake_group = 'edms_at_ati'` (exported as `ATI_ETL_Config_{PRD,UAT}.csv`).

Last updated 2026-09-15. Everything below marked *verified* was reproduced locally by running the
notebooks' own Python against the real xlsx files and config — no Azure calls.

---

## Observed errors (pipeline 21, activity `copy-bronze-to-silver`)

| Flow | Run | Error column | Position in mapping |
|---|---|---|---|
| Airline List | 20 Aug / 3 Sep | `CODE` | **1st** |
| Freighter | 20 Aug (failed) / 3 Sep (succeeded) | `AIRLINE` | **1st** |
| City Links | 3 Sep | `OpsType_Codeshare` | **12th of 13** |
| Pax | 3 Sep | — succeeded | — |

All read `...while read data from worksheet ''`.

`delete-from-silver_landing` → `PathNotFound` on `parquet-landing/<flow>/partition=<date>` appears
alongside every failure. That is a first-load symptom (nothing to delete yet), not a cause — but it
is currently allowed to fail the run.

---

## Key diagnostic: the position of the reported column

ADF reports the **first** mapped column when it finds *none* of the mapped columns in the sheet it
read. It reports a **middle** column when it found the rest and that one is genuinely absent.

So the errors are two different situations:

- **`CODE`, `AIRLINE`** — ADF read a sheet with zero matching columns. These are *not* column problems.
- **`OpsType_Codeshare`** — ADF read the correct sheet, found 12 of 13 columns. Genuine missing column.

---

## Confirmed — `OpsType_Codeshare` is missing from the client's file (City Links only)

*Verified.* The July City Links sheet has 12 header columns; the 2017 file has 13, with
`OpsType_\nCodeshare` (literal newline, confirmed in `sharedStrings.xml`) at column 12. Both PRD and
UAT mappings still expect `OpsType_\nCodeshare`. The mapping cannot drop it — older files carry it.

**Fix applied:** NOTEBOOK B now backfills missing mapped columns as empty (see *Changes applied*).

---

## Ruled out (verified)

- **Notebooks mangling the header.** NOTEBOOK A's `df.columns = df.iloc[0]` does take the report
  title as the header (all four sheets have a title at row 1), but NOTEBOOK B's schema-matching
  re-finds the real header and repairs it. Simulated end-to-end on the July file: all four sheets
  repair correctly; `CODE` and `AIRLINE` are present in the output. The actual cleaned file from the
  end of pipeline 00 (`ATI Airline List_20260707.xlsx`) has one sheet, `Sheet1`, `CODE` at A1,
  151 rows — matches the simulation exactly.
- **The `None`-filename bug.** `update_filename()` does return `None` for names not matching
  `%d %b %y` (`ATI 07 July 26`, `ATI 07 Jul 2026`, `ATI_20260707`), and there is no guard before the
  copy+delete. But for these runs the split worked and files were correctly named. Real latent
  defect, not this incident.
- **UAT/PRD divergence.** Notebook code identical apart from storage account / key vault names.
  Config equivalent: City Links `dynamic_data_range = 2`, Airline List `= 1` in both — CSV rows are
  only ordered differently.
- **Folder-level read.** Pipeline 21 passes a specific `file_name` to `ds_excel_source`;
  `recursive: true` is a no-op.
- **Static `ds_excel_source` misconfiguration.** Would break every file, including the ATSS flows
  and the months that succeed.
- **Trailing `Unnamed: N` columns.** Pax has them and succeeds; Freighter has none and failed.
- **Airline List mandatory-column truncation.** `silver_mandatory` is `CARRIER,FSC_LCC,2_LETTER`, not
  `CODE`. Extraction stops correctly at the footnote block (~123-151 rows depending on month).

---

## Open — why `CODE` / `AIRLINE` fail

The decisive fact any explanation must meet: **Freighter failed on 20 Aug and succeeded on 3 Sep
with the same file and config.** That rules out anything based on file content.

**Leading hypothesis:** pipeline 21 was handed the **original client workbook** (`ATI_20260707.xlsx`)
rather than the split file. The 2026 exports carry a new `Cognos_Office_Connection_Cache` sheet at
index 0 — a 1×1 sheet with an empty A1. The 2017 file has no such sheet. A `sheetIndex`-based
dataset (consistent with `worksheet ''`) would read zero columns and report the first mapped column.
Whether the original is still in the landing path when the file loop enumerates is state-dependent,
which fits the 20 Aug / 3 Sep difference.

**To confirm — one query:** pipeline 21 logs
`p_source_name = @concat(p_bronze_landing_path, p_file_name)` through `99_pl_job_log_4`, which
succeeded in every failed run. Read `source_name` for those runs:

- `.../raw/ATI_20260707.xlsx` → confirmed.
- `.../raw/ATI Airline List_20260707.xlsx` → hypothesis wrong; next look at `ds_excel_source`
  (`sheetName` / `sheetIndex`, `firstRowAsHeader`).

---

## Changes applied

1. **NOTEBOOK B** (`00_remove_invalid_headers_xlsx_dynamic_PRD.ipynb` and `_UAT.ipynb`),
   `excel_blob_to_dataframes`: after concatenating tables, any mapped source column absent from the
   sheet is added as empty with a warning. More than two missing raises instead — that pattern means
   a wrong header row, not schema drift. Names come from `extract_source_columns_from_mapping`, so
   the newline in `OpsType_\nCodeshare` is preserved. Same logic verified on the July City Links
   sheet: missing `['OpsType_\nCodeshare']` → `[]`, 181 rows kept. The edited cell itself is
   checked only for valid JSON and Python syntax — not yet executed locally (pandas isn't
   installed on this machine). Pre-edit copies are in `backup/`.

## Recommended, not applied

- **Pipeline 00:** change the dependencies from `Completed` to `Succeeded` so a notebook
  failure stops the run before the copy.
- **Pipeline 21:** make `delete-from-silver_landing` tolerate a missing partition (Get
  Metadata + If on `exists`). Until then, leave `copy-bronze-to-silver`'s `Completed`
  dependency on the delete as it is — switching it to `Succeeded` would stop every first load.
- **NOTEBOOK A hardening (latent data-loss bug):** guard `if not new_filename: continue` before
  the copy+delete; add `%Y%m%d` and `%d %B %y` fallback formats; derive `main_identifier` from
  `tokens[0]` rather than the renamed file.
- **If the job log confirms the original workbook was read:** restrict pipeline 21's file loop to
  split outputs (names starting with the flow's `source_name`), and/or select the sheet by name.

---

## Client communication

- **City Links** — the client dropped `OpsType_Codeshare`. The backfill makes this non-blocking on
  our side; worth confirming with them whether the removal is permanent.
- **Airline List / Freighter** — ours. Nothing for the client to change; a re-upload will not fix it.
- The client was told early on that headers had line breaks needing removal. That was wrong — they
  checked and were right.
