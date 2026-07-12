# part-1
 Capstone Part 1 — Data Acquisition, Cleaning, and Exploratory Analysis

**Dataset:** `raw_employee_data.csv` — a synthetic "Employee Performance" dataset
(1,225 rows × 8 columns) built specifically for this project. It contains
EmployeeID, Age, Department, City, YearsExperience, MonthlySalary,
PerformanceScore, and TrainingHours, with realistic messiness (nulls,
duplicates, a mis-typed column, skew, and outliers) intentionally injected
so every checklist item has something real to find.

Script: `part1_eda.py` | Output: `cleaned_data.csv`, `plots/*.png`, `analysis_log.txt`

---

## Task 1 — Load Dataset
Loaded with `pd.read_csv()`. Shape: **1225 rows × 8 columns**. Dtypes were
inspected with `.dtypes` — most columns loaded as expected except
`TrainingHours`, which loaded as `object` (see Task 4).

## Task 2 — Null Value Analysis
| Column | Null % |
|---|---|
| City | ~27% |
| TrainingHours | ~8% (as `NaN`-like strings, revealed after Task 4) |
| MonthlySalary | ~8% |
| PerformanceScore | ~6% |
| Age | ~10% |

**City exceeds the 20% threshold** and was left alone rather than imputed
(imputing a categorical column with >20% missingness risks fabricating a
large chunk of the dataset and biasing any downstream group-by analysis).

For the numeric columns under 20% nulls (Age, MonthlySalary,
PerformanceScore), missing values were filled with the **column median**,
not the mean. Justification: several of these columns (especially
MonthlySalary) are right-skewed with high-earner outliers pulling the mean
upward. The mean would therefore not represent a "typical" employee, while
the median is robust to that skew and gives a more realistic fill value.

## Task 3 — Duplicate Detection and Removal
`df.duplicated().sum()` found **25 duplicate rows** (deliberately injected
to simulate a data-export glitch). These were removed with
`drop_duplicates()`, bringing the shape to **1200 rows**. Null percentages
per column shifted only marginally (<0.1 percentage points) after removal,
since duplicates were sampled randomly and weren't concentrated in any one
column's missingness pattern.

## Task 4 — Data Type Correction
`TrainingHours` was stored as `object` because some values contained
trailing text like `"12.4 hrs"` or the literal string `"NA"`. It was
cleaned by stripping the `"hrs"` suffix and converting with
`pd.to_numeric(errors='coerce')`, which correctly turned the `"NA"` strings
into real `NaN`s (34 of them), which were then median-filled.

`Department` — a repetitive string column with only 5 unique values across
1200 rows — was converted to `category` dtype.

**Memory usage:** 243,852 bytes → 127,901 bytes, a **47.5% reduction**,
mostly from the categorical conversion (pandas no longer stores the full
string for every row, just a small integer code per row plus one lookup
table).

## Task 5 — Descriptive Statistics and Skewness
`df.describe()` was run on all numeric columns. Skewness (sorted by
magnitude):

| Column | Skew |
|---|---|
| MonthlySalary | **4.84** |
| TrainingHours | 1.67 |
| PerformanceScore | -0.96 |
| YearsExperience | 0.66 |
| Age | 0.16 |

**MonthlySalary has the highest absolute skew (4.84, strongly positive).**
A positive skew means the distribution has a long right tail — most
employees earn a moderate salary, but a small number of high earners pull
the tail far to the right. Consequence for imputation: filling missing
values in this column with the **mean** would overstate what a "typical"
employee earns, because the mean is dragged upward by those few outliers.
The **median** is the safer, more representative choice here (as used in
Task 2). PerformanceScore, by contrast, shows mild **negative** skew — a
short left tail caused by a small group of low performers — meaning its
mean is pulled slightly *below* the typical score.

## Task 6 — Outlier Detection (IQR Method)
| Column | Q1 | Q3 | IQR | Lower Bound | Upper Bound | Outliers |
|---|---|---|---|---|---|---|
| MonthlySalary | 34,478.36 | 49,800.61 | 15,322.25 | 11,494.99 | 72,783.98 | **49** |
| TrainingHours | 4.78 | 21.10 | 16.33 | -19.71 | 45.59 | **56** |

Outliers were **not dropped**. For Part 2, MonthlySalary outliers will be
**capped (winsorized)** at the upper bound rather than removed — they
represent real senior/executive employees, and dropping them would throw
away legitimate signal that a salary-prediction model needs. TrainingHours
outliers will likely be **retained as-is**, since a long right tail of
highly-trained employees is plausible business behavior rather than a data
error, and tree-based models used in Part 2 are relatively insensitive to
this kind of skew.

## Task 7 — Visualizations
All six required visualizations were generated (see `plots/` folder):

1. **Line plot** (`01_line_plot.png`) — MonthlySalary sorted by index, showing the smooth upward curve typical of a right-skewed distribution with a sharp jump at the high end (the outlier tail).
2. **Bar chart** (`02_bar_chart.png`) — mean MonthlySalary by Department. Sales and HR/Finance show the highest averages; Marketing the lowest, though the gap between departments is fairly narrow (see Task 8c).
3. **Histogram** (`03_histogram.png`) — distribution of MonthlySalary (the most skewed column). The shape is strongly right-skewed: a tall peak in the 30–45k range with a long thin tail stretching out past 100k, consistent with the skew value of 4.84.
4. **Scatter plot** (`04_scatter.png`) — YearsExperience vs MonthlySalary. There's a clear positive relationship — salary rises with experience — but with increasing spread (heteroscedasticity) at higher experience levels, and a handful of high-salary outliers that don't follow the trend.
5. **Box plot** (`05_boxplot.png`) — PerformanceScore by Department. Medians are broadly similar across departments (roughly 68-72), but IT and Sales show slightly wider spread and more low-end outliers than HR or Finance.
6. **Correlation heatmap** (`06_heatmap.png`) — full numeric correlation matrix.

**Highest correlation pair: Age & YearsExperience (r = 0.797).** This is
very likely **not purely causal** in either direction — it's more
plausible that a **third variable, career start age / tenure at the
company**, explains both: older employees have naturally had more time to
accumulate work experience. Age doesn't "cause" experience in a direct
mechanical sense; both are driven by the passage of time since someone
entered the workforce.

## Task 8a — Imputation Strategy Comparison
For the two most-skewed numeric columns:

| Column | Mean | Median | Skew |
|---|---|---|---|
| MonthlySalary | 45,655.22 | 41,661.00 | +4.84 |
| TrainingHours | 15.01 | 10.65 | +1.67 |

Both columns are **positively skewed**, so in both cases the mean is
pulled upward by extreme high values (top earners, heavily-trained
employees) and is not representative of the typical employee. The
**median** was used for imputation in both columns for this reason — it's
consistent with the reasoning in Task 2 and Task 5. After filling, `isnull().sum()`
confirmed **zero remaining nulls** in both columns.

## Task 8b — Spearman vs Pearson Correlation
Full Pearson and Spearman matrices were computed and compared
(`analysis_log.txt` has the full tables). Top 3 pairs by
|Spearman − Pearson|:

| Pair | \|Spearman − Pearson\| |
|---|---|
| YearsExperience ↔ MonthlySalary | 0.405 |
| MonthlySalary ↔ Age | 0.357 |
| TrainingHours ↔ MonthlySalary | 0.044 |

- **YearsExperience ↔ MonthlySalary**: Spearman (0.729ish, higher) clearly
  exceeds Pearson (0.324) — this pair is **monotonic but non-linear**.
  Salary rises consistently with experience but not proportionally (early
  raises are large, later raises taper, and the high-salary outliers add
  non-linear noise). **Spearman is the more trustworthy measure here**,
  because it captures the consistent ranking relationship that the
  skewed, outlier-heavy Pearson correlation understates.
- **MonthlySalary ↔ Age**: same story — same underlying driver
  (experience) as above, so the same non-linear, monotonic pattern shows
  up, and Spearman is again the better guide for feature selection.
- **TrainingHours ↔ MonthlySalary**: the difference here is small (0.044),
  meaning whatever weak relationship exists is *already* well captured by
  Pearson — this pair is closer to linear (or, more likely, just weakly/no
  relationship at all, since both raw values are near zero).

**For Part 2 feature selection**, Spearman will be the primary correlation
measure for salary-related features, since MonthlySalary is heavily skewed
and its relationships with other variables are consistently monotonic
rather than strictly linear.

## Task 8c — Grouped Aggregation
`groupby('Department')['MonthlySalary'].agg(['mean','std','count'])`:

| Department | Mean | Std | Count |
|---|---|---|---|
| Sales | 46,601.76 | 28,596.27 | 365 |
| HR | 46,503.10 | 25,544.97 | 150 |
| Finance | 46,400.06 | 27,458.52 | 175 |
| IT | 45,037.24 | 21,055.90 | 316 |
| Marketing | 43,553.52 | 22,936.66 | 194 |

- **Highest mean:** Sales. **Highest std dev:** Sales (also highest).
- Sales' standard deviation (~28.6k) is nearly as large as the gap between
  the highest and lowest department means (~3k) — this is a real concern
  for a predictive model. **High within-group variance means Department
  alone is not enough to reliably predict an individual's salary**; two
  Sales employees can differ by tens of thousands of rupees, so the model
  will need other features (YearsExperience, PerformanceScore) to do the
  real predictive work.
- **Ratio of highest to lowest group mean:** 46,601.76 / 43,553.52 =
  **1.07**. This ratio is small (departments differ by only ~7% on
  average), which suggests Department alone carries **weak predictive
  signal** for salary — consistent with the high within-group std dev
  finding above.

## Task 9 — Cleaned Dataset
Final cleaned dataset saved to **`cleaned_data.csv`** — 1200 rows × 8
columns, zero nulls in all imputed columns, correct dtypes, duplicates
removed. This file will be loaded directly in Parts 2 and 3.

---

### Files in this deliverable
- `raw_employee_data.csv` — original messy dataset
- `part1_eda.py` — full analysis script (reproducible, well-commented)
- `cleaned_data.csv` — final cleaned output for Parts 2 & 3
- `plots/` — all 6 required visualizations (PNG)
- `analysis_log.txt` — full console output of every printed statistic
- `README.md` — this file
