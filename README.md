# Does College Pay Off?
### Modeling the Return on Investment of U.S. Higher Education Using the College Scorecard

**DS5110 — Data Management and Processing | Northeastern University | Spring 2026**

**Team:** Sangeeth Deleep Menon · Benoy Joseph · Rohit Biju · Bryan Samuel James

---

## Research Question

> *Which institutional characteristics and program features best predict whether a college degree delivers strong earnings relative to its cost, and how do these factors differ across institution types and fields of study?*

We define **ROI = Median Earnings (10 years after entry) / Average Net Price**.

---

## Hypotheses

| ID | Hypothesis |
|----|-----------|
| H1 | Net price meaningfully predicts median earnings 10yr after enrollment, even after controlling for selectivity |
| H2 | ROI varies significantly by institution type; for-profit schools deliver weaker earnings relative to cost |
| H3 | Field of study is a stronger earnings predictor than institutional characteristics |
| H4 | Higher graduation rates predict better earnings even after controlling for selectivity |

---

## Data

**Source:** [U.S. Department of Education College Scorecard](https://collegescorecard.ed.gov/data/)

Two files are required (place in `data/`):

| File | Download Name |
|------|--------------|
| Institution-level | Most Recent Institution-Level Data (CSV) |
| Field of Study | Most Recent Field of Study Data (CSV) |

> Data files are **not included** in this repo due to size. Download directly from the College Scorecard website linked above.

**Key variables:** `MD_EARN_WNE_P10` (primary outcome), `NPT4_PUB`/`NPT4_PRIV` (net price), `ADM_RATE`, `SAT_AVG`, `CONTROL` (institution type), `C150_4` (graduation rate), `PCIP*` (field shares), `REGION`

---

## Methods Pipeline

```
Data Preprocessing → EDA → Hypothesis Testing → Linear Regression
→ Cross-Validation → Logistic Classification → PCA → K-Means Clustering
```

| Stage | Method | Reference |
|-------|--------|-----------|
| 1 | Multiple linear regression + stepwise selection | ISLR Ch. 3 |
| 2 | 10-fold cross-validation | ISLR Ch. 5 |
| 3 | t-tests, ANOVA, partial F-tests | PSDS Ch. 2–3 |
| 4 | Logistic regression (binary ROI classification) | ISLR Ch. 4 |
| 5 | PCA + k-means clustering | ISLR Ch. 12 |

---

## Project Structure

```
.
├── DS5110_Project.Rmd        # Main analysis — knit to produce HTML report
├── DS5110_Project.html       # Rendered output (after knitting)
├── PROJECT_GUIDE.md          # Phase-by-phase implementation guide
├── README.md                 # This file
├── data/
│   ├── README.md             # Data download instructions
│   ├── Most-Recent-Cohorts-Institution.csv     # (download separately)
│   └── Most-Recent-Cohorts-Field-of-Study.csv  # (download separately)
└── output/                   # Saved figures and model outputs
```

---

## Setup

### R Packages Required

```r
install.packages(c(
  "tidyverse",   # data wrangling + ggplot2
  "corrplot",    # correlation heatmap
  "caret",       # model training + cross-validation
  "pROC",        # ROC curves
  "MASS",        # stepwise selection
  "factoextra",  # PCA/clustering visualization
  "scales",      # axis formatting
  "knitr",       # report rendering
  "kableExtra"   # table formatting
))
```

### Running the Analysis

1. Download data from https://collegescorecard.ed.gov/data/ into `data/`
2. Open `DS5110_Project.Rmd` in RStudio
3. Knit to HTML: `rmarkdown::render("DS5110_Project.Rmd")`

---

## Team Roles

| Team Member | Responsibility |
|-------------|---------------|
| Sangeeth Deleep Menon | Linear regression modeling (Stages 1–2), cross-validation, report writing |
| Rohit Biju | Data acquisition, preprocessing, feature engineering, SQL joins |
| Benoy Joseph | EDA, visualization, statistical hypothesis testing (Stage 3) |
| Bryan Samuel James | Classification (Stage 4), PCA and clustering (Stage 5) |

---

## Timeline

| Week | Dates | Milestone |
|------|-------|-----------|
| 1 | Mar 17–23 | Data download, cleaning, joins, initial EDA |
| 2 | Mar 24–30 | Complete EDA; begin regression modeling |
| 3 | Mar 31–Apr 6 | Cross-validation; hypothesis testing |
| 4 | Apr 7–13 | ROI classification (logistic regression) |
| 5 | Apr 14–20 | PCA and clustering; draft report |
| 6 | Apr 21+ | Final revisions, peer review, submission |

---

## References

- U.S. Department of Education. (2025). *College Scorecard Data*. https://collegescorecard.ed.gov/data/
- James, G., Witten, D., Hastie, T., & Tibshirani, R. (2021). *An Introduction to Statistical Learning* (2nd ed.). Springer.
- Bruce, P., Bruce, A., & Gedeck, P. (2020). *Practical Statistics for Data Scientists* (2nd ed.). O'Reilly.
- Wickham, H., & Grolemund, G. (2023). *R for Data Science* (2nd ed.). O'Reilly.
