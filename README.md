# Maternal & Child Health Analysis — Kenya

## Project Overview
This project analyzes trends in Kenya's maternal and reproductive health indicators using WHO data, exploring how key outcomes have changed over time and where progress has been strongest or weakest.

**Core question:** How have Kenya's key maternal and reproductive health indicators changed between 1993 and 2023, and which areas show the strongest vs. weakest progress? *(Secondary angle: do improvements in one indicator, e.g. antenatal care coverage, track alongside improvements in another, e.g. facility births?)*

**Tools used:** Python (Pandas, Matplotlib/Seaborn), SQL, Power BI

---

## Stage 1: Problem Framing

Initial candidate questions considered:
1. What factors are most associated with skilled birth attendance / facility delivery?
2. How does antenatal care coverage vary by wealth quintile, education, or region?
3. What's the relationship between maternal nutrition/BMI and child stunting/underweight?
4. How have maternal/child health service coverage indicators changed over time in Kenya?

After inspecting the actual dataset (Stage 2), the project was scoped around **Question 4**, since the data available is national-level, pre-aggregated WHO indicator data — not individual-level survey microdata — making trend and cross-indicator comparison the right fit rather than individual-level predictive analysis.

---

## Stage 2: Data Collection

**Source:** [HDX — WHO Data for Kenya](https://data.humdata.org/dataset/who-data-for-ken)
**File:** `maternal_and_reproductive_health_indicators_ken.csv`
**Format:** Long-format WHO Global Health Observatory (GHO) export — one row per indicator + year (+ sub-dimension where applicable)

**Initial shape:** 195 rows × 17 columns

**Indicators included:**
- Births attended by skilled health personnel (%)
- Antenatal care coverage — at least four visits (%)
- Adolescent birth rate (per 1000 women)
- Proportion of births delivered in a health facility (%)
- Births by caesarean section (%)
- Number of women of reproductive age with anaemia (thousands)
- Prevalence of anaemia in women of reproductive age (%)

**Year range:** 1993–2023 (non-continuous — coverage varies by indicator)

---

## Stage 3: Data Cleaning & Preprocessing

### Columns dropped
Constant, redundant, or non-informative for this analysis: `GHO (CODE)`, `GHO (URL)`, `REGION (CODE)`, `REGION (DISPLAY)`, `COUNTRY (CODE)`, `COUNTRY (DISPLAY)`, `DIMENSION (TYPE)`, `DIMENSION (CODE)`, `DIMENSION (NAME)`, `Value` (redundant, rounded display-string version of `Numeric`), `STARTYEAR`, `ENDYEAR`.

### Columns renamed
`GHO (DISPLAY)` → `indicator`, `YEAR (DISPLAY)` → `year`, `Numeric` → `value`, `Low`/`High` → `ci_low`/`ci_high`.

### Missing values
`ci_low`/`ci_high` are missing for facility/service-coverage indicators (skilled attendance, antenatal care, facility births, caesarean rate, adolescent birth rate) — these come from routine health-system reporting rather than sampled surveys, so there's no sampling uncertainty to report. This is a structural pattern, not random missingness, so these were left as `NaN` rather than imputed.

### Data quality issue #1 — Bundled adolescent birth rate
WHO's indicator code `MDG_0000000003` ("Adolescent birth rate (per 1000 women)") bundles two age bands — 10-14 and 15-19 years — under a single code and label, with no distinguishing column in this export. Resolved by splitting on value magnitude (a clean, non-overlapping gap: max of one cluster was 8.1, min of the other was 43.6), cross-validated against known Kenya adolescent birth rates (World Bank data). Values under 20 were labeled "ages 10-14"; values 20+ were labeled "ages 15-19."

### Data quality issue #2 — Bundled anaemia prevalence/counts
Both anaemia indicators (code `NUTRITION_ANAEMIA_REPRODUCTIVEAGE_PREV` and its count equivalent) bundle three sub-populations (pregnant, non-pregnant, all reproductive-age women) under one code, again with no distinguishing column. Unlike the adolescent birth rate case, these three tracks were too close in value to reliably separate three ways. Resolved by:
- Labeling the highest value per year as "pregnant women" (a consistent, physiologically-expected ~10+ point gap above the other two every year)
- Averaging the remaining two values per year into a single "non-pregnant/all women (combined)" category
- Dropping confidence intervals for the combined category (no longer a real published estimate)

This is a documented, judgment-based split rather than a distinction confirmed by WHO source metadata — noted here for transparency.

### Result
- Zero duplicate `indicator` + `year` rows after both splits
- 9 final distinct indicators (7 original + 1 adolescent birth rate split into 2 + 2 anaemia indicators each split into 2, minus originals)
- Clean, renamed, analysis-ready dataset (`df_clean`)

---

## Stage 4: Exploratory Data Analysis

### Summary statistics
Ran `df_clean.groupby('indicator')['value'].describe()` across all 9 cleaned indicators. Notable observations:
- **Caesarean section rate** has only 1 data point (8.7%) — unusable for trend analysis, kept only as a limitation note.
- **Facility births** shows the widest relative spread (40.1%–88.1%), suggesting a dramatic shift rather than gradual change.
- The anaemia split held up under scrutiny: pregnant women consistently higher than non-pregnant/all women across every summary statistic, reinforcing that the Stage 3 splitting logic was directionally sound.

### Chart 1 — Anaemia prevalence trend (pregnant vs. non-pregnant/all women)
Reveals a **U-shaped pattern**, not a simple decline:
- **2000–~2011**: steady improvement for both groups (pregnant women: 45.8% → ~38.9%; non-pregnant/all: 33.7% → ~26.7%)
- **~2012 onward**: both reverse and worsen again through 2023, with non-pregnant/all women climbing back up more steeply (26.8% → 32.0%) than pregnant women (38.6% → 39.8%)
- The gap between the two groups stays consistently 10-13 percentage points throughout — a persistent, structural difference in anaemia risk during pregnancy that hasn't closed or widened over time.
- Timing coincides loosely with COVID-19-era health service disruption, but the reversal starts a few years earlier, so this is noted as a plausible contributing factor, not a proven cause.

### Chart 2 — Maternal health service coverage (antenatal care, skilled attendance, facility births)
Plotted as markers only (not connected lines), since data points are irregularly spaced (5-13 years apart) and a connected line would misleadingly imply continuous trends between known points.
- **Skilled birth attendance and facility births move almost identically together** at every observed time point — intuitive, since a skilled attendant is typically present specifically because a birth occurred in a facility.
- **Antenatal care coverage follows a distinct, separate trajectory** — it started higher in the 1990s (64%), *declined* independently through the 2000s (down to 47% by 2009), and by 2022 (66%) has fallen behind skilled attendance (89%) and facility births (88%), which caught up and surpassed it.
- **Key insight:** getting mothers into antenatal care and getting them to deliver in a facility with skilled staff appear to be two distinct systemic challenges in Kenya, not one and the same — antenatal care coverage has not kept pace with the sharp gains made in facility delivery and skilled attendance, especially post-2014.

---

## Next steps (Stage 5 onward)
Feature engineering (e.g. rate-of-change columns between survey years), correlation/statistical testing between the service coverage indicators, and a final Power BI dashboard.
