# L02 · Exercise 1: my own bronze bucket

- **Who:** Till Lüder (`ME=luder`)
- **Date:** 2026-10-09
- **Where:** bucket `gs://data-engineering-i1-luder` (`europe-west1`, Standard,
  uniform access), BigQuery dataset `data-engineering-i1.luder_bronze`
- **Data:** `legacy_csv` v1.0.0. Parquet written by pandas 3.0.6 and pyarrow
  25.0.1, snappy codec

## Step 2: CSV vs Parquet

| File | Bytes |
|---|---|
| `order_items.csv` | 10,373,925 |
| `order_items.parquet` (snappy) | 3,546,854 |
| **Ratio** | **2.92×**: the Parquet file is 34 % of the CSV |

## Step 5: the error message

I could not reach the expected error. Creating the service account was refused
first:

```
Permission 'iam.serviceAccounts.create' denied on resource
'//cloudresourcemanager.googleapis.com/projects/data-engineering-i1'
```

My account cannot create service accounts in the course project. Once the
teacher grants that right, I rerun step 5 and record the
`does not have storage.objects.create access` message here.

## Step 7: one month, two layouts

August 2026: 9,893 order lines, 33,231 units. Query: units per day for
1–7 August. Both layouts return the same totals (e.g. 1,031 units on
2026-08-01).

| # | What | `month_by_day` (`dt`) | `month_by_day_product` (`dt`, `p_id`) | Factor |
|---|---|---|---|---|
| 1 | Objects | 31 | 8,649 | 279× |
| 2 | Total bytes | 372,214 | 32,662,322 | 88× |
| 3 | Query elapsed time (`JOBS_BY_USER`, 3 runs) | 300 ms avg (335, 270, 296) | 1,114 ms avg (1,172, 895, 1,274) | 3.7× |
| 4 | Bytes processed | 17,056 | 17,056 | 1× |

Also measured:

| What | `month_by_day` | `month_by_day_product` |
|---|---|---|
| Engine work (`total_slot_ms`, average) | 487 | 155,408 (~320×) |
| Upload time (`gcloud storage cp -r`) | 4.6 s | 18.9 s |
| Listing every object (`gcloud storage ls -r`) | seconds | several minutes |

`JOBS_BY_PROJECT` was refused (it needs `bigquery.jobs.listAll`), so the times
come from `INFORMATION_SCHEMA.JOBS_BY_USER`, which holds the same fields for
my own jobs.

## Step 8: what it costs

| Cost | Calculation | Amount |
|---|---|---|
| Monthly storage of everything I created | 73,310,514 bytes = 0.0683 GiB × $0.020 | **$0.0014 per month** |
| Step 7 write operations | (31 + 8,649) objects ÷ 1,000 × $0.005 | **$0.0434**, once |
| **Larger** | | **The writes, about 32× one month of storage** |

## My partitioning rule

> *To write myself, in two sentences: when is another partition worth adding,
> and when does it start costing? Quote one number from above.*
