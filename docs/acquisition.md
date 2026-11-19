# Acquisition

`sources/catalog.toml` is the list of files the rebuild may fetch. One `[[document]]` is one file.

| Field | Meaning |
| --- | --- |
| `id` | Stable catalog id. Unique. |
| `source_id` | Family, and the first directory under `data/raw/` |
| `source_url` | Bytes to fetch |
| `page_url` | PDE page the file was found on |
| `parser` | Name in `pa_edu.refine.PARSERS` |
| `status` | `stable` or `review` |
| `school_year` | Four-digit start year (`2024` is 2024–25). Omitted on a multi-year file. Taken from this field, not from a header string |
| `fetch` | `http`, or `edna` for the directory form |
| `filename` | Saved name, when the URL does not carry one |

`pa.gov` returns 403 without the User-Agent in `pa_edu.common.HEADERS`. Every fetch uses that header.

## stable and review

`stable` is downloaded if needed, extracted, and parsed into facts.

`review` is downloaded and, when it is a workbook, extracted. It is not parsed. A new or moved header stays `review` until a parser rule and a check exist.

## Raw cache

`python -m pa_edu.refine` walks the catalog. For each id, `materialize` returns the file already stored for that id when the bytes are still on disk. A missing file is fetched and saved at:

```text
data/raw/<source_id>/<sha256>/<filename>
```

The hash is the SHA-256 of the bytes. The same URL with new bytes is a new directory. The previous hash stays. `document_id` is `<catalog id>-<first 12 hex of the sha256>`.

## Sheet extract

A workbook (`.xls`, `.xlsx`, `.xlsm`) is written to `data/extract/<document_id>/`, one parquet file per sheet, plus `meta.json`. `refine` and `personnel` read that extract. When the stored hash and the extract code still match, the extract is kept and Excel is not opened.

`python -m pa_edu.check` still opens the workbook. Its cell checks re-read the listed cells in `sources/checks.toml` from the file, and compare them to the parquet.

## A second run

When the catalog ids, the hashes, the parser code, and the parquet files match the stamp in `data/refined/_inputs.json`, `refine` prints `kept refined tables` and does not parse again. It still rebuilds `data/pde.duckdb` from `sql/model.sql` and then `sql/portal.sql`.

`personnel` does the same with `data/refined/_personnel.json`. A match prints `kept personnel tables`.

## Discover

`python -m pa_edu.discover` prints links ending in `.xls`, `.xlsx`, or `.pdf` on the pages in `pa_edu.discover.PAGES` that are not already a catalog `source_url`.

On a full catalog run, `refine` also downloads workbook links from those pages that are not in the inventory. They are stored as `status = review` under `source_id` `discovered`, extracted, and left out of the fact tables until a catalog entry names a parser.

## LEA names

The EdNA directory extract (`parser = "edna"`, `fetch = "edna"`) is the LEA source. It writes `lea`: one row per AUN, with the directory name, category, status, and county. `in_directory` is true on those rows.

An AUN that shows up on a fact file and is missing from that extract is added to `lea` with `in_directory` false, so the fact still has a directory key. The added name comes from the newest finance row, then enrollment, then assessment.

The name printed on a finance or enrollment file stays on that file's rows. Finance names for a year are `lea_year` (`lea_name`, `county`, `inst_cat`, and the `document_id` of the file). Enrollment, assessment, and the other fact tables keep their own `lea_name` column. Those columns are the name on that file. They are not a copy of the EdNA name.

Professional staff is a separate command, `python -m pa_edu.personnel`. It uses the same raw-cache and extract rules, with `source_id` `personnel`. The staff grain is in [schema.md](schema.md).
