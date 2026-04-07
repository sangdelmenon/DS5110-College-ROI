# DS5110 Project Guide: Does College Pay Off?
**Modeling the Return on Investment of U.S. Higher Education Using the College Scorecard**

Team: Sangeeth Deleep Menon, Benoy Joseph, Rohit Biju, Bryan Samuel James

---

## Research Question
> Which institutional characteristics and program features best predict whether a college degree delivers strong earnings relative to its cost, and how do these factors differ across institution types and fields of study?

## Four Hypotheses to Test
| ID | Hypothesis |
|----|-----------|
| H1 | Net price meaningfully predicts median earnings 10 years after enrollment, even after controlling for selectivity (admission rate, SAT scores) |
| H2 | ROI varies significantly by institution type; for-profit schools deliver weaker earnings relative to cost |
| H3 | Field of study is a stronger earnings predictor than institutional characteristics (size, location) |
| H4 | Higher graduation rates predict better earnings even after controlling for selectivity |

---

## Data Sources

### Files to Download
From https://collegescorecard.ed.gov/data/:
1. **Institution-level CSV** — "Most Recent Institution-Level Data"
2. **Field-of-study CSV** — "Most Recent Field of Study Data"

### Key Variables
| Variable | Description | Role |
|----------|-------------|------|
| `MD_EARN_WNE_P10` | Median earnings 10 years after entry | **Primary outcome** |
| `EARN_MDN_5YR` | Median earnings 5 yrs after completion (by field) | Secondary outcome |
| `NPT4_PUB` / `NPT4_PRIV` | Average net price (public / private) | Key predictor |
| `ADM_RATE` | Admission rate | Selectivity control |
| `SAT_AVG` | Average SAT equivalent score | Selectivity control |
| `CONTROL` | Institution type (1=public, 2=private NP, 3=private FP) | Grouping variable |
| `C150_4` | 4-year completion rate (150% time) | Predictor |
| `UGDS` | Total undergraduate enrollment | Predictor |
| `REGION` | Geographic region (Census) | Predictor |
| `PCIP*` | Fraction of degrees by field (26 PCIP fields) | Predictor set |
| `DEBT_MDN` | Median cumulative debt at graduation | Predictor |
| `PREDDEG` | Predominant degree type (filter to 3 = bachelor's) | Filter variable |
| `OPEID` | Institution ID — use for joining the two files | Join key |

---

## Phase-by-Phase Implementation

---

### Phase 1 — Data Acquisition & Preprocessing
**Owner: Rohit** | Week 1 (Mar 17–23) — *should be complete*

#### Steps
1. **Download** both CSVs from the College Scorecard website
2. **Load** into R:
   ```r
   library(tidyverse)
   inst <- read_csv("institution_data.csv", na = c("", "NA", "PrivacySuppressed"))
   fos  <- read_csv("field_of_study_data.csv", na = c("", "NA", "PrivacySuppressed"))
   ```
3. **Filter** to bachelor's-granting institutions only:
   ```r
   inst <- inst |> filter(PREDDEG == 3)
   ```
4. **Drop suppressed rows** — remove institutions where `MD_EARN_WNE_P10` is NA (small sample sizes < 30 students are suppressed by DoE)
5. **Join** institution + field-of-study data:
   ```r
   combined <- inst |> inner_join(fos, by = "OPEID")
   ```
6. **Coerce numeric columns** — many columns load as character due to `PrivacySuppressed`; coerce after replacing with NA
7. **Handle missingness:**
   - Drop any variable missing > 40% of values
   - Use complete-case analysis or multiple imputation for the rest
8. **Engineer features:**
   - **ROI metric:** `ROI = MD_EARN_WNE_P10 / NPT4` (where NPT4 = NPT4_PUB for public, NPT4_PRIV for private)
   - **STEM indicator:** Create binary flag using PCIP field shares (e.g., PCIP11 + PCIP14 + PCIP15 + PCIP26 + PCIP27 > threshold)
   - **Net price unified column:** Combine NPT4_PUB and NPT4_PRIV into one column based on CONTROL

#### Deliverable
- Clean, joined, tidy data frame ready for EDA
- Summary of missingness per variable

---

### Phase 2 — Exploratory Data Analysis
**Owner: Benoy** | Week 2 (Mar 24–30) — *should be complete*

#### Visualizations to Produce

1. **Distribution plots** — histograms/density of `net_price`, `MD_EARN_WNE_P10`, and `ROI`, faceted by `CONTROL` (institution type)
2. **Scatter plot** — net price vs. median earnings, colored by institution type, with per-group regression smoothers (ggplot2 version of the preliminary Python plot)
3. **Geographic analysis** — bar chart or choropleth of median ROI by Census `REGION`
4. **Correlation heatmap** — pairwise correlations among numeric predictors using `corrplot`
5. **Box plots** — ROI and earnings distributions by institution type (for H2 visual check)
6. **Field-of-study earnings** — bar chart of median earnings by CIP code / field (for H3 visual check)

#### Summary Statistics
- Mean/median/SD of ROI by institution type
- Earnings by top 10 and bottom 10 fields of study
- Missing data summary table

#### Deliverable
- EDA R Markdown section with all plots and commentary

---

### Phase 3 — Statistical Inference & Hypothesis Testing
**Owner: Benoy** | Week 3 (Mar 31–Apr 6) — *should be complete*

#### Tests to Run

**H1 — Does net price predict earnings after controlling for selectivity?**
```r
# Fit model with and without net price
model_base <- lm(MD_EARN_WNE_P10 ~ ADM_RATE + SAT_AVG, data = df)
model_full  <- lm(MD_EARN_WNE_P10 ~ ADM_RATE + SAT_AVG + net_price, data = df)
# Check significance of net_price coefficient
summary(model_full)
anova(model_base, model_full)
```

**H2 — Does ROI differ significantly by institution type?**
```r
# One-way ANOVA
aov_result <- aov(ROI ~ factor(CONTROL), data = df)
summary(aov_result)
# Post-hoc t-tests (Tukey)
TukeyHSD(aov_result)
```

**H3 — Field of study vs. institution characteristics**
- Compare R² of a model using only institutional predictors vs. one using field-of-study predictors
- Use partial F-tests or model comparison

**H4 — Graduation rate predicts earnings after controlling for selectivity?**
```r
model_h4 <- lm(MD_EARN_WNE_P10 ~ ADM_RATE + SAT_AVG + C150_4, data = df)
summary(model_h4)  # check C150_4 coefficient significance
```

#### Deliverable
- Test statistics, p-values, and written interpretation for each hypothesis

---

### Phase 4 — Linear Regression Modeling
**Owner: Sangeeth** | Weeks 2–3 — *should be complete or near complete*

#### Stage 1: Multiple Linear Regression
```r
# Full model
model_lm <- lm(
  MD_EARN_WNE_P10 ~ net_price + ADM_RATE + SAT_AVG +
    factor(CONTROL) + C150_4 + UGDS + factor(REGION),
  data = train_df
)
summary(model_lm)

# Forward stepwise selection
library(MASS)
model_step <- stepAIC(model_lm, direction = "forward")
```

#### Evaluation
```r
# On held-out test set
preds <- predict(model_lm, newdata = test_df)
rmse  <- sqrt(mean((test_df$MD_EARN_WNE_P10 - preds)^2))
r2    <- cor(test_df$MD_EARN_WNE_P10, preds)^2
```

#### Stage 2: Cross-Validation
```r
library(caret)
ctrl <- trainControl(method = "cv", number = 10)
model_cv <- train(
  MD_EARN_WNE_P10 ~ net_price + ADM_RATE + SAT_AVG +
    factor(CONTROL) + C150_4 + UGDS + factor(REGION),
  data = df,
  method = "lm",
  trControl = ctrl
)
model_cv$results  # RMSE and R² from CV
```

#### Deliverable
- Final regression model with selected predictors
- RMSE, R², residual diagnostic plots (QQ, fitted vs. residuals)
- CV-estimated prediction error vs. train-test split estimate

---

### Phase 5 — ROI Classification (Logistic Regression)
**Owner: Bryan** | Week 4 (Apr 7–13) — *current week*

#### Setup
```r
# Create binary ROI label
median_roi <- median(df$ROI, na.rm = TRUE)
df <- df |> mutate(high_roi = ifelse(ROI > median_roi, 1, 0))

# Train/test split
set.seed(42)
train_idx  <- sample(nrow(df), 0.8 * nrow(df))
train_df   <- df[train_idx, ]
test_df    <- df[-train_idx, ]
```

#### Model
```r
model_logit <- glm(
  high_roi ~ net_price + ADM_RATE + SAT_AVG +
    factor(CONTROL) + C150_4 + UGDS + factor(REGION),
  data = train_df,
  family = binomial
)
summary(model_logit)
```

#### Evaluation
```r
library(pROC)
probs  <- predict(model_logit, newdata = test_df, type = "response")
preds  <- ifelse(probs > 0.5, 1, 0)

# Confusion matrix
confusionMatrix(factor(preds), factor(test_df$high_roi))

# ROC curve
roc_obj <- roc(test_df$high_roi, probs)
auc(roc_obj)
plot(roc_obj)
```

#### Deliverable
- Logistic regression model with interpretation of significant coefficients
- Confusion matrix with accuracy, precision, recall, F1
- ROC curve and AUC value

---

### Phase 6 — PCA & Clustering
**Owner: Bryan** | Week 5 (Apr 14–20)

#### PCA on Field-Share Variables
```r
# Select the 26 PCIP field-share columns
pcip_cols <- df |> select(starts_with("PCIP"))

# Run PCA
pca_result <- prcomp(pcip_cols, center = TRUE, scale. = TRUE)
summary(pca_result)  # variance explained by each PC

# Scree plot
plot(pca_result, type = "l", main = "Scree Plot")

# Biplot
biplot(pca_result, scale = 0)
```

#### K-Means Clustering
```r
# Use PCA scores + cost/selectivity/outcome variables
cluster_features <- data.frame(
  pca_result$x[, 1:5],  # first 5 PCs
  net_price = df$net_price,
  SAT_AVG   = df$SAT_AVG,
  ROI        = df$ROI
) |> scale()

# Elbow method to choose k
wss <- map_dbl(1:10, ~kmeans(cluster_features, centers = .x, nstart = 25)$tot.withinss)
plot(1:10, wss, type = "b", xlab = "k", ylab = "WSS")

# Fit final model (choose k based on elbow)
set.seed(42)
km <- kmeans(cluster_features, centers = 4, nstart = 25)
df$cluster <- km$cluster

# Visualize in PCA space
df_pca <- data.frame(pca_result$x[, 1:2], cluster = factor(km$cluster))
ggplot(df_pca, aes(PC1, PC2, color = cluster)) + geom_point(alpha = 0.5)
```

#### Deliverable
- Scree plot showing variance explained
- Cluster profiles (mean ROI, net price, earnings, selectivity per cluster)
- PCA biplot and cluster visualization
- Interpretation: what do the clusters represent?

---

## Current Status (as of Apr 7, 2026)

| Phase | Status | Owner |
|-------|--------|-------|
| Phase 1: Data Preprocessing | Should be complete | Rohit |
| Phase 2: EDA | Should be complete | Benoy |
| Phase 3: Hypothesis Testing | Should be complete | Benoy |
| Phase 4: Linear Regression + CV | Should be complete | Sangeeth |
| **Phase 5: Classification** | **In progress (this week)** | **Bryan** |
| Phase 6: PCA & Clustering | Up next (Apr 14–20) | Bryan |
| Report Writing | Apr 21 onward | All |

---

## R Markdown Report Structure

```
1. Introduction & Research Question
2. Data Description & Key Variables
3. Data Preprocessing
   - Filtering, tidying, missingness
   - Feature engineering (ROI, STEM flag)
4. Exploratory Data Analysis
   - Distribution plots
   - Scatter plots (cost vs. earnings)
   - Geographic analysis
   - Correlation heatmap
5. Hypothesis Testing
   - H1: Net price + selectivity → earnings
   - H2: ROI by institution type (ANOVA/t-tests)
   - H3: Field vs. institution predictors
   - H4: Graduation rate → earnings
6. Linear Regression Modeling
   - Model selection (stepwise)
   - CV evaluation (RMSE, R²)
   - Residual diagnostics
7. ROI Classification
   - Logistic regression
   - Confusion matrix, ROC/AUC
8. PCA & Clustering
   - Variance explained
   - Cluster profiles & interpretation
9. Discussion & Conclusions
   - Answers to research hypotheses
   - Policy implications
   - Limitations
10. References
```

---

## R Packages to Install

```r
install.packages(c(
  "tidyverse",    # data wrangling + ggplot2
  "caret",        # model training + CV
  "tidymodels",   # alternative to caret
  "corrplot",     # correlation heatmap
  "pROC",         # ROC curves
  "MASS",         # stepwise selection
  "factoextra",   # PCA/clustering visualization
  "mice"          # multiple imputation (if needed)
))

# Optional: pre-tidied Scorecard data as cross-check
install.packages("collegeScorecard")
```

---

## Key Decisions & Notes

- **ROI formula:** `ROI = MD_EARN_WNE_P10 / net_price` — higher = better return
- **`PrivacySuppressed`** must be converted to `NA` at load time (use `na = c("", "NA", "PrivacySuppressed")` in `read_csv`)
- **Join key:** `OPEID` connects institution-level to field-of-study data
- **Filter:** Keep only `PREDDEG == 3` (bachelor's-granting) for comparability
- **Random seeds:** Set `set.seed(42)` before any train/test split, CV, or k-means
- **Net price:** Public schools use `NPT4_PUB`; private schools use `NPT4_PRIV` — unify into one column
- **STEM indicator:** Sum relevant PCIP shares (PCIP11, PCIP14, PCIP15, PCIP26, PCIP27, PCIP40) and threshold (e.g., > 0.3 = STEM-heavy)
