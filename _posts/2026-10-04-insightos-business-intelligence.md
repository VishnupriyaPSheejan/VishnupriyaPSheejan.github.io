---
categories:
- Artificial Intelligence
- Machine Learning
- Python
- Data Analytics
date: 2026-10-04
layout: post
tags:
- Python
- Streamlit
- Docker
- GitHub Actions
- Business Intelligence
- Data Analytics
- AI Engineering
title: "InsightOS: From Raw Sales Data to Business Insights with Python,
  Streamlit, Docker, and GitHub Actions"
---

# InsightOS: Turning Raw Business Data into Actionable Insights 📊

*What if business users could explore sales data, understand trends, and
get useful analytical answers without writing SQL queries or building
reports manually?*

That question inspired me to build **InsightOS**, a lightweight business
intelligence application that turns sales data into interactive
visualizations and easy-to-understand summaries.

As an AI/ML Engineer, I wanted to go beyond a notebook or a standalone
model. I wanted to package a complete, reproducible application---from
data processing and visualization to Docker and automated testing.

> **Project note:** InsightOS currently uses a deterministic
> Python-based question router, not a live LLM or autonomous multi-agent
> system. The included dataset is synthetic demo data.

## The problem

Business data is often available in spreadsheets, databases, and data
warehouses, but turning it into useful insights can still require
technical skills. Users may need to write queries, calculate metrics,
build charts, and compare performance across regions or product
categories.

InsightOS aims to make this exploration easier through a simple
interface.

## What is InsightOS?

InsightOS is a local-first business intelligence demo built with Python
and Streamlit. It loads a bundled sales dataset, processes it with
Pandas, and displays interactive charts using Plotly.

Users can:

-   View total revenue, units sold, and unique order counts.
-   Filter the dataset by region and product category.
-   Explore revenue trends over time.
-   Compare revenue across regions.
-   Ask supported questions about the selected data.
-   Download the filtered data as a CSV file.

The application does not require an API key, cloud account, or external
dataset.

## Architecture

``` text
                 User
                  |
                  v
          Streamlit Interface
                  |
          Region / Category Filters
                  |
                  v
             Pandas Data Layer
                  |
        +---------+----------+
        |                    |
        v                    v
    KPI Analytics       Question Router
        |                    |
        v                    v
    Plotly Charts       Python Logic
        |                    |
        +---------+----------+
                  |
                  v
           Results / CSV Export
```

The CSV data source keeps the setup straightforward and makes the
project easy to reproduce locally.

## Technology stack

  Technology       Purpose
  ---------------- ----------------------------------------------------
  Python           Application and analytics logic
  Pandas           Data loading, filtering, grouping, and aggregation
  Streamlit        Interactive dashboard and user interface
  Plotly           Interactive charts
  Docker           Reproducible application environment
  Pytest           Unit testing
  GitHub Actions   Automated tests and Docker build checks

## Exploring the dashboard

### KPI overview

The dashboard displays total revenue, units sold, and unique orders for
the selected filters. These metrics provide a quick summary of the
selected business segment.

### Revenue trends

A time-series chart groups revenue by month so users can inspect how it
changes over the selected period.

### Regional performance

A bar chart compares revenue across regions and helps identify the
leading region in the filtered dataset.

### Ask the analyst

The question interface supports common prompts such as:

-   What is the total revenue?
-   Which region has the highest revenue?
-   Which category leads by revenue?
-   How did revenue change over the selected period?

The current implementation uses predefined Python logic to answer these
questions. Trend comparisons are descriptive; they do not establish why
a change occurred or prove causation.

## Making it reproducible with Docker

The repository includes a `Dockerfile` and `docker-compose.yml`. With
Docker installed, launch the app using:

``` bash
docker compose up --build
```

Then open <http://localhost:8501>.

This packages the runtime and dependencies so that users do not need to
manually configure the Python environment.

## Continuous integration with GitHub Actions

The included GitHub Actions workflow runs when code is pushed or a pull
request is opened. It:

1.  Checks out the repository.
2.  Sets up Python.
3.  Installs dependencies.
4.  Runs the unit tests.
5.  Builds the Docker image.

This provides a basic CI foundation. Cloud deployment is a separate next
step and requires choosing a hosting target and configuring the required
GitHub secrets.

## Run locally in VS Code

### Prerequisites

-   Python 3.11 or later
-   VS Code
-   Git

### 1. Clone the repository

``` bash
git clone YOUR_GITHUB_REPOSITORY_URL
cd insightos
```

### 2. Create and activate a virtual environment

``` bash
python -m venv .venv
```

Windows PowerShell:

``` powershell
.venv\Scripts\Activate.ps1
```

macOS/Linux:

``` bash
source .venv/bin/activate
```

### 3. Install dependencies and run

``` bash
pip install -r requirements.txt
streamlit run app.py
```

Open the local URL shown in the terminal, usually
`http://localhost:8501`.

The sales dataset is already included in the repository, so no separate
download is needed.

## What I learned

Building InsightOS reinforced that delivering a useful AI/ML application
involves more than model development or data analysis. It also requires
a usable interface, modular code, reproducible setup, automated tests,
packaging, and clear documentation.

I also wanted the project to be honest about its current capabilities.
The first version focuses on dependable descriptive analytics; more
advanced LLM-based reasoning should be added only with appropriate
safeguards and evaluation.

## What's next?

Potential future iterations include:

-   **LLM-powered analytics:** interpret a wider range of
    natural-language questions.
-   **Text-to-SQL:** generate read-only queries with validation and
    execution safeguards.
-   **Multi-agent workflow:** separate data retrieval, analysis, insight
    generation, and verification responsibilities.
-   **Evaluation and observability:** measure answer correctness,
    grounding, latency, and failure cases.
-   **Cloud deployment:** extend GitHub Actions to publish and deploy
    the application.

These are roadmap ideas, not features currently included in this starter
release.

## Explore the project

-   **GitHub repository:** \[Add your repository URL\]
-   **Live demo:** \[Add your deployed URL when available\]

If you find the project useful, feel free to explore the code, open an
issue, or share feedback.

Thanks for reading! 🚀

------------------------------------------------------------------------

**Author:** Vishnupriya\
AI/ML Engineer \| Python \| Machine Learning \| Generative AI
