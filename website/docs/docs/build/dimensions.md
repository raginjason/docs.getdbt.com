---
title: Dimensions
id: dimensions
description: "DDimensions determine the level of aggregation for a metric, and are non-aggregatable expressions."
sidebar_label: "Dimensions"
tags: [Metrics, Semantic Layer]
---

Dimensions are non-aggregatable expressions that define the level of aggregation for a metric used to define how data is sliced or grouped in a metric, Since dimensions can't be aggregated, they're considered to be a property of the primary or unique entity of the table.

Dimensions are defined within semantic models, alongside entities and measures, and correspond to non-aggregatable columns in your dbt model that provide categorical or time-based context. In SQL, dimensions are typically included in the GROUP BY clause.

For the following examples, we will refer to the following semantic model.

```yaml
semantic_model:
  - name: transactions
    description: A record for every transaction that takes place. Carts are considered multiple transactions for each SKU. 
    model: {{ ref("fact_transactions") }}

# --- ENTITIES ---
  Entities: 
    - name: transaction_id
      type: primary
    - name: customer_id
      type: foreign
    - name: store_id
      type: foreign
    - name: product_id
      type: foreign

# --- MEASURES --- 
  measures: 
    - name: revenue
      description:  
      expr: price * quantity
      agg: sum

# --- DIMENSIONS ---
  dimensions:
    - name: ds
      type: time
      expr: date_trunc('day', ts)
      type_params:
        is_primary: true
        time_granularity: day
    - name: is_bulk_transaction
      type: categorical
      expr: case when quantity > 10 then true else false end
```
All dimensions require a `name`, `type` and in most cases, an `expr` parameter. 

| Name | Parameter | Field type |
| --- | --- | --- |
| `name` |  Refers to the name of the dimension that will be visible to the user in downstream tools. It can also serve as an alias if the column name or SQL query reference is different and provided in the `expr` parameter. <br /><br /> &mdash; Dimension names should be unique within a semantic model, but they can be non-unique across different models as MetricFlow uses [joins](/docs/build/join-logic) to identify the right dimension. | Required |
| `type` | Specifies the type of dimension created in the semantic model. There are three types:<br /><br />&mdash; Categorical: Group rows in a table by categories like geography, product type, color, and so on. <br />&mdash; Time: Point to a date field in the data platform, and must be of type TIMESTAMP or equivalent in the data platform engine. <br />&mdash; Slowly-changing dimensions: Analyze metrics over time and slice them by dimensions that change over time, like sales trends by a customer's country. | Required |
| `expr` | Defines the underlying column or SQL query for a dimension. If no `expr` is specified, MetricFlow will use the column with the same name as the dimension. You can use `expr` to input a SQL expression, including a case statement, or the column name itself. | Optional |

## Dimensions types

Dimensions have three types. This section further explains the definitions and provides examples.

1. [Categorical dimensions](#categorical)
1. [Time dimensions](#time)
1. [Slowly changing dimensions](#scd-type-ii)

### Categorical

Category dimensions are used to group metrics by different categories such as product type, color, or geographical area. They can refer to existing columns in your dbt model or be calculated using a SQL expression with the `expr` parameter. An example of a category dimension is `is_bulk_transaction`, which is a dimension created by applying a case statement to the underlying column `quantity`. This allows users to group or filter the data based on bulk transactions.

```yaml
dimensions: 
  - name: is_bulk_transaction
    type: categorical
    expr: case when quantity > 10 then true else false end
```

### Time

Time dimension has additional parameters specified under the `type_params` section.

:::tip use datetime data type if using BigQuery
To use BigQuery as your data platform, time dimension columns need to be in the datetime data type. If they are stored in another type, you can cast them to datetime using the `expr` property. Time dimensions are used to group metrics by different levels of time, such as day, week, month, quarter, and year. MetricFlow supports these granularities, which can be specified using the `time_granularity` parameter.
:::

<Tabs>

<TabItem value="is_primary" label="is_primary">

To specify the default time dimension for a measure or metric in MetricFlow, set the `is_primary` parameter to True. If you have multiple time dimensions in your semantic model, the non-primary ones should have `is_primary` set to False. To assign a non-primary time dimension to a measure, use the `agg_time_dimension` parameter and refer to the time dimension defined in the dimension section. 

In the provided example, the semantic model has two time dimensions, `created_at` and `deleted_at`, with `created_at` being the primary time dimension through `is_primary: True`. The `users_created` measure defaults to the primary time dimension, while the `users_deleted` measure uses `deleted_at` as its time dimension. 

```yaml
dimensions: 
  - name: created_at
    type: time
    expr: date_trunc('day', ts_created) #ts_created is the underlying column name from the table 
    is_partition: True 
    type_params:
      is_primary: True
      time_granularity: day
  - name: deleted_at
    type: time
    expr: date_trunc('day', ts_deleted) #ts_deleted is the underlying column name from the table 
    is_partition: True 
    type_params:
      is_primary: False
      time_granularity: day

measures:
  - name: users_deleted
    expr: 1
    agg: sum
    agg_time_dimension: deleted_at
    create_metric: True 
  - name: users_created
    expr: 1
    agg: sum
    create_metric: True 
```

When querying one or more metrics in the MetricFlow CLI, the default time dimension for a single metric is the primary time dimension, which can be referred to as metric_time or the dimension's name. Multiple time dimensions can be used in separate metrics, such as users_created which uses created_at, and users_deleted which uses deleted_at.

* MetricFlow 

      ```
      #TODO update with new syntax
       mf query --metrics users_created,users_deleted --dimensions metric_time --order metric_time 
      ```

</TabItem>

<TabItem value="time_gran" label="time_granularity">

`time_granularity` specifies the smallest level of detail that a measure or metric should be reported at, such as daily, weekly, monthly, quarterly, or yearly. Different granularity options are available, and each metric must have a specified granularity. For example, a metric that is specified with weekly granularity couldn't be aggregated to a daily grain. 

The current options for time granularity are day, week, month, quarter, and year. 

Aggregation between metrics with different granularities is possible, with MetricFlow returning results at the highest granularity by default. For example, when querying two metrics with daily and monthly granularity, the resulting aggregation will be at the monthly level.

```yaml
dimensions: 
  - name: created_at
    type: time
    expr: date_trunc('day', ts_created) #ts_created is the underlying column name from the table 
    is_partition: True 
    type_params:
      is_primary: True
      time_granularity: day
  - name: deleted_at
    type: time
    expr: date_trunc('day', ts_deleted) #ts_deleted is the underlying column name from the table 
    is_partition: True 
    type_params:
      is_primary: False
      time_granularity: day

measures:
  - name: users_deleted
    expr: 1
    agg: sum
    agg_time_dimension: deleted_at
    create_metric: True 
  - name: users_created
    expr: 1
    agg: sum
    create_metric: True 
```

</TabItem>

<TabItem value="is_partition" label="is_partition">

Use `is_partition: True` to indicate that a dimension exists over a specific time window. For example, a date-partitioned dimensional table. When you query metrics from different tables, MetricFlow will use this parameter to ensure that the correct dimensional values are joined to measures. 

In addition, MetricFlow allows for easy aggregation of metrics at query time. For example, you can aggregate the `messages_per_month` measure, where the original `time_granularity` of the time dimension `metrics_time`, at a yearly granularity by specifying it in the query in the CLI.

```
TODO update syntax
mf query --metrics messages_per_month --dimensions metric_time --order metric_time --time-granularity year  
```


```yaml
dimensions: 
  - name: created_at
    type: time
    expr: date_trunc('day', ts_created) #ts_created is the underlying column name from the table 
    is_partition: True 
    type_params:
      is_primary: True
      time_granularity: day
  - name: deleted_at
    type: time
    expr: date_trunc('day', ts_deleted) #ts_deleted is the underlying column name from the table 
    is_partition: True 
    type_params:
      is_primary: False
      time_granularity: day

measures:
  - name: users_deleted
    expr: 1
    agg: sum
    agg_time_dimension: deleted_at
    create_metric: True 
  - name: users_created
    expr: 1
    agg: sum
    create_metric: True 
```

</TabItem>

</Tabs>


### SCD Type II 

:::caution 
Currently, there are limitations in supporting SCD's. 
:::

MetricFlow supports joins against dimension values in a semantic model built on top of an SCD Type II table (slowly changing dimension) Type II table. This is useful when you need a particular metric sliced by a dimension that changes over time, such as the historical trends of sales by a customer's country. 

As their name suggests SCD Type II are dimensions that change values at a coarser time granularity. This results in a range of valid rows with different dimension values for a given metric or measure. MetricFlow associates the metric with the first (minimum) available dimension value within a coarser time window, such as month. By default, MetricFlow uses the dimension that is valid at the beginning of the time granularity.

The following basic structure of an SCD Type II data platform table is supported:

| entity_key | dimension_1 | dimension_2 | ... | dimension_x | valid_from | valid_to |
|------------|-------------|-------------|-----|-------------|------------|----------|  

* `entity_key` (required): An entity_key (or some sort of identifier) must be present 
* `valid_from` (required): A timestamp indicating the start of a changing dimension value must be present
* `valid_to` (required): A timestamp indicating the end of a changing dimension value must be present

**Note**: The SCD dimensions table must have `valid_to` and `valid_from` columns.

This is an example of SQL code that shows how a sample metric called `num_events` is joined with versioned dimension data (stored in a table called `scd_dimensions`) using a natural key made up of the `entity_key` and `timestamp` columns. 


```sql
SELECT metric_time, dimension_1, SUM(1) AS num_events
FROM events a
LEFT OUTER JOIN scd_dimensions b
ON 
  a.entity_key = b.entity_key 
  AND a.metric_time >= b.valid_from 
  AND (a.metric_time < b. valid_to OR b.valid_to IS NULL)
GROUP BY 1, 2
```

<!-- this section below is quite long so i may turn both examples into tabs -->

<Tabs>

<TabItem value="example" label="SCD table example 1">

This example shows how to create slowly changing dimensions (SCD) using a semantic model. The SCD table contains information about sales persons' tier and the time length of that tier. Suppose you have the underlying SCD table:

| sales_person_id | tier | start_date | end_date | 
|-----------------|------|------------|----------|
| 111             | 1    | 2019-02-03 | 2020-01-05| 
| 111             | 2    | 2020-01-05 | 2048-01-01| 
| 222             | 2    | 2020-03-05 | 2048-01-01| 
| 333             | 2    | 2020-08-19 | 2021-10-22| 
| 333             | 3    | 2021-10-22 | 2048-01-01|  

Take note of the extra arguments under `validity_params`: `is_start` and `is_end`. These arguments indicate the columns in the SCD table that contain the start and end dates for each tier (or beginning or ending timestamp column for a dimensional value).

```yaml 
semantic_model:
  name: sales_person_tiers
  description: SCD Type II table of tiers for sales people 
  model: {{ref(sales_person_tiers)}}

  dimensions:
    - name: tier_start
      type: time
      expr: start_date
      type_params:
        time_granularity: day
        validity_params:
          is_start: True
    - name: tier_end 
      type: time
      expr: end_date
      type_params:
        time_granularity: day
        validity_params:
          is_end: True
    - name: tier
      type: categorical

  entities:
    - name: sales_person
      type: natural 
      expr: sales_person_id
```

The following code represents a separate semantic model that holds a fact table for `transactions`:  

```yaml
semantic_model: 
  name: transactions 
  description: |
    Each row represents one transaction.
    There is a transaction, product, sales_person, and customer id for 
    every transaction. There is only one transaction id per 
    transaction. The `ds` or date is reflected in UTC.
  model: {{ ref(fact_transactions) }}

  entities:
    - name: transaction_id
      type: primary
    - name: customer
      type: foreign
      expr: customer_id
    - name: product
      type: foreign
      expr: product_id
    - name: sales_person
      type: foreign
      expr: sales_person_id

  measures:
    - name: transactions
      expr: 1
      agg: SUM
      create_metric: True
    - name: gross_sales
      expr: sales_price
      agg: SUM
      create_metric: True
    - name: sales_persons_with_a_sale
      expr: sales_person_id
      agg: COUNT_DISTINCT
      create_metric: True

  dimensions:
    - name: ds
      type: time
      is_partition: true
      type_params:
        is_primary: True
        time_format: YYYY-MM-DD
        time_granularity: day
    - name: sales_geo
      type: categorical
```

You can now access the metrics in the `transactions` semantic model organized by the slowly changing dimension of `tier`. 

In the sales tier example,  For instance, if a salesperson was Tier 1 from 2022-03-01 to 2022-03-12, and gets promoted to Tier 2 from 2022-03-12 onwards, all transactions from March would be categorized under Tier 1 since since the dimension value of Tier 1 comes earlier (and is the default starting point), even though the salesperson was promoted to Tier 2 on 2022-03-12.

</TabItem>

<TabItem value="example2" label="SCD table example 2">

THIS EXAMPLE NEEDS FLESHING OUT

This example shows how to create slowly changing dimensions (SCD) using a semantic model. The SCD table contains information about sales persons' tier and the time length of that tier. Suppose you have the underlying SCD table:

| sales_person_id | tier | start_date | end_date | 
|-----------------|------|------------|----------|
| 111             | 1    | 2019-02-03 | 2020-01-05| 
| 111             | 2    | 2020-01-05 | 2048-01-01| 
| 222             | 2    | 2020-03-05 | 2048-01-01| 
| 333             | 2    | 2020-08-19 | 2021-10-22| 
| 333             | 3    | 2021-10-22 | 2048-01-01|  

In the sales tier example, if sales_person_id 456 is Tier 2 from 2022-03-08 onwards, but there is no associated tier level dimension for this person from 2022-03-01 to 2022-03-08, then all transactions associated with sales_person_id 456 for the month of March will be grouped under 'NA' since no tier is present prior to Tier 2.

The following command or code represents how to return the count of transactions generated by each sales tier per month:

```
TODO: Update syntax
# MetricFlow 
mf query --metrics transactions --dimensions metric_time__month,sales_person__tier --order metric_time__month --order sales_person__tier

```

</TabItem>
</Tabs>