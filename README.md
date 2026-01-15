# Reddit Data Pipeline - ETL Lakehouse Project

A data engineering project implementing a modern data lakehouse architecture for Reddit posts using Apache Airflow, PySpark, Delta Lake, and Great Expectations.

## Table of Contents

1. [Architecture](#architecture)
2. [Technology Stack](#technology-stack)
3. [Data Flow](#data-flow)
4. [Table Structure](#table-structure)
5. [CI/CD Pipeline](#cicd-pipeline)
6. [Getting Started](#getting-started)

---

## Architecture

![Project Architecture](Project%20Architechture.png)

### Dashboard Sample

![Dashboard Sample](report/sample/dashboard.png)

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Orchestration | Apache Airflow (Astronomer Runtime 12.6.0) | Workflow scheduling & monitoring |
| Processing | Apache Spark 3.5.3 | Distributed data processing with Structured Streaming |
| Storage | Delta Lake 3.2.1 | ACID transactions, CDC, time travel |
| Data Quality | Great Expectations 1.0.0 | Validation with quarantine system |
| Object Storage | MinIO | S3-compatible lakehouse storage |
| Query Engine | Trino | Distributed SQL on Delta Lake |
| CI/CD | GitHub Actions | Automated build & deployment |

---

## Data Flow

The pipeline follows a **medallion architecture** pattern:

```
Raw CSV Files → Bronze Layer → Silver Layer → Gold Layer
    (Landing)   (Validated)    (Cleaned)      (Star Schema)
```

### Key Features

- **Bronze**: CSV ingestion with Great Expectations validation, failing rows quarantined
- **Silver**: Type casting, CDC enabled, MERGE upserts with conditional updates
- **Gold**: Star schema with fact table (`fact_posts_gl`) and dimensions (`dim_authors_gl`, `dim_flairs_gl`, `dim_domains_gl`)

---

## Airflow DAG

### DAG: `reddit_etl_pipeline`

| Property | Value |
|----------|-------|
| Schedule | `0 8 * * *` (Daily at 8:00 AM SGT) |
| Timezone | Asia/Singapore |
| Catchup | Disabled |
| Tags | reddit, spark, lakehouse |

### Task Flow

```
run_scrapy_notebook → run_spark_etl_notebook → refresh_power_bi
     (Scraping)           (ETL Pipeline)          (BI Refresh)
```

| Task | Description | Timeout |
|------|-------------|---------|
| `run_scrapy_notebook` | Scrape Reddit posts via Papermill | 10 min |
| `run_spark_etl_notebook` | Run Bronze → Silver → Gold ETL | 20 min |

---

## Table Structure

### Database: `reddit_db`

---

### Bronze Layer Tables

#### `reddit_posts_bz`

**Purpose**: Raw data landing zone with validation. Stores Reddit posts as ingested from CSV files.

**Storage Format**: Delta Lake  
**Location**: `s3a://{bucket}/reddit_db/reddit_posts_bz`

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| `post_id` | STRING | No | Unique identifier for Reddit post |
| `title` | STRING | Yes | Post title |
| `author` | STRING | Yes | Reddit username of post author |
| `score` | INT | Yes | Net upvotes/downvotes score |
| `upvote_ratio` | DOUBLE | Yes | Ratio of upvotes to total votes (0.0-1.0) |
| `comments` | INT | Yes | Number of comments on the post |
| `flair` | STRING | Yes | Post flair/tag category |
| `is_video` | STRING | Yes | Whether post contains video (raw string: "True"/"False") |
| `is_self` | STRING | Yes | Whether post is a self-post (raw string: "True"/"False") |
| `domain` | STRING | Yes | Domain of linked content (e.g., "self.singapore", "youtube.com") |
| `url` | STRING | Yes | Full URL of the post or linked content |
| `created_utc` | STRING | Yes | Post creation timestamp in UTC (format: "yyyy-MM-dd HH:mm:ss") |
| `selftext` | STRING | Yes | Post body text content (for self-posts) |
| `extracted_time` | TIMESTAMP | Yes | Timestamp when data was scraped/extracted |
| `load_time` | TIMESTAMP | Yes | Timestamp when data was loaded into Bronze layer |

---

### Silver Layer Tables

#### `reddit_posts_sl`

**Purpose**: Cleaned and transformed data layer with CDC tracking.

**Storage Format**: Delta Lake (with CDC enabled)  
**Location**: `s3a://{bucket}/reddit_db/reddit_posts_sl`  
**Change Data Capture**: Enabled (`delta.enableChangeDataFeed = true`)

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| `post_id` | STRING | No | Unique identifier for Reddit post |
| `title` | STRING | Yes | Post title |
| `author` | STRING | Yes | Reddit username of post author |
| `score` | INT | Yes | Net upvotes/downvotes score |
| `upvote_ratio` | DOUBLE | Yes | Ratio of upvotes to total votes (0.0-1.0) |
| `comments` | INT | Yes | Number of comments on the post |
| `flair` | STRING | Yes | Post flair/tag category |
| `is_video` | BOOLEAN | Yes | Whether post contains video (converted from STRING) |
| `is_self` | BOOLEAN | Yes | Whether post is a self-post (converted from STRING) |
| `domain` | STRING | Yes | Domain of linked content |
| `url` | STRING | Yes | Full URL of the post or linked content |
| `created_utc` | TIMESTAMP | Yes | Post creation timestamp (converted from STRING) |
| `selftext` | STRING | Yes | Post body text content |
| `extracted_time` | TIMESTAMP | Yes | Timestamp when data was scraped/extracted |
| `load_time` | TIMESTAMP | Yes | Timestamp when data was loaded into Bronze layer |
| `update_time` | TIMESTAMP | Yes | Timestamp when row was last updated in Silver layer |

**MERGE Logic**:
```sql
MERGE INTO reddit_posts_sl a USING reddit_posts_sl_delta b ON a.post_id = b.post_id
WHEN MATCHED AND (a.score != b.score OR a.comments != b.comments OR a.upvote_ratio != b.upvote_ratio) 
THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

---

### Gold Layer Tables (Star Schema)

#### `fact_posts_gl` (Fact Table)

**Purpose**: Central fact table for analytics and reporting.

**Storage Format**: Delta Lake  
**Location**: `s3a://{bucket}/reddit_db/fact_posts_gl`

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| `post_id` | STRING | No | Unique identifier for Reddit post (primary key) |
| `title` | STRING | Yes | Post title |
| `author` | STRING | Yes | Reddit username (FK to `dim_authors_gl`) |
| `score` | INT | Yes | Net upvotes/downvotes score |
| `upvote_ratio` | DOUBLE | Yes | Ratio of upvotes to total votes (0.0-1.0) |
| `comments` | INT | Yes | Number of comments on the post |
| `flair` | STRING | Yes | Post flair category (FK to `dim_flairs_gl`) |
| `domain` | STRING | Yes | Domain of linked content (FK to `dim_domains_gl`) |
| `format` | STRING | Yes | Post format: "text" (self-post), "video", or "Others" |
| `url` | STRING | Yes | Full URL of the post or linked content |
| `created_utc` | TIMESTAMP | Yes | Post creation timestamp |
| `selftext` | STRING | Yes | Post body text content |
| `extracted_time` | TIMESTAMP | Yes | Timestamp when data was scraped/extracted |
| `update_time` | TIMESTAMP | Yes | Timestamp when row was last updated |

#### `dim_authors_gl` (Dimension Table)

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| `author` | STRING | Reddit username (primary key) |
| `update_time` | TIMESTAMP | Last update timestamp |

#### `dim_flairs_gl` (Dimension Table)

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| `flair` | STRING | Post flair category (primary key) |
| `update_time` | TIMESTAMP | Last update timestamp |

#### `dim_domains_gl` (Dimension Table)

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| `domain` | STRING | Content domain (primary key) |
| `update_time` | TIMESTAMP | Last update timestamp |

---

### Data Quality Tables

#### `data_quality_quarantine`

**Purpose**: Quarantine table for rows failing Great Expectations validation.

**Storage Format**: Delta Lake  
**Location**: `s3a://{bucket}/reddit_db/data_quality_quarantine`

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| `table_name` | STRING | No | Source table where validation failed |
| `gx_batch_id` | STRING | No | Great Expectations batch identifier |
| `violated_rules` | STRING | Yes | Concatenated list of failed expectations |
| `raw_data` | STRING | Yes | JSON representation of the failed row |
| `ingestion_time` | TIMESTAMP | Yes | Timestamp when row was quarantined |

---

## CI/CD Pipeline

GitHub Actions workflow (`.github/workflows/deploy.yml`) automates deployment:

### Workflow Triggers
- Push to `main` branch

### Pipeline Steps

1. **Build & Push**
   - Multi-architecture Docker build (amd64/arm64)
   - Push to Docker Hub with GitHub Actions cache

2. **Deploy**
   - Connect via Tailscale VPN
   - SSH to server, pull latest code and image
   - Restart services with `docker compose up -d`
   - Auto-sync Airflow permissions

### Required Secrets
- `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- `TAILSCALE_AUTHKEY`
- `SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY`

---

## Getting Started

### Prerequisites
- Docker & Docker Compose
- Python 3.8+
- Astronomer CLI (optional)

### Quick Start

```bash
# Clone and start
git clone <repository-url>
cd airflow-02-reddit-project
astro dev start  # or: docker-compose up -d

# Access Airflow UI
# URL: http://localhost:8080 (admin/admin)
```

### Environment Variables

```bash
AWS_ENDPOINT_URL=http://minio:9000
AWS_ACCESS_KEY_ID=admin
AWS_SECRET_ACCESS_KEY=admin_password
MINIO_BUCKET=reddit-warehouse
```

---

## Best Practices

- **Idempotency**: All operations support re-runs without duplication
- **Exactly-Once Processing**: Checkpoint-based streaming
- **Data Quality Gates**: Validation before ingestion with quarantine
- **Incremental Processing**: CDC-based updates
- **Schema Evolution**: Delta Lake handles changes gracefully
