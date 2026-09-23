---
layout: post
title: "Building FinAgent: A Financial Time-Series Forecasting Platform"
date: 2026-09-23
categories: [AI, Machine Learning, Time Series]
tags: [Python, XGBoost, FastAPI, Pydantic, Docker, CI/CD, Machine Learning, Time Series]
---

# Building FinAgent: A Financial Time-Series Forecasting Platform

I wanted to build a machine learning project that went beyond training a model inside a Jupyter notebook.

A typical machine learning project can look something like this:

```text
Dataset → Model → Prediction → Done
```

That's useful for experimentation, but I wanted to explore what happens when we start treating the model as part of an actual software system.

That led me to **FinAgent** — a financial time-series forecasting platform designed around a simple problem:

> **Can we predict the next day's transaction volume using historical transaction behavior, financial indicators, and calendar patterns?**

The first version of FinAgent focuses on the fundamentals:

```text
Data
 ↓
Feature Engineering
 ↓
Time-Series Modeling
 ↓
Evaluation
 ↓
Model Persistence
 ↓
FastAPI
 ↓
Docker
 ↓
Testing
 ↓
CI/CD
```

This is intentionally the first stage of the project. I want to establish a clean and reproducible foundation before moving into more advanced approaches.

---

## 1. The Problem

Financial transaction activity changes over time.

Transaction volume can be influenced by several factors:

- Historical transaction behavior
- Day of the week
- Weekends
- Holidays
- Month-end effects
- Quarter-end effects
- Interest rates
- Inflation
- Unemployment
- Market volatility
- Overall market conditions

A model that only looks at the previous day's transaction volume may miss many of these patterns.

So I defined the initial problem as:

> **Given historical transaction activity and relevant financial/calendar features, can we predict the transaction volume for the following day?**

The target variable in the dataset is:

```text
target_next_day_volume
```

---

## 2. Dataset

For the initial version, I created a synthetic financial transaction dataset so that the complete project could be reproduced without access to confidential or proprietary financial data.

The dataset contains approximately 2,800 daily observations covering multiple years.

Some of the important variables are:

```text
transaction_volume
transaction_value_usd
avg_transaction_value

digital_transaction_share
international_transaction_share

fed_funds_rate
inflation_rate
unemployment_rate
vix
sp500_index

day_of_week
month
quarter
is_weekend
is_month_end
is_quarter_end
is_holiday

target_next_day_volume
```

### A note about the data

The transaction-related data is **synthetic** and is used only for development and demonstration.

It is not JPMorgan customer data or any other confidential financial dataset.

One of my goals is to eventually experiment with public financial and economic datasets while keeping the project reproducible.

---

## 3. Why Time-Series Forecasting?

Time-series problems require a slightly different way of thinking about validation.

In a normal machine learning problem, we might randomly divide our data into training and testing sets:

```text
Random rows
     ↓
Train / Test
```

For time-series forecasting, this can cause data leakage.

Imagine trying to predict values in 2025 while allowing the model to train on observations from 2026.

That wouldn't represent the real forecasting problem.

Instead, FinAgent uses a chronological split:

```text
                 Time ───────────────────────>

Training Data                         Test Data
───────────────────────────────      ─────────────
            Past                       Future
```

The model learns from historical observations and is evaluated on a later period.

This better reflects how the model would behave when making predictions in production.

---

## 4. Feature Engineering

Raw time-series data isn't always enough.

I created several groups of features to give the model information about historical behavior, seasonality, and financial conditions.

### Calendar features

```text
day_of_week
month
quarter
is_weekend
is_month_end
is_quarter_end
is_holiday
```

For example, transaction behavior on a weekend can be very different from transaction behavior on a weekday.

Month-end and quarter-end effects can also introduce recurring patterns.

---

## 5. Lag Features

One of the most important concepts in time-series forecasting is using historical observations as features.

FinAgent creates:

```text
lag_1
lag_7
lag_14
lag_28
```

For example:

```text
lag_1  → yesterday's transaction volume
lag_7  → transaction volume seven days ago
lag_14 → transaction volume fourteen days ago
lag_28 → transaction volume twenty-eight days ago
```

This allows the model to learn both short-term behavior and recurring patterns.

A simplified representation looks like this:

```text
Previous observations
        │
        ├── Yesterday
        ├── 7 days ago
        ├── 14 days ago
        └── 28 days ago
                │
                ▼
          Forecast model
                │
                ▼
       Next-day prediction
```

---

## 6. Rolling Statistics

I also introduced rolling statistical features.

These include:

```text
rolling_mean_7
rolling_std_7

rolling_mean_28
rolling_std_28
```

For example, instead of looking at only yesterday's transaction volume, the model can also understand the recent average:

```text
Last 7 days

1.02M
1.08M
1.05M
1.10M
1.04M
1.07M
1.06M

       ↓

7-day rolling mean
```

The rolling standard deviation gives the model information about recent volatility.

This becomes particularly useful when the underlying series becomes more unstable.

---

## 7. Financial and Economic Features

The dataset also contains financial and economic variables such as:

```text
Federal Funds Rate
Inflation
Unemployment
VIX
S&P 500
```

The idea is to allow the model to learn relationships between transaction activity and broader financial conditions.

Conceptually:

```text
Historical transaction behavior
              +
Calendar effects
              +
Economic conditions
              +
Market conditions
              ↓
        Forecasting Model
              ↓
       Next-day forecast
```

These variables aren't being treated as proof of causation. They are simply additional features that may contain predictive information.

---

## 8. Why XGBoost?

For the initial implementation, I selected **XGBoost** as the forecasting model.

This doesn't mean XGBoost is universally the best algorithm for time-series forecasting.

I selected it because it provides a strong baseline for a problem containing:

- Lag features
- Rolling statistics
- Calendar features
- External numerical variables
- Non-linear relationships

Instead of feeding the raw time series directly into XGBoost, the time-series structure is first represented through engineered features.

The initial model uses parameters such as:

```text
n_estimators = 500
max_depth = 6
learning_rate = 0.05
```

The important part isn't the parameters themselves.

The important part is establishing a reproducible baseline that can later be compared against other approaches.

---

## 9. Model Evaluation

FinAgent evaluates the model using three metrics.

### MAE — Mean Absolute Error

```text
MAE = average(|actual - predicted|)
```

This provides an intuitive measure of the average prediction error.

### RMSE — Root Mean Squared Error

RMSE gives greater weight to larger errors.

This makes it useful when large forecasting mistakes are particularly important.

### MAPE — Mean Absolute Percentage Error

MAPE expresses the error as a percentage, which can make the results easier to interpret.

The evaluation pipeline therefore produces:

```text
MAE
RMSE
MAPE
```

The evaluation is performed on the future portion of the time series rather than using a random split.

---

## 10. From Model to API

A machine learning model sitting inside a Python script isn't particularly useful to other applications.

So I wanted to expose the model through an API.

For this, I used **FastAPI**.

The architecture is:

```text
Client
  │
  ▼
FastAPI
  │
  ▼
Pydantic Validation
  │
  ▼
Feature Preparation
  │
  ▼
XGBoost Model
  │
  ▼
Prediction
```

The main endpoint is:

```text
POST /forecast
```

A client can send the required features and receive a predicted next-day transaction volume.

This also creates a clean separation between the machine learning model and the application consuming it.

---

## 11. Why Pydantic?

An API shouldn't blindly accept arbitrary input.

For example:

```text
digital_transaction_share
```

should logically be between:

```text
0 and 1
```

Similarly:

```text
month → 1–12
quarter → 1–4
day_of_week → 0–6
```

Pydantic allows these constraints to be expressed directly in the API schema.

This gives us:

- Input validation
- Structured request models
- Structured responses
- Better API documentation
- Easier maintenance

FastAPI also automatically generates an interactive Swagger interface.

---

## 12. Model Persistence

The trained model is saved using Joblib.

Instead of retraining the model every time the API starts:

```text
Training
   ↓
Model artifact
   ↓
Save
   ↓
API loads model
   ↓
Prediction
```

This separates:

**Model training**

from

**Model serving**

which is an important concept when moving from experimentation toward production machine learning.

---

## 13. Docker

I also wanted the application to be reproducible across different environments.

So the project includes Docker support.

Conceptually:

```text
Docker Container
│
├── Python
├── Dependencies
├── FinAgent source code
├── Dataset
└── FastAPI
```

The goal is to reduce the classic:

> "It works on my machine."

problem.

The same application environment can be packaged and run consistently wherever Docker is available.

---

## 14. Automated Testing

The repository includes automated tests for important parts of the application.

The current tests cover areas such as:

```text
Data validation
Feature engineering
Model training
API health endpoint
```

This is important because a machine learning project is still software.

A change to a feature-processing function shouldn't silently break the API or training pipeline.

---

## 15. CI With GitHub Actions

The repository also includes a GitHub Actions workflow.

The current CI flow is approximately:

```text
Developer pushes code
        ↓
GitHub Actions
        ↓
Install dependencies
        ↓
Training smoke test
        ↓
Run tests
        ↓
Lint
        ↓
Build status
```

This gives the project a basic automated quality gate.

It also means that the repository isn't dependent entirely on manual testing.

---

## 16. Project Architecture

The current Phase 1 architecture looks like this:

```text
                    ┌─────────────────┐
                    │   Transaction   │
                    │      Data       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Data Validation │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Feature      │
                    │   Engineering   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    XGBoost      │
                    │     Model       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Evaluation   │
                    │ MAE/RMSE/MAPE   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Model Artifact  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    FastAPI      │
                    │      API        │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Docker + CI/CD  │
                    └─────────────────┘
```

---

## 17. Repository Structure

I wanted the repository to reflect the separation between experimentation, production code, data, and tests.

```text
FinAgent/
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── configs/
│   └── config.yaml
│
├── data/
│   └── raw/
│       ├── transactions.csv
│       └── data_dictionary.csv
│
├── docker/
│   └── docker-compose.yml
│
├── notebooks/
│
├── scripts/
│   ├── train.py
│   └── predict.py
│
├── src/
│   └── finagent/
│       ├── api/
│       ├── data.py
│       ├── features.py
│       ├── evaluation.py
│       └── model.py
│
├── tests/
│
├── Dockerfile
├── Makefile
├── pyproject.toml
├── requirements.txt
└── README.md
```

---

## 18. Running the Project

Clone the repository:

```bash
git clone https://github.com/VishnupriyaPSheejan/FinAgent.git
cd FinAgent
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate the environment and install the dependencies:

```bash
pip install -r requirements.txt
```

Train the model:

```bash
python scripts/train.py
```

Run the API:

```bash
uvicorn finagent.api.main:app --reload
```

Then open:

```text
http://127.0.0.1:8000/docs
```

The FastAPI Swagger interface can be used to test the forecasting endpoint.

---

## 19. What I Learned

The biggest takeaway from this project isn't simply how to train an XGBoost model.

It is understanding the difference between:

```text
Machine Learning Experiment
```

and:

```text
Machine Learning Application
```

A model is only one part of the system.

To make it usable, we also need to think about:

- Data validation
- Feature engineering
- Time-aware validation
- Model evaluation
- Model persistence
- API design
- Input validation
- Testing
- Containerization
- Continuous integration

That shift in perspective is probably the most valuable part of building FinAgent.

---

## 20. Current Status

The current version focuses on the **core forecasting and engineering foundation**.

Implemented:

- Data ingestion
- Data validation
- Time-aware train/test split
- Calendar features
- Lag features
- Rolling features
- XGBoost forecasting
- MAE / RMSE / MAPE evaluation
- Model persistence
- FastAPI
- Pydantic validation
- Docker
- Automated tests
- GitHub Actions CI

I am intentionally keeping this first version focused.

Rather than adding technologies simply to make the project look larger, I want each component to have a clear purpose and be properly implemented.

---

## Conclusion

FinAgent started with a simple question:

> **Can I build a financial forecasting model?**

But that quickly evolved into a more interesting question:

> **Can I turn that model into a reproducible, testable and consumable machine learning application?**

That change in perspective shaped the architecture of the project.

The current implementation establishes the forecasting and engineering foundation. More advanced approaches will be explored separately rather than mixing multiple architectures into the initial project.

For now, the goal is simple:

**Build the foundation correctly before building on top of it.**

---

## Project Repository

**FinAgent — Financial Time-Series Forecasting Platform**

[View the project on GitHub](https://github.com/VishnupriyaPSheejan/FinAgent)

---

### Technologies

`Python` · `Pandas` · `NumPy` · `Scikit-learn` · `XGBoost` · `FastAPI` · `Pydantic` · `Docker` · `Pytest` · `GitHub Actions` · `Git`
