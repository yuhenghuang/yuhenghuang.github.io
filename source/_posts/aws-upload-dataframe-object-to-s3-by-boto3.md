---
title: Upload DataFrame Object to AWS S3 Directly
top: false
cover: false
toc: true
mathjax: false
date: 2020-07-04 11:53:54
password:
summary:
tags:
  - S3
  - IO
categories: AWS
---

When uploading pandas dataframe to *S3*, there are times that it's better to upload the data object directly than to temporarily save the file and upload the temporary file. This post suggests one relatively sufficient way to communicate with *S3* directory without dumping data to hard disk, thus all the operations are done in memory.

### Upload data objects

* Without `gzip` compression

```python
import boto3
import pandas as pd

import gzip
from io import StringIO, BytesIO

client = boto3.client('s3', aws_access_key_id='aws_id', aws_secret_access_key='aws_key')

df = read.csv('/path/to/your/file') #...and other parameters

buff_str = StringIO()

# save .tsv file to string buffer
df.to_csv(buff_str, sep='\t', index=False) #...and other parameters

buff_byte = BytesIO(buff_str.getvalue().encode('uft-8'))

# why buff_byte.seek(0) does not matter here...
buff_byte.seek(0)
client.upload_fileobj(Bucket='your_bucket', Key='/your/s3/key.tsv', Fileobj=buff_byte)

```

* With `gzip` compression


```python
buff_str = StringIO()

df.to_csv(buff_str, sep='\t', index=False)

buff_byte = BytesIO()

with gzip.open(buff_byte, 'wb') as f:
  f.write(buff_str.getvalue().encode('uft-8'))

# this time the reset of pointing can not be ignored...
buff_byte.seek(0)
client.upload_fileobj(Bucket='your_bucket', Key='/your/s3/key.tsv.gz', Fileobj=buff_byte)

```

### Download data objects

* Without `gzip` compression

```python

buff_byte = BytesIO()
client.download_fileobj(Bucket='your_bucket', Key='/your/s3/key.tsv', Fileobj=buff_byte)

# reset the pointer of the buffer
buff_byte.seek(0)

df = pd.read_csv(buff_byte, sep='\t', encoding='utf-8') # and other parameters

```

* With `gzip` compression

```python

buff_byte = BytesIO()
client.download_fileobj(Bucket='your_bucket', Key='/your/s3/key.tsv.gz', Fileobj=buff_byte)

# reset the pointer of the buffer
buff_byte.seek(0)

df = pd.read_csv(buff_byte, compression='gzip', sep='\t', encoding='utf-8') # and other parameters
```