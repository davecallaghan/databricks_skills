---
name: vector-search-builder
description: Databricks Vector Search endpoint and index creation, embedding pipelines, RAG (Retrieval-Augmented Generation) patterns, and similarity search with filters. Read this skill when the user asks to build Vector Search, create RAG pipelines, configure embeddings, or perform semantic search.
---

# Vector Search Builder

## Architecture Overview
Vector Search enables semantic similarity search over Delta tables. The pipeline:
1. Create a Vector Search endpoint
2. Create an index on a Delta table with an embedding column
3. Query the index with text or vector inputs

## Creating a Vector Search Endpoint
```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()

endpoint = vsc.create_endpoint(
    name="coe-rag-endpoint",
    endpoint_type="STANDARD",
)
```

## Creating a Delta Sync Index
```python
index = vsc.create_delta_sync_index(
    endpoint_name="coe-rag-endpoint",
    index_name="coe_tpch.gold.knowledge_base_index",
    source_table_name="coe_tpch.gold.knowledge_base",
    pipeline_type="TRIGGERED",
    embedding_source_column="content",
    embedding_model_endpoint_name="databricks-gte-large-en-v1",
    primary_key="id",
)
```

## RAG Pattern
```python
# Step 1: Chunk documents
from pyspark.sql.functions import explode, seq

documents = spark.table("coe_tpch.gold.governance_docs")
chunked = documents.withColumn("chunk_id", expr("uuid()")) \
    .withColumn("content", col("document_text"))  # pre-chunked

# Step 2: Write to Delta table with embeddings
chunked.write.mode("overwrite").saveAsTable("coe_tpch.gold.knowledge_base")

# Step 3: Sync the index
index.sync()

# Step 4: Query for RAG
results = index.similarity_search(
    query_text="What are the GDPR requirements for data retention?",
    num_results=5,
    columns=["content", "document_title", "source"],
    filter_json={"source_type": "policy"}
)
```

## Direct Vector Index (Custom Embeddings)
For cases where you compute embeddings yourself:
```python
index = vsc.create_direct_access_index(
    endpoint_name="coe-rag-endpoint",
    index_name="coe_tpch.gold.custom_embeddings",
    primary_key="id",
    embedding_dimension=1536,
    embedding_vector_column="embedding",
)
```

## CoE Context
Q3 accelerator `rag_knowledge_base` builds a RAG pipeline over governance documentation. The Vector Search endpoint and index are parameterized through the Q3 DAB variables (`vector_search_endpoint`, `embedding_model`). Q1 governance KB content is the source corpus.

## Best Practices
- Use `TRIGGERED` pipeline type for batch updates; `CONTINUOUS` for near-real-time
- Chunk size: 512-1024 characters with 50-100 character overlap
- Filter columns improve precision: tag by document type, domain, sensitivity
- Always front Vector Search through AI Gateway for governance (see `ai-gateway-config` skill)