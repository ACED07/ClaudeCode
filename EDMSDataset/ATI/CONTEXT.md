# ATI — monthly client workbook, bronze to silver

**Last updated:** 2026-09-15 16:04 +0800

One job: hold the exported pipeline pieces for ATI ingestion and the durable facts needed to
change them safely. Current incident state is in `ATI_ingestion_handoff.md`, not here.

## Inputs

- Reference (every read): the exports in this folder — they're the source of truth *for this
  repo*, but can lag Synapse. Re-verify a load-bearing claim before acting on it.
- Working: the client's sample workbooks (`ATI_20170808.xlsx` ingested fine;
  `ATI_20260707.xlsx` failed) and one cleaned output (`ATI Airline List_20260707.xlsx`, as
  it looked at the end of pipeline 00).

## Files

| File | What it is |
|---|---|
| `00_pl_edms_bronze_to_silver_AT_{PRD,UAT}.json` | pipeline 00: split + header cleanup, then hands off to the copy |
| `00_split_excel_worksheets_{PRD,UAT}.ipynb` | **Notebook A** — renames the upload, splits it into one file per worksheet |
| `00_remove_invalid_headers_xlsx_dynamic_{PRD,UAT}.ipynb` | **Notebook B** — finds the real header by schema match, rewrites each split file in place |
| `21_pl_edms_bronze_to_silver-excel_fileloop_PRD.json` | pipeline 21: per-file copy to silver parquet (no UAT export yet) |
| `ATI_ETL_Config_{PRD,UAT}.csv` | export of `config.ETL_Lakehouse_Config` for `datalake_group = 'edms_at_ati'` |
| `backup/` | dated pre-edit copies of notebooks |

Not in this repo: the standard (non-dynamic) header notebook
`00_remove_invalid_headers_xlsx_standard`, the dataset `ds_excel_source`, and the
`01_extract_partition_date` notebook.

## Flow

1. **Pipeline 00** → `get-config worksheet` (only rows with `flag=1`, i.e. the pax_breakdown
   row) → **Notebook A** runs once and splits all four sheets.
2. `get-config` → `ForEach2` over every config row → `IfCondition`
   `@greater(coalesce(item().dynamic_data_range, 0), 0)` → **Notebook B** (dynamic), else the
   standard notebook.
3. **Pipeline 21**, once per file: `get_partition` (notebook `01_extract_partition_date`) →
   `extract_partition_token` → `delete-from-silver_landing` → `copy-bronze-to-silver` from
   `ds_excel_source` (passed `file_name`, no sheet parameter). The copy depends on the delete
   with `Completed`, so it runs even when the delete fails. On copy success:
   `copy-bronze-raw-to-archive` → `delete-from-raw`. On copy failure: `99_pl_job_log_4`, which
   logs `bronze_landing_path + p_file_name`.
4. Every `dependsOn` in pipeline 00 is `Completed`, never `Succeeded`.

## Config fields that matter

| Field | Notes |
|---|---|
| `worksheet`, `original_source_name`, `flag` | set only on the pax_breakdown row; that row drives Notebook A for all four sheets |
| `date_format`, `date_normalization_enabled` | `%d %b %y`, `True` — the upload must look like `ATI 07 Jul 26.xlsx` |
| `dynamic_data_range` | Airline List / Pax / Freighter = `1` (all tables); City Links = `2` (first table only); same in PRD and UAT |
| `silver_schema_mapping` | source → sink names; the source names are what must appear in the header row |
| `silver_mandatory` | sink names, resolved to source names; a blank in **any** of them ends a table |
| `source_name` | e.g. `ATI Airline List` — the filename prefix Notebook B uses to pick its file |

PRD and UAT CSVs store rows in a different order. Compare them by `datalake_flow_name`;
a line diff reports differences that aren't there.

## Landmines

1. **Notebook A can destroy the upload.** `update_filename()` returns `None` when no token
   parses against `date_format` (`ATI 07 July 26`, `ATI 07 Jul 2026`, `ATI_20260707`). There
   is no guard: the file is copied to a blob literally named `None` and the original is
   deleted. Split outputs become `None <sheet>.xlsx`, which Notebook B skips.
2. **Notebook A takes row 1 as the header** (`df.columns = df.iloc[0]`). All four sheets have a
   report title there. Notebook B repairs this by schema match; the standard notebook is unverified.
3. **Split files have one sheet, `Sheet1`** (pandas default — Notebook A calls `to_excel`
   without a sheet name). Notebook B keeps that name.
4. **2026 exports start with an empty sheet**, `Cognos_Office_Connection_Cache` at index 0.
   The 2017 file has none. Anything that reads the *original* workbook by sheet index gets
   zero columns.
5. **Reading `ExcelInvalidColumnName`.** If ADF names the *first* column in the mapping, it
   found none of the columns — the header or sheet is wrong, not one column. If it names a
   *middle* column, the rest matched and that one is genuinely absent. `worksheet ''` means
   the dataset selects by index.
6. **`OpsType_\nCodeshare` has a literal newline** in both the client header and the mapping.
   Never hand-type it; take names from `extract_source_columns_from_mapping`.
7. **`delete-from-silver_landing` fails with `PathNotFound` on a first load** (no partition
   yet). The copy still runs because it depends on the delete with `Completed`, but the failed
   delete is reported against the run. Don't switch that dependency to `Succeeded` until the
   delete tolerates a missing path, or every first load stops.
8. **The archive can't tell you what the client uploaded.** Notebook A archives under the
   renamed name.
9. **Airline List ends in a footnote block** (a legend row with `CODE` alone in column A).
   The mandatory-column check stops there, which is correct.

## Verifying locally

No Azure calls. Run the notebook's own code, not a re-implementation:

1. `pip install pandas openpyxl` (not installed by default here).
2. Load the config row from the CSV by `datalake_flow_name`.
3. Read the cell source out of the `.ipynb` JSON and `exec` it with `p_dynamic_data_range`
   set from that row.
4. Reproduce Notebook A's split (`df.columns = df.iloc[0]; df = df[1:]`, `to_excel` with no
   sheet name) on the sample workbook.
5. Feed the bytes to `excel_blob_to_dataframes` and compare the output columns with
   `extract_source_columns_from_mapping(...)`.

## Editing notebooks

1. **Back up first**, from the committed version:
   `git cat-file --filters HEAD:<path> > backup/<name>_<YYYYMMDD>_<reason>.ipynb`, then confirm
   `git hash-object` of the backup equals `git rev-parse HEAD:<path>`.
2. **Edit as a text substitution on the raw JSON**, starting from the committed blob. Don't use
   NotebookEdit — it re-serialises the whole file into a ~900-line diff.
3. Apply the same change to the `_PRD` and `_UAT` notebooks.
4. Check the result parses as JSON and the edited cell compiles; `git diff --stat` should
   show only the intended lines, identical in both files.
5. Record what changed in `ATI_ingestion_handoff.md`.

## Outputs

- Edited notebooks for the user to apply in Synapse.
- `ATI_ingestion_handoff.md` kept current with the incident state.

## Human check

The user applies changes in Synapse; nothing here deploys itself. Before editing, confirm the
export in this folder still matches the live notebook — the repo copy can lag.
