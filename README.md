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


Power outages cost the U.S. economy an estimated $150 billion per year. 
Understanding what drives them — and being able to predict their cause the 
moment they are detected — can help utilities respond faster, reduce damage, 
and protect critical infrastructure.
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

<iframe src="assets/anomaly_by_cause.html" width="800" height="500" frameborder="0"></iframe>

Severe weather outages tend to occur at more extreme anomaly levels (both positive 
and negative), suggesting they are associated with El Niño and La Niña climate 
episodes. Intentional attacks show a tighter, near-zero distribution, confirming 
they are not climate-dependent — a finding that directly supports our classifier 
using `ANOMALY.LEVEL` as a feature to distinguish these cause types.

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

The `CAUSE.CATEGORY.DETAIL` column is likely **MNAR** (Missing Not at Random). 
When broken down by cause category, `islanding` and `public appeal` events have 
a **100% missingness rate** in this column, while `intentional attack` is missing 
only 11.5% of the time. This pattern cannot be explained by any other observed 
variable in the dataset — the missingness is directly related to the value of 
the cause itself, which is the defining characteristic of MNAR. Operators likely 
omit details for islanding and public appeal events because no specific incident 
detail applies to those cause types. To make this column MAR, we would want 
access to NERC incident report filings, which would reveal whether the absence 
of detail was a deliberate omission or simply not applicable.

### Missingness Dependency

We analyze the missingness of `CUSTOMERS.AFFECTED`, which is missing in 
**655 of 1,534 rows (42.7%)**. The missingness rate varies dramatically by cause: 
intentional attacks are missing 95.5% of the time, while severe weather outages 
are missing only 7.2% of the time — a difference that is unlikely to be random.

**Depends on: `CAUSE.CATEGORY`**  
We used TVD as our test statistic since `CAUSE.CATEGORY` is categorical. Our 
observed TVD was **0.7558** with a p-value of **< 0.001** across 500 permutations. 
Since p < 0.05, the missingness of `CUSTOMERS.AFFECTED` **does** depend on 
`CAUSE.CATEGORY`. This makes sense from the data generating process: utilities 
may not report customer counts for intentional attacks to avoid revealing the 
scale of grid vulnerabilities.

**Does not depend on: `ANOMALY.LEVEL`**  
We used absolute difference in means since `ANOMALY.LEVEL` is numeric. Our 
observed difference was **0.0497** with a p-value of **0.198**. Since p > 0.05, 
the missingness of `CUSTOMERS.AFFECTED` **does not** depend on the climate 
anomaly level at the time of the outage — whether it is an El Niño or La Niña 
year has no bearing on whether customer counts get recorded.

<iframe src="assets/missingness_permtest.html" width="800" height="500" frameborder="0"></iframe>

<iframe src="assets/missingness_distribution.html" width="800" height="500" frameborder="0"></iframe>

The distribution of `ANOMALY.LEVEL` is nearly identical whether `CUSTOMERS.AFFECTED` 
is missing or not, visually confirming our permutation test result that these 
two variables are independent.

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
attack). We chose median over mean because `OUTAGE.DURATION` is heavily 
right-skewed with extreme outliers, making median a more robust measure of center. 
A directional test is appropriate because our alternative hypothesis specifically 
predicts one group is longer, not merely different.

**Significance Level:** 0.05 — the standard threshold that balances false 
positive risk against the cost of missing a real effect.

**Result:** The severe weather median duration was **2,464 minutes** vs. 
**92 minutes** for intentional attacks — an observed difference of **2,372 minutes**. 
After 1,000 permutations, we obtained a p-value of **< 0.001**. Since p < 0.05, 
we reject the null hypothesis. This suggests that severe weather outages tend to 
last substantially longer than intentional attack outages, though we cannot 
conclude this is a causal relationship. The result is consistent with our 
intuition: physical infrastructure damage from storms requires extensive field 
repairs, while targeted attacks can often be isolated and resolved more quickly.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

We aim to **predict the cause category (`CAUSE.CATEGORY`) of a power outage** — 
a **multiclass classification** problem with 7 possible classes: severe weather, 
intentional attack, system operability disruption, public appeal, equipment 
failure, fuel supply emergency, and islanding.

We chose `CAUSE.CATEGORY` as our response variable because it is the core of 
our EDA question — understanding what characteristics are associated with each 
cause. A classifier that can predict cause at the moment an outage is detected 
allows utility companies to dispatch the right response team immediately.

We evaluate using **weighted F1-score** over other metrics for the following 
reasons. Accuracy is misleading here because the classes are heavily imbalanced 
— a naive classifier that always predicts "severe weather" would achieve ~50% 
accuracy while failing on every other class. Macro F1 treats all 7 classes 
equally regardless of how frequently they appear, which over-weights rare classes 
like islanding (46 examples) relative to severe weather (763 examples). Weighted 
F1 computes F1 per class and averages by support, rewarding good performance 
proportionally to how often each class actually occurs.

All features used are limited to information available **at the moment an outage 
begins**: static geographic properties (`CLIMATE.REGION`, `NERC.REGION`), 
published climate data (`ANOMALY.LEVEL`), the outage start timestamp (`SEASON`), 
and state-level demographic and economic data (`POPPCT_URBAN`, `TOTAL.PRICE`). 
We explicitly exclude `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and 
`CUSTOMERS.AFFECTED` — all of which are only known after the outage ends.
---

## Baseline Model

Our baseline model is a **Logistic Regression classifier** implemented in a 
single `sklearn` Pipeline. It uses two features:

- `CLIMATE.REGION` — **nominal**, one-hot encoded with `OneHotEncoder` into 9 
binary indicator columns (one per climate region). One-hot encoding is appropriate 
here because climate regions have no natural ordering.
- `ANOMALY.LEVEL` — **quantitative**, standardized with `StandardScaler` to zero 
mean and unit variance. Standardization helps Logistic Regression converge and 
ensures this feature is on the same scale as the encoded indicators.

**Performance:**

| Split | Weighted F1 | Accuracy |
|---|---|---|
| Train | 0.4761 | 0.5683 |
| Test | 0.4951 | 0.5921 |

The train and test F1 scores are nearly identical (gap of -0.019), indicating 
the model generalizes well — but it is simply underfitting rather than overfitting. 
We do not consider this a good model. On the test set, it only predicts two of 
the seven possible classes ("severe weather" and "intentional attack"), completely 
failing on five minority classes. With only two features and a linear classifier, 
it lacks the complexity to distinguish cause categories that overlap in 
geographic and climate space. It does, however, meaningfully beat the majority 
class baseline F1 of 0.3297, confirming that climate region carries some 
predictive signal.

---

## Final Model

We improved upon the baseline by switching to a **Random Forest classifier** 
and adding four features to the baseline's two:

- `SEASON` (nominal, OHE) — Outage causes vary strongly by season; severe 
weather spikes in summer and winter, while equipment failures are more uniform. 
Derived from `OUTAGE.START` timestamp.
- `NERC.REGION` (nominal, OHE) — The WECC (Western) region has disproportionately 
more intentional attacks than other regions, a pattern `CLIMATE.REGION` alone 
cannot capture since multiple NERC regions overlap each climate region.
- `POPPCT_URBAN` (quantitative, StandardScaler) — Urban areas concentrate 
infrastructure, making them more likely targets of intentional attacks. Rural 
areas have more exposed lines prone to weather damage.
- `TOTAL.PRICE` (quantitative, StandardScaler) — Electricity price reflects 
regional infrastructure maturity and fuel dependency, both of which correlate 
with cause type.

We tuned `n_estimators`, `max_depth`, and `min_samples_split` using `GridSearchCV` 
with 5-fold cross-validation scored on weighted F1. Both baseline and final 
models were evaluated on the **exact same train/test split** for a valid comparison.

**Best hyperparameters:** `n_estimators=200`, `max_depth=None`, `min_samples_split=2`

| Model | Train F1 | Test F1 | Improvement |
|---|---|---|---|
| Baseline (Logistic Regression) | 0.4763 | 0.4861 | — |
| Final (Random Forest) | 0.9211 | 0.6420 | +0.1559 |

The final model predicts all 7 classes (vs. only 2 for the baseline), with 
strong performance on severe weather (F1=0.79) and intentional attack (F1=0.73). 
The train-test gap of 0.28 indicates some overfitting from fully grown trees, 
but the test F1 improvement of +0.156 over the baseline confirms meaningful 
generalization.

<iframe src="assets/confusion_matrix.html" width="800" height="600" frameborder="0"></iframe>

---

## Fairness Analysis

**Group X:** High urbanization states (`POPPCT_URBAN` ≥ 84.05%, n=159)  
**Group Y:** Low urbanization states (`POPPCT_URBAN` < 84.05%, n=143)  
**Metric:** Weighted F1-score

**Null Hypothesis:** Our model is fair. Its weighted F1 for high-urbanization 
and low-urbanization states are roughly the same, and any differences are due 
to random chance.

**Alternative Hypothesis:** Our model is unfair. Its weighted F1 differs 
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
limitation of the model for urban utility operators.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>