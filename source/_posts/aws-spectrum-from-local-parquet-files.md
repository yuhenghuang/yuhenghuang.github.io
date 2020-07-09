---
title: Create ASW Spectrum From Local Parquet Files
top: false
cover: false
toc: true
mathjax: False
date: 2020-06-27 09:14:51
password:
summary:
tags:
  - Spectrum
  - Athena
categories: AWS
---

This post gives an example of uploading local parquet files to s3 and using *Spectrum* to make the data accessible from *Redshift*.

Generally it is more common to unload data directly from *Redshift* than create data by oneself. But this example helps deepen understanding of the whole process and thus can be easily applied to other scenarios.

### Requirements

* Python libraries `boto3` and `pyspark` (along with *Spark*) installed
* AWS account that enables you to use *S3*, *Redshift* and *Athena*
* Sufficient priorities of the account


### Create parquet files and upload to *S3*

First step is to prepare data by transforming other data format to `parquet` files.

```python
import boto3
from spark.sql import SparkSession

import os

# create/fetch spark session
ss = SparkSession.builder.master("local[*]").getOrCreate()

# can also load other data formats
# see Spark documents for more details
df = ss.read.csv('/path/to/your/data.csv', header=True, inferSchema=True, encoding='utf-8')

# check if data types are correctly inferred
df.printSchema()

# save to parquet files
# partitionsBy is optional, but it is widely used for large datasets
df.writer.parquet('/path/to/target/directory/', partitionBy=['your_partition_key'])

# upload to s3
# this part can also be done from ASW CLI, and shall be more convenient
# only for consistency of the block
client = boto3.client('s3',
                      aws_access_key_id='your_access_key',
                      aws_secret_access_key='your_access_secret')

for root, dirs, files in os.walk('/path/to/your/target/director'):
  for file in files:
    filepath = os.path.join(root, file)
    client.upload_file(Bucket='your_s3_bucket',
                       Filename=filepath,
                       Key=os.path.join('your_s3_bucket_prefix', filepath))

```


### Create *Spectrum*, or External Table from *S3* files

Executing the following query in *Redshift*.

```SQL
CREATE EXTERNAL TABLE table_name (
  var1 VARCHAR(32),
  var2 INTEGER(36),
  var3 TIMESTAMP WITHOUT TIME ZONE
)
-- PARTITIONED BY (var4 INTEGER)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '|'
STORED AS parquet
  LOCATION 's3://your_prefix/your_dir_name/'
;

SELECT
  *
FROM table_name
LIMIT 100
;

```

If one didn't specify the partition key, the above sql query is sufficient to load the data in s3 to the external table.

The case is that if partition key is specified, the sql query would only construct an **Empty** external table waiting to be feeded with data. 

In case the test query can't fetch any rows, it's not the table that is broken. It is the process that remains unfinished.

### Feed External Table with data

The command that AWS prepared for us is way too simple, only one line.

```SQL
MSCK REPAIR TABLE table_name;

```

This will automatically load all partitions in specified path to the external table.

The command might be straightforward and convenient when initializing a new table, but when updating the table by newly added partitions, there is a more explicit way of doing this.

Supposedly now you already added a new partition to *S3*.

```SQL
-- new_key_value is the partition_key's value that you wanted to add to the table
ALTER TABLE table_name ADD
PARTITION (partition_key=$new_key_value)
LOCATION 's3://your_prefix/your_dir_name/partition_key=$new_partition_key/'

```

<sup>The ways of adding partition, either unloading from *Redshift* or uploading from local, might be different, but the data must be put in the right place. Writing a script for updating external table is a good practice to avoid mistakes.<sup>

### Reference

[SPECTRUM EXTERNAL TABLES](https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-tables.html)

[MSCK REPAIR TABLE](https://docs.aws.amazon.com/athena/latest/ug/msck-repair-table.html)

[ADD PARTITION](https://docs.aws.amazon.com/athena/latest/ug/alter-table-add-partition.html)