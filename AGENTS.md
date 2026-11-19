# Agent instructions

This repo acquires Pennsylvania Department of Education (PDE) public files and keeps the refined schema. The path is catalog → raw bytes → parsers → parquet → DuckDB.

The website stays in [`bobruddy/pa_education_data`](https://github.com/bobruddy/pa_education_data). This repo does not build the LEA browser, does not run `pa_edu.publish`, and does not deploy container `pde-data` or ports 8090 and 8091. Do not edit that repository.

Cursor rules under [`.cursor/rules/`](.cursor/rules/) are the short, enforceable version.

## Goals

1. **Cite the source.** Every refined fact row has a `document_id` that resolves to `document.source_url` and to the retained bytes it came from.
2. **Rebuild from the catalog.** `sources/catalog.toml` + `pip install -e .` + `python -m pa_edu.refine` rebuilds `data/refined/*.parquet`. The repo does not ship data.
3. **Keep every downloaded hash.** Bytes land under `data/raw/<source_id>/<sha256>/`. A later PDE edit does not erase what we already fetched.
4. **Parse `stable` only.** Catalog status `stable` is parsed unattended. `review` is downloaded and left alone until a human confirms the layout.
5. **Keep PDE’s published grain.** Budget and actuals stay apart. Do not add them. Do not add a parent account to its children.

## Layout

```text
sources/catalog.toml     known documents (URL, parser, stable|review, school_year)
sources/checks.toml      AUNs / cells that must round-trip from workbook → parquet
pa_edu/extract.py        workbook bytes → one table per sheet, beside data/raw
pa_edu/parse_*.py        one parser family per source shape
pa_edu/refine.py         download → extract stable workbooks → refined parquet
pa_edu/personnel.py      professional assignment and staff tables
pa_edu/check.py          offline identity, grain, and cell checks against the workbooks
pa_edu/serve.py          refined Parquet over HTTP, and /status for the data service
sql/model.sql            DuckDB views over local parquet
sql/portal.sql           convenience views (school district, fund 10, labeled lines)
```

`school_year` is the **four-digit start year** (`2024` = 2024–25). Take it from the catalog entry, not from a workbook header string.

`pa.gov` returns 403 without the project User-Agent in `pa_edu.common.HEADERS`.

Identity stays text. AUN is a 9-digit string. School numbers stay text. Empty is not zero: a reported zero is a row, and a missing or suppressed cell is absence, not `0`.

## Commands

Always use `.venv/bin/python`.

| Command | Role |
| --- | --- |
| `python -m pa_edu.refine` | Fetch missing raw files, extract sheets, build parquet and the local DuckDB file |
| `python -m pa_edu.personnel` | Professional assignment and staff tables |
| `python -m pa_edu.check` | Offline checks; cell round-trip still opens the workbook |
| `python -m pa_edu.discover` | Print uncataloged links on watched PDE pages |
| `pytest tests/` | Tiny fixture workbooks for the parsers and the data service |

`check`, `refine`, and `personnel` write a run record under `data/status/runs.json`. The data service reads it at `/` and `/status.json`. That page does not serve workbooks.

## Schema and DuckDB

`sql/model.sql` is views over the local parquet. `sql/portal.sql` adds the labeled lines and the other convenience views. `refine` applies `model.sql`, then `portal.sql`. A view keeps published grain. It does not add a parent account to its children, and it does not add budget to actuals.

Local analysis is DuckDB on `data/pde.duckdb`: views over `data/refined`. Do not kill an interactive `duckdb` session that holds that file. Parquet can still be written. `check` reads parquet.

The downloadable DuckDB file is specified in the schema contract. Until that contract lands, the file is forthcoming. Do not document or build the website’s download database in its place.

## Working agreements

- Do only what Robert asked for. "Can we…", "could we…", "would that work…", and "same as we did for…" are questions: answer whether the method fits, what it would change, and what it would leave, then stop. Do not edit code, tests, or config until he asks for the change ("fix it", "do it", "implement it", "go ahead", "make the change"). "Open an issue" means research enough to write it, then file it. A hover, a card, or any other fix mentioned in that request goes in the issue body and is not implemented. "Implement a ticket" means read the ticket, understand it, ask any clarifying questions, and implement only after those are answered. If he objects that you did something he did not ask for, apologize and stop. Do not revert or otherwise undo the work unless he explicitly says to undo it.
- Do not commit or push unless Robert asks.
- When a commit is requested, author and committer are Robert Ruddy <bob@ruddy.net> only. No Co-authored-by trailer. Do not attribute the commit to Cursor or any agent. If a tool appends a trailer, remove it before pushing.
- Do not edit `docs/` unless he asks. This `AGENTS.md` and `.cursor/rules/` are the standing exception when the request is about agent guidance.
- Prefer proving a number from the workbook or the parquet.
