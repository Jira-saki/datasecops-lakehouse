[![DataSecOps CI Pipeline](https://github.com/Jira-saki/aws-dataops-platform/actions/workflows/ci.yaml/badge.svg)](https://github.com/Jira-saki/aws-dataops-platform/actions/workflows/ci.yaml)
![Architecture](https://img.shields.io/badge/Architecture-Serverless%20Lakehouse-orange?style=flat-square)
![FinOps](https://img.shields.io/badge/FinOps-Zero--Idle--Cost-green?style=flat-square)
![Security](https://img.shields.io/badge/Security-Salted%20SHA--256%20PII-blue?style=flat-square)

# EP3: Unified Batch & Streaming DataSecOps Lakehouse (Serverless & FinOps)

> A production-grade, zero-idle-cost Lakehouse platform engineered for data privacy compliance (GDPR/APPI), dual-engine ingestion, and serverless threat hunting without operational cluster overhead.

The platform bridges local developer iteration and enterprise cloud infrastructure through an architectural continuum: developers execute 100% offline using Python (>= 3.11), the `dlt` (data load tool) ELT framework, DuckDB OLAP engine, and an S3-compatible emulator (Floci). When deploying to AWS, modular HashiCorp Terraform provisions a 100% serverless data lakehouse featuring SSE-AES256 encrypted S3 storage, an AWS Glue Data Catalog configured with **Zero-Crawler Partition Projection**, and serverless AWS Athena threat hunting suites.

---

## 🎯 Architectural Highlights & Key Pillars

* **Dual-Engine Ingestion Strategy:** Unifies local-first batch ingestion (`dlt` + Parquet) with managed real-time streaming (**Amazon Data Firehose** *[In Progress]*), terminating into a single partitioned Lakehouse format.
* **Deterministic PII Masking & Privacy Engineering:** Client IP addresses are pseudonymized at ingestion time using a salted SHA-256 cryptographic hasher (`src/utils/hasher.py`), generating a fixed 12-character identifier (`user_masked_id`) to ensure GDPR compliance while retaining cross-table correlation capabilities.
* **Pure FinOps & Zero-Idle Cost:** 100% Serverless architecture on AWS. Zero EC2/EKS compute charges when idle, sub-kilobyte query scanning via Snappy Parquet, and zero metadata crawler costs.
* **AWS Glue Partition Projection:** Eliminates daily Glue Crawler executions and expensive metastore listing operations by deterministically projecting partition metadata directly within Athena queries.
* **Non-Blocking Dead Letter Queue (DLQ):** Unparseable payloads, malformed headers, and regex mismatches are automatically segregated into an isolated DLQ Parquet dataset, allowing clean records to process continuously without pipeline failures.
* **Unified Threat Hunting Interface:** Cross-environment SQL analytics supporting local zero-copy queries via DuckDB and enterprise cloud security investigation via AWS Athena.

---

## 🏗 System Architecture


![DATASEC-OPS PLATFORM Architecture Data Flow](assets/dataops.png)




```text
               [ Raw Multi-Tenant Logs / Audit Trails ]
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼ (Batch Historical Ingestion)                    ▼ (Near Real-Time Stream)
┌───────────────────────────────────┐             ┌──────────────────────────────────┐
│  DataSecOps Ingestion Gateway     │             │ Amazon Data Firehose (Stream)    │
│  (dlt + DuckDB Engine)            │             │ [Roadmap / In Progress]          │
└─────────────────┬─────────────────┘             └────────────────┬─────────────────┘
                  │                                                │
                  │   • Deterministic Salted SHA-256 PII Hasher    │
                  │   • Non-Blocking DLQ Anomaly Filter            │
                  ▼                                                ▼
        ┌───────────────────┐                            ┌───────────────────┐
        │ Clean Access Logs │                            │  Corrupted (DLQ)  │
        └─────────┬─────────┘                            └─────────┬─────────┘
                  └───────────────────────┬────────────────────────┘
                                          ▼
                         [ Hive-Partitioned Apache Parquet ]
                           (year=YYYY / month=MM / day=DD)
                                          │
       ┌──────────────────────────────────┴──────────────────────────────────┐
       ▼ (Local Emulation)                                                   ▼ (AWS Production - 100% Serverless)
• Floci S3 Emulation                                                  • Encrypted S3 Data Lake (SSE-AES256)
• DuckDB OLAP Analytics                                               • AWS Glue Partition Projection (Zero Crawler)
• Pytest Test Automation                                              • Athena Threat Hunting Analytics

```

---

## 📁 Repository Structure

```
.
├── Makefile                          # Unified build, test, and pipeline automation
├── pytest.ini                        # Pytest configuration and Python path resolver
├── README.md                         # Platform documentation
├── requirements.txt                  # Python dependencies (dlt, pyarrow, boto3, duckdb, pytest)
├── assets/
│   ├── aws_athena_proof.png          # Live AWS Athena query execution evidence
│   ├── aws_glue_proof.png            # Live AWS Glue Table schema proof
│   ├── aws_glue_proof-adv.png        # Live AWS Glue Partition Projection config proof
│   └── dataops.png                   # Architecture Data Flow diagram
├── sql/
│   └── athena_threat_hunting.sql     # Production Athena Threat Hunting SQL suite
├── src/
│   ├── pipelines/
│   │   └── xserver_pipeline.py       # Core dlt DataSecOps ELT pipeline
│   └── utils/
│       ├── hasher.py                 # Deterministic Salted SHA-256 PII Hasher
│       └── parser.py                 # High-throughput regex access & audit log parsers
├── terraform/
│   ├── environments/
│   │   ├── aws/                      # AWS Production Environment (S3 + Glue)
│   │   └── local/                    # Local-first emulation provisioning
│   └── modules/
│       ├── glue_catalog/             # Glue DB, Partition Projection Tables
│       └── s3/                       # Hardened S3 Bucket with SSE-AES256
└── tests/
    └── test_parser.py                # Unit test suite for parsing, hashing, and DLQ
```

---

## 🚀 Quickstart Guide

### 1. Environment & Local Provisioning

Initialize Python virtual environment and provision local emulation infrastructure:

```bash
# Set up Python virtual environment & install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Provision local infrastructure emulation (S3/Floci object store)
cd terraform/environments/local/
terraform init
terraform apply -auto-approve
cd ../../../
```

### 2. Run Quality Assurance & Unit Tests

Execute the unit test suite covering deterministic PII masking, regex parsing edge cases, and DLQ anomaly routing:

```bash
make test
# or: pytest -v tests/
```

### 3. Execute DataSecOps Pipeline

Run the pipeline to parse raw access logs, apply salted SHA-256 IP masking, route unparseable records to DLQ, and write Hive-partitioned Parquet files to `data/lakehouse`:

```bash
# Process raw logs and export Parquet Lakehouse locally
make run-parquet

# Inspect generated Hive partition structure
tree data/lakehouse
```

### 4. Threat Hunting Verification (Local Analytics)

Query the generated Parquet files using the local DuckDB OLAP engine to analyze status code distributions and identify anomaly patterns:

```bash
python -c "
import duckdb
con = duckdb.connect()
print(con.sql('''
    SELECT 
        tenant_id, 
        user_masked_id, 
        request_path, 
        http_method, 
        status_code, 
        COUNT(*) AS attempts
    FROM 'data/lakehouse/dataops_lakehouse/xserver_access_logs/*/*/*/*.parquet'
    WHERE status_code IN (403, 404)
    GROUP BY ALL
    ORDER BY attempts DESC
''').df())
"
```

### 5. Infrastructure Validation

Validate AWS production Terraform configurations prior to deployment:

```bash
# Validate AWS Production Configuration
make tf-validate

# Deploy Infrastructure to AWS (requires configured AWS credentials)
cd terraform/environments/aws
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## ☁️ Infrastructure as Code & Partition Projection

The cloud target uses AWS Glue Data Catalog configured with **Partition Projection**. This eliminates metastore synchronization overhead by projecting date-based partition paths dynamically:

```hcl
resource "aws_glue_catalog_table" "xserver_access_logs" {
  name          = "xserver_access_logs"
  database_name = var.database_name
  table_type    = "EXTERNAL_TABLE"

  parameters = {
    "classification"            = "parquet"
    "projection.enabled"        = "true"
    "projection.year.type"      = "integer"
    "projection.year.range"     = "2024,2030"
    "projection.month.type"     = "integer"
    "projection.month.range"    = "1,12"
    "projection.month.digits"   = "2"
    "projection.day.type"       = "integer"
    "projection.day.range"      = "1,31"
    "projection.day.digits"     = "2"
    "storage.location.template" = "s3://${var.s3_bucket_id}/dataops_lakehouse/xserver_access_logs/year=$${year}/month=$${month}/day=$${day}/"
  }
  # ... storage_descriptor & column definitions ...
}
```

---

## 🛡 DataSecOps & Threat Hunting Showcase

Production query snippet from `sql/athena_threat_hunting.sql` executed against AWS Athena to detect reconnaissance probing (`.env`, `wp-config`, administrative endpoints):

```sql
-- Detect Reconnaissance & Exploitation Probing
SELECT 
    tenant_id,
    user_masked_id,
    request_path,
    http_method,
    status_code,
    COUNT(*) AS scan_attempts,
    MIN(timestamp) AS first_attempt_utc,
    MAX(timestamp) AS last_attempt_utc
FROM dataops_lakehouse_prod.xserver_access_logs
WHERE (year = '2026' AND month = '08')
  AND (
      request_path LIKE '%/.env%'
      OR request_path LIKE '%/wp-config%'
      OR request_path LIKE '%/wp-login%'
      OR status_code IN (401, 403, 404)
  )
GROUP BY tenant_id, user_masked_id, request_path, http_method, status_code
ORDER BY scan_attempts DESC;
```

---

## 📊 Live AWS Verification & Architecture Evidence

The infrastructure and DataSecOps pipeline were validated against live AWS resources using a zero-idle-cost FinOps lifecycle (`terraform apply` ➔ Ingest & Query ➔ `terraform destroy`):

| 1. Glue Table Schema | 2. Partition Projection Config | 3. Athena Query Execution |
| :---: | :---: | :---: |
| ![Glue Schema](assets/aws_glue_proof.png) | ![Glue Advanced Config](assets/aws_glue_proof-adv.png) | ![Athena Query](assets/aws_athena_proof.png) |

### Key Architectural Highlights:
* **Serverless Metastore Scaling:** AWS Glue Partition Projection (`projection.enabled = true` & custom `storage.location.template`) eliminates periodic crawler runs and enables instant partition discovery.
* **DataSecOps & Privacy Compliance:** Client IPs are deterministically pseudonymized at ingestion via Salted SHA-256 (`user_masked_id`), complying with GDPR/APPI without sacrificing analytics capability.
* **FinOps Storage Efficiency:** Columnar Parquet with multi-level Hive partitioning reduced analytical query data scan to **0.50 KB** (sub-kilobyte footprint).

---

## 📦 Requirements & Tooling

| Component | Technology | Version / Specification | Role in Architecture |
| :--- | :--- | :--- | :--- |
| **Language** | Python | `>= 3.11` | Pipeline logic & custom log parsers |
| **ELT Framework** | dlt (data load tool) | `>= 1.0.0` | Ingestion, schema inference, & filesystem load |
| **Storage Engine** | Apache Parquet (PyArrow) | Columnar (Snappy / ZSTD) | Data lake storage with Hive partitioning |
| **Local Query Engine** | DuckDB | `>= 1.0.0` | Zero-copy SQL analytics on local Parquet files |
| **IaC** | HashiCorp Terraform | `>= 1.5.0` (AWS Provider `~> 5.0`) | Declarative infrastructure provisioning |
| **Cloud Target** | AWS S3, Glue, Athena | Hive Partition Projection | Production encrypted data lake & query engine |
| **Testing** | Pytest | Unit & Anomaly Routing Tests | Test coverage for parsing, hashing, & DLQ |
