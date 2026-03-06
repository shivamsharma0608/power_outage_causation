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