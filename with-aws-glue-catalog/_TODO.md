
**We can query AWS Glue Catalog iceberg table `sales_iceberg_table` directly from Snowflake** without copying or ingesting the data into Snowflake's internal storage.

Because AWS Glue manages the Iceberg metadata and S3 holds the Parquet files, Snowflake treats it as an **externally managed Apache Iceberg table** (read-only in Snowflake, with metadata updates synced from Glue).

---

### Architecture Overview

```text
+-------------------------------------------------------------------+
| AWS S3: s3://<BUCKET>/      |
|   └── sales_iceberg_warehouse/ (Parquet data + Iceberg metadata)  |
+-------------------------------------------------------------------+
             ▲                                      ▲
             │ (Reads S3 via External Volume)       │ (Metadata pointer)
             │                                      │
+------------+--------------+          +------------+---------------+
| Snowflake                 |          | AWS Glue Data Catalog      |
|  - External Volume        | ───────▶ |  - DB: db-output           |
|  - Catalog Integration    |  (Reads  |  - Table: sales_iceberg_   |
|  - Iceberg Table (Read)   |  Catalog)|           table            |
+---------------------------+          +----------------------------+

```

---

### Step-by-Step Setup

To connect Snowflake to your table in `us-east-2` (`db-output.sales_iceberg_table`), configure two trust relationships in AWS IAM and two integration objects in Snowflake:

#### 1. Create an IAM Policy & Role for Snowflake

In AWS IAM (`us-east-2` / global), create a role (e.g., `snowflake-iceberg-role`) with permissions to read both your S3 warehouse folder and the Glue Data Catalog:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3IcebergRead",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::<BUCKET>",
                "arn:aws:s3:::<BUCKET>/sales_iceberg_warehouse/*"
            ]
        },
        {
            "Sid": "AllowGlueCatalogRead",
            "Effect": "Allow",
            "Action": [
                "glue:GetDatabase",
                "glue:GetDatabases",
                "glue:GetTable",
                "glue:GetTables"
            ],
            "Resource": [
                "arn:aws:glue:us-east-2:<ACCOUNT_ID>:catalog",
                "arn:aws:glue:us-east-2:<ACCOUNT_ID>:database/db-output",
                "arn:aws:glue:us-east-2:<ACCOUNT_ID>:table/db-output/sales_iceberg_table"
            ]
        }
    ]
}

```

*(Temporarily set the role's Trust Policy to your own AWS account root ARN; you will update it with Snowflake's generated IAM user ARN and External ID in Step 3.)*

---

#### 2. Create the External Volume & Catalog Integration in Snowflake

Run these commands in a Snowflake worksheet using a role with `ACCOUNTADMIN` (or `CREATE EXTERNAL VOLUME` / `CREATE INTEGRATION` privileges):

**A. Create the External Volume (for S3 file access):**

```sql
CREATE OR REPLACE EXTERNAL VOLUME iceberg_s3_vol
  STORAGE_LOCATIONS = (
    (
      NAME = 'us-east-2-iceberg-s3'
      STORAGE_PROVIDER = 'S3'
      STORAGE_BASE_URL = 's3://<BUCKET>/sales_iceberg_warehouse/'
      STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<ACCOUNT_ID>:role/snowflake-iceberg-role'
    )
  )
  ALLOW_WRITES = FALSE;

```

**B. Create the Catalog Integration (for AWS Glue metadata access):**

```sql
CREATE OR REPLACE CATALOG INTEGRATION glue_iceberg_cat_int
  CATALOG_SOURCE = GLUE
  TABLE_FORMAT = ICEBERG
  GLUE_AWS_ROLE_ARN = 'arn:aws:iam::<ACCOUNT_ID>:role/snowflake-iceberg-role'
  GLUE_CATALOG_ID = '<ACCOUNT_ID>'
  GLUE_REGION = 'us-east-2'
  CATALOG_NAMESPACE = 'db-output'
  ENABLED = TRUE;

```

---

#### 3. Complete the AWS IAM Trust Policy

Snowflake provisions an AWS IAM user and External IDs for both objects. Retrieve them in Snowflake:

```sql
DESCRIBE EXTERNAL VOLUME iceberg_s3_vol;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID

DESCRIBE CATALOG INTEGRATION glue_iceberg_cat_int;
-- Copy GLUE_AWS_IAM_USER_ARN and GLUE_AWS_EXTERNAL_ID

```

Update the **Trust Relationship** of `snowflake-iceberg-role` in AWS IAM to allow Snowflake's IAM user ARN(s) with the returned External IDs in the `sts:ExternalId` condition list:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": [
                        "<STORAGE_AWS_EXTERNAL_ID>",
                        "<GLUE_AWS_EXTERNAL_ID>"
                    ]
                }
            }
        }
    ]
}

```

---

#### 4. Register and Query the Iceberg Table in Snowflake

Create the Iceberg table pointer in your Snowflake database/schema:

```sql
CREATE OR REPLACE ICEBERG TABLE sales_iceberg_table
  EXTERNAL_VOLUME = 'iceberg_s3_vol'
  CATALOG = 'glue_iceberg_cat_int'
  CATALOG_TABLE_NAME = 'sales_iceberg_table'
  CATALOG_NAMESPACE = 'db-output'
  AUTO_REFRESH = TRUE;

```

Now you can query it like any native Snowflake table:

```sql
SELECT
    code,
    "date",
    customer,
    price,
    year,
    month,
    day
FROM sales_iceberg_table
WHERE year = '2026'
LIMIT 10;

```

### Key Behaviors to Keep in Mind

* **Read-Only in Snowflake:** Because AWS Glue is the catalog owner, DML writes (`INSERT`, `UPDATE`, `DELETE`) must still happen on the AWS side (Athena, Glue ETL, or EMR).
* **Automatic Syncing:** Setting `AUTO_REFRESH = TRUE` tells Snowflake to poll the Glue Catalog (every 30 seconds by default) so new partitions or snapshots written via Athena appear automatically in Snowflake.

