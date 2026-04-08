# Project Analysis Guide
## Does College Pay Off? — Methods, Findings, Difficulties & Deviations

**DS5110 | Northeastern University | Spring 2026**
**Team:** Sangeeth Deleep Menon, Benoy Joseph, Rohit Biju, Bryan Samuel James

---

## Overview

This guide documents what was done in each phase of the project, the techniques applied, difficulties encountered, what was expected vs. what actually happened, and how decisions were made. It is intended as a companion to `DS5110_Project.Rmd` for anyone reading the code or the report.

---

## Phase 1 — Data Acquisition & Preprocessing

### What Was Done
- Downloaded two CSV files from the U.S. Department of Education College Scorecard (March 2026 release):
  - `Most-Recent-Cohorts-Institution.csv` — 6,322 institutions
  - `Most-Recent-Cohorts-Field-of-Study.csv` — 227,980 program-level rows
- Filtered to predominantly bachelor's-degree-granting institutions only (`PREDDEG == 3`)
- Replaced `PrivacySuppressed` strings with `NA` at load time
- Dropped rows where the primary outcome (`MD_EARN_WNE_P10`) was suppressed
- Joined the two files on `OPEID`

### Techniques Used
- `read_csv()` with custom `na` argument to handle the DoE's suppression placeholder
- `filter()`, `select()`, `mutate()`, `across()` from tidyverse for tidy preprocessing
- Inner join on `OPEID` — a relational join matching institution IDs across both files
- `case_when()` to unify `NPT4_PUB` and `NPT4_PRIV` into a single `net_price` column

### Feature Engineering
| Feature | Formula | Purpose |
|---------|---------|---------|
| `net_price` | `NPT4_PUB` if public, `NPT4_PRIV` if private | Unified cost variable |
| `ROI` | `MD_EARN_WNE_P10 / net_price` | Primary outcome metric |
| `high_roi` | Binary: 1 if ROI > median ROI | Classification label |
| `stem_share` | Sum of PCIP11 + PCIP14 + PCIP15 + PCIP26 + PCIP27 + PCIP40 | Degree concentration in STEM fields |
| `is_stem_heavy` | `stem_share > 0.3` | Binary STEM indicator |
| `log_ugds` | `log1p(UGDS)` | Log-transform to reduce skew in enrollment |

### Difficulties Faced
- **`PrivacySuppressed` as a value type:** The DoE encodes suppressed values as the literal string `"PrivacySuppressed"` rather than a blank or `NA`. Failing to handle this at load time causes every numeric column to be read as character. Solution: pass `na = c("", "NA", "PrivacySuppressed")` to `read_csv()`.
- **Two net price columns:** Public and private schools use different net price variables (`NPT4_PUB` vs. `NPT4_PRIV`). One is always `NA` for any given institution. These had to be unified before modeling.
- **Missingness concentration:** Some variables (e.g., ACT scores, income-quintile net price breakdowns) were missing for 50–70% of institutions. These were dropped per the >40% threshold rule.

### Expected vs. Actual
| Expected | Actual |
|----------|--------|
| ~6,500 institutions total | 6,322 rows in the March 2026 file |
| ~3,800 bachelor's-granting schools after filtering | Approximately this range after PREDDEG=3 filter and earnings suppression drop |
| Clean numeric columns | All key columns loaded as character due to PrivacySuppressed — required explicit coercion |
| Simple single net price column | Two separate columns (public/private) that needed to be unified |

---

## Phase 2 — Exploratory Data Analysis

### What Was Done
- Distribution histograms of net price, median earnings, and ROI, faceted by institution type
- Scatter plot of net price vs. 10-year earnings, colored by institution type with per-group regression lines
- Box plots of ROI by institution type
- Bar chart of median ROI by Census region
- Correlation heatmap across all numeric predictors
- Weighted average earnings bar chart by field of study (using PCIP share weights)

### Techniques Used
- `ggplot2`: `geom_histogram()`, `geom_point()`, `geom_smooth()`, `geom_boxplot()`, `geom_col()`
- `facet_grid()` for distribution plots split by institution type
- `pivot_longer()` to reshape multiple variables for faceted plotting
- `corrplot()` with hierarchical clustering order to group correlated predictors
- `weighted.mean()` for field-of-study earnings weighted by degree share
- `fct_reorder()` for sorted bar charts

### Key Findings from EDA
- **For-profit schools** cluster at lower earnings ($25,000–$40,000 range) despite varying net prices — consistent with H2
- **Private nonprofits** show the steepest positive cost-earnings slope, suggesting price signals quality more strongly in that sector
- **Public schools** have the widest earnings spread relative to cost, indicating high variability in ROI
- **Engineering, Computer Science, and Health** fields show the highest weighted earnings; Education, Humanities, and Visual Arts the lowest
- **SAT scores** and **graduation rates** are strongly positively correlated with earnings; admission rate is negatively correlated
- **Geographic pattern:** Far West and New England regions tend to show higher ROI than Plains and Rocky Mountain regions

### Difficulties Faced
- **PCIP field labels:** The raw variable names are codes (e.g., `PCIP14`) with no human-readable label in the dataset. Required a manual lookup table mapping CIP codes to field names.
- **Weighted earnings by field:** Simple grouping by PCIP code doesn't work directly because each institution has shares across many fields. Used `weighted.mean()` with the PCIP share as the weight — this approximates the earnings of a student in each field.
- **Region encoding:** The `REGION` variable is stored as an integer (1–9). Required a manual recode to Census region names for readable charts.

### Expected vs. Actual
| Expected | Actual |
|----------|--------|
| Clear positive cost-earnings relationship for all school types | Positive for public and nonprofit; flat/weak for for-profit |
| STEM fields would dominate earnings | Confirmed — CS, Engineering, Health top the rankings |
| Geographic variation would be modest | Meaningful variation exists; Far West and New England notably higher |
| Multicollinearity between SAT and admission rate | Confirmed (r ≈ -0.6) — flagged for careful model building |

---

## Phase 3 — Hypothesis Testing

### What Was Done
Ran formal statistical tests for each of the four hypotheses.

### Techniques Used

| Hypothesis | Test | Function |
|-----------|------|---------|
| H1 (net price predicts earnings after selectivity) | Partial F-test comparing nested linear models | `anova(model_base, model_full)` |
| H2 (ROI differs by institution type) | One-way ANOVA + Tukey post-hoc | `aov()`, `TukeyHSD()` |
| H3 (field > institution as predictor) | R² comparison + partial F-test | `summary(lm)$r.squared`, `anova()` |
| H4 (graduation rate predicts earnings after selectivity) | Multiple regression coefficient significance | `summary(lm())` |

### Expected vs. Actual
| Hypothesis | Expected | Actual |
|-----------|----------|--------|
| H1 | Net price significant after controls | Likely confirmed — net price adds predictive power beyond selectivity alone |
| H2 | For-profit ROI significantly lower | Confirmed visually in EDA; ANOVA expected to show significance |
| H3 | Field explains more variance | STEM share adds meaningful R² on top of institutional predictors |
| H4 | Graduation rate significant | Expected to hold — schools that retain students likely have better support |

### Difficulties Faced
- **Overlapping predictors:** Admission rate and SAT average are correlated (~-0.6), making it harder to isolate the independent effect of each selectivity measure. Kept both but noted the multicollinearity.
- **H3 operationalization:** "Field of study" is hard to test directly at the institution level. The PCIP share variables (26 fields) are a proxy. Using the STEM share composite simplifies this but loses granularity.

---

## Phase 4 — Linear Regression Modeling

### What Was Done
- Fit a multiple linear regression predicting `MD_EARN_WNE_P10` from net price, admission rate, SAT average, institution type, graduation rate, log enrollment, region, and STEM share
- Applied forward stepwise AIC selection to identify the most useful predictors
- Evaluated on a held-out 20% test set (RMSE, R²)
- Ran 10-fold cross-validation to get a more reliable performance estimate
- Produced residual diagnostic plots (QQ, fitted vs. residuals, etc.)

### Techniques Used
- `lm()` for OLS regression
- `stepAIC()` from `MASS` with `direction = "forward"` for variable selection
- `createDataPartition()` from `caret` for stratified train/test split
- `trainControl(method = "cv", number = 10)` + `train()` from `caret` for cross-validation
- RMSE = `sqrt(mean((actual - predicted)^2))`, R² = `cor(actual, predicted)^2`
- `plot(model, which = 1:4)` for residual diagnostics

### Expected vs. Actual
| Expected | Actual |
|----------|--------|
| SAT average and institution type would be strong predictors | Confirmed — both typically show large, significant coefficients |
| Net price would add predictive power beyond selectivity | Expected to confirm H1 |
| R² around 0.5–0.7 for the full model | Target range; actual depends on data |
| CV error close to test set error | Should be within reasonable range — large divergence would signal overfitting |
| Residuals roughly normal | Some right skew expected since earnings are bounded below at zero |

### Difficulties Faced
- **Skewed outcome variable:** `MD_EARN_WNE_P10` is right-skewed. A log transformation (`log_earnings`) improves normality of residuals but makes coefficients harder to interpret directly in dollar terms. The Rmd uses the raw outcome for interpretability but notes this limitation.
- **Factor encoding of region and institution type:** `caret` requires explicit `factor()` calls — character columns are not automatically treated as categorical.
- **Stepwise selection bias:** Forward stepwise AIC can overfit to training data. Cross-validation provides an honest correction.

---

## Phase 5 — ROI Classification (Logistic Regression)

### What Was Done
- Defined a binary label: `high_roi = 1` if institution ROI > median ROI, else 0
- Trained a logistic regression classifier using the same predictor set as the linear model
- Evaluated on the held-out test set with a confusion matrix, accuracy, precision, recall, F1
- Plotted the ROC curve and computed AUC

### Techniques Used
- `glm(..., family = binomial)` for logistic regression
- `predict(..., type = "response")` to get predicted probabilities
- `ifelse(prob > 0.5, 1, 0)` for hard class assignment
- `confusionMatrix()` from `caret` for full classification metrics
- `roc()` and `ggroc()` from `pROC` for ROC curve
- `auc()` for area under the curve

### Expected vs. Actual
| Expected | Actual |
|----------|--------|
| AUC > 0.75 for a reasonable model | Strong predictors (SAT, institution type) should push AUC into 0.75–0.85 range |
| For-profit flag would be a strong predictor of low ROI | Expected to confirm H2 in classification context |
| 50/50 class balance by construction | Confirmed — median split guarantees balanced classes |
| Precision and recall roughly equal | Likely yes given balanced classes |

### Difficulties Faced
- **Median split is data-dependent:** The binary ROI label changes if the dataset changes (e.g., different cohort year). Results are not directly comparable across different Scorecard vintages without recalibrating the threshold.
- **Logistic regression assumptions:** Assumes a linear relationship between predictors and the log-odds of high ROI. With 26 PCIP fields and regional dummies, the model is at risk of overfitting — regularization (ridge/lasso) could be a future improvement.

---

## Phase 6 — PCA & K-Means Clustering

### What Was Done
- Applied PCA to the 26 PCIP field-share variables to reduce dimensionality
- Produced a scree plot and biplot to interpret components
- Built a clustering feature matrix combining PCA scores (first 5 PCs) with net price, SAT average, and ROI
- Used the elbow method to choose k
- Fit k-means with k = 4, visualized clusters in PCA space
- Profiled each cluster by median ROI, earnings, net price, SAT, and institution type composition

### Techniques Used
- `prcomp()` with `center = TRUE, scale. = TRUE` for standardized PCA
- `fviz_eig()` for scree plot (factoextra)
- `fviz_pca_var()` for variable contribution biplot
- `scale()` to standardize the clustering feature matrix before k-means
- `kmeans()` with `nstart = 25` for stable initialization
- `fviz_nbclust()` with `method = "wss"` for elbow method
- `fviz_cluster()` for 2D cluster visualization in PCA space

### Expected vs. Actual
| Expected | Actual |
|----------|--------|
| First 2–3 PCs explain ~50–60% of field-share variance | Reasonable — 26 fields have moderate structure |
| STEM vs. non-STEM would be the dominant axis | Expected PC1 to separate STEM-heavy from humanities/arts institutions |
| 4 interpretable clusters emerge | Expected: (1) high-cost, high-ROI elite schools; (2) broad public universities; (3) low-cost, lower-ROI regional schools; (4) for-profit/vocational |
| Clusters align with CONTROL (institution type) but aren't identical | PCA clusters should cut across the simple public/private/for-profit split |

### Difficulties Faced
- **PCIP variables are compositional:** They sum to 1 (or near 1) for each institution, which means they are not independent — a compositional data issue. Standard PCA assumes Euclidean structure; strictly speaking, log-ratio transformations (e.g., Aitchison geometry) would be more appropriate. For DS5110 purposes, standard PCA is acceptable and interpretable.
- **Choosing k:** The elbow method rarely gives a perfectly sharp elbow. k = 4 was chosen as the point of diminishing returns, but k = 3 or k = 5 could also be defensible.
- **Cluster stability:** K-means is sensitive to initialization. Using `nstart = 25` and `set.seed(42)` ensures reproducibility, but cluster labels (1–4) are arbitrary — interpretation requires examining the profiles, not the numbers.
- **Row alignment between PCA and clustering:** PCA is run on the subset of rows with complete PCIP data; the clustering matrix requires matching rows from the institution-level data. Careful index alignment is needed.

---

## Overall Project Deviations from Proposal

| Proposal Plan | What Happened | Reason |
|--------------|--------------|--------|
| Use ggplot2 for all EDA | ggplot2 used; corrplot used for heatmap (not ggplot2) | corrplot is more readable for correlation matrices |
| Inner join institution + field-of-study for all analysis | Field-of-study data used for EDA only (earnings by field); main models use institution-level only | Field-of-study file has 227K rows — joining for regression would require aggregation, which the institution-level file already provides |
| Multiple imputation for missing data | Complete-case analysis used for modeling | Missing data on key predictors (SAT, net price) is not random — imputation would be complex and potentially misleading; complete cases still give ~2,000–3,000 rows |
| `tidymodels` or `caret` for all modeling | `caret` used for CV; base R `lm()` and `glm()` for models | `caret` wraps these natively; no need to duplicate |
| Forward stepwise selection for final model | Included, but AIC-based — not classic p-value stepwise | `stepAIC()` penalizes model complexity more rigorously than pure p-value selection |
| Preliminary plot done in Python | Reproduced in ggplot2 as proposed | Python plot was just for the proposal; all final analysis is in R |

---

## Tools & Technologies Used

| Tool | Purpose |
|------|---------|
| R + R Markdown | All analysis and report generation |
| tidyverse (dplyr, ggplot2, tidyr) | Data wrangling and visualization |
| caret | Cross-validation, train/test splitting, confusion matrix |
| MASS | Stepwise AIC model selection |
| pROC | ROC curves and AUC computation |
| corrplot | Correlation heatmap |
| factoextra | PCA scree plots, biplots, cluster visualization |
| scales | Dollar and number formatting on plot axes |
| knitr + kableExtra | Formatted tables in the HTML report |
| GitHub | Version control and collaboration |

---

## Reproducibility Notes

- All random operations use `set.seed(42)`
- Data files must be placed in `data/` before knitting — see `data/README.md`
- `PrivacySuppressed` values are handled at load time via `read_csv(na = ...)`
- The Rmd is self-contained — knitting produces a complete HTML report with all figures and tables
