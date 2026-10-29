---
name: mlflow-lifecycle
description: MLflow experiment tracking, model registry, autolog, model versioning, and deployment lifecycle on Databricks. Read this skill when the user asks to track experiments, register models, use MLflow, or manage the ML model lifecycle.
---

# MLflow Lifecycle Management

## Experiment Tracking
```python
import mlflow
import mlflow.sklearn

# Set experiment (creates if not exists)
mlflow.set_experiment("/coe-program/customer_churn_prediction")

# Start a run
with mlflow.start_run(run_name="xgboost_v1") as run:
    # Log parameters
    mlflow.log_params({"max_depth": 6, "n_estimators": 100, "learning_rate": 0.1})
    
    # Train model
    model = xgb.XGBClassifier(**params)
    model.fit(X_train, y_train)
    
    # Log metrics
    predictions = model.predict(X_test)
    mlflow.log_metric("accuracy", accuracy_score(y_test, predictions))
    mlflow.log_metric("f1", f1_score(y_test, predictions))
    
    # Log model
    mlflow.sklearn.log_model(model, "model", registered_model_name="coe_models.customer_churn")
```

## Model Registry in Unity Catalog
```python
# Register a model in Unity Catalog
mlflow.register_model(
    model_uri=f"runs:/{run_id}/model",
    name="coe_tpch.models.customer_churn",
)

# Transition model version stage
client = mlflow.tracking.MlflowClient()
client.transition_model_version_stage(
    name="coe_tpch.models.customer_churn",
    version=1,
    stage="Production",
)
```

## Autolog Patterns
```python
# Enable autologging for supported frameworks
mlflow.autolog(log_input_examples=True, log_model_signatures=True)

# Framework-specific autolog
mlflow.xgboost.autolog()
mlflow.sklearn.autolog()
```

## Model Serving via MLflow Deployments
```python
import mlflow.deployments

# Create an endpoint
mlflow.deployments.create_endpoint(
    name="customer-churn-endpoint",
    config={
        "served_entities": [{
            "name": "churn-model",
            "entity_name": "coe_tpch.models.customer_churn",
            "entity_version": "1",
            "workload_size": "Small",
            "scale_to_zero_enabled": True,
        }]
    }
)

# Query the endpoint
client = mlflow.deployments.get_deploy_client("databricks")
response = client.predict(
    endpoint="customer-churn-endpoint",
    inputs={"dataframe_split": {"columns": feature_cols, "data": [[...]]}},
)
```

## CoE Context
Q3 accelerator `mlflow_experiment_lifecycle` demonstrates the full MLflow lifecycle on TPC-H data: experiment creation, parameter/metric logging, model registry in Unity Catalog, and endpoint deployment. Uses `coe_tpch.models` schema for model storage.

## Evaluation Patterns
Use MLflow evaluation for model quality:
```python
mlflow.evaluate(
    model=logged_model_uri,
    data=eval_dataset,
    targets="label",
    model_type="classifier",
    evaluators=["default"],
)
```

## Best Practices
- Always use Unity Catalog registered models (`catalog.schema.model_name`)
- Log model signatures for input validation
- Use model stages (Staging → Production) or aliases
- Enable scale-to-zero for dev/test endpoints
- Track data lineage with `mlflow.log_artifact` for training data references