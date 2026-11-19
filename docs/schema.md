# Schema

Schema version `1`. This page is the contract. A later change builds the DuckDB files to match it. The parquet files are the stored form. `refine.COLUMNS` is the column list. This page does not add a column or a grain.

`school_year` on a fact is the four-digit start year (`2024` is 2024–25). Unless a table says otherwise, it is the catalog `school_year` of the file that produced the row, not a year parsed out of a header.

A missing cell stays null. A reported zero is a row. `aun` and `school_number` are text. Amounts are numbers. A suppressed percent is null, not zero.

## Version

`dataset.schema_version` is a string. This contract is `1`.

- Adding a column or a view bumps the version.
- Changing a grain bumps the version and is a separate change. It is not folded into a column or a view being added.

## Do not add these together

- General Fund Budget (`gfb_expenditure`, `gfb_revenue`) and Annual Financial Report (`afr_expenditure`, `afr_revenue`) are different files. A budget amount and an actual are different measures.
- A parent function and the functions under it. AFR rollup codes are attributes on the detail row. They are not a second amount.
- `enrollment_gender` or `enrollment_race` and `enrollment`. Gender and race are the same students counted again. `enrollment_race` has no school number.
- `afr_expenditure_school` and `afr_expenditure`. The school file is the building split of the LEA actual for the years a building file was published. It is not more dollars.
- `professional_assignment.annual_salary`. That figure is the person's total at the LEA, repeated on every assignment. Sum `professional_staff.annual_salary`.

## Types that survive a round trip

| Column | Type |
| --- | --- |
| `aun` | text, nine digits |
| `school_number` | text. `9999`, `0`, and `LEAW` stay text |
| amounts (`amount`, `total_amount`, `elementary_amount`, `secondary_amount`, `federal_amount`, `annual_salary`, and the other published counts and rates) | numbers |
| suppressed or unpublished percents (`pct_advanced`, `pct_proficient`, `pct_basic`, `pct_below_basic`, `pct_proficient_and_above`, `pct_low_income`, `pct_iep`) | null |

## Lookups with no `aun`

`document`, `act1_base`, `grade`, `account`, `revenue_account`, and `chart_description` have no `aun`. `nces_district` is keyed by `nces_leaid` and carries `aun` on the row.

## Tables

`check.GRAIN` is the uniqueness check. Where it exists, it uses the same columns as `refine.SORT`. `check.py` does not uniqueness-check the lookups or the two personnel tables.

### `document`

Grain `document_id`. One retained file. `check.py` does not enforce the key.

There is nothing to sum. Filter on `document_id`, `catalog_id`, `source_id`, or `school_year`. `source_url` is the citation. `sha256` and `retrieved_at` identify the retained bytes. `school_year` is the catalog field, and it is null on a multi-year file. The row is the citation target. It does not cite another `document_id`.

### `lea`

Grain `aun`. `check.py` does not enforce the key.

Filter on `aun`, `category`, `status`, `county`, `in_directory`. There is nothing to sum. `lea_name` is the EdNA name, or the name taken from the frame that added an AUN missing from EdNA.

No `school_year`. `document_id` is the EdNA extract for a directory row. On a row added for a missing AUN, it is the `document_id` of the frame that supplied the name.

The EdNA extract is the directory. An AUN missing from it is added only from `lea_year`, `enrollment`, `assessment_result`, `enrollment_iep`, `enrollment_low_income`, or `cohort_graduation`. The newest `school_year` wins, so a later enrollment year beats an older finance row. Within one year the order is finance, then enrollment, then assessment, then IEP, then low income, then cohort graduation. The added row has `in_directory` false and `status` unknown. An AUN that appears only on another table, such as `charter_adm` or `real_estate_tax`, is not added.

### `lea_year`

Grain `school_year` × `aun` × `document_id`. `check.py` does not enforce the key.

Filter on those three. There is nothing to sum. `lea_name`, `county`, and `inst_cat` are the name, county, and category printed on that finance file.

`school_year` is the catalog year of that file. `document_id` is that file.

### `school`

Grain `aun` × `school_number`. `check.py` does not enforce the key.

Filter on `aun` and `school_number`. There is nothing to sum. `school_name` is the name from the newest enrollment year for that pair.

No `school_year` column and no `document_id`.

### `grade`

Grain `grade_code`. No `aun`. `check.py` does not enforce the key.

Filter on `grade_code` or `band`. There is nothing to sum. `label`, `band`, and `session` are the code list.

No `school_year` and no `document_id`.

### `account`

Grain `function_code` × `object_code`. No `aun`. `check.py` does not enforce the key.

Filter on `function_code`, `object_code`, `major_function`, or `major_object`. There is nothing to sum. `function_name` and `object_name` are chart labels. `major_function` is the chart name of the function code with its last three digits set to `000` (`1100` belongs to `1000`). `major_object` is the chart name of the object code with its last two digits set to `00` (`120` belongs to `100`). A code that already ends that way keeps its own name. The columns hold names, not the major codes.

No `school_year` and no `document_id`. The names come from the chart of accounts.

### `revenue_account`

Grain `account_code`. No `aun`. `check.py` does not enforce the key.

Filter on `account_code` or a rollup code. There is nothing to sum. `name` is the chart label. `rollup1_account_code`, `rollup2_account_code`, and `rollup3_account_code` are the parent codes on the chart. They are not amounts.

No `school_year` and no `document_id`.

### `chart_description`

Grain `kind` × `code`. `kind` is `function`, `object`, or `revenue`. No `aun`. `check.py` does not enforce the key.

Filter on `kind` and `code`. There is nothing to sum. `name` is the heading. `description` is the paragraph under it.

No `school_year` and no `document_id`.

### `nces_district`

Grain `nces_leaid`. `aun` is on the row. `check.py` does not enforce the key.

Filter on `nces_leaid` or `aun`. There is nothing to sum. `ccd_name` is the CCD name. This is the crosswalk that puts `aun` on `district_population`.

No `school_year`. `document_id` is the CCD file.

### `gfb_expenditure`

Grain `aun` × `school_year` × `function_code` × `object_code` × `program_code`. `check.GRAIN` enforces that key.

Sum `amount`. That is a budget amount. `fund_code` is `10` on every row. Filter on the grain, or on `fund_code`. A blank program is an empty string, and it is part of the key.

`school_year` is the catalog year of the General Fund Budget file. `document_id` is that file.

`lea_year` carries the name printed on the same file. This table does not.

### `gfb_revenue`

Grain `aun` × `school_year` × `account_code`. `check.GRAIN` enforces that key.

Sum `amount`. That is a budget amount. `fund_code` is `10`. Filter on the grain. Fund-balance columns (`0810` through `0850`) are not in this table.

`school_year` is the catalog year of the General Fund Budget file. `document_id` is that file.

### `afr_expenditure`

Grain `aun` × `school_year` × `fund_code` × `function_code` × `object_code` × `program_code`. `check.GRAIN` enforces that key.

Sum one amount column: `elementary_amount`, `secondary_amount`, `federal_amount`, or `total_amount`. They are separate published fields. `total_amount` is the row total. Filter on the grain.

Attributes, not amounts: `rollup_fund_code`, `rollup1_account_code`, `rollup2_account_code`, `rollup3_account_code`, `rollup1_object_code`, `rollup2_object_code`.

`school_year` is the catalog year of the Annual Financial Report file. `document_id` is that file.

### `afr_expenditure_school`

Grain `aun` × `school_number` × `school_year` × `fund_code` × `function_code` × `object_code` × `program_code`. `check.GRAIN` enforces that key.

Sum the same amount columns as `afr_expenditure`. `school_name` is the name on the building file. The rollup codes are attributes, as on the LEA table.

`school_year` and `document_id` are the Annual Financial Report file, the same file as the LEA rows for that year.

### `afr_revenue`

Grain `aun` × `school_year` × `fund_code` × `account_code` × `program_code`. `check.GRAIN` enforces that key.

Sum `amount`. That column is `total_amount` when the workbook filled it, and `amount_1` when `total_amount` is null. `amount_1`, `amount_2`, `amount_3`, and `total_amount` are the published fields. Do not add `amount` to `total_amount`.

Attributes: `rollup_fund_code`, `rollup1_account_code`, `rollup2_account_code`, `rollup3_account_code`.

`school_year` is the catalog year of the Annual Financial Report file. `document_id` is that file.

### `enrollment`

Grain `aun` × `school_number` × `school_year` × `grade_code`. `check.GRAIN` enforces that key.

Sum `enrollment`. That is the headcount in one grade cell. Filter on the grain. `school_name`, `lea_name`, `lea_type`, and `county` are the values printed on that file.

`school_year` is the catalog year of the enrollment file. `document_id` is that file.

### `enrollment_gender`

Grain `aun` × `school_number` × `school_year` × `gender` × `grade_code`. `check.GRAIN` enforces that key.

Sum `enrollment` only within one gender, or across genders when the question is the same headcount split by Female and Male. Adding the result to `enrollment` counts the students twice. `school_name`, `lea_name`, `lea_type`, and `county` are the values printed on that file.

`school_year` is the catalog year of the enrollment file. `document_id` is that file.

### `enrollment_race`

Grain `aun` × `school_year` × `race` × `grade_code`. No school number. `check.GRAIN` enforces that key.

Sum `enrollment` only as the race split of the LEA headcount. Adding the result to `enrollment` counts the students twice. `lea_name`, `lea_type`, and `county` are the values printed on that file.

`school_year` is the catalog year of the enrollment file. `document_id` is that file.

### `enrollment_low_income`

Grain `aun` × `school_year`. `check.GRAIN` enforces that key.

`low_income_enrollment` and `total_enrollment` are the published counts. `pct_low_income` is the published percentage. A blank percentage is null. Do not add `low_income_enrollment` to `enrollment`. `lea_name` and `lea_type` are the values printed on that file.

`school_year` is the catalog year of that file. `document_id` is that file.

### `enrollment_iep`

Grain `aun` × `school_year`. `check.GRAIN` enforces that key.

`iep_enrollment` and `total_enrollment` are the published counts. `pct_iep` is the published percentage. A blank percentage is null. Do not add `iep_enrollment` to `enrollment`. `lea_name` is the name printed on that file.

`school_year` is the catalog year of that file. `document_id` is that file.

### `assessment_result`

Grain `aun` × `school_year` × `assessment` × `subject` × `grade` × `student_group`. `check.GRAIN` enforces that key.

`number_scored` is the count. The `pct_` columns are the published percents for that student group. A suppressed percent is null. Each student group is its own row. Do not add groups together and call the result All Students. `lea_name` and `county` are the values printed on the district file.

`school_year` is the catalog year of the district assessment file. The workbook's displayed year is that start year plus one. `document_id` is that file.

### `assessment_result_school`

Grain `aun` × `school_number` × `school_year` × `assessment` × `subject` × `grade` × `student_group`. `check.GRAIN` enforces that key.

Same measures as `assessment_result`, plus `growth` when the school file published it. `school_name`, `lea_name`, and `county` are the values printed on the school file. The stored `pct_proficient_and_above` is unchanged when the workbook published it. The view, not the parquet, fills a null combined percent. See Views.

`school_year` is the catalog year of the school assessment file. `document_id` is that file.

### `act1_index`

Grain `aun` × `school_year`. `check.GRAIN` enforces that key.

`adjusted_index` is the rate as a decimal (`0.053` is 5.3%). An empty rate is not a row. `lea_name` and `county` are the values printed on that row.

`school_year` is the start year in the column header (`2024-25` is `2024`), not a single catalog year. One workbook holds many years, and every row from it cites that workbook's `document_id`.

### `act1_base`

Grain `school_year`. No `aun`. `check.GRAIN` enforces that key.

`base_index`, `saww_increase`, and `eci_increase` are three published rates. They are not added to `adjusted_index`.

`school_year` is the start year on the history page. Every row cites that PDF's `document_id`.

### `real_estate_tax`

Grain `aun` × `school_year` × `county` × `locality`. `check.GRAIN` enforces that key.

`mills` and `college_mills` are separate published rates. A row with no rate is not stored.

`school_year` is the catalog year of the millage file. `document_id` is that file.

### `charter_adm`

Grain `aun` × `school_year`. `check.GRAIN` enforces that key.

Sum `adm` when the question is the published charter average daily membership for that district. It is not a count of students to add to `enrollment`. `lea_name` and `county` are the values printed on that file.

`school_year` is the catalog year. `document_id` is that file.

### `home_education`

Grain `aun` × `school_year`. `check.GRAIN` enforces that key.

`students`, `private_tutor`, `age_5_11`, and `age_12_plus` are the published affidavit counts on that row. They are not added to `enrollment`. `lea_name` and `county` are the values printed on that file.

`school_year` is the catalog year. `document_id` is that file.

### `district_population`

Grain `aun` × `school_year`. `check.GRAIN` enforces that key.

`total_population`, `population_5_17`, and `children_5_17_in_poverty` are separate Census estimates. `nces_leaid` and `census_name` are attributes. `aun` comes from `nces_district`.

`school_year` is the catalog year of the SAIPE file. `document_id` is the SAIPE file, not the CCD file.

### `cohort_graduation`

Grain `aun` × `school_year` × `span`. `check.GRAIN` enforces that key.

`graduates` and `cohort` are the published counts. `rate` is the published rate. A blank rate is null. `lea_name` and `lea_type` are the values printed on that file.

`school_year` is the catalog year. `span` is the cohort length on that file. `document_id` is that file.

### `cohort_graduation_group`

Grain `aun` × `school_year` × `span` × `student_group`. `check.GRAIN` enforces that key.

`rate` is the published rate for that group. Do not add it to `cohort_graduation.rate`.

`school_year` is the catalog year. `document_id` is that file.

### `professional_assignment`

Grain `school_year` × `aun` × `public_id` × `assignment_code` × `school_number`. `check.py` does not enforce the key. `python -m pa_edu.personnel` writes the file. `refine` does not replace an existing one.

Do not sum `annual_salary`. It is that person's total at the LEA, repeated on every assignment. `fte_percent` is the assignment's FTE. Filter on the grain, `job_class`, or `primary_assignment`.

The name, degree, years, status, LEA, school, assignment, and position columns are attributes of the assignment. `source_layout` is the column map for that year. `source_file` is the workbook name under `data/raw/personnel/`.

`school_year` is the start year in the workbook filename. The sheet label must be that same year. There is no `document_id`.

### `professional_staff`

Grain `school_year` × `aun` × `public_id`. `check.py` does not enforce the key. Written by `personnel`, and left in place by `refine`.

Sum `annual_salary`. One row is one person at one LEA in one year. `row_rule` is `primary_assignment` when the file marked a primary assignment, otherwise `highest_fte`. `assignment_count` is how many assignment rows were folded into this row. The other columns are the chosen assignment's attributes, including the repeated `annual_salary`.

`school_year` is the same start year as `professional_assignment`. There is no `document_id`. `source_file` names the workbook.

## Views

A view ships because it names a published grain or attaches a label. A view that adds a parent account to its children, or budget to actuals, does not ship. No new view is added in this version except `dataset`, which is build metadata and is created by the change that builds the DuckDB files.

### One view per parquet table

`sql/model.sql` defines one view per parquet file. Each view is `SELECT *` from that file, except `assessment_result_school`.

`assessment_result_school` keeps a published `pct_proficient_and_above`. When that cell is null and both `pct_advanced` and `pct_proficient` are present, the view uses their sum. The sum is not rounded. A row with only one of those two percents stays null. The parquet cell is unchanged.

### `afr_general_fund`

Fund `10` of `afr_expenditure`. `sql/model.sql` owns this view. `sql/portal.sql` repeats the same statement so `afr_general_fund_line` can name it. The two statements stay identical. `refine` applies `model.sql` and then `portal.sql`, so a change to the copy in `portal.sql` would replace the owned view.

### Labels and other published grains

`sql/portal.sql` also defines:

| View | What it is |
| --- | --- |
| `school_district` | `lea` where `category` is `school_district` |
| `enrollment_headcount` | Sum of `enrollment` grade cells for one `aun`, `school_number`, and `school_year`. The same students as `enrollment`, not a new count |
| `gfb_line` | Each budget amount with the directory name and the chart labels. The amount is still `gfb_expenditure.amount` |
| `afr_general_fund_line` | Each general-fund actual with the directory name and the chart labels. The amounts are still the AFR amount columns |
| `assessment_all_students` | `assessment_result` where `student_group` is `All Students` |
| `assessment_lea` | One assessment row per LEA. A school district keeps the district workbook. Any other LEA is copied from its single school row that year |

### `dataset`

One row. Build metadata, not a fact table. No `aun` and no `document_id`.

| Column | Value |
| --- | --- |
| `schema_version` | This contract. `1` until a column, a view, or a grain changes |
| `git_sha` | The commit that built the refined tables |
| `built_at` | The UTC time that build finished writing the parquet |

The view is not in the SQL yet. The change that builds the DuckDB files adds it.

## Two DuckDB files

Both files use this schema. Neither file is a copy of the parquet.

### Local

`data/pde.duckdb`. `python -m pa_edu.refine` writes it. `python -m pa_edu.personnel` refreshes it after replacing the staff parquet. The views read the local parquet directory. This is the file for a machine that has already run `refine`.

Open it from the repository root:

```bash
duckdb data/pde.duckdb
```

A row cites the file it came from:

```sql
SELECT g.aun, g.school_year, g.amount, d.source_url
FROM gfb_expenditure AS g
JOIN document AS d USING (document_id)
LIMIT 20;
```

### Downloadable

One small file, named `pde.duckdb`. Its views read the data service's parquet with `httpfs`:

```text
read_parquet('<service>/refined/<table>.parquet')
```

`<service>` is the data service that file was built against. The service answers range requests, so DuckDB fetches the row groups a query needs. The file does not copy table bodies into the database. A file that materializes every table for offline use is not part of version 1.

`dataset` is in this file too, with the schema version, git SHA, and build time of the parquet the views read.

## Worked queries

Each total below is a published measure. Each one selects `source_url`.

General-fund actuals for one district and year. The sum is `total_amount` on fund `10`.

```sql
SELECT
    a.aun,
    a.school_year,
    sum(a.total_amount) AS general_fund_actual,
    d.source_url
FROM afr_general_fund AS a
JOIN document AS d USING (document_id)
WHERE a.aun = '101260303'
  AND a.school_year = 2024
GROUP BY a.aun, a.school_year, d.source_url;
```

The same year's general-fund budget, at function × major object code. `budget_amount` is not the actual above. A function ending in `000` is a parent and is left out. The major object code is the object code with its last two digits set to `00`. `account.major_object` is the chart name of that code, not the code itself.

```sql
SELECT
    g.aun,
    g.school_year,
    g.function_code,
    CASE
        WHEN right(lpad(g.object_code, 3, '0'), 2) = '00' THEN lpad(g.object_code, 3, '0')
        ELSE left(lpad(g.object_code, 3, '0'), 1) || '00'
    END AS major_object_code,
    sum(g.amount) AS budget_amount,
    d.source_url
FROM gfb_expenditure AS g
JOIN document AS d USING (document_id)
WHERE g.aun = '101260303'
  AND g.school_year = 2024
  AND right(g.function_code, 3) <> '000'
GROUP BY g.aun, g.school_year, g.function_code, major_object_code, d.source_url;
```

Enrollment headcount for one school. This is the sum of the grade cells, the same total as `enrollment_headcount`. Gender is those students again, in `enrollment_gender`, and is not added to this total.

```sql
SELECT
    e.aun,
    e.school_number,
    e.school_year,
    sum(e.enrollment) AS headcount,
    d.source_url
FROM enrollment AS e
JOIN document AS d USING (document_id)
WHERE e.aun = '124150002'
  AND e.school_number = '7691'
  AND e.school_year = 2024
GROUP BY e.aun, e.school_number, e.school_year, d.source_url;
```

Salary for one district and year, from `professional_staff`. Summing `professional_assignment.annual_salary` repeats each person once per assignment. Staff rows have no `document_id`. The personnel file for that year is the `document` row with `catalog_id` `personnel-` plus the year. When more than one hash was kept, use the latest `retrieved_at` so the salary is not repeated once per hash.

```sql
SELECT
    s.aun,
    s.school_year,
    sum(s.annual_salary) AS salary,
    d.source_url
FROM professional_staff AS s
JOIN (
    SELECT school_year, source_url
    FROM document
    WHERE catalog_id = 'personnel-' || CAST(school_year AS VARCHAR)
    QUALIFY row_number() OVER (PARTITION BY school_year ORDER BY retrieved_at DESC) = 1
) AS d
    ON d.school_year = s.school_year
WHERE s.aun = '101260303'
  AND s.school_year = 2024
GROUP BY s.aun, s.school_year, d.source_url;
```
