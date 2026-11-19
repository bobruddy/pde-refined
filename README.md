# pde-refined

Citeable Pennsylvania Department of Education tables. A clone is the catalog, the parsers, and the DuckDB views. The workbooks and the parquet are not in git. They are rebuilt into `data/`, which git ignores.

## Install

Requires Python 3.11 or newer.

```bash
python -m venv .venv
.venv/bin/python -m pip install -e '.[dev]'
```

## Rebuild

Use `.venv/bin/python` for each command.

| Command | Role |
| --- | --- |
| `python -m pa_edu.refine` | Download catalog files that are not on disk, extract workbooks, write parquet, and build the local DuckDB file |
| `python -m pa_edu.personnel` | Professional assignment and staff tables |
| `python -m pa_edu.check` | Identity, grain, and cell checks. Cell checks open the workbook |
| `python -m pa_edu.discover` | Print workbook links on the watched PDE pages that are not in the catalog |

Bytes land here:

| Path | Contents |
| --- | --- |
| `data/raw/<source_id>/<sha256>/` | Retained download for that hash |
| `data/extract/<document_id>/` | One parquet file per sheet |
| `data/refined/*.parquet` | One file per table |
| `data/pde.duckdb` | Views over `data/refined` |

The first `refine` needs a network connection. A later run with the same files and the same code keeps the cache. See [docs/acquisition.md](docs/acquisition.md).

## Query

From the repository root, after `refine`:

```bash
duckdb data/pde.duckdb
```

```sql
SELECT g.aun, g.school_year, g.function_code, g.object_code, g.amount, d.source_url
FROM gfb_expenditure AS g
JOIN document AS d USING (document_id)
LIMIT 20;
```

`document.source_url` is the file that row was parsed from. Grain, the views, and what a query must not add together are in [docs/schema.md](docs/schema.md). The downloadable DuckDB file is described there as well.

## Source of this copy

This tree was copied from [pa_education_data](https://github.com/bobruddy/pa_education_data) at `ac3a7222dc1801e53e3e0d3f29b8c35852e9e84d`.
