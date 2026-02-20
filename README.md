# 🎬 MovieLens Lakehouse: Multi-Source Recommendation Engine

A production-ready data lakehouse built with PySpark and Databricks, following the Medallion Architecture. Raw data is transformed through Bronze → Silver → Gold layers into a collaborative filtering recommendation engine powered by Spark MLlib's ALS algorithm.

---

## Table of Contents

- [Description](#description)
- [Architecture](#architecture)
- [Data Sources](#data-sources)
- [Installation](#installation)
- [Usage](#usage)
- [Pipeline Overview](#pipeline-overview)
- [Gold Layer: Business Value](#gold-layer-business-value)
- [Recommendation Model](#recommendation-model)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Contributors](#contributors)
- [Timeline](#timeline)

---

## Description

This project simulates a real-world streaming platform's data infrastructure. Data is fragmented across multiple sources — internal transactional databases (ratings), static metadata (movies, links), user-generated content (tags).

This is a pipeline that ingests, cleans, joins, and transforms this raw data into:
- A **star schema** data warehouse (fact + dimension tables)
- A **collaborative filtering recommendation engine** generating personalized top-10 movie suggestions per user
- An **automated end-to-end pipeline** orchestrated via Databricks Workflows

**Tech stack:** PySpark, Delta Lake, Databricks (Free Edition), Unity Catalog, Spark MLlib (ALS)

---

## Architecture

The project follows the **Medallion Architecture**, ensuring data quality at every stage:

```
┌────────────────────────────────────────────────────────────────────┐
│                        MEDALLION ARCHITECTURE                      │
│                                                                    │
│  ┌───────────┐     ┌───────────────┐     ┌─────────────────────┐   │
│  │  BRONZE   │     │    SILVER     │     │       GOLD          │   │
│  │           │     │               │     │                     │   │
│  │ Raw CSV   │────▶│ Type casting  │────▶│ fact_ratings        │   │
│  │           │     │ Dedup (SCD1)  │     │ dim_users           │   │
│  │ No changes│     │ Genre parsing │     │ dim_movies_enriched │   │
│  │           │     │ Left joins    │     │ user_recommendations│   │
│  │ Preserve  │     │ Make data     │     │ Make data           │   │
│  │ truth     │     │ trustworthy   │     │ actionable          │   │
│  └───────────┘     └───────────────┘     └─────────────────────┘   │
│                                                    │               │
│                                          ┌─────────▼──────────┐    │
│                                          │   ALS MODEL        │    │
│                                          │   Top-10 recs/user │    │
│                                          └────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

| Layer | Goal | Format |
|-------|------|--------|
| **Bronze** | Controlled raw ingestion — preserve truth | Delta (schema unchanged from source) |
| **Silver** | Data quality & structure — make data trustworthy & relational | Delta (typed, deduped, joined) |
| **Gold** | Business intelligence & analytics — make data actionable | Delta (aggregated, segmented, ML-ready) |

---

##  Data Sources

| Dataset | Source | Format | Key Columns | DE Challenge |
|---------|--------|--------|-------------|--------------|
| **Ratings** | [MovieLens](https://grouplens.org/datasets/movielens/) | CSV | userId, movieId, rating, timestamp | 32M rows — high-volume type casting & dedup |
| **Movies** | [MovieLens](https://grouplens.org/datasets/movielens/) | CSV | movieId, title, genres | Pipe-separated genres → ArrayType parsing |
| **Tags** | [MovieLens](https://grouplens.org/datasets/movielens/) | CSV | userId, movieId, tag | No unique key, messy text, CSV quoting issues |
| **Links** | [MovieLens](https://grouplens.org/datasets/movielens/) | CSV | movieId, imdbId, tmdbId | Cross-referencing IDs for scraping phase |
**Dataset used:** |[ml-25m](https://grouplens.org/datasets/movielens/32m/)

---

## Installation

### Prerequisites

- A [Databricks Free Edition](https://signup.databricks.com) account (free, no credit card required)
- Git installed locally

### Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/your-team/movielens-lakehouse.git
   cd movie-recommender
   ```

2. **Sign up for Databricks Free Edition:**
   - Go to [signup.databricks.com](https://signup.databricks.com)
   - Choose Free Edition (not Free Trial)
   - Sign in with Microsoft, Google, or email

3. **Download the MovieLens dataset:**
   - Download [ml-32m](https://grouplens.org/datasets/movielens/32m/) from GroupLens
   - Unzip to get: `ratings.csv`, `movies.csv`, `links.csv`, `tags.csv`

4. **Upload data to Databricks:**
   ```sql
   -- Create a volume for raw data
   CREATE VOLUME IF NOT EXISTS workspace.default.movielens_raw;
   ```
   - In Databricks: **New → Add or upload data → Upload files to volume**
   - Upload all CSV files to `workspace.default.movielens_raw`

5. **Connect your GitHub repo:**
   - In Databricks: **Workspace → Repos → Add Repo**
   - Paste your GitHub repo URL

6. **Set environment version (required for SparkML):**
   - Open any notebook → right sidebar gear icon → Environment version → **4** (or latest)

---

## Usage

1. Go to **Jobs & Pipelines** in the sidebar
2. Select the `MovieLens_Pipeline` job
3. Click **Run now**
4. The job chains all four notebooks automatically:
   ```
   ingest_bronze → clean_silver → transform_gold → train_model
   ```

---

## Pipeline Overview

### Bronze (Raw Ingestion)
- Ingests CSV and JSON files with no schema modifications
- Adds `ingestion_timestamp` and `source` metadata columns
- Saves as Delta tables for ACID transaction guarantees
- Handles CSV quoting issues (tags containing commas)

### Silver (Cleaning & Joining)
- **Type casting**: userId/movieId → Integer, rating → Float, timestamp → Timestamp
- **Deduplication**: SCD Type 1 — one rating per user-movie pair
- **Genre parsing**: Pipe-separated string → trimmed, lowercased ArrayType
- **Tag cleaning**: Lowercase, trim, deduplicate per movie, aggregate into arrays via `collect_set`
- **Joins**: Left joins preserving dimensional integrity (ratings + movies + tags)
- **Data quality report**: Null checks, orphan detection, row count validation

### Gold (Business-Ready)
- **fact_ratings**: Lean event table (userId, movieId, rating, timestamp) — direct ALS input
- **dim_users**: User behavior profiles with segmentation (power user flag via percentile ranking)
- **dim_movies_enriched**: Complete movie catalog with pre-computed aggregates, genres array, tags array
- **user_recommendations**: Top-10 ALS predictions per user

### Model (Serving)
- **Algorithm**: ALS (Alternating Least Squares) collaborative filtering
- **Training**: 80/20 split, 10 iterations, rank 10
- **Cold start strategy**: Drop (handles users/items not in training set)
- **Evaluation**: RMSE ≈ 0.81
- **Output**: Top-10 recommendations generated for power user segment

---

## Gold Layer: Business Value

The Gold layer follows a **star schema** design:

```
              dim_users
             (who rated?)
                  │
                  │ userId
                  │
fact_ratings ─────┤
(what happened?)  │
                  │ movieId
                  │
             dim_movies_enriched
             (what was rated?)
```

### Questions the Gold layer answers immediately:

- Who are our most engaged users? Who's at risk of churning?
- What are the highest-rated genres among power users vs. casual users?
- Which movies have the most passionate fanbase?
- What tags are most associated with top-rated movies?
- What should we recommend to each user next?

---

## Recommendation Model

### How ALS Works

ALS decomposes the sparse user-movie rating matrix into two smaller matrices (user factors and item factors). Multiplying them back together predicts ratings for unseen user-movie pairs. It learns purely from rating patterns — no content features needed.

### Results

- **RMSE**: ~0.81
- **Recommendations generated** for top 100 power users (10 movies each)
- **Cold start handling**: Users/movies not in training set are dropped from evaluation

### Limitations & Future Improvements

- Recommendations generated for a sample (not all users) due to compute constraints
- `recommendForAllUsers()` not supported on serverless — workaround uses `crossJoin` + ranking
- `.cache()` not supported on Free Edition — would improve training performance on classic clusters
- Content-based hybrid approach (using genres, tags, director) could complement collaborative filtering

---

## Key Engineering Decisions

| Decision | Why |
|----------|-----|
| **Delta format everywhere** | ACID transactions prevent half-written tables in the automated pipeline |
| **Left joins (not inner)** | Preserve all ratings even when movie metadata is missing — more training data for ALS |
| **Genres as ArrayType** | Enables filtering/exploding without runtime string parsing |
| **Tags lowercased + trimmed + collect_set** | "Sci-Fi", " sci-fi ", "SCI-FI" all become one tag |
| **Power user by percentile (not fixed threshold)** | Adapts to dataset size — always the top 10% |
| **Star schema in Gold** | Storage efficiency, update independence, query flexibility |
| **SCD Type 1 deduplication** | Latest rating per user-movie pair overwrites previous — simple and sufficient |

---

## Project Structure

```
movielens-lakehouse/
├── notebooks/
│   ├── 01_ingest_bronze.ipynb        # Raw CSV → Delta tables
│   ├── 02_clean_silver.ipynb         # Type casting, dedup, joins, genre/tag parsing
│   ├── 03_transform_gold.ipynb       # Fact + dimension tables, user segmentation
│   └── 04_train_model.ipynb          # ALS training, evaluation, top-10 recommendations 
└── README.md
```

---


## Resources

- [MovieLens Dataset](https://grouplens.org/datasets/movielens/)
- [Spark MLlib ALS Documentation](https://spark.apache.org/docs/latest/ml-collaborative-filtering.html)
- [Databricks Free Edition](https://signup.databricks.com)
- [Delta Lake Documentation](https://docs.delta.io/)
