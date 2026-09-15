# ATI bronze-to-silver ingestion failure — handoff

**Incident:** 2026-09-03, file `ATI_20260707.xlsx`. Two of four sheets failed silver ingestion.
**Platform:** Azure Synapse. Storage account `saedmsprdizadls01`, containers `caas-edms-bronze` / `caas-edms-silver`.
**Pipeline:** `00_pl_edms_bronze_to_silver_AT`
**Config table:** `config.ETL_Lakehouse_Config`, `datalake_group = 'edms_at_ati'`

---

## What happened

| Pipeline | Time (SGT) | Result |
|---|---|---|
| AT_ATI_Airline_List_Ingestion | 19:40:38 | Fail — `ExcelInvalidColumnName` on `CODE` |
| AT_ATI_City_Links_Ingestion | 19:41:32 | Fail — `ExcelInvalidColumnName` on `OpsType_Codeshare` |
| AT_ATI_Pax_Breakdown_Ingestion | 20:00:23 | Success |
| AT_ATI_Freighter_Breakdown_Ingestion | 20:09:17 | Success |

Also seen: delete activity `delete-from-silver_landing` failed with `PathNotFound` on
`parquet-landing/at_ati_airline_list/partition=20260707` at 11:38:31 GMT (= 19:38 SGT).
This is a **downstream symptom** — the partition was never written because the copy failed.

Split notebook log showed:
```
Skipping None Airline List.xlsx — does not match source: ATI PAX_BREAKDOWN
Skipping None City Links.xlsx — does not match source: ATI PAX_BREAKDOWN
Skipping None FREIGHTER_BREAKDOWN.xlsx — does not match source: ATI PAX_BREAKDOWN
Skipping None PAX_BREAKDOWN.xlsx — does not match source: ATI PAX_BREAKDOWN
```

---

## Pipeline structure

Execution order (all `dependsOn` use **`Completed`**, not `Succeeded`):

```
ForEach1 (inactive)
  -> get-config worksheet        (Lookup: worksheet, original_source_name,
                                  date_format, date_normalization_enabled)
  -> ForEach Worksheet
       -> 00_split_excel_worksheets           [NOTEBOOK A]
  -> get-config                  (Lookup: silver_schema_mapping, silver_mandatory,
                                  dynamic_data_range)
  -> ForEach2
       -> IfCondition: dynamic_data_range > 0
            true  -> 00_remove_invalid_headers_xlsx_dynamic   [NOTEBOOK B]
            false -> 00_remove_invalid_headers_xlsx
  -> 00_pl_edms_bronze_to_silver (ExecutePipeline — the ADF Copy activities)
```

Notebook A params: `p_bronze_container_src` and `p_bronze_container_dst` both resolve to
`@concat(item().bronze_container,'/',item().bronze_landing_path)` — **the same path**.

---

## Root causes (confirmed by analysis of both xlsx files)

### 1. `update_filename()` returns `None` implicitly — NOTEBOOK A

```python
def update_filename(file_name, fmt):
    for i in range(len(tokens)):          # reads GLOBAL tokens; file_name arg unused
        for j in range(i+1, min(i+4, len(tokens))+1):
            ...
            try:
                parsed = datetime.strptime(candidate, fmt_clean)
                ...
                return new_filename
            except ValueError:
                continue
    # <-- no return here. Python returns None.
```

`fmt` for this source is `%d %b %y` (**two-digit year**). Verified behaviour:

| Input basename | Result |
|---|---|
| `ATI 04 Apr 26` | `ATI_20260404.xlsx` ✅ |
| `ATI 07 Jul 26` | `ATI_20260707.xlsx` ✅ |
| `ATI 07 Jul 2026` | `None` ❌ |
| `ATI_20260707` | `None` ❌ |
| `ATI Airline List_20260707` | `None` ❌ |
| `ATI PAX_BREAKDOWN_20260707` | `None` ❌ |

### 2. No guard before the destructive copy+delete — NOTEBOOK A

```python
new_filename = update_filename(file_name, fmt)   # -> None
if path:
    new_filename = f"{path}/{new_filename}"      # -> ".../None"   (f-string renders None)
...
new_file.start_copy_from_url(copy_source)
old_file.delete_blob()                           # ORIGINAL DESTROYED
```

### 3. `main_identifier` derived from the renamed file — NOTEBOOK A

```python
original_file_name = os.path.splitext(new_filename.split('/')[-1])[0]  # "None"
main_identifier = original_file_name.split('_')[0]                     # "None"
output_filename = f"{main_identifier} {worksheet_name}{remaining_name}.xlsx"
```

`tokens[0]` already holds `"ATI"` and does not depend on the rename succeeding.

### 4. Header row assumed to be row 0 — NOTEBOOK A  ← **causes the `CODE` error**

```python
df = pd.DataFrame(data)
df.columns = df.iloc[0]
df = df[1:]
```

Airline List sheet: row 1 = `CARRIERS OPERATING SCHEDULED SERVICES INTO SINGAPORE`,
row 2 = blank, **row 3 = the real header** (`CODE`, `CARRIER`, ...).
City Links sheet: row 3 = `Table 1: Citylinks`, **row 4 = the real header**.

So the split output's header becomes the report title and `CODE` is absent → ADF throws.

### 5. Self-reprocessing loop — NOTEBOOK A

Final loop writes split outputs back into `p_bronze_container_src_path`. The rename block at
the top of the blob loop runs against **every** blob in that path before any filtering, so
leftover split outputs (`ATI Airline List_20260707.xlsx`) get re-parsed next run, fail
`%d %b %y`, and are destroyed as `None`. This fires independently of what the client uploads.

### 6. `OpsType_Codeshare` genuinely removed from source — CLIENT SIDE

Verified against the raw files:

| | Columns | Col 12 header | Col 13 header |
|---|---|---|---|
| `ATI_20260505.xlsx` (pass) | 15 | `'OpsType_\nCodeshare'` | `'Region'` |
| `ATI_20260707.xlsx` (fail) | 14 | `'Region'` | — |

**The copy activity mapping expects source name `"OpsType_\nCodeshare"` — WITH a literal
newline.** The error message reports the *sink* name (`OpsType_Codeshare`, no newline),
which is why this initially looked like a header-formatting problem. It is not.

Cannot remove the mapping entry: older files contain the column and must remain reprocessable.

### 7. Dependencies use `Completed` not `Succeeded`

A notebook failure does not stop the copy activity, so damage surfaces as a cryptic ADF
error several steps downstream instead of failing where it happened.

---

## Ruled out (do not re-investigate)

- **Newlines / whitespace in `CODE`.** Raw sharedStrings XML is `<t>CODE</t>` in both files.
  Cell A3, byte-identical. No NBSP, no trailing space, no zero-width chars.
- **The Airline List source file changed.** Header row, sheet name, sheet index, merged cells,
  and blank-`CODE` row count (69) are identical across both months. Only 2 extra data rows.
- **The silver schema mapping for Airline List.** All 10 source names match row 3 exactly,
  including `FSC/LCC`, `2 LETTER`, `3-LETTER`, `DOP (LATEST)`, `GHA (Check-in)`.
- **The newline in `OpsType_\nCodeshare` being the problem.** The May file has that exact
  newline and ingested fine.
- **NOTEBOOK B (dynamic) header detection.** It matches City Links row 4 at 12/13 = 0.92,
  above its 0.60 threshold, and produces correct output minus the one missing column.

---

## Still unconfirmed

1. **The actual filename the client uploaded in July.** The files on hand (`ATI_20260505.xlsx`,
   `ATI_20260707.xlsx`) are post-rename names. Check `caas-edms-bronze/air transport/ati/archive/`
   or the NOTEBOOK A run log for the `print(current_filename)` / `print(new_filename)` pair
   (Monitor → Apache Spark applications, run ~19:38 SGT 2026-09-03).
   - If input was `ATI 07 Jul 2026.xlsx` → client changed year format, they are the trigger.
   - If input was a leftover split file → cause #5, entirely our bug, client is innocent.
2. **Whether UAT and PRD notebook code have diverged.** Diff the exported notebook JSON.
3. **Whether `ATI_20260707.xlsx` still exists in the landing path** or survives only as a
   `None` blob. The original was deleted by cause #2.
4. **Pax/Freighter succeeded reading correctly-named paths** while the same run produced
   `None` outputs — suggests more than one execution that day. Check trigger history for
   duplicate runs.

---

## Fixes to implement

Priority order. Items 1–2 are the ones that prevent data loss.

1. **Guard before copy+delete** — `if not normalized_date: print(...); continue`.
2. **Add `%Y%m%d` as a fallback format** so already-normalised names and split outputs pass
   through untouched. Neutralises cause #5.
3. **`main_identifier = tokens[0]`** — take the identifier from the original filename.
4. **Replace `df.iloc[0]`** with header detection (port `is_schema_header_row` from
   NOTEBOOK B: 60% match against the mapping's source column names).
5. **Backfill missing mapped columns as empty** rather than raising, so a single ETL config
   stays valid across old and new files:
   ```python
   missing = [c for c in expected_columns if c not in df.columns]
   if missing:
       print(f"WARNING: backfilling missing column(s) {missing}")
       for c in missing:
           df[c] = ""
   if len(missing) > 2:
       raise ValueError("too many missing columns — likely wrong header row, not schema drift")
   ```
   Column names must come from `extract_source_columns_from_mapping` so the `\n` in
   `OpsType_\nCodeshare` is preserved — do not hand-type it.
6. **Refactor `update_filename` to take `tokens`/`ext` as arguments** instead of reading
   module globals. The `file_name` parameter is currently accepted and ignored, which makes
   any standalone test of this function misleading.
7. **Change copy-activity dependencies to `Succeeded`** so notebook failures stop the pipeline.
8. **Make the delete activity tolerate a missing partition** (Get Metadata + If Condition on
   `exists`, or treat 404 as success).

---

## Known data-quality bug (separate, pre-existing)

NOTEBOOK B's `is_mandatory_empty` treats *any* blank mandatory cell as end-of-table. If
`CODE` is configured as the mandatory column for Airline List, ingestion stops at the first
blank — and `CODE` is blank on 69 of ~149 rows in both files.

Observed output: **3 rows** from the May file, **1 row** from the July file, out of ~150 carriers.

This has been silently truncating Airline List every month. It did not error because the
columns were correct — only the data was missing. Check what is actually in the silver table.

`CARRIER` is populated throughout and would be a better mandatory column.

---

## Client communication status

Client was told (incorrectly, early on) that the headers had line breaks needing removal.
They checked and found the headers fine — they were right. An acknowledgement has been sent;
they have not yet re-uploaded.

Note for the eventual call:
- **Airline List** — ours. Nothing for the client to do. A re-upload will not fix it.
- **City Links** — theirs (missing column), but if they restore it, the current mapping needs
  the header to contain a literal newline again. Better to backfill on our side (fix #5) than
  to ask them to reproduce an invisible character.
