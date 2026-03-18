# Behind the Blackout: Uncovering the Causes of Major U.S. Power Outages

**By Shivam Sharma**

---
## Introduction

This project analyzes a dataset of major power outages in the continental U.S.
from January 2000 to July 2016. The dataset contains **1,534 outages** across
all major U.S. climate regions, with information about each outage's cause,
location, duration, customers affected, and associated economic and climate
conditions.

Power outages cost the U.S. economy an estimated $150 billion per year.
Understanding what drives them and being able to predict their cause the
moment they are detected can help utilities respond faster, reduce damage,
and protect critical infrastructure.

The central question guiding this project is: **What characteristics:
location, climate, timing, and economic factors are associated with each
cause category of power outage?**

The relevant columns for the analysis are:

| Column | Description |
|---|---|
| `CAUSE.CATEGORY` | Category of the event causing the outage (the target variable) |
| `CLIMATE.REGION` | U.S. climate region where the outage occurred |
| `NERC.REGION` | North American reliability region of the outage |
| `ANOMALY.LEVEL` | Oceanic El Niño/La Niña index (climate anomaly indicator) |
| `CLIMATE.CATEGORY` | Climate episode category (Warm, Cold, or Normal) |
| `U.S._STATE` | State where the outage occurred |
| `OUTAGE.START` | Date and time the outage began |
| `OUTAGE.DURATION` | Duration of the outage in minutes |
| `CUSTOMERS.AFFECTED` | Number of customers affected by the outage |
| `POPPCT_URBAN` | Percentage of the state's population that is urban |
| `TOTAL.PRICE` | Average electricity price in the state (cents/kWh) |
| `SEASON` | Season the outage occurred (engineered from `OUTAGE.START`) |

---

## Data Cleaning and Exploratory Data Analysis

I performed the following cleaning steps:

- **Merged timestamp columns:** Combined `OUTAGE.START.DATE` and `OUTAGE.START.TIME`
into a single `pd.Timestamp` column called `OUTAGE.START`. Did the same for
`OUTAGE.RESTORATION`. This makes time-based feature engineering straightforward.
- **Replaced 0s with NaN** in `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and
`CUSTOMERS.AFFECTED`, since a value of 0 in these columns indicates missing data
rather than a true zero — a power outage lasting 0 minutes is not a real event.
- **Engineered a `SEASON` column** from the outage start timestamp, since cause
categories like severe weather vary strongly by time of year.
- **Dropped the units row** that appeared as the first row of the raw Excel file.

Here is the head of the cleaned DataFrame:

| YEAR | U.S._STATE | CLIMATE.REGION | CAUSE.CATEGORY | OUTAGE.DURATION | CUSTOMERS.AFFECTED | SEASON |
| --- | --- | --- | --- | --- | --- | --- |
| 2011 | Minnesota | East North Central | severe weather | 3060 | 70000 | Summer |
| 2014 | Minnesota | East North Central | intentional attack | 1 | nan | Spring |
| 2010 | Minnesota | East North Central | severe weather | 3000 | 70000 | Fall |
| 2012 | Minnesota | East North Central | severe weather | 2550 | 68200 | Summer |
| 2015 | Minnesota | East North Central | severe weather | 1740 | 250000 | Summer |

### Univariate Analysis

<iframe src="assets/cause_distribution.html" width="800" height="500" frameborder="0"></iframe>

Severe weather is by far the most common cause of major outages, accounting for
nearly half of all 1,534 events (763 outages). Intentional attack is the second
most common with 418 outages, followed by system operability disruption (127) and
equipment failure (60). This strong class imbalance motivates our use of weighted
F1-score rather than accuracy when evaluating our classifier.

<iframe src="assets/duration_distribution.html" width="800" height="500" frameborder="0"></iframe>

Outage durations are heavily right-skewed as the majority of outages resolve
within a few thousand minutes, but a small number last tens of thousands of
minutes. This suggests extreme outlier events driven by catastrophic causes such
as fuel supply emergencies or major hurricanes.

<iframe src="assets/outage_map.html" width="800" height="500" frameborder="0"></iframe>

Power outages are concentrated in highly populated states like California, Texas,
and Michigan. The West and Northeast show particularly high outage counts,
reflecting both dense population centers and aging grid infrastructure.


### Bivariate Analysis

<iframe src="assets/duration_by_cause.html" width="800" height="500" frameborder="0"></iframe>

Fuel supply emergencies and severe weather tend to have the longest median
durations, while intentional attacks are resolved more quickly on average.
Physical infrastructure damage from storms requires extensive repair work, while
targeted attacks can often be isolated and contained more quickly.

<iframe src="assets/cause_by_climate.html" width="800" height="500" frameborder="0"></iframe>

The West and East North Central climate regions see the highest absolute counts
of outages. Severe weather dominates in most regions, but intentional attacks are
proportionally more prominent in the West and Northwest, suggesting geographic
patterns in grid vulnerability that our classifier can exploit.

<iframe src="assets/anomaly_by_cause.html" width="800" height="500" frameborder="0"></iframe>

Severe weather outages tend to occur at more extreme anomaly levels (both positive
and negative), suggesting association with El Niño and La Niña climate episodes.
Intentional attacks show a tighter near-zero distribution, confirming they are
not climate-dependent, supporting the use of `ANOMALY.LEVEL` as a distinguishing
feature.

### Interesting Aggregates

The table below shows the **median number of customers affected**, broken down by
cause category and season:

| CAUSE.CATEGORY                |   Fall |   Spring |   Summer |   Winter |
|:------------------------------|-------:|---------:|---------:|---------:|
| equipment failure             | 900000 |    80915 |    45452 |    52000 |
| fuel supply emergency         |    nan |      nan |      nan |        1 |
| intentional attack            |   9200 |     5852 |     1100 |     2500 |
| islanding                     |   7077 |     9700 |      606 |     6635 |
| public appeal                 |    nan |      nan |     8000 |    18600 |
| severe weather                | 118000 |   102568 |   109000 |   118000 |
| system operability disruption | 104000 |    82500 |    33500 |    51982 |

Severe weather outages consistently affect over 100,000 customers regardless of
season, reflecting their broad geographic impact. Intentional attacks affect far
fewer customers across all seasons, consistent with their targeted nature.

---

## Assessment of Missingness

### MNAR Analysis

The `CAUSE.CATEGORY.DETAIL` column is likely **MNAR** (Missing Not at Random).
When broken down by cause category, `islanding` and `public appeal` events have
a **100% missingness rate** in this column, while `intentional attack` is missing
only 11.5% of the time. This pattern cannot be explained by any other observed
variable, the missingness is directly related to the value of the cause itself.
Operators likely omit details for islanding and public appeal events because no
specific incident detail applies to those cause types. To make this column MAR,
we would want access to NERC incident report filings that would reveal whether
the absence of detail was a deliberate omission or simply not applicable.

### Missingness Dependency

I analyzed the missingness of `CUSTOMERS.AFFECTED`, which is missing in
**655 of 1,534 rows (42.7%)**. The missingness rate varies dramatically by cause:
intentional attacks are missing 95.5% of the time, while severe weather outages
are missing only 7.2% of the time.

**Depends on: `CAUSE.CATEGORY`**
I used TVD as the test statistic since `CAUSE.CATEGORY` is categorical. The
observed TVD was **0.7558** with a p-value of **< 0.001** across 500 permutations.
Since p < 0.05, the missingness of `CUSTOMERS.AFFECTED` **does** depend on
`CAUSE.CATEGORY` as utilities likely do not report customer counts for intentional
attacks to avoid revealing the scale of grid vulnerabilities.

**Does not depend on: `ANOMALY.LEVEL`**
I used absolute difference in means since `ANOMALY.LEVEL` is numeric. The
observed difference was **0.0497** with a p-value of **0.198**. Since p > 0.05,
the missingness of `CUSTOMERS.AFFECTED` **does not** depend on the climate
anomaly level, whether it is an El Niño or La Niña year has no bearing on
whether customer counts get recorded.

<iframe src="assets/missingness_permtest.html" width="800" height="500" frameborder="0"></iframe>

<iframe src="assets/missingness_distribution.html" width="800" height="500" frameborder="0"></iframe>

The distribution of `ANOMALY.LEVEL` is nearly identical whether `CUSTOMERS.AFFECTED`
is missing or not, visually confirming that these two variables are independent.

---

## Hypothesis Testing

**Question:** Do severe weather outages last significantly longer than intentional
attack outages?

**Null Hypothesis:** The distribution of outage durations is the same for severe
weather and intentional attack outages. Any observed difference in median duration
is due to random chance.

**Alternative Hypothesis:** Severe weather outages have longer median durations
than intentional attack outages.

**Test Statistic:** Difference in group medians (severe weather minus intentional
attack). I decided to use median rather than mean because `OUTAGE.DURATION` is heavily
right-skewed with extreme outliers, making median a more robust measure of center.
A directional test is appropriate because our alternative hypothesis specifically
predicts one group is longer.

**Significance Level:** 0.05

**Result:** The severe weather median duration was **2,464 minutes** vs.
**92 minutes** for intentional attacks, an observed difference of **2,372 minutes**.
After 1,000 permutations, I obtained a p-value of **< 0.001**. Since p < 0.05,
we reject the null hypothesis. This suggests that severe weather outages tend to
last substantially longer than intentional attack outages, though we cannot
conclude this is a causal relationship.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

I aim to **predict the cause category (`CAUSE.CATEGORY`) of a power outage** —
a **multiclass classification** problem with 7 possible classes: severe weather,
intentional attack, system operability disruption, public appeal, equipment
failure, fuel supply emergency, and islanding.

I chose `CAUSE.CATEGORY` as our response variable because identifying the likely
cause at the moment an outage begins allows utility companies to dispatch the
right response teams immediately: field crews for weather damage, law enforcement
for attacks, engineers for equipment issues.

I will evaluate using **weighted F1-score** rather than accuracy because the classes
are heavily imbalanced. A naive classifier always predicting "severe weather"
would achieve ~50% accuracy while failing on every other class. Weighted F1
computes F1 per class and averages by support, rewarding good performance
proportionally to how often each class actually occurs.

All features are limited to information available **at the moment an outage
begins**: static geographic properties (`CLIMATE.REGION`, `NERC.REGION`),
published climate data (`ANOMALY.LEVEL`), the outage start timestamp (`SEASON`),
and state-level demographic and economic data (`POPPCT_URBAN`, `TOTAL.PRICE`).
We explicitly exclude `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and
`CUSTOMERS.AFFECTED` — all only known after the outage ends.

---

## Baseline Model

The baseline model is a **Logistic Regression classifier** implemented in a
single `sklearn` Pipeline. It uses two features:

- `CLIMATE.REGION` — **nominal**, one-hot encoded with `OneHotEncoder` into 9
binary indicator columns. One-hot encoding is appropriate because climate regions
have no natural ordering.
- `ANOMALY.LEVEL` — **quantitative**, standardized with `StandardScaler` to zero
mean and unit variance. Standardization helps Logistic Regression converge and
ensures equal feature scaling.

**Performance:**

| Split | Weighted F1 | Accuracy |
|---|---|---|
| Train | 0.4763 | 0.5683 |
| Test | 0.4861 | 0.5921 |

This is not considered a good model. On the test set it only predicts two of
seven classes ("severe weather" and "intentional attack"), completely failing on
five minority classes. With only two features and a linear classifier it cannot
capture the complex interactions between geography, climate, and outage cause.
It does beat the majority class baseline F1 of 0.3297, confirming that climate
region carries some predictive signal, and establishes a clear performance floor
to improve upon.

---

## Final Model

I improved upon the baseline by switching to a **Random Forest classifier**
and adding four features:

- `SEASON` (nominal, OHE) — Outage causes vary strongly by season; severe weather
spikes in summer and winter while equipment failures are more uniform. Derived
from the `OUTAGE.START` timestamp.
- `NERC.REGION` (nominal, OHE) — The WECC region has disproportionately more
intentional attacks than other regions — a pattern `CLIMATE.REGION` alone cannot
capture since multiple NERC regions overlap each climate region.
- `POPPCT_URBAN` (quantitative, StandardScaler) — Urban areas concentrate
infrastructure, making them more likely targets of intentional attacks. Rural
areas have more exposed lines prone to weather damage.
- `TOTAL.PRICE` (quantitative, StandardScaler) — Electricity price reflects
regional infrastructure maturity and fuel dependency, both of which correlate
with cause type.

I tuned `n_estimators`, `max_depth`, and `min_samples_split` using `GridSearchCV`
with 5-fold cross-validation scored on weighted F1. Both models were evaluated on
the **exact same train/test split** for a valid comparison.

**Best hyperparameters:** `n_estimators=200`, `max_depth=None`, `min_samples_split=2`

| Model | Train F1 | Test F1 | Improvement |
|---|---|---|---|
| Baseline (Logistic Regression) | 0.4763 | 0.4861 | — |
| Final (Random Forest) | 0.9211 | 0.6420 | +0.1559 |

The final model predicts all 7 classes (vs. only 2 for the baseline), with
strong performance on severe weather (F1=0.79) and intentional attack (F1=0.73).
The train-test gap of 0.28 indicates some overfitting from fully grown trees,
but the test F1 improvement of +0.156 over the baseline confirms meaningful
generalization beyond training data.

<iframe src="assets/confusion_matrix.html" width="800" height="600" frameborder="0"></iframe>

---

## Fairness Analysis

**Group X:** High urbanization states (`POPPCT_URBAN` ≥ 84.05%, n=159)
**Group Y:** Low urbanization states (`POPPCT_URBAN` < 84.05%, n=143)
**Metric:** Weighted F1-score

**Null Hypothesis:** The model is fair. Its weighted F1 for high-urbanization
and low-urbanization states are roughly the same, and any differences are due
to random chance.

**Alternative Hypothesis:** The model is unfair. Its weighted F1 differs
between high-urbanization and low-urbanization states.

**Test Statistic:** Absolute difference in weighted F1 scores
**Significance Level:** 0.05

**Results:**

| Group | Weighted F1 |
|---|---|
| High urbanization | 0.5704 |
| Low urbanization | 0.7198 |
| Observed difference | 0.1494 |
| p-value | 0.016 |

Since p = 0.016 < 0.05, we reject the null hypothesis. The evidence suggests
our model tends to perform worse on outages in highly urbanized states. This
disparity likely reflects the fact that high-urbanization states have a more
diverse and harder-to-classify mix of cause categories — including more
intentional attacks and system operability disruptions — while low-urbanization
states are more dominated by severe weather, which our model predicts well.
We cannot conclude this difference is causal, but it points to a meaningful
limitation for urban utility operators.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Conclusion

This project explored what characteristics are associated with each cause category 
of major U.S. power outages, and built a machine learning model to predict cause 
at the moment an outage is detected.

The exploratory analysis revealed clear patterns: severe weather dominates outage 
causes across all regions and seasons, intentional attacks are disproportionately 
concentrated in the West, and outage duration varies dramatically by cause, with 
fuel supply emergencies and severe weather lasting far longer than targeted attacks.

The final Random Forest classifier achieved a weighted F1 of 0.642, improving 
substantially over the logistic regression baseline (F1 = 0.486). The model 
performs well on the two most common classes, severe weather and intentional 
attack, but struggles with rare classes like fuel supply emergency and islanding, 
where more data and additional features would be needed.

The fairness analysis found that the model performs significantly better for 
low-urbanization states than high-urbanization states, pointing to a meaningful 
limitation for urban utility operators and an avenue for future improvement.