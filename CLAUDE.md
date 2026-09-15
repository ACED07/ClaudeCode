# ClaudeCode / EDMS — working conventions

**Last updated:** 2026-09-15 16:04 +0800

Workspace for investigating and fixing EDMS bronze-to-silver ingestion on Azure Synapse
(storage `saedmsprdizadls01` / `saedmsuatizadls01`). It holds **exported copies** of
notebooks, pipeline JSON and ETL config, plus sample client files. The live versions are in
the Synapse workspace and `config.ETL_Lakehouse_Config` — this repo can lag them, so ask
whether an export is current before relying on it.

One shelf per data source under `EDMSDataset/<SOURCE>/`, each with its own `CONTEXT.md`.
Today there is one: **[EDMSDataset/ATI/](EDMSDataset/ATI/CONTEXT.md)**.

`EDMSDataset/<SOURCE>/*_handoff.md` is in-flight incident state; `backup/` folders are
pre-edit copies. Neither is a routing target.

## Where to start for any given ask

Don't read a shelf cover-to-cover — this table plus your task decide what to open.

| You're asked to... | Read (in this order) | Stop when |
|---|---|---|
| investigate or fix an ATI ingestion failure | [EDMSDataset/ATI/CONTEXT.md](EDMSDataset/ATI/CONTEXT.md), then [ATI_ingestion_handoff.md](EDMSDataset/ATI/ATI_ingestion_handoff.md) for current state | you know which pipeline activity failed and which file it read |
| edit a notebook | [EDMSDataset/ATI/CONTEXT.md](EDMSDataset/ATI/CONTEXT.md) → *Editing notebooks* | a dated backup exists and the PRD and UAT diffs are identical |
| check a config value (`dynamic_data_range`, `silver_mandatory`, `date_format`, ...) | [EDMSDataset/ATI/CONTEXT.md](EDMSDataset/ATI/CONTEXT.md) → *Config fields*, then the CSV row | you've matched rows by `datalake_flow_name`, not by line diff |
| verify a change without touching Azure | [EDMSDataset/ATI/CONTEXT.md](EDMSDataset/ATI/CONTEXT.md) → *Verifying locally* | the notebook's own code ran against the sample xlsx |
| add a new data source | copy the ATI layout: a folder, a `CONTEXT.md`, PRD/UAT-suffixed exports | |

## Hard rules

- **Never edit notebook code directly — back up first.** Copy the committed version to
  `<source>/backup/<name>_<YYYYMMDD>_<reason>.ipynb` before any edit. Procedure in
  [EDMSDataset/ATI/CONTEXT.md](EDMSDataset/ATI/CONTEXT.md).
- **No git remote actions unless asked.** No push, no PRs, nothing on GitHub without an
  explicit instruction in the current conversation. Commit only when asked.
- **No live Azure calls.** Verify by running the notebooks' own Python locally against the
  sample files and the CSV config.
- **PRD and UAT move together.** An edit to one `_PRD` file gets the same edit in its `_UAT`
  twin; only environment values (storage account, key vault) may differ.
- **Secrets stay in Key Vault.** Notebooks read keys through `mssparkutils.credentials.getSecret`;
  keep it that way. Global rule in `~/.claude/CLAUDE.md`.

## Landmines — pointers only, never restated here

- Notebook A's rename can delete the client's original file: [EDMSDataset/ATI/CONTEXT.md](EDMSDataset/ATI/CONTEXT.md) → landmine 1
- Reading ADF `ExcelInvalidColumnName` errors correctly: landmine 5, same file
- `OpsType_\nCodeshare` contains a literal newline: landmine 6, same file
- NotebookEdit re-serialises a whole `.ipynb`: *Editing notebooks*, same file

## Local environment

- Windows, Git Bash + PowerShell. `python` is 3.13; `pandas` and `openpyxl` are **not**
  installed — `pip install pandas openpyxl` before a local simulation.
- `core.autocrlf=true`: blobs are LF, the working copy is CRLF. Judge edits by
  `git diff --stat`, not by file size.

## Keeping these context files current

Hub files carry a `**Last updated:** YYYY-MM-DD HH:MM +ZZZZ` line under their title:
`CLAUDE.md` (this file) and each `EDMSDataset/<SOURCE>/CONTEXT.md`. When you change a hub
file's substance, update its line in the same edit using `date "+%Y-%m-%d %H:%M %z"`, not a guess.

**Shelf files are replaced, not appended to.** A dated "resolved on ..." paragraph is
run-state — it belongs in the source's `*_handoff.md`, not in `CLAUDE.md` or a `CONTEXT.md`.
