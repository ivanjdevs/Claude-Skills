---
name: data-wrangling
description: >
  Use this skill whenever the user wants to clean, fix, or wrangle tabular data.
  Triggers include: messy datasets, dirty data, data quality issues, missing values,
  duplicates, wrong data types, outliers, inconsistent formats, normalization, binning,
  discretization, or any mention of "data cleaning", "data wrangling", "limpiar datos",
  "limpiar dataset", "datos sucios", "datos faltantes", "duplicados", "nulos",
  "discretizar", "binning", or similar. Also trigger when the user uploads a CSV, TSV,
  or Excel file and asks to "fix", "clean", "process", or "prepare" it. Always use
  this skill before doing any analysis on raw/untrusted data.
  Outputs a cleaned file (.csv or .xlsx) AND a reusable Python script.
---

# Data Wrangling & Cleaning Skill

## Overview

This skill guides Claude through a structured, interactive data cleaning workflow for CSV, TSV, and Excel files.

The flow is:
1. Inspect the data (automatic)
2. Diagnose quality issues (automatic)
3. **Present a cleaning plan and wait for user approval** ← interactive block
4. Execute the approved plan
5. Validate results
6. Deliver outputs

The output is always:
1. **A cleaned data file** (same format as input, or as requested)
2. **A reusable Python script** that documents and reproduces every transformation

---

## Step 0 — Read the File First

Before anything else, inspect the data:

```python
import pandas as pd
import numpy as np

# CSV / TSV
df = pd.read_csv('file.csv')          # or sep='\t' for TSV
# Excel
df = pd.read_excel('file.xlsx')       # add sheet_name=... if needed

print(df.shape)
print(df.dtypes)
print(df.head())
print(df.isnull().sum())
print(df.duplicated().sum())
```

Always report to the user:
- Shape (rows × columns)
- Column names and inferred types
- Missing value counts per column
- Duplicate row count
- A brief sample (first 3–5 rows)

---

## Step 1 — Diagnose Data Quality Issues

Run a full diagnostic and surface findings to the user before making changes.

### Checklist

| Issue | How to detect |
|---|---|
| Missing values (NaN, None, empty strings) | `df.isnull().sum()`, `(df == '').sum()` |
| Placeholder strings used as nulls (`?`, `-`, `N/A`) | See heuristic scan below |
| Columns with >50% nulls | `df.isnull().mean()` — flag any column above 0.5 |
| Duplicate rows (full) | `df.duplicated().sum()` |
| Duplicate rows (by key column) | See key column scan below |
| Wrong data types (numbers stored as `object`) | `df.dtypes` |
| Inconsistent string formats | `df['col'].value_counts()` for low-cardinality columns |
| Leading/trailing whitespace | `df['col'].str.strip()` comparison |
| Outliers (numeric) | `df.describe()`, IQR method |
| Mixed formats (dates, currencies) | `df['col'].sample(10)` |
| Columns with all nulls | `df.columns[df.isnull().all()]` |

### Placeholder detection — heuristic scan

Do not rely only on the known list (`?`, `-`, `N/A`, etc.). Run this scan on every `object` column to catch non-standard placeholders the user may not be aware of:

```python
KNOWN_PLACEHOLDERS = {'?', '-', '--', 'N/A', 'n/a', 'NA', 'null', 'NULL', 'None', 'none', '', ' '}

suspicious = {}
for col in df.select_dtypes(include='object').columns:
    unique_vals = df[col].dropna().unique()
    flagged = []
    for v in unique_vals:
        s = str(v).strip()
        # Flag: very short (1-2 chars), non-alphabetic, or known placeholder
        if s in KNOWN_PLACEHOLDERS or (len(s) <= 2 and not s.isalpha()):
            flagged.append(v)
    if flagged:
        suspicious[col] = flagged

if suspicious:
    print("Possible non-standard placeholders detected:")
    for col, vals in suspicious.items():
        print(f"  {col}: {vals}")
```

Report any flagged values to the user in the diagnosis summary. Include them in the cleaning plan (Step 2) and ask the user to confirm whether they should be treated as nulls.

Also flag columns that pandas read as `object` but appear to contain numeric values — these almost always have a hidden placeholder blocking the type inference:

```python
for col in df.select_dtypes(include='object').columns:
    converted = pd.to_numeric(df[col], errors='coerce')
    pct_valid = converted.notna().mean()
    if pct_valid > 0.8:  # >80% of values look numeric
        print(f"'{col}' looks numeric but is stored as object — likely has a placeholder")
```

### Temporary numeric preview — required for ANY statistic on an `object` column

This is a general technique, not just for the placeholder scan above. Any time Step 1 or
Step 2 needs to calculate a statistic (mean, median, skew, quartiles/IQR for outliers) on a
column that is still `object` dtype in the real dataframe, use a **temporary, non-destructive
coercion** — never modify `df` itself at this stage:

```python
# Example: outlier diagnosis on a column still stored as object
preview = pd.to_numeric(df['price'], errors='coerce')  # temporary series, df untouched

Q1 = preview.quantile(0.25)
Q3 = preview.quantile(0.75)
IQR = Q3 - Q1
lower, upper = Q1 - 1.5 * IQR, Q3 + 1.5 * IQR
outlier_count = ((preview < lower) | (preview > upper)).sum()

# Same logic applies to skew, mean, and median calculations used to suggest
# an imputation strategy in Step 2 — always compute on `preview`, never on df[col] directly
# until the real conversion happens in Step 3d.
```

> ⚠️ This creates a real risk of confusing the user: the Step 2 plan may show a mean/median
> suggestion for a column that the SAME plan also lists as "stored as object, needs conversion."
> Both statements are true, but presented together they look contradictory. Whenever this
> happens, Claude MUST add an explicit note in the plan clarifying that the suggested values
> were computed from a temporary numeric preview, and that the real, permanent conversion runs
> later during execution (Step 3d) — see the note template in Step 2's plan format.

### Key column duplicate scan

Full-row duplicates are caught by `drop_duplicates()`, but real datasets often have duplicates based on a key column (e.g. same Order ID appearing twice with slightly different values elsewhere). Detect candidate key columns:

```python
for col in df.columns:
    n_unique = df[col].nunique()
    n_rows = len(df)
    uniqueness_ratio = n_unique / n_rows
    if uniqueness_ratio > 0.9:  # >90% unique values — likely a key column
        dupes = df[df.duplicated(subset=[col], keep=False)]
        if not dupes.empty:
            print(f"'{col}' looks like a key column and has {len(dupes)} duplicate entries")
```

Report candidate key columns to the user and ask for confirmation in Step 2.

Summarize all findings internally. Do NOT apply any transformation yet. Proceed to Step 2.

---

## Step 2 — Present the Cleaning Plan (interactive block)

After the diagnosis, present a complete cleaning plan to the user and **wait for approval before executing anything**.

### Pre-plan questions

Before presenting the full plan, always ask these two questions:

> **1.** "Are there any columns in your dataset that are critical and should never be imputed — such as your target variable, a primary key, or an ID column? If so, please name them."

> **2.** "Are there any values in your dataset that represent missing data beyond the ones I may have detected automatically? For example, codes like `#`, `!`, `999`, `0`, or any domain-specific sentinel values?"

Use the answers to refine the plan before presenting it. If the user says no to both, proceed directly to the plan.

### Format for the plan

Present the plan in two clearly separated sections:

**Section A — Outlier decisions (handle first)**
**Section B — Imputation and all other decisions (depends on Section A)**

Add this note at the top of the plan:

> *"I'm showing outlier decisions first because the fill values for imputation (mean/median) are calculated after outliers are resolved. Please confirm Section A before I finalize Section B."*

**Example:**

> **🧹 Proposed Cleaning Plan**
>
> Before I start cleaning, here's what I found and what I propose to do.
>
> **Section A — Outliers (confirm first)**
>
> | Column | Issue | Options | My suggestion |
> |---|---|---|---|
> | `engine_size` | 3 outliers above IQR upper bound (<1% of rows) | Cap (winsorize) / Remove / Leave | **Cap** — preserves row count |
>
> *Once you confirm Section A, I will calculate the mean/median for imputation using the cleaned values.*
>
> **Section B — Everything else**
>
> | Column | Issue | Options | My suggestion |
> |---|---|---|---|
> | *(all columns)* | Column names have spaces | Rename to snake_case | Fixed — standard practice |
> | `normalized_losses` | 41 nulls (20%) — numeric | Mean / Median / Drop rows | **Mean** — symmetric distribution |
> | `bore`, `stroke` | 4 nulls each — numeric | Mean / Median / Drop rows | **Median** — possible skew |
> | `num_of_doors` | 2 nulls — low-cardinality categorical | Mode / 'Unknown' / Drop rows | **Mode** ('four') |
> | `salesperson_name` | 8 nulls — identity/free-text column | 'Unknown' / Drop rows / Leave as NaN | **'Unknown'** — mode meaningless here |
> | `order_id` | 3 nulls — identity column | Drop rows / 'Unknown' / Leave as NaN | **Drop rows** — ID is critical |
> | `price` | 4 nulls — ⚠️ HIGH NULL RATE (55%) | Drop column / Impute with median / Leave as NaN | **Drop column** — too sparse to impute reliably |
> | `horsepower`, `price` | Stored as object, should be float | Convert to float | Fixed — needed for analysis |
> | *(all string cols)* | Leading/trailing whitespace | Strip whitespace | Fixed — consistency |
> | Duplicate rows (full) | 0 found | — | No action needed |
> | `order_id` | Key column — 3 duplicate entries | Drop duplicates on `order_id` / Review manually | **Drop** — keeping first occurrence |
> | Discretization | No numeric column clearly requires binning | — | Skip |
>
> ⚠️ *Nota: `normalized_losses`, `bore`, `stroke` y `price` aparecen arriba con sugerencias de
> media/mediana, pero la tabla también indica que están guardadas como tipo texto (`object`).
> Ambas cosas son ciertas — las sugerencias se calcularon con una vista previa numérica
> temporal, sin modificar los datos todavía. La conversión real y permanente ocurre durante
> la ejecución (antes de aplicar la imputación de verdad), no en este momento.*
>
> **Confirm Section A first, then approve or adjust Section B.**

> 📌 Whenever the plan shows a mean/median/skew-based suggestion for a column that the SAME
> plan also lists under "stored as object, needs conversion," Claude MUST include a note like
> the one above. Do not present these two facts about the same column without that
> clarification — to the user they look contradictory even though both are accurate.

### Decision rules for the plan (use these to fill in the table automatically)

```
Placeholder strings ('?', '-', 'N/A', etc.) detected?
  → Always replace with NaN (not negotiable, propose it as fixed)

Columns with >50% nulls:
  → Flag prominently with ⚠️ HIGH NULL RATE
  → Propose: Drop column
  → Always offer: Drop column / Impute anyway (with warning) / Leave as NaN
  → Never silently impute a column with >50% missing values

Outliers (IQR method) — always present in Section A:
  → If <5% of rows affected: propose cap (winsorize)
  → If ≥5% of rows affected: flag for user, propose review before action
  → If user chooses Leave: add note to Section B — "Since outliers will be kept,
    mean will be skewed — recommending median instead for imputation"

Null values — present in Section B, calculated AFTER Section A is confirmed:
  Is the column named by user as critical / primary key / target variable?
    YES → Propose: Drop rows (fixed, not negotiable)
  Is it numeric?
    YES → Calculate skew AFTER outlier decision is applied
          if abs(skew) < 1 → propose mean; else → propose median
          Always offer: Mean / Median / Drop rows
  Is it an object/string column?
    Is it an identity or free-text column?
    (signals: high cardinality — most values unique; column name contains
    'name', 'id', 'email', 'address', 'description', 'comment', 'note', 'code';
    or values are long strings with no repetition pattern)
      YES → Propose: Fill with 'Unknown'
            Always offer: 'Unknown' / Drop rows / Leave as NaN
    NO — low-cardinality categorical (status, type, category, gender, etc.)
      YES → Check if values are grouped/ordered before proposing mode:

            ```python
            def is_grouped(series):
                s = series.dropna()
                transitions = (s != s.shift()).sum()
                max_transitions = len(s.unique()) * 2
                return transitions <= max_transitions
            ```

            If `is_grouped(df[col])` is True — values repeat in contiguous blocks rather
            than being scattered randomly (e.g. a `make`/brand column sorted alphabetically,
            where assigning the most frequent brand to a null inside a different brand's
            block would be wrong) — do NOT propose mode.
              → Propose: Fill with 'Unknown', explaining why mode is inappropriate here
                (e.g. "asignar 'Toyota' por ser la marca más frecuente no sería razonable
                para un nulo ubicado en el bloque de Volvo")
                Always offer: 'Unknown' / Drop rows / Leave as NaN
            If `is_grouped(df[col])` is False — values are scattered, mode is meaningful:
              → Propose: Fill with mode
                Always offer: Mode / 'Unknown' / Drop rows

Wrong data types:
  → Propose conversion (astype float, to_datetime, etc.) — mark as fixed

Duplicates (full rows):
  → Always propose drop_duplicates

Duplicates (key column):
  → If detected, present as a separate row in the plan
  → Always offer: Drop duplicates on key column (keep first) / Review manually / Leave

Discretization:
  → Do NOT propose by default. Only include in plan if user explicitly requested it,
    or if a column clearly benefits from grouping (e.g. age, income, score ranges).
    If proposing, suggest number of bins and labels.

Normalization / scaling:
  → Do NOT propose by default. Only include if user requested it or if dataset
    context clearly suggests it (e.g. preparing for ML model).
```

### Handling user response

- **"Aprobado" / "Proceed" / "Looks good"** → Execute the plan exactly as proposed.
- **Partial change** (e.g. "for `bore` use mean instead of median") → Update that row, confirm, then execute.
- **"Don't discretize" / "Skip normalization"** → Remove those steps and execute the rest.
- **"Cancel"** → Stop. Do not apply any changes.

> ⚠️ CRITICAL — Answering a pointed question does NOT equal approving the plan.
> If the plan includes specific open questions (e.g. "which row do you want to keep for
> duplicate ID 1205?" or "do you approve the outlier suggestions in Section A?"), the user's
> answer to those questions resolves ONLY that point. It is NOT a general green light.
> Claude must NOT execute the plan until the user gives an explicit, overall confirmation
> ("aprobado", "procede", "adelante", "ejecuta el plan", or equivalent). If the user answers
> the pointed questions but does not explicitly approve the full plan, Claude must ask:
> "Gracias, anoto esas decisiones. ¿Apruebas el resto del plan tal como está para que proceda
> con la ejecución?" — and wait for that confirmation before touching the data.

> ⚠️ If any single operation would remove more than 10% of total rows, highlight it in the plan
> with a warning and explicitly ask the user to confirm that specific step before proceeding.

---

## Step 3 — Execute the Approved Plan

Apply transformations in this fixed order — do not change the sequence:

1. Structural fixes (column names, empty rows/cols)
2. Placeholder replacement (→ NaN)
3. Duplicate removal (full rows, then key column if applicable)
4. Data type conversions + MANDATORY hard-stop null check ← moved before outliers/imputation
5. Outlier handling ← now runs on confirmed-numeric columns only
6. Missing value imputation ← uses clean statistics on fully-typed data
7. String normalization
8. Normalization / scaling (if approved)
9. Discretization / binning (if approved)
10. Dtype cleanup — downcast whole-number floats back to int (automatic)

> ⚠️ Why this order matters: outlier detection (IQR/quantiles) and imputation (mean/median)
> are statistical operations that require a column to already be numeric. If a column is
> still `object` dtype — for example `price` with mixed values like `13495` and `$22.470` —
> these operations either fail outright or silently produce wrong results. Type conversion
> must happen first so any formatting issue surfaces immediately, not after outliers or
> imputation have already run on unreliable data.

### 3a. Structural fixes

```python
# Strip whitespace from column names and convert to snake_case
df.columns = df.columns.str.strip().str.lower().str.replace(' ', '_')

# Drop fully empty rows or columns
df.dropna(how='all', inplace=True)
df.dropna(axis=1, how='all', inplace=True)
```

### 3b. Replace placeholder strings with NaN

Use the combined list: known standards + any values detected by the heuristic scan in Step 1 + any additional values confirmed by the user in Step 2.

```python
# Base list of known placeholders
placeholders = ['?', '-', '--', 'N/A', 'n/a', 'NA', 'null', 'NULL', 'None', 'none', '', ' ']

# Add any non-standard placeholders confirmed by the user or detected by the scan
# e.g. placeholders += ['#', '999', '!']

df.replace(placeholders, np.nan, inplace=True)

# Verify
print("Nulls after placeholder replacement:")
print(df.isnull().sum())
```

> This must happen BEFORE type conversions — a column with `?` values will fail `astype(float)`
> until the placeholders are replaced.

### 3c. Duplicates

```python
# Full-row duplicates
before = len(df)
df.drop_duplicates(inplace=True)
print(f"Removed {before - len(df)} full duplicate rows")

# Key column duplicates (if confirmed by user in Step 2)
before = len(df)
df.drop_duplicates(subset=['order_id'], keep='first', inplace=True)
print(f"Removed {before - len(df)} duplicate entries on key column")
```

### 3d. Data type conversions + MANDATORY hard-stop null check

This step now runs BEFORE outliers and imputation — see explanation above.

```python
# Record null counts before conversion (these are the ORIGINALLY approved nulls
# from placeholder replacement — the ones the user already saw in the plan)
nulls_before = df.isnull().sum()

# Numeric columns — use astype when column is confirmed clean
df['horsepower'] = df['horsepower'].astype('float')

# Use pd.to_numeric with errors='coerce' as fallback when unsure
df['price'] = pd.to_numeric(df['price'], errors='coerce')

# Remove currency/formatting symbols before converting
df['revenue'] = pd.to_numeric(df['revenue'].str.replace(r'[\$,]', '', regex=True), errors='coerce')

# Date columns
df['date'] = pd.to_datetime(df['date'], dayfirst=True, errors='coerce')

# Boolean-like columns
df['active'] = df['active'].map({'yes': True, 'no': False, '1': True, '0': False})

# MANDATORY: check for new nulls introduced by coerce
nulls_after = df.isnull().sum()
new_nulls = nulls_after[nulls_after > nulls_before]
```

> ⚠️ HARD STOP — this is not optional and not just a print statement.
>
> If `new_nulls` is non-empty, Claude must **STOP the entire execution sequence immediately**.
> Do NOT proceed to outlier handling (3e), imputation (3f), or any later step. The dataframe
> at this point is in an intermediate, unconfirmed state.
>
> Identify the exact row(s) and original raw value(s) that caused the new null (e.g. "row with
> `auto_id=1204`, original value `$22.470` in column `price`"). Present this to the user as a
> distinct, separate issue — explicitly NOT part of the already-approved plan — and ask how to
> proceed. Typical resolution: clean the formatting and recover the value (e.g. strip `$` and
> normalize the decimal separator, then reparse) rather than treating it as a real missing value.
>
> Only resume execution (outliers → imputation → remaining steps) once the user has explicitly
> resolved every new null. Never let a newly discovered null get silently swept into a
> previously-approved `dropna()` or imputation step meant for the original, already-known nulls.

### 3e. Outlier handling ← now runs on confirmed-numeric columns

```python
# IQR method — safe now that the column is guaranteed numeric
Q1 = df['col'].quantile(0.25)
Q3 = df['col'].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR

# Option A: Cap (winsorize) — preserves row count
df['col'] = df['col'].clip(lower, upper)

# Option B: Remove
df = df[df['col'].between(lower, upper)]
```

> Outliers are handled here — after type conversion, before imputation — so that mean/median
> calculations in the next step reflect clean, correctly-typed, non-extreme data.

### 3f. Missing values ← uses clean, fully-typed, outlier-resolved data

Before filling numeric nulls with mean or median, match the decimal precision already used
by the existing values in that column — otherwise the imputed value can end up with raw
floating-point precision noise (e.g. `121.67469879518072` instead of a clean `121.67`),
which looks unpolished and is more prone to downstream formatting glitches.

```python
def infer_decimal_places(series):
    non_null = series.dropna()
    decimals = non_null.apply(lambda x: len(str(x).split('.')[-1]) if '.' in str(x) else 0)
    return int(decimals.mode()[0]) if not decimals.empty else 2

# Drop rows with nulls in critical columns (user-confirmed, originally approved nulls)
df.dropna(subset=['price'], inplace=True)

# Fill numeric with mean — rounded to match the column's existing decimal precision
n_decimals = infer_decimal_places(df['normalized_losses'])
df['normalized_losses'] = df['normalized_losses'].fillna(round(df['normalized_losses'].mean(), n_decimals))

# Fill numeric with median — same rounding logic
n_decimals = infer_decimal_places(df['bore'])
df['bore'] = df['bore'].fillna(round(df['bore'].median(), n_decimals))

# Fill categorical with mode
df['num_of_doors'] = df['num_of_doors'].fillna(df['num_of_doors'].mode()[0])

# Fill identity/free-text column
df['salesperson_name'] = df['salesperson_name'].fillna('Unknown')
```

> Apply `infer_decimal_places()` + `round()` to every mean/median imputation, not just the
> examples above. This is a small, mandatory step — not optional — for any numeric fill value.

> ⚠️ pandas 3.0 compatibility: always use `df[col] = df[col].fillna(value)`.
> Never use `df[col].replace(np.nan, value, inplace=True)` — it is deprecated.

### 3g. String normalization

```python
df['name'] = df['name'].str.strip().str.lower()
df['status'] = df['status'].str.strip().str.title()
```

### 3h. Normalization / scaling (only if approved)

```python
# Min-Max normalization (scales to [0, 1])
df['col'] = (df['col'] - df['col'].min()) / (df['col'].max() - df['col'].min())

# Or with sklearn
from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
df[['col1', 'col2']] = scaler.fit_transform(df[['col1', 'col2']])
```

### 3i. Discretization / Binning (only if approved)

```python
# Equal-width bins using linspace
bins = np.linspace(df['col'].min(), df['col'].max(), 4)  # 4 edges = 3 bins
group_labels = ['Low', 'Medium', 'High']
df['col_group'] = pd.cut(df['col'], bins=bins, labels=group_labels, include_lowest=True)

print(df['col_group'].value_counts())
```

> Use `include_lowest=True` to ensure the minimum value falls in the first bin.
> For custom bin edges, pass a list directly: `bins=[0, 100, 200, 500]`.

### 3j. Dtype cleanup — downcast whole-number floats back to int

A common pandas quirk: if a column ever held a NaN (even one, even temporarily), pandas
upcasts it to `float64` to be able to represent that NaN — and it stays `float64` even after
all nulls are gone, causing whole numbers to display as `13495.0` instead of `13495`. This is
purely cosmetic but looks unpolished in the final deliverable. Run this automatically, after
all other transformations, right before validation:

```python
# Downcast float columns back to int when no decimal precision is actually needed
for col in df.select_dtypes(include='float').columns:
    if df[col].notna().all() and (df[col] % 1 == 0).all():
        df[col] = df[col].astype('int64')
```

> This is automatic — mark as "Fixed" in the plan, no user decision needed, the same way
> type conversions are handled. It correctly skips genuinely fractional columns (e.g. `bore`
> with values like `3.78`) since `3.78 % 1 != 0`, so only columns that are whole numbers in
> disguise get downcast. Mention in the cleaning summary which columns were downcast.

---

## Step 4 — Validate the Cleaned Data

```python
print("=== Post-cleaning report ===")
print(f"Shape: {df.shape}")
print(f"Nulls remaining:\n{df.isnull().sum()}")
print(f"Duplicates remaining: {df.duplicated().sum()}")
print(df.dtypes)
print(df.describe())
```

Report results to the user before saving.

---

## Step 5 — Save Outputs

### Cleaned file

```python
# CSV
df.to_csv('cleaned_data.csv', index=False, encoding='utf-8-sig')

# Excel
df.to_excel('cleaned_data.xlsx', index=False)
```

Save to `/mnt/user-data/outputs/` and use `present_files` to share with user.

### Python script

Generate a standalone `clean_data.py` that:
- Declares the input file path as a variable at the top: `INPUT_FILE = 'your_file.csv'`
- Runs every approved transformation in the correct order (outliers before imputation)
- Includes the post-conversion null check after type conversions
- Uses pandas 3.0-compatible syntax throughout
- Prints a before/after summary at the end
- Is well-commented and saves the output file

Save to `/mnt/user-data/outputs/clean_data.py` and present alongside the cleaned file.

---

## Output Format for the User

After delivering the files, present a **cleaning summary**:

> **✅ Cleaning complete**
>
> | | Before | After |
> |---|---|---|
> | Rows | 205 | 199 |
> | Columns | 26 | 25 |
>
> **What was done:**
> - Placeholders replaced: 47 `?` values across 6 columns → NaN
> - Columns dropped: `price` (55% nulls — too sparse to impute)
> - Outliers: 3 values in `engine_size` capped via IQR (before imputation)
> - Nulls imputed: `normalized_losses` (mean), `bore` / `stroke` (median), `num_of_doors` (mode), `salesperson_name` ('Unknown')
> - Rows dropped: 6 (nulls in critical column `order_id`)
> - Full duplicates removed: 0
> - Key column duplicates removed: 2 (on `order_id`)
> - Type conversions: `horsepower`, `price` → float; ⚠️ 2 new nulls detected post-conversion in `revenue` → filled with median
> - Discretization: `horsepower_group` created (Low / Medium / High)
>
> **Files generated:** `cleaned_data.csv`, `clean_data.py`

---

## Important Principles

- **Never apply any transformation before the user approves the plan** (Step 2)
- **Answering a pointed question is not approving the plan** — wait for explicit overall confirmation before executing anything
- **Never silently drop data** — always explain what was removed and why
- **Type conversion before outliers and imputation** — both require numeric data to work correctly
- **Outliers before imputation** — always resolve outliers first so statistics are clean
- **Never silently impute columns with >50% nulls** — flag and ask
- **Post-conversion null check is a hard stop** — if Step 3d finds new nulls, halt the entire
  execution sequence and resolve them with the user before touching outliers or imputation
- **Key column duplicates** — always check, don't rely only on full-row deduplication
- **Critical columns** — always ask the user which columns should never be imputed
- **Order matters** — always follow: placeholders → duplicates → type conversion → outliers → imputation
- **Prefer conservative strategies** — when in doubt, flag rather than delete
- **Destructive operations >10% of rows** — highlight in plan and ask for explicit confirmation
- **Handle encoding issues** — try `encoding='latin-1'` or `encoding='cp1252'` if UTF-8 fails
- **pandas 3.0 compatibility** — use `df[col] = df[col].fillna(value)`, never `.replace(np.nan, ..., inplace=True)`

---

## Reference files

- `references/common_issues.md` — Quick lookup for common messy data patterns and fixes

If you encounter any of the following situations not covered by the steps above, **consult `references/common_issues.md` before proceeding**:

- File fails to load due to encoding errors (garbled characters, UnicodeDecodeError)
- Excel file has merged cells or multi-row headers
- ID/code columns are losing leading zeros
- Date column has mixed formats within the same column
- Currency or percentage columns with symbols that block numeric conversion
- File is too large to load fully into memory
- Multiple sheets in an Excel file that all need cleaning
