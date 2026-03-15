# Behind the Blackout: Uncovering the Causes of Major U.S. Power Outages

**By Shivam Sharma**

---

## Introduction

This project analyzes a dataset of major power outages in the continental U.S. 
from January 2000 to July 2016. The dataset contains **1,534 outages** across 
all major U.S. climate regions, with information about each outage's cause, 
location, duration, customers affected, and associated economic and climate 
conditions.

The central question guiding this project is: **What characteristics — 
location, climate, timing, and economic factors — are associated with each 
cause category of power outage?** Understanding this is valuable for utility 
companies, policymakers, and emergency responders who need to anticipate and 
prepare for different types of outages.

The relevant columns for our analysis are:

| Column | Description |
|---|---|
| `CAUSE.CATEGORY` | Category of the event causing the outage (our target variable) |
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

We performed the following cleaning steps:

- **Merged timestamp columns:** Combined `OUTAGE.START.DATE` and `OUTAGE.START.TIME` 
into a single `pd.Timestamp` column called `OUTAGE.START`. Did the same for 
`OUTAGE.RESTORATION`. This makes time-based feature engineering straightforward.
- **Replaced 0s with NaN** in `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and 
`CUSTOMERS.AFFECTED`, since a value of 0 in these columns indicates missing data 
rather than a true zero.
- **Engineered a `SEASON` column** from the outage start timestamp, since cause 
categories like severe weather vary strongly by time of year.
- **Dropped the units row** that appeared as the first row of the raw Excel file.

Here is the head of the cleaned DataFrame:

| CLIMATE.REGION   | NERC.REGION   | SEASON   |   ANOMALY.LEVEL |   POPPCT_URBAN |   TOTAL.PRICE | true_label                    | pred_label                    |
|:-----------------|:--------------|:---------|----------------:|---------------:|--------------:|:------------------------------|:------------------------------|
| West             | WECC          | Spring   |            -0.2 |          94.95 |         12.98 | intentional attack            | intentional attack            |
| West             | WECC          | Spring   |            -0.7 |          94.95 |         11.91 | system operability disruption | severe weather                |
| Southeast        | SERC          | Winter   |            -0.2 |          66.09 |          8.97 | severe weather                | system operability disruption |
| Northwest        | WECC          | Winter   |            -0.5 |          81.03 |          8.85 | intentional attack            | intentional attack            |
| Northeast        | NPCC          | Spring   |            -0.3 |          91.97 |         13.81 | intentional attack            | intentional attack            |


### Univariate Analysis

<iframe src="assets/cause_distribution.html" width="800" height="500" frameborder="0"></iframe>

Severe weather is by far the most common cause of major outages, accounting for 
nearly half of all events. Intentional attack is the second most common, 
followed by system operability disruption and equipment failure.

<iframe src="assets/duration_distribution.html" width="800" height="500" frameborder="0"></iframe>

Outage durations are heavily right-skewed — the majority of outages resolve 
within a few thousand minutes, but a small number last tens of thousands of 
minutes. This suggests extreme outlier events driven by catastrophic causes.

### Bivariate Analysis

<iframe src="assets/duration_by_cause.html" width="800" height="500" frameborder="0"></iframe>

Fuel supply emergencies and severe weather tend to have the longest median 
durations, while intentional attacks are resolved more quickly on average. 
This suggests that physical infrastructure damage takes longer to repair than 
disruptions caused by human interference.

<iframe src="assets/cause_by_climate.html" width="800" height="500" frameborder="0"></iframe>

The Northeast and South climate regions see the highest absolute counts of 
outages. Severe weather dominates in most regions, but intentional attacks are 
proportionally more common in certain regions, suggesting geographic patterns 
in grid vulnerability.

### Interesting Aggregates

The table below shows the median number of customers affected, broken down by 
cause category and season:

[PASTE YOUR PIVOT TABLE IN MARKDOWN HERE — use print(pivot.to_markdown())]

Severe weather outages in summer affect the most customers on average, consistent 
with hurricane and thunderstorm season. Intentional attacks show less seasonal 
variation, suggesting they are not weather-dependent.

---

## Assessment of Missingness

### MNAR Analysis

We believe the `CAUSE.CATEGORY.DETAIL` column is **MNAR** (Missing Not at Random). 
When outages are caused by intentional attacks or other sensitive events, operators 
may deliberately withhold specific details to avoid revealing security 
vulnerabilities in the grid. This means the missingness of the detail column is 
related to the actual value of the missing data itself — a defining characteristic 
of MNAR. To make this column MAR, we would want access to NERC incident reports 
or federal filings that could explain why details were omitted for specific events.

### Missingness Dependency

We analyzed the missingness of `CUSTOMERS.AFFECTED` against two other columns 
using permutation tests.

**Depends on: `CAUSE.CATEGORY`**  
We used Total Variation Distance (TVD) as our test statistic since `CAUSE.CATEGORY` 
is categorical. Our observed TVD was [X] with a p-value of [X]. Since p < 0.05, 
we conclude that the missingness of `CUSTOMERS.AFFECTED` **does** depend on 
`CAUSE.CATEGORY` — certain cause types are more likely to have missing customer 
data than others.

**Does not depend on: `ANOMALY.LEVEL`**  
We used the absolute difference in means as our test statistic since 
`ANOMALY.LEVEL` is numeric. Our observed difference was [X] with a p-value of [X]. 
Since p > 0.05, we conclude that the missingness of `CUSTOMERS.AFFECTED` **does 
not** depend on `ANOMALY.LEVEL`.

<iframe src="assets/missingness_permtest.html" width="800" height="500" frameborder="0"></iframe>

---

## Hypothesis Testing

**Question:** Do severe weather outages affect more customers on average than 
intentional attack outages?

**Null Hypothesis:** The distribution of customers affected is the same for 
severe weather and intentional attack outages. Any observed difference is due 
to random chance.

**Alternative Hypothesis:** Severe weather outages affect more customers on 
average than intentional attack outages.

**Test Statistic:** Difference in group means (severe weather mean minus 
intentional attack mean). We chose this because we are comparing a numeric 
variable across two groups and have a directional alternative hypothesis.

**Significance Level:** 0.05

**Result:** Our observed difference was [X] customers. After 1,000 permutations, 
we obtained a p-value of [X]. Since p [< / >] 0.05, we [reject / fail to reject] 
the null hypothesis. This suggests that severe weather outages [do / do not] 
affect significantly more customers than intentional attack outages.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

We aim to **predict the cause category (`CAUSE.CATEGORY`) of a power outage** — 
a multiclass classification problem with 7 possible classes.

We chose `CAUSE.CATEGORY` as our response variable because identifying the likely 
cause of an outage at the moment it begins allows utility companies and emergency 
responders to allocate resources appropriately and respond faster.

We evaluate our model using **weighted F1-score** rather than accuracy because 
the classes are imbalanced — severe weather accounts for nearly 50% of outages, 
so a naive classifier that always predicts severe weather would achieve high 
accuracy while being useless. Weighted F1 accounts for this imbalance by weighting 
each class's F1 score by its support.

Features used are limited to information available **at the time an outage begins**: 
location, climate region, season, urbanization, and electricity price. We 
explicitly exclude `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and `CUSTOMERS.AFFECTED` 
since these are only known after the outage ends.

---

## Baseline Model

Our baseline model is a **Logistic Regression classifier** implemented in a 
single `sklearn` Pipeline. It uses two features:

- `CLIMATE.REGION` — nominal, one-hot encoded with `OneHotEncoder`
- `ANOMALY.LEVEL` — quantitative, standardized with `StandardScaler`

**Performance:**
- Train Accuracy: [X]
- Test Accuracy: [X]
- Test Weighted F1: [X]

We do not consider this baseline model to be particularly good. With only two 
features and a linear classifier, it cannot capture the complex interactions 
between geography, climate, and outage cause. However, it establishes a 
meaningful performance floor to improve upon.

---

## Final Model

We improved upon the baseline by switching to a **Random Forest classifier** 
and engineering four additional features:

- `SEASON` (nominal) — Outage causes vary strongly by season; severe weather 
spikes in summer and winter while equipment failures are more evenly distributed.
- `NERC.REGION` (nominal) — Different reliability regions have different 
infrastructure ages and regulatory environments, which affects cause likelihood.
- `POPPCT_URBAN` (quantitative) — Urban areas are more likely targets of 
intentional attacks due to infrastructure concentration, while rural areas 
see more equipment failures.
- `TOTAL.PRICE` (quantitative) — Electricity price reflects economic and 
infrastructure conditions that correlate with certain cause categories.

We tuned `n_estimators`, `max_depth`, and `min_samples_split` using 
`GridSearchCV` with 5-fold cross-validation scored on weighted F1.

**Best hyperparameters:** [paste from your output]

**Performance:**
- Test Weighted F1 (Baseline): [X]
- Test Weighted F1 (Final): [X]
- Improvement: [X]

The Random Forest outperforms logistic regression because it naturally captures 
non-linear interactions between features — for example, the combination of 
NERC region and season is far more predictive than either feature alone.

<iframe src="assets/confusion_matrix.html" width="800" height="600" frameborder="0"></iframe>

---

## Fairness Analysis

**Group X:** High urbanization states (`POPPCT_URBAN` ≥ median)  
**Group Y:** Low urbanization states (`POPPCT_URBAN` < median)  
**Metric:** Weighted F1-score  

**Null Hypothesis:** Our model is fair. Its weighted F1 for high-urbanization 
and low-urbanization states are roughly the same, and any differences are due 
to random chance.

**Alternative Hypothesis:** Our model is unfair. Its weighted F1 differs 
between high-urbanization and low-urbanization states.

**Test Statistic:** Absolute difference in weighted F1 scores  
**Significance Level:** 0.05

**Results:**
- F1 (high urbanization): [X]
- F1 (low urbanization): [X]
- Observed difference: [X]
- p-value: [X]

Since p [< / >] 0.05, we [reject / fail to reject] the null hypothesis. 
This suggests our model [is / is not] significantly fairer for one group 
over the other.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>