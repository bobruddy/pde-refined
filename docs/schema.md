# Schema

Each name in `refine.COLUMNS` is one parquet file, `data/refined/<name>.parquet`. `refine.SORT` is the row order and the natural key. `check.GRAIN` is the uniqueness check. Where both exist, they use the same columns. `check.py` does not uniqueness-check the lookup tables or the two personnel tables.

A fact row that has `document_id` cites `document.source_url`. A missing cell stays null. A reported zero is a row.

`sql/model.sql` defines one view per parquet file, then `afr_general_fund`. `refine` applies that file, then `sql/portal.sql`. The local file is `data/pde.duckdb`.

The downloadable DuckDB file is decided in the schema-contract issue. This page does not specify it.

## Do not add these together

- General Fund Budget (`gfb_expenditure`, `gfb_revenue`) and Annual Financial Report (`afr_expenditure`, `afr_revenue`) are different files. A budget amount and an actual are different measures.
- A parent function or a parent object and its children. The chart stores both. Adding a parent line to the lines under it counts the same dollars twice.
- `enrollment_gender` or `enrollment_race` and `enrollment`. Gender and race are the same students counted again. `enrollment_race` has no school number.
- `afr_expenditure_school` and `afr_expenditure`. The school file is the building split of the LEA actual for the years PDE published one. It is not more dollars.

## Tables

### Directory and files

| Table | Grain |
| --- | --- |
| `document` | `document_id`. One retained file. `check.py` does not uniqueness-check it |
| `lea` | `aun`. EdNA directory, plus fact AUNs missing from that extract. `check.py` does not uniqueness-check it |
| `lea_year` | `school_year` × `aun` × `document_id`. Name and county printed on that finance file. `check.py` does not uniqueness-check it |
| `nces_district` | `nces_leaid`. CCD id, with `aun` on the row. `check.py` does not uniqueness-check it |
| `school` | `aun` × `school_number`. School name from the newest enrollment year for that pair. No `document_id`. `check.py` does not uniqueness-check it |
| `grade` | `grade_code`. Code, label, and band. No `document_id`. `check.py` does not uniqueness-check it |

### Finance

| Table | Grain |
| --- | --- |
| `gfb_expenditure` | `aun` × `school_year` × `function_code` × `object_code` × `program_code`. One budget amount |
| `gfb_revenue` | `aun` × `school_year` × `account_code`. One budget amount |
| `afr_expenditure` | `aun` × `school_year` × `fund_code` × `function_code` × `object_code` × `program_code`. Amount columns are the published elementary, secondary, federal, and total fields |
| `afr_expenditure_school` | `aun` × `school_number` × `school_year` × `fund_code` × `function_code` × `object_code` × `program_code`. Same amount columns, at the building |
| `afr_revenue` | `aun` × `school_year` × `fund_code` × `account_code` × `program_code`. The amount columns are separate published fields |
| `account` | `function_code` × `object_code`. Chart names, including `major_function` and `major_object`. No `document_id`. `check.py` does not uniqueness-check it |
| `revenue_account` | `account_code`. Name and rollup codes. No `document_id`. `check.py` does not uniqueness-check it |
| `chart_description` | `kind` × `code`. Heading and paragraph from the chart. No `document_id`. `check.py` does not uniqueness-check it |

### Enrollment

| Table | Grain |
| --- | --- |
| `enrollment` | `aun` × `school_number` × `school_year` × `grade_code`. One grade cell. `lea_name` is the name on that file |
| `enrollment_gender` | `aun` × `school_number` × `school_year` × `gender` × `grade_code`. The same students, split by Female and Male |
| `enrollment_race` | `aun` × `school_year` × `race` × `grade_code`. LEA file. No school number |
| `enrollment_low_income` | `aun` × `school_year`. Count and the percentage PDE published |
| `enrollment_iep` | `aun` × `school_year`. Count and the percentage PDE published |

### Assessment, tax, and population

| Table | Grain |
| --- | --- |
| `assessment_result` | `aun` × `school_year` × `assessment` × `subject` × `grade` × `student_group`. District PSSA and Keystone. Each student group is its own row |
| `assessment_result_school` | `aun` × `school_number` × `school_year` × `assessment` × `subject` × `grade` × `student_group`. School file, including district buildings, charter schools, and career and technical centers |
| `act1_index` | `aun` × `school_year`. Adjusted index for a district |
| `act1_base` | `school_year`. Statewide base index. No `aun` |
| `real_estate_tax` | `aun` × `school_year` × `county` × `locality`. Mills on that line |
| `charter_adm` | `aun` × `school_year`. Charter average daily membership for a district |
| `home_education` | `aun` × `school_year`. Affidavit counts for a district |
| `district_population` | `aun` × `school_year`. Census SAIPE estimates. `aun` comes from `nces_district` |
| `cohort_graduation` | `aun` × `school_year` × `span`. Counts and rate |
| `cohort_graduation_group` | `aun` × `school_year` × `span` × `student_group`. Rate for that group |

### Professional staff

Written by `python -m pa_edu.personnel`. `refine` leaves an existing file in place. `check.py` does not uniqueness-check these two tables. There is no `document_id`. `source_file` is the workbook name under `data/raw/personnel/`.

| Table | Grain |
| --- | --- |
| `professional_assignment` | `school_year` × `aun` × `public_id` × `assignment_code` × `school_number`. One assignment. `annual_salary` is that person's total at the LEA and is repeated on every assignment. Do not sum it here |
| `professional_staff` | `school_year` × `aun` × `public_id`. One person at one LEA in one year, so `annual_salary` can be summed. `row_rule` is `primary_assignment` when PDE marked a primary assignment, otherwise `highest_fte`. `assignment_count` is how many assignment rows were folded into this one |

## Views

`sql/model.sql` is `SELECT *` from each parquet file, with two exceptions:

- `assessment_result_school` keeps a published `pct_proficient_and_above`. When that cell is null and both `pct_advanced` and `pct_proficient` are present, the view uses their sum. The sum is not rounded. The parquet cell is unchanged.
- `afr_general_fund` is `afr_expenditure` where `fund_code` is `10`.

`sql/portal.sql` defines the same `afr_general_fund` again, and these views:

| View | What it is |
| --- | --- |
| `school_district` | `lea` where `category` is `school_district` |
| `enrollment_headcount` | Sum of `enrollment` grade cells for one `aun`, `school_number`, and `school_year` |
| `gfb_line` | Each budget amount with the directory name and the chart labels |
| `afr_general_fund_line` | Each general-fund actual with the directory name and the chart labels |
| `assessment_all_students` | `assessment_result` where `student_group` is `All Students` |
| `assessment_lea` | One assessment row per LEA. A school district keeps the district workbook. Any other LEA is copied from its single school row that year |

`refine` runs `model.sql` and then `portal.sql`, so the `afr_general_fund` text that remains is the one in `portal.sql`. Both texts are fund 10 of `afr_expenditure`.
