# 🦠 Who Gets the Vaccine? — Hackathon Dataset C

**Predicting who will accept a COVID vaccine, and using those predictions to keep a town's total bill as low as possible.**

7PAM2002 Hackathon · University of Hertfordshire · Group Q

---

## Table of Contents

- [Overview](#overview)
- [The Problem](#the-problem)
- [Our Approach](#our-approach)
- [Key Results](#key-results)
- [Final Submissions](#final-submissions)
- [What Drives the Decisions](#what-drives-the-decisions)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Notebook Contents](#notebook-contents)
- [Limitations and Next Steps](#limitations-and-next-steps)
- [Team](#team)

---

## Overview

Winter parties are about to infect a whole town. We can post a vaccine to any resident, but a dose sent to someone who refuses it is wasted. Our task is **not** to guess who says "yes" as accurately as possible. It is to decide **who to post a vaccine to so the town's total cost is minimised**.

Accuracy and F1 are therefore the wrong targets. We derive a task-specific cost metric and build decisions directly around it.

## The Problem

Every resident falls into one of the following cost cases:

| Event | Cost |
|---|---|
| A vaccine is posted (and refused, so the dose is wasted) | **$1,500** |
| A willing person is missed (the town pays them cash) | **$1,500** |
| A hospital stay (unvaccinated people are far more likely to need one) | **$41,000** |

- **Data:** 4,756 surveyed residents for training (about 1 in 3 said yes) and 4,749 residents in the test town.
- **Target:** `covid_vaccine` (1 = accepts a vaccine).
- **Goal:** for each test resident, output *send / don't send* so that expected total cost is as low as possible.

## Our Approach

### 1. Estimate hospitalisation risk (from research)

We combine published estimates into a per-person hospital-risk score:

| Factor | Effect |
|---|---|
| Age | 2% (18–34) up to 18% (65+) chance of hospitalisation if infected and unvaccinated (CDC estimates) |
| Chronic conditions | ×2 to ×4 for one, two, or three or more conditions |
| Sex | ×1.33 for men (meta-analysis) |
| Vaccine effectiveness | 85% fewer hospital stays when vaccinated |

Because different sources and AI assistants give different numbers, every result is **re-scored under five sets of assumptions** as a robustness check.

### 2. Predict acceptance

We prepared the data four different ways and trained six model families:

- Logistic Regression
- Random Forest
- XGBoost
- LightGBM
- CatBoost
- Ensemble of four models

Hyper-parameters were tuned automatically and judged with 5-fold cross-validation on 80% of the data, keeping 20% as a hold-out set. We tuned for **well-calibrated probabilities**, because the decision rule compares them with a threshold.

### 3. Turn probabilities into decisions

Every resident gets a personal "bar" to clear:

```
bar = 1,500 / (1,500 + 41,000 × hospital_risk × 0.85)
```

**Send a vaccine if the person's chance of saying yes is above their bar.**

- High hospital risk → low bar (a used dose can save $41,000).
- Low risk → high bar (only send if they will probably use it).

| Resident | Bar |
|---|---|
| Aged 18–34, no health problems | 73% |
| Aged 45–54, one health problem | 38% |
| Aged 65+, three or more health problems | 9% |

A single 50% cut-off for everyone is never the cheapest choice.

## Key Results

**Data preparation.** Treating a blank answer as its own category (instead of filling it with the most common answer) saved about **$45 per resident** (around $200,000 for the test town) and improved ROC-AUC from 0.835 to 0.86. People who skip questions are much less likely to accept (for example, 43% acceptance among those who answered the health-insurance question vs 17% among those who skipped it).

**Model choice barely matters.** All tuned models were within about **$10 per resident** of each other. The survey data, not the algorithm, limits accuracy.

**Decision rule matters a lot.** Same predictions, three ways of deciding (average cost per resident, lower is better):

| Decision rule | Cost per resident |
|---|---|
| One bar at 50% (the usual default) | $3,149 |
| Best single bar (34%) | $3,118 |
| **Personal bars** | **$3,040** |

That is about **$110 saved per resident** versus the 50% default, roughly half a million dollars for the test town. This is bigger than any difference between models.

**Unseen hold-out performance (cost per resident, lower is better):**

| Strategy | Cost |
|---|---|
| Send to nobody | $3,944 |
| Send to everybody | $3,855 |
| **Our model (XGBoost + personal bars)** | **$3,207** |
| Perfect knowledge (impossible) | $2,845 |

We capture **67% of the possible saving**, and personal bars were cheapest in **4 of 5** hospital-risk scenarios.

**Expected cost for the test town:** $14.93M with our top submission, vs $18.53M if no vaccines were sent (about **$3.6M saved**).

## Final Submissions

Five ranked submission files are generated, best first:

| # | Model | Decision rule | Why it is here |
|---|---|---|---|
| 1 | XGBoost | Personal bars | Best combined score (cross-validation + hold-out) |
| 2 | LightGBM | Personal bars | Effectively tied with #1 |
| 3 | Ensemble of four | Personal bars | Lowest cross-validated cost |
| 4 | Logistic Regression | Personal bars | Best on hold-out, most robust, easy to explain |
| 5 | Logistic Regression | One bar (34%) | Safety net if markers use one flat hospital risk |

> **Caveat:** submissions 1–4 differ by only about $5 per resident, well inside the noise, so their order could swap on the test town.

Selection rule (fixed before looking at the test set): combined score = 0.8 × repeated-CV cost + 0.2 × hold-out cost. Slots 1–4 are the best personal-bar candidates, and slot 5 is the best single-bar candidate as a hedge.

## What Drives the Decisions

Three explanation methods (SHAP, logistic-regression odds ratios, permutation importance) agree on the top four drivers:

1. **Doctor's recommendation** (70% accept when recommended vs 22% when not; scrambling it would cost the town about $85–105 per resident)
2. **Health insurance** (mostly whether people answered the question at all)
3. **How risky COVID feels**
4. **Belief that the vaccine works**

Race, sex, and household size matter much less.

**Same acceptance chance, different decision.** Two residents with roughly a 33% chance of saying yes:

- Older resident, 31.5% hospital risk → bar is 12% → **send a vaccine**
- Young resident, 1.4% hospital risk → bar is 76% → **don't send**

## Repository Structure

```
.
├── Hackathon_DatasetC_Covid_Vaccine.ipynb   # Full analysis, modelling and submissions
├── data/                                    # Place the 3 CSV files here (not tracked)
│   ├── dataset_C_training.csv
│   ├── dataset_C_testing.csv
│   └── dataset_C_example_submission.csv
├── figures/                                 # Auto-generated plots (PNG)
├── submissions/                             # Auto-generated: five ranked prediction files
├── Who_Gets_the_Vaccine___Hackathon_Dataset_C_-_Group_Q.pdf   # Presentation slides
└── README.md
```

`figures/` and `submissions/` are created automatically when the notebook runs.

## Getting Started

### Option A: Google Colab (easiest)

1. Open the notebook in Colab: **[add Colab link here]**
2. Run all cells. If `./data` is empty, you will be prompted to upload the three CSV files.

### Option B: Run locally

```bash
# 1. Clone the repository
git clone https://github.com/Group-Q-Hackathon-Research-Methods/Group-Q---Hacktathon.git
cd Group-Q---Hacktathon

# 2. (Optional) create a virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install numpy pandas matplotlib seaborn scipy scikit-learn xgboost lightgbm catboost shap jupyter

# 4. Add the data files to ./data (see Repository Structure)

# 5. Launch the notebook
jupyter notebook Hackathon_DatasetC_Covid_Vaccine.ipynb
```

The first notebook cell also installs any missing library automatically (for example CatBoost on Colab). All results are seeded (`RANDOM_STATE = 42`) for reproducibility.

> **Note:** hyper-parameter search with repeated cross-validation is the slowest part of the notebook. Allow extra time on a laptop or use Colab.

## Notebook Contents

| Section | Content |
|---|---|
| 1 | Set-up (works locally and in Google Colab) |
| 2 | Understanding the task and deriving the task-specific cost metric, with hospitalisation-risk research |
| 3 | Exploratory data analysis: data quality, missingness, distributions, associations, train/test shift |
| 4 | Pre-processing: cleaning, feature engineering, and a comparison of four pre-processing options |
| 5 | Modelling: six model families plus an ensemble, hyper-parameter tuning, repeated cross-validation |
| 6 | Turning probabilities into decisions: three decision strategies, threshold analysis, calibration |
| 7 | Hold-out performance, bootstrap uncertainty, and robustness to hospitalisation assumptions |
| 8 | Final model selection, re-training on all data, and writing the five ranked submission files |
| 9 | Explainable AI: SHAP, odds ratios, permutation importance (including cost-based importance) |
| 10 | Conclusions, limitations, and a non-technical summary |

### Data-quality findings

- Some 1–5 opinion scores were stored as `0` (hidden blanks).
- A blank `employment_sector` means "not employed", which is a real answer.
- `doctor_recc_covid` has no blanks, unlike every other survey answer, so missing was probably coded as `0`.
- The test town looks like the training sample (no meaningful distribution shift).

## Limitations and Next Steps

- Hospitalisation risks come from published estimates and a literature-based multiplier model, not from this town's data. Different assumptions change the numbers, which is why we test five scenarios.
- Survey answers are the limiting factor, so model tuning gives only small gains.
- Top submissions are statistically very close; the exact ranking on the test town could differ.
- The expected cost for the test town uses model probabilities, since true labels are unknown.

## Team

**Group Q**, University of Hertfordshire (7PAM2002)

| Name |
|---|
| Mathujan |
| Kokila Sivakumar |
| Rohitha R |
| Lakshmi N.A |
| Krisha S.G |
| Mohit V.S |
| Shaik C |
| Rohit B.H |
| Hemant |

## Acknowledgements

Dataset and task provided as part of the 7PAM2002 module at the University of Hertfordshire.
