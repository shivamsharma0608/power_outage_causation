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

|   YEAR | U.S._STATE   | CLIMATE.REGION     | CAUSE.CATEGORY     |   OUTAGE.DURATION |   CUSTOMERS.AFFECTED | SEASON   |
|-------:|:-------------|:-------------------|:-------------------|------------------:|---------------------:|:---------|
|   2011 | Minnesota    | East North Central | severe weather     |              3060 |                70000 | Summer   |
|   2014 | Minnesota    | East North Central | intentional attack |                 1 |                  nan | Spring   |
|   2010 | Minnesota    | East North Central | severe weather     |              3000 |                70000 | Fall     |
|   2012 | Minnesota    | East North Central | severe weather     |              2550 |                68200 | Summer   |
|   2015 | Minnesota    | East North Central | severe weather     |              1740 |               250000 | Summer   |

### Univariate Analysis

<iframe src="assets/cause_distribution.html" width="800" height="500" frameborder="0"></iframe>

Severe weather is by far the most common cause of major outages, accounting for 
nearly half of all 1,534 events (763 outages). Intentional attack is the second 
most common with 418 outages, followed by system operability disruption (127) and 
equipment failure (60). This strong class imbalance motivates our use of weighted 
F1-score rather than accuracy when evaluating our classifier.

<iframe src="assets/duration_distribution.html" width="800" height="500" frameborder="0"></iframe>

Outage durations are heavily right-skewed — the majority of outages resolve 
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
This makes intuitive sense — physical infrastructure damage from storms requires 
extensive repair work, while targeted attacks may be easier to isolate and contain.

<iframe src="assets/cause_by_climate.html" width="800" height="500" frameborder="0"></iframe>

The West and East North Central climate regions see the highest absolute counts of 
outages. Severe weather dominates in most regions, but intentional attacks are 
proportionally more prominent in the West and Northwest, suggesting geographic 
patterns in grid vulnerability that our classifier should be able to exploit.

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
fewer customers across all seasons, consistent with their targeted nature. Note 
that fuel supply emergencies and public appeal outages have many missing values 
in certain seasons, reflecting their rarity.

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

We analyzed the missingness of `CUSTOMERS.AFFECTED` (missing in 42.7% of rows) 
against two other columns using permutation tests.

**Depends on: `CAUSE.CATEGORY`**  
We used Total Variation Distance (TVD) as our test statistic since `CAUSE.CATEGORY` 
is categorical. Our observed TVD was **0.7558** with a p-value of **< 0.001**. 
Since p < 0.05, we conclude that the missingness of `CUSTOMERS.AFFECTED` **does** 
depend on `CAUSE.CATEGORY` — intentional attack outages, for instance, are far 
more likely to have missing customer counts than severe weather outages.

**Does not depend on: `ANOMALY.LEVEL`**  
We used the absolute difference in means as our test statistic since 
`ANOMALY.LEVEL` is numeric. Our observed difference was **0.0497** with a 
p-value of **0.18**. Since p > 0.05, we conclude that the missingness of 
`CUSTOMERS.AFFECTED` **does not** depend on `ANOMALY.LEVEL` — the climate 
anomaly index at the time of an outage has no bearing on whether customer 
counts are recorded.

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

**Result:** Our observed difference was **172,219 customers**. After 1,000 
permutations, we obtained a p-value of **< 0.001**. Since p < 0.05, we 
**reject the null hypothesis**. This suggests that severe weather outages 
**do** affect significantly more customers than intentional attack outages on 
average, and this difference is extremely unlikely to be due to random chance. 
This aligns with the targeted vs. broad-impact nature of these two cause types.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

We aim to **predict the cause category (`CAUSE.CATEGORY`) of a power outage** — 
a multiclass classification problem with 7 possible classes: severe weather, 
intentional attack, system operability disruption, public appeal, equipment 
failure, fuel supply emergency, and islanding.

We chose `CAUSE.CATEGORY` as our response variable because identifying the likely 
cause of an outage at the moment it begins allows utility companies and emergency 
responders to allocate resources appropriately and respond faster.

We evaluate our model using **weighted F1-score** rather than accuracy because 
the classes are heavily imbalanced — severe weather accounts for nearly 50% of 
outages, so a naive classifier that always predicts severe weather would achieve 
high accuracy while being useless for minority classes. Weighted F1 accounts for 
this imbalance by weighting each class's F1 score by its support.

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
- Train Accuracy: 0.5683
- Test Accuracy: 0.5921
- Test Weighted F1: 0.4951

We do not consider this baseline model to be particularly good. With only two 
features and a linear classifier, it cannot capture the complex interactions 
between geography, climate, and outage cause. A weighted F1 of 0.4951 indicates 
the model struggles especially with minority classes like fuel supply emergency 
and islanding. However, it establishes a meaningful performance floor to improve upon.

---

## Final Model

We improved upon the baseline by switching to a **Random Forest classifier** 
and engineering four additional features:

- `SEASON` (nominal) — Outage causes vary strongly by season; severe weather 
spikes in summer and winter while equipment failures are more evenly distributed. 
Derived from the `OUTAGE.START` timestamp.
- `NERC.REGION` (nominal) — Different reliability regions have different 
infrastructure ages and regulatory environments, which affects cause likelihood. 
For example, the WECC region has disproportionately more intentional attacks.
- `POPPCT_URBAN` (quantitative) — Urban areas are more likely targets of 
intentional attacks due to higher infrastructure concentration, while rural areas 
tend to see more equipment failures and severe weather impacts.
- `TOTAL.PRICE` (quantitative) — Electricity price reflects economic and 
infrastructure conditions; areas with higher prices may have older infrastructure 
(more equipment failures) or be more susceptible to fuel supply disruptions.

We tuned `n_estimators`, `max_depth`, and `min_samples_split` using 
`GridSearchCV` with 5-fold cross-validation scored on weighted F1.

**Best hyperparameters:** `n_estimators=200`, `max_depth=None`, `min_samples_split=2`

**Performance:**
- Test Weighted F1 (Baseline): 0.4951
- Test Weighted F1 (Final): 0.6420
- Improvement: +0.1469

The Random Forest substantially outperforms logistic regression because it 
naturally captures non-linear interactions between features — for example, 
the combination of NERC region and season is far more predictive of intentional 
attacks than either feature alone. The best model used fully grown trees 
(`max_depth=None`), suggesting the feature interactions benefit from deep splits.

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
- F1 (high urbanization): 0.5704
- F1 (low urbanization): 0.7198
- Observed difference: 0.1494
- p-value: 0.0130

Since p < 0.05, we **reject the null hypothesis**. Our model performs 
significantly better on outages in low-urbanization states (F1 = 0.72) than 
high-urbanization states (F1 = 0.57). This is likely because high-urbanization 
states have a more diverse mix of cause categories — including more intentional 
attacks — making classification harder, while low-urbanization states are more 
dominated by severe weather which the model predicts well. This disparity 
suggests that future work should focus on better features to distinguish cause 
categories in dense urban areas.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>
