**For Iceberg tables.** Snowflake connects to Glue through a **catalog integration**, then reads the data directly from S3. Glue tracks the table metadata, and the data stays in your bucket.

Regular Glue tables (Hive-style CSV, JSON, or plain Parquet) can't be read this way. For those, use a Snowflake external table on the same S3 path, or convert them to Iceberg.

Glue's **Iceberg REST endpoint** is recommended, shown below. 
Replace the account ID, region, bucket, role, and `sales_db`.



## 1. Create the IAM role

Create `snowflake-glue-role` with this permissions policy (to access AWS GLUE Catalog and to access S3 bucket with parquet files):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["glue:GetCatalog", "glue:GetDatabase", "glue:GetDatabases", "glue:GetTable", "glue:GetTables"],
      "Resource": [
        "arn:aws:glue:us-east-1:123456789012:catalog",
        "arn:aws:glue:us-east-1:123456789012:database/sales_db",
        "arn:aws:glue:us-east-1:123456789012:table/sales_db/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::my-bucket/iceberg/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::my-bucket"
    }
  ]
}
```



## 2. Create the Snowflake objects

```sql
USE ROLE ACCOUNTADMIN;

-- Access to the Iceberg files in S3
CREATE OR REPLACE EXTERNAL VOLUME glue_iceberg_vol
  STORAGE_LOCATIONS = ((
    NAME = 's3-iceberg'
    STORAGE_PROVIDER = 'S3'
    STORAGE_BASE_URL = 's3://my-bucket/iceberg/'
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-glue-role'
  ))
  ALLOW_WRITES = FALSE;

-- Connection to Glue
CREATE OR REPLACE CATALOG INTEGRATION glue_rest_int
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  CATALOG_NAMESPACE = 'sales_db'
  REST_CONFIG = (
    CATALOG_URI = 'https://glue.us-east-1.amazonaws.com/iceberg'
    CATALOG_API_TYPE = AWS_GLUE
    CATALOG_NAME = '123456789012'      -- AWS account ID
  )
  REST_AUTHENTICATION = (
    TYPE = SIGV4
    SIGV4_IAM_ROLE = 'arn:aws:iam::123456789012:role/snowflake-glue-role'
    SIGV4_SIGNING_REGION = 'us-east-1'
  )
  ENABLED = TRUE;

-- Get the Snowflake principals and external IDs for the trust policy
DESC EXTERNAL VOLUME glue_iceberg_vol;       -- STORAGE_AWS_IAM_USER_ARN, STORAGE_AWS_EXTERNAL_ID
DESC CATALOG INTEGRATION glue_rest_int;      -- API_AWS_IAM_USER_ARN, API_AWS_EXTERNAL_ID
```

## 3. Update the role's trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "<STORAGE_AWS_IAM_USER_ARN>" },
      "Action": "sts:AssumeRole",
      "Condition": { "StringEquals": { "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>" } }
    },
    {
      "Effect": "Allow",
      "Principal": { "AWS": "<API_AWS_IAM_USER_ARN>" },
      "Action": "sts:AssumeRole",
      "Condition": { "StringEquals": { "sts:ExternalId": "<API_AWS_EXTERNAL_ID>" } }
    }
  ]
}
```

Then test both connections:

```sql
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('glue_iceberg_vol');
SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('glue_rest_int');
```







## 4. Choose how to expose the tables

**Option A: link one table**

```sql
CREATE OR REPLACE ICEBERG TABLE my_db.schema_b.orders
  EXTERNAL_VOLUME = 'glue_iceberg_vol'
  CATALOG = 'glue_rest_int'
  CATALOG_NAMESPACE = 'sales_db'
  CATALOG_TABLE_NAME = 'orders'
  AUTO_REFRESH = TRUE;

SELECT * FROM my_db.schema_b.orders LIMIT 10;
```






**Option B: link the whole Glue database**

A **catalog-linked database** automatically discovers tables in the allowed namespaces:

```sql
CREATE DATABASE glue_linked_db
  LINKED_CATALOG = (
    CATALOG = 'glue_rest_int',
    ALLOWED_NAMESPACES = ('sales_db')
  )
  EXTERNAL_VOLUME = 'glue_iceberg_vol';

SELECT SYSTEM$CATALOG_LINK_STATUS('glue_linked_db');

-- Glue names are lowercase, so quote them
SELECT * FROM glue_linked_db."sales_db"."orders" LIMIT 10;
```






Use **A** for a few controlled tables and **B** when new Glue tables should appear automatically.

## Things to watch for

- **Lake Formation:** if it governs these tables, the role also needs Lake Formation permissions, or calls can fail even with correct IAM policies.
- **Encryption:** with SSE-KMS, add `kms:Decrypt` on the key.
- **Freshness:** Snowflake sees changes after a metadata refresh, so `AUTO_REFRESH` can lag slightly. Run `ALTER ICEBERG TABLE ... REFRESH` to force one.
- **Region:** keep Snowflake and the S3 bucket in the same region when possible to avoid data transfer charges and added latency.
- **Access:** grant `SELECT` to roles such as ROLE_A as usual. Limit `USAGE` on the integration and external volume to admin or deployment roles.