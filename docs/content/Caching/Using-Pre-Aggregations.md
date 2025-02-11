---
title: Using Pre-Aggregations
permalink: /caching/using-pre-aggregations
category: Caching
menuOrder: 3
---

Pre-aggregations is a powerful way to speed up your Cube.js queries. There are
many configuration options to consider. Please make sure to also check [this
Pre-Aggregations page in the data schema section][ref-preaggs].

## Refresh Strategy

Refresh strategy can be customized by setting the
[refreshKey][ref-preaggs-refresh-key] property for the pre-aggregation.

The default value of `refreshKey` is `every: '1 hour'`. It can be redefined
either by providing SQL:

```javascript
cube(`Orders`, {
  // ...

  preAggregations: {
    amountByCreated: {
      type: `rollup`,
      measureReferences: [amount],
      timeDimensionReference: createdAt,
      granularity: `month`,
      refreshKey: {
        sql: `SELECT MAX(created_at) FROM orders`,
      },
    },
  },
});
```

Or by providing a refresh time interval:

```javascript
cube(`Orders`, {
  // ...

  preAggregations: {
    amountByCreated: {
      type: `rollup`,
      measureReferences: [amount],
      timeDimensionReference: createdAt,
      granularity: `month`,
      refreshKey: {
        every: `12 hour`,
      },
    },
  },
});
```

## Background Refresh

You can refresh pre-aggregations in the background by setting
`scheduledRefresh: true`. You can find more information about this setting in
the [Pre-Aggregation Reference][ref-preagg-sched-refresh].

In development mode, Cube.js enables background refresh by default and will
refresh all pre-aggregations marked with the
[`scheduledRefresh`](/pre-aggregations#scheduled-refresh) parameter.

Please consult the [Production Checklist][ref-production-checklist-refresh] for
best practices on running background refresh in production environments.

```js
cube(`Orders`, {
  // ...

  preAggregations: {
    amountByCreated: {
      type: `rollup`,
      measureReferences: [amount],
      timeDimensionReference: createdAt,
      granularity: `month`,
      scheduledRefresh: true,
    },
  },
});
```

## Read Only Data Source

In some cases, it may not be possible to stage pre-aggregation query results in
materialized tables in the source database. For example, the database driver may
not support it, or the source database may be read-only.

To fallback to a strategy where the pre-aggregation query results are downloaded
without first being materialized, set the `readOnly` property of
[`driverFactory`][ref-config-driverfactory] in your configuration:

```javascript
const PostgresDriver = require('@cubejs-backend/postgres-driver');

module.exports = {
  driverFactory: () =>
    new PostgresDriver({
      readOnly: true,
    }),
};
```

<!-- prettier-ignore-start -->
[[warning |]]
| Read only pre-aggregations are only suitable for small datasets
| since they require loading all the data into Cube.js process memory. We **do not**
| recommend using `readOnly` mode for production workloads.
<!-- prettier-ignore-end -->

By default, Cube.js uses temporary tables to extract data types from executed
query while `readOnly` is `false`. If the driver is used in `readOnly` mode, it
will use heuristics to extract data types from the database's response, but this
strategy has certain limitations:

- The aggregation results can be empty, and Cube.js will throw an exception
  because it is impossible to detect types
- Data types can be incorrectly inferred, in rare cases

We highly recommend leaving `readOnly` unset or explicitly setting it to
`false`.

## Partitioning

[Partitioning][wiki-partitioning] is an extremely effective optimization for
improving data access. It effectively "shards" the data between multiple tables,
splitting them by a defined attribute. An incoming query would be checked for
this attribute, and **only** valid partitions required to satisfy it are
selected. This results in faster refresh times due to unnecessary data not being
scanned and processed, and possibly even reduced cost, depending on your
database solution.

Cube.js supports [both time][ref-preagg-time-part] and
[segment-based][ref-preagg-segment-part] partitioning. However, it must first be
enabled for each pre-aggregation.

[Time-based partitioning][ref-preagg-time-part] is especially helpful for
incremental refreshes; when configured, Cube.js will only refresh partitions as
necessary. Without incremental refreshing, Cube.js will re-calculate the entire
pre-aggregation whenever [the refresh key][ref-preaggs-refresh-key] changes.

## Garbage Collection

When pre-aggregations are refreshed, Cube.js will create new pre-aggregation
tables each time a version change is detected. This allows for seamless,
transparent hot swapping of tables for users of any database, even for those
without DDL transactions support.

However, it does lead to orphaned tables which need to be collected over time.
By default, Cube.js will store all content versions for 10 minutes and all
structure versions for 7 days. Then it will retain only the most recent ones and
orphaned tables are dropped from the database.

## Inspecting Pre-Aggregations

Cube Store partially supports the MySQL protocol. This allows you to execute
simple queries using a familiar SQL syntax. To check which pre-aggregations are
managed by Cube Store, for example, you could run the following query:

```sql
SELECT * FROM information_schema.tables;
```

These pre-aggregations are stored as Parquet files under the `.cubestore/`
folder in the project root during development.

## Pre-Aggregations Storage

The default pre-aggregations storage in Cube.js is its own purpose-built storage
layer: Cube Store.

Alternatively, you can store pre-aggregations either **internally** in the
source database, or **externally** in databases such as MySQL or Postgres.

In order to make external pre-aggregations work outside of Cube Store, you
should set the [`externalDriverFactory`][ref-config-extdriverfactory] and
[`externalDbType`][ref-config-extdbtype] properties in your `cube.js`
configuration file. These properties can also be set through the environment
variables.

```bash
CUBEJS_EXT_DB_HOST=<YOUR_DB_HOST_HERE>
CUBEJS_EXT_DB_PORT=<YOUR_DB_PORT_HERE>
CUBEJS_EXT_DB_NAME=<YOUR_DB_NAME_HERE>
CUBEJS_EXT_DB_USER=<YOUR_DB_USER_HERE>
CUBEJS_EXT_DB_PASS=<YOUR_DB_PASS_HERE>
CUBEJS_EXT_DB_TYPE=<SUPPORTED_DB_TYPE_HERE>
```

<!-- prettier-ignore-start -->
[[warning |]]
| Please be aware of the limitations when using internal and external (outside of Cube Store) pre-aggregations.
<!-- prettier-ignore-end -->

![](https://raw.githubusercontent.com/cube-js/cube.js/master/docs/content/Caching/pre-aggregations.png)

#### Some known limitations when using Postgres/MySQL as a storage layer listed below.

**Performance issues with high cardinality rollups:** Queries over billions of
datapoints or higher start exhibiting severe latency issues, negatively
impacting end-user experience.

**Lack of HyperLogLog support:** The HyperLogLog algorithm makes fast work of
distinct counts in queries, a common analytical operation. Unfortunately,
support between database vendors varies greatly, and therefore cannot be
guaranteed.

**Degraded performance for big `UNION ALL` queries:** A practical example of
this would be when querying over a date range using a pre-aggregation with a
`partitionGranularity`. The query would use several partitioned tables to
deliver the result set, and therefore needs to join all valid partitions.

**Poor JOIN performance across rolled up tables:** This often affects workloads
which require cross database joins.

**Table/schema name length mismatches:** A common issue when working across
different database types is that different databases have different length
limits on table names. Cube.js allows working around the issue with `sqlAlias`
but this becomes cumbersome with lots of pre-aggregations.

**SQL type differences between source and external database:** Different
databases often specify types differently, which can cause type mismatch issues.
This is also a common issue and source of frustration which Cube Store resolves.

#### Internal pre-aggregations

Internal pre-aggregations are not considered production-ready due to several
shortcomings, as noted below.

**Concurrency:** Databases (especially RDBMs) generally cannot handle high
concurrency without special configuration. Application databases such as MySQL
and Postgres do support concurrency, but often cannot cope with the demands of
analytical queries without significant tuning. On the other hand, data
warehousing solutions such as Athena and BigQuery often limit the number of
concurrent connections too.

**Latency:** Data warehousing solutions (such as BigQuery or Redshift) are often
slow to return results.

**Cost:** Some databases charge by the amount of data scanned for each query
(such as AWS Athena and BigQuery). Repeatedly querying for this data can easily
rack up costs.

[wiki-partitioning]: https://en.wikipedia.org/wiki/Partition_(database)
[ref-config-env]: /reference/environment-variables#cube-store
[ref-schema-timedimension]: /types-and-formats#dimensions-types-time
[ref-preaggs]: /pre-aggregations
[ref-preagg-sched-refresh]: /pre-aggregations#scheduled-refresh
[ref-preagg-time-part]: /pre-aggregations#rollup-time-partitioning
[ref-preagg-segment-part]: /pre-aggregations#rollup-segment-partitioning
[ref-preaggs-refresh-key]: /pre-aggregations#refresh-key
[ref-config-extdbtype]: /config#options-reference-external-db-type
[ref-config-driverfactory]: /config#options-reference-driver-factory
[ref-config-extdriverfactory]: /config#options-reference-external-driver-factory
[ref-production-checklist-refresh]:
  /deployment/production-checklist#set-up-refresh-worker
