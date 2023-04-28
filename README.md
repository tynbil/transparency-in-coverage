# Transparency in Coverage

## What is this data?

Beginning July 1st, 2022, CMS (Centers for Medicare and Medicaid) requires all healthcare payers and insurance plans to publish their negotiated rates with healthcare providers (doctors, hospitals, pharmacies, etc.) every month. This has the potential to _greatly_ change Americans' relationship with healthcare since, up to now, the real price has been hidden from view.

The source data files are generally available on the web. However, they are often many 10's of gigabytes per file and, in total across all payers, are over 100 terabytes of data. Downloading, storing, processing, and searching this dataset is a daunting task.

We at Tynbil have been hard at work these past few months refining a processing pipeline to reduce a payer's terabytes of source data down to an easily queryable 10's of gigabytes. This repository points to the processed parquet files which you can download and use.

## How can I use this data?

We are offering these processed files under the [CC BY-NC-SA license](https://creativecommons.org/licenses/by-nc-sa/4.0/) which means that you can use them for any noncommercial use, must credit Tynbil in any published work, and must use the same or similar license if you republish the files.

If you are interested in licensing these files for commercial use, please [contact us](mailto:brendan@tynbil.com).

## Where is the data?

The various `sources.txt` files in this repository contain links to the parquet files. There is also an associated `notes.txt` file which details any unexpected issues with the data import you should be aware of.

## How are the data structured?

Now for the fun part! The full schema for the source files is available from [CMS's Price Transparency Guide](https://github.com/CMSgov/price-transparency-guide/). The Tynbil parquet files are normalized versions of the same data so when looking up the meaning of a field (say `negotiation_arrangement`) you can refer to the same field in [CMS's in-network schema](https://github.com/CMSgov/price-transparency-guide/tree/master/schemas/in-network-rates).

For exploration, we recommend [DuckDB](https://duckdb.org/) as it has excellent parquet support and is pleasantly fast.

Let's start with the `agg_prices.parquet` file from Cigna's April 2023 data.

```sql
D select * from agg_prices.parquet limit 10;
┌───────┬─────────┬───────────────┬─────────────────┬─────────────────┐
│  id   │ code_id │ price_meta_id │ negotiated_rate │ negotiated_type │
│ int32 │  int32  │     int32     │  decimal(15,2)  │     varchar     │
├───────┼─────────┼───────────────┼─────────────────┼─────────────────┤
│   114 │   11489 │            58 │          346.89 │ fee schedule    │
│   116 │   11489 │            58 │          432.13 │ fee schedule    │
│   163 │   11489 │            58 │          105.03 │ fee schedule    │
│   211 │   11489 │            58 │          677.02 │ fee schedule    │
│   238 │   11489 │            58 │          518.35 │ fee schedule    │
│   274 │   11489 │            58 │          470.32 │ fee schedule    │
│   313 │   11489 │            58 │          478.69 │ fee schedule    │
│   349 │   11489 │            58 │          359.43 │ fee schedule    │
│   355 │   11489 │            58 │          204.93 │ fee schedule    │
│   356 │   11489 │            58 │          526.02 │ fee schedule    │
├───────┴─────────┴───────────────┴─────────────────┴─────────────────┤
│ 10 rows                                                   5 columns │
└─────────────────────────────────────────────────────────────────────┘
```

These are all negotiated rates for `billing_code_id` 11489. To find out what code that it, query `agg_codes.parquet`

```sql
D select * from agg_codes.parquet where id = 11489;
┌───────┬──────────────┬───────────────────┬───────────────────────────┬─────────────────────────┐
│  id   │ billing_code │ billing_code_type │ billing_code_type_version │ negotiation_arrangement │
│ int32 │   varchar    │      varchar      │          varchar          │         varchar         │
├───────┼──────────────┼───────────────────┼───────────────────────────┼─────────────────────────┤
│ 11489 │ 95816        │ CPT               │ 2023                      │ ffs                     │
└───────┴──────────────┴───────────────────┴───────────────────────────┴─────────────────────────┘
```

CPT[^1] code 95816 is for a routine EEG. Let's look at the `price_meta_id` 58.

[^1]: CPT codes are registered trademarks of the American Medical Association which makes a lot of money from licensing their use and aggressively demands payment. This is why we don't provide a human readable description of the code. Sorry.

```sql
D select * from agg_price_metas.parquet where id = 58;
┌───────┬───────────────┬─────────────────┬──────────────────────┬────────────────────────────────────────────────────────────────────────────────┐
│  id   │ billing_class │ expiration_date │ billing_code_modif…  │                                  service_code                                  │
│ int32 │    varchar    │      date       │      varchar[]       │                                   varchar[]                                    │
├───────┼───────────────┼─────────────────┼──────────────────────┼────────────────────────────────────────────────────────────────────────────────┤
│    58 │ professional  │ 9999-12-31      │ [TC]                 │ [01, 02, 03, 04, 05, 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19,…  │
└───────┴───────────────┴─────────────────┴──────────────────────┴────────────────────────────────────────────────────────────────────────────────┘
```

The `billing_class` can either be `professional` or `institutional`. In pre-computer times, physicians submitted claims on a CMS-1500 form while hospitals, nursing homes, etc. submitted on a UB-04 form. The number and size of the little boxes on those forms now dictate how we bill for healthcare services.

The `TC` modifier stands for "technical component" which means there was just a bunch of button pushing on the EEG machine and very little that required a medical degree. Later on, when the doctor points to a spike on the chart and says "that must be when you were thinking about lunch!" she'll bill for a separate encounter.

The service codes describe where/how the service was delivered. There's a [frustratingly formatted PDF](https://www.cms.gov/medicare/medicare-fee-for-service-payment/physicianfeesched/downloads/website-pos-database.pdf) CMS maintains which details each service code. Cigna likes listing all of them all the time. Other payers rarely list any. 🤷‍♂️

Returning to the price distribution, we see that someone's getting paid $526.02 for this EEG while someone else only gets $105.03 (suckers). Let's see who's getting paid what.

```sql
D select * from read_parquet('./agg_price_group_file_*.parquet') where price_id in (163, 356) order by price_id;
┌──────────┬──────────┬──────────┐
│ price_id │ group_id │ file_ids │
│  int32   │  int32   │ int32[]  │
├──────────┼──────────┼──────────┤
│      163 │    52548 │ [64]     │
│      356 │    36346 │ [64]     │
└──────────┴──────────┴──────────┘
```

Looking up the two `group_id`'s, we get a list of NPIs.

```sql
D select * from agg_groups.parquet where id in (52548, 36346);
┌───────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  id   │                                                                  npis                                                                   │
│ int32 │                                                                 int32[]                                                                 │
├───────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 36346 │ [1003266313, 1194353763, 1225317795, 1386669497, 1396213476, 1396354320, 1427032127, 1447245840, 1467403287, 1477821445, 1497861983, …  │
│ 52548 │ [1003561234, 1003895095, 1013008366, 1013149079, 1013205947, 1013995034, 1013995042, 1013996941, 1023320645, 1023471406, 1023767753, …  │
└───────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

NPIs are National Provider Identifiers and are assigned by the gov't to every healthcare professional. In group 36346, we can look up [NPI 1003266313](https://npiregistry.cms.hhs.gov/provider-view/1003266313) who appears to be employed by Prevea Ashwaubenon Health Center in Green Bay, WI. (Go Pack!) Other NPIs in the list are employed at Prevea's other clinics.

In group 52548, we see several NPIs who work at Mankato Clinic which appears to be associated with the Mayo Clinic. They get paid a more reasonable $105.03 for an EEG.

Next we have the `file_ids` column which tells us the source file this rate came from.

```sql
D select * from agg_files.parquet where id = 64;
┌───────┬─────────────────────────────────────────────────────────────────────────────────────┐
│  id   │                                      filename                                       │
│ int32 │                                       varchar                                       │
├───────┼─────────────────────────────────────────────────────────────────────────────────────┤
│    64 │ 2023-04-01_upper-midwest-affiliation-partner_cigna-open-access_in-network-rates.zip │
└───────┴─────────────────────────────────────────────────────────────────────────────────────┘
```

From the file name we can surmise that this file is part of several different plans. Let's see which plans reference this file.

```sql
D select
>   p.*
> from
>   agg_plan_file.parquet pf
>   join agg_plans.parquet p on p.id = pf.plan_id
> where
>   file_id = 64
> limit
>   10;
┌───────┬──────────────────────────────────────────────────────────────┬──────────────┬───────────┬──────────────────┐
│  id   │                          plan_name                           │ plan_id_type │  plan_id  │ plan_market_type │
│ int64 │                           varchar                            │   varchar    │  varchar  │     varchar      │
├───────┼──────────────────────────────────────────────────────────────┼──────────────┼───────────┼──────────────────┤
│  1894 │ PATHWELL PPO Nicolson Construction, Inc.                     │ ein          │ 900026476 │ group            │
│  1895 │ PATHWELL PPO Method Studio                                   │ ein          │ 800138187 │ group            │
│  1896 │ PATHWELL PPO MFOC Holdco, Inc.                               │ ein          │ 454078144 │ group            │
│  1897 │ PATHWELL PPO Signifyd, Inc.                                  │ ein          │ 453073312 │ group            │
│  1899 │ PATHWELL PPO Renren U.S. HoldCo, Inc                         │ ein          │ 822062432 │ group            │
│  1900 │ PATHWELL PPO Parker-Migliorini International, LLC            │ ein          │ 453860089 │ group            │
│  1901 │ PATHWELL PPO Acme Construction Supply Co., Inc.              │ ein          │ 930805825 │ group            │
│  1902 │ PATHWELL PPO New Orleans-Baton Rouge Steamship Pilots Assoc. │ ein          │ 720389381 │ group            │
│  1904 │ PATHWELL PPO CCI Network Services, LLC                       │ ein          │ 870709990 │ group            │
│  1919 │ PATHWELL PPO BirdEye, Inc.                                   │ ein          │ 454836337 │ group            │
├───────┴──────────────────────────────────────────────────────────────┴──────────────┴───────────┴──────────────────┤
│ 10 rows                                                                                                  5 columns │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

We take this moment to remind the general public that most folks with employer sponsored health care are _insured_ by their employer, not Cigna, Blue Cross, Humana, etc. Those companies merely administer the billing and negotiate rates on behalf of the employers which is why we normally refer to them as "payers". In this instance, Cigna runs the plans for these employers and includes the upper midwest open access network in all of them.

## Where's the rest of the data?

You mean for all the other 100+ insurers and payers in this great big country of ours? We're working on it! We decided to publish this sample from April, 2023, as a start and we'll be ramping up our ingestion pipeline to collect as much as we can for May, 2023, onwards.

If there is a particular payer you are interested in, please [let us know](mailto:brendan@tynbil.com).
