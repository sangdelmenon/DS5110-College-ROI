# Data Directory

Data files are **not included** in this repository due to file size.

## Download Instructions

1. Go to https://collegescorecard.ed.gov/data/
2. Download the following two files and place them in this `data/` directory:

| Required Filename | Download Label |
|-------------------|---------------|
| `Most-Recent-Cohorts-Institution.csv` | Most Recent Institution-Level Data |
| `Most-Recent-Cohorts-Field-of-Study.csv` | Most Recent Field of Study Data |

> Note: The College Scorecard uses `PrivacySuppressed` as a placeholder for suppressed values. The R code handles this automatically by treating it as `NA` at load time.

## Additional Reference

- [College Scorecard Data Dictionary](https://collegescorecard.ed.gov/assets/CollegeScorecardDataDictionary.xlsx)
- [Technical Documentation](https://collegescorecard.ed.gov/assets/InstitutionDataDocumentation.pdf)
