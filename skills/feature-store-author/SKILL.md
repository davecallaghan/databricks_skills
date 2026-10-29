---
name: feature-store-author
description: Databricks Feature Store and Feature Views — creating features, training sets, point-in-time joins, and feature serving for ML models. Read this skill when the user asks to create feature tables, use FeatureLookup, build training datasets, or serve features for inference.
---

# Feature Store Author

## Feature Views (Declarative API)
Use Feature Views (databricks-feature-engineering >= 0.16.0) for declarative, time-windowed features:

```python
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup

fe = FeatureEngineeringClient()

# Create a feature table
fe.create_table(
    name="coe_tpch.features.customer_features",
    primary_keys=["c_custkey"],
    df=customer_feature_df,
    schema=customer_feature_df.schema,
    description="Customer-level features for churn prediction",
)

# Create a training set with point-in-time joins
training_df = spark.table("coe_tpch.gold.orders_summary")

training_set = fe.create_training_set(
    df=training_df,
    feature_lookup_objects=[
        FeatureLookup(
            table_name="coe_tpch.features.customer_features",
            lookup_key="c_custkey",
            feature_names=["avg_order_value", "order_count_30d", "days_since_last_order"],
            timestamp_lookup_key="o_orderdate",
        ),
    ],
    label="churn_label",
    exclude_columns=["o_orderdate"],
)

training_data = training_set.load_df()
```

## Point-in-Time Join Pattern
The key pattern: join features to training data at the timestamp of the event, not at the current time. This prevents data leakage.

```sql
-- Point-in-time join: get feature values AS OF the order date
SELECT o.o_orderkey, o.o_custkey, o.o_orderdate,
       f.avg_order_value, f.order_count_30d
FROM coe_tpch.gold.orders o
ASOF JOIN coe_tpch.features.customer_features f
  ON o.c_custkey = f.c_custkey
  AND o.o_orderdate >= f.feature_timestamp
```

## Time-Windowed Aggregation Features (Feature Views API)
```python
from databricks.feature_engineering import (
    FeatureEngineeringClient,
    AggregationFunction,
    RollingWindow,
)

fe = FeatureEngineeringClient()

# Define rolling window features
rolling_spec = RollingWindow(
    window="30d",
    offset="0d",
    time_column="o_orderdate",
)

fe.create_feature(
    name="coe_tpch.features.customer_rolling",
    primary_keys=["c_custkey"],
    aggregation_functions=[
        AggregationFunction(column="o_totalprice", function="avg", alias="avg_order_value_30d"),
        AggregationFunction(column="o_orderkey", function="count", alias="order_count_30d"),
    ],
    rolling_windows=[rolling_spec],
    source_table="coe_tpch.gold.orders",
)
```

## Feature Serving
```python
# Serve features for online inference
fe.serve_features(
    name="customer-features-endpoint",
    feature_spec=feature_spec,
    workload_size="Small",
    scale_to_zero_enabled=True,
)
```

## CoE Context
Q3 accelerator `feature_store_setup` creates feature tables on TPC-H customer and order data. Features include customer aggregation features (avg order value, order frequency, days since last order) and supplier features (supply reliability, avg supply cost). The training set joins these features to the orders table using point-in-time joins.

## Best Practices
- **Always use point-in-time joins**: Prevent data leakage in training data
- **Primary keys are mandatory**: Every feature table needs a primary key
- **Timestamp column for ASOF joins**: Include a `feature_timestamp` column
- **Use Feature Views over classic Feature Store**: The declarative API is the recommended approach
- **Log features to MLflow**: Track which features were used in each training run