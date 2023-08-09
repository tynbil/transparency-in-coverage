# Transparency in Coverage

## What is this data?

Beginning July 1st, 2022, CMS (Centers for Medicare and Medicaid) requires all healthcare payers and insurance plans to publish their negotiated rates with healthcare providers (doctors, hospitals, pharmacies, etc.) every month. This has the potential to _greatly_ change Americans' relationship with healthcare since, up to now, the real price has been hidden from view.

The source data files are generally available on the web. However, they are often many 10's of gigabytes per file and, in total across all payers, is around 1 petabyte of data per month. Downloading, storing, processing, and searching this dataset is a daunting task.

We at [Tynbil](https://tynbil.com) have been hard at work these past few months refining a processing pipeline to reduce a payer's terabytes of source data down to an easily queryable 10's of gigabytes. This repository points to a sample of the processed Parquet files which you can download and explore.

If you would like to see the full list of payers we process each month (a list which is constantly growing), it's at the bottom of [https://tynbil.com/payers](https://tynbil.com/payers).

## How can I use this data?

We are offering these processed files under the [CC BY-NC-SA license](https://creativecommons.org/licenses/by-nc-sa/4.0/) which means that you can use them for any noncommercial use, must credit Tynbil in any published work, and must use the same or similar license if you republish the files.

If you are interested in licensing these files for commercial use, please [contact us](mailto:brendan@tynbil.com).

## Where is the data?

The various `sources.txt` files in this repository contain links to the parquet files. There may also be an associated `notes.txt` file which details any unexpected issues with the data import you should be aware of.

## How are the data structured?

Now for the fun part! The full schema for the source files is available from [CMS's Price Transparency Guide](https://github.com/CMSgov/price-transparency-guide/). The Tynbil parquet files are normalized versions of the same data so when looking up the meaning of a field (say `negotiation_arrangement`) you can refer to the same field in [CMS's in-network schema](https://github.com/CMSgov/price-transparency-guide/tree/master/schemas/in-network-rates).

For exploration, we recommend [Clickhouse Local](https://clickhouse.com/blog/extracting-converting-querying-local-files-with-sql-clickhouse-local) as it has excellent parquet support and is pleasantly fast.

Let's start with Highmark Pennsylvania's July 2023 data. First, we'll create a view to make examining the prices a little easier.

```sql
:) CREATE VIEW prices AS
SELECT
    p.id AS price_id,
    c.billing_code_type AS billing_code_type,
    c.billing_code AS billing_code,
    p.negotiated_rate AS negotiated_rate,
    c.billing_code_type_version AS billing_code_type_version,
    c.negotiation_arrangement AS negotiation_arrangement,
    pm.billing_class AS billing_class,
    pm.negotiated_type AS negotiated_type,
    pm.billing_code_modifier AS billing_code_modifier,
    pm.service_code AS service_code,
    c.bundled_codes AS bundled_codes,
    c.covered_services AS covered_services
FROM file('prices.parquet') AS p
LEFT JOIN file('price_metas.parquet') AS pm ON pm.id = p.price_meta_id
INNER JOIN file('codes.parquet') AS c ON p.code_id = c.id;

:) SELECT *
FROM prices
WHERE billing_code = '99214'
AND billing_class = 'professional'
LIMIT 10
FORMAT Vertical;

Row 1:
──────
price_id:                  -2800171617385289562
billing_code_type:         CPT
billing_code:              99214
negotiated_rate:           100.39
billing_code_type_version: 2023
negotiation_arrangement:   ffs
billing_class:             professional
negotiated_type:           derived
billing_code_modifier:     ['25','95']
service_code:              ['10','11']
bundled_codes:             ᴺᵁᴸᴸ
covered_services:          ᴺᵁᴸᴸ
```

From this we can see some of the actual reimbursement rates for an Established Patient Evaluation and Management Visit, 30-39 minutes.

We've restricted the `billing_class` to `professional` (as opposed to `institutional`) to see the physician reimbursement, not the hospital or clinic fee.

The service codes describe where/how the service was delivered. There's a [frustratingly formatted PDF](https://www.cms.gov/medicare/medicare-fee-for-service-payment/physicianfeesched/downloads/website-pos-database.pdf) CMS maintains which details each service code. Service codes 10 and 11 are for telehealth and in-office. Highmark, like most of the Blues, does a pretty good job of setting these correctly. Other payers seem to ignore this field. 🤷‍♂️

Billing code modifier 95 is for a synchronous telemedicine visit. Modifier 25 means this Evaluation and Management visit was likely unplanned and tacked on to a different procedure. This seems like a special case.

As you work with this data, you'll find a lot of rates like this which aren't particularly useful as benchmarks since they're so specific. Let's drill down a bit to find the real bread and butter rates for a planned office visit.

```sql
:) SELECT *
FROM prices
WHERE (billing_code = '99214') AND (billing_class = 'professional') AND empty(billing_code_modifier) AND has(service_code, '11')
LIMIT 10
FORMAT Vertical;

Row 1:
──────
price_id:                  5370871758677784184
billing_code_type:         CPT
billing_code:              99214
negotiated_rate:           123.09
billing_code_type_version: 2023
negotiation_arrangement:   ffs
billing_class:             professional
negotiated_type:           derived
billing_code_modifier:     []
service_code:              ['11']
bundled_codes:             ᴺᵁᴸᴸ
covered_services:          ᴺᵁᴸᴸ

Row 2:
──────
price_id:                  8414204045595673939
billing_code_type:         CPT
billing_code:              99214
negotiated_rate:           103.27
billing_code_type_version: 2023
negotiation_arrangement:   ffs
billing_class:             professional
negotiated_type:           derived
billing_code_modifier:     []
service_code:              ['11','20']
bundled_codes:             ᴺᵁᴸᴸ
covered_services:          ᴺᵁᴸᴸ
```

Much better. Now let's find who gets to bill these rates.

```sql
SELECT DISTINCT
    pgf.price_id AS price_id,
    arrayJoin(npis) AS npi,
    pgf.file_id as file_id
FROM file('price_group_file.parquet') AS pgf
INNER JOIN file('groups.parquet') AS g ON g.id = pgf.group_id
WHERE pgf.price_id IN (5370871758677784184, 8414204045595673939)
ORDER BY
    1 ASC,
    2 ASC;

┌────────────price_id─┬────────npi─┬──────────────file_id─┐
│ 5370871758677784184 │ 1023077856 │ 13197186521333479035 │
│ 5370871758677784184 │ 1023077856 │ 12740363278053471596 │
└─────────────────────┴────────────┴──────────────────────┘
┌────────────price_id─┬────────npi─┬─────────────file_id─┐
│ 8414204045595673939 │ 1538361621 │ 2575527443172353476 │
│ 8414204045595673939 │ 1538361621 │ 1576220056349810356 │
│ 8414204045595673939 │ 1689882110 │ 2575527443172353476 │
│ 8414204045595673939 │ 1689882110 │ 1576220056349810356 │
└─────────────────────┴────────────┴─────────────────────┘
```

The first rate of $123.09 per office visit is a [rheumatologist](https://npiregistry.cms.hhs.gov/provider-view/1023077856) in Allentown, PA. The second rate of $103.27 is for an RN and PA in West Virginia.

Let's see which plans these rates belong to by looking up their file_id's.

```sql
:) SELECT DISTINCT
    f.id AS file_id,
    f.filename AS filename,
    p.plan_name AS plan_name
FROM file('files.parquet') AS f
INNER JOIN file('plan_file.parquet') AS pf ON pf.file_id = f.id
INNER JOIN file('plans.parquet') AS p ON p.plan_id = pf.plan_id
WHERE f.id IN (13197186521333479035, 12740363278053471596, 2575527443172353476, 1576220056349810356)

┌──────────────file_id─┬─filename───────────────────────────────────────────────┬─plan_name─────────────────────────────────────────────────────┐
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ HDHP BCBS PPO:Highmark, Inc.                                  │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ HDHP BCBS PPO:Highmark, Inc.:378000008010                     │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ HDHP PPO BLUE:Highmark, Inc.:378000008010                     │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ PPO BLUE:Highmark, Inc.:378000008010                          │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ Community Blue Flex HDHP PPO:Highmark, Inc.:378000000088      │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ Community Blue Flex HDHP PPO:Highmark, Inc.:378000000089      │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ Community Blue Flex HDHP PPO:Highmark, Inc.:378000000106      │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ HDHP PPO BLUE:Highmark, Inc.:378000000328                     │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ PPO BLUE:Highmark, Inc.:378000000328                          │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ BLUE CARD:Highmark, Inc.:378000008010                         │
│ 13197186521333479035 │ 2023-07_378_49T0_in-network-rates_01_of_06.json.gz     │ Community Blue Flex HDHP PPO:Highmark, Inc.:378000008010      │
```

And this is where things start to get a little messy. These rates are in Highmark's PPO network and appear to be in most, if not all, the PPO variations. Unfortunately, there is no standard way of referring to a payer's network, but we at Tynbil can help you with the mapping if you wish.

## Where's the rest of the data?

You mean for all the other 100+ payers in this great big country of ours? We process them each month! The current list of processed payers is available on [our website](https://tynbil.com/payers).

If there is a particular payer you are interested in, please [let us know](mailto:brendan@tynbil.com).
