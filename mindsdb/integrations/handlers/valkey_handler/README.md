# Valkey Handler

This is the implementation of the Valkey handler for MindsDB, providing vector store capabilities using the [Valkey Search](https://valkey.io/) module.

## Prerequisites

- **Valkey Server** 9.0+ with the Search module enabled (e.g., `valkey/valkey-bundle` Docker image)
- **Python packages**: `valkey-glide>=2.4.0`, `numpy>=1.21.0`

## Installation

Install the handler dependencies:

```bash
pip install valkey-glide numpy
```

Or install via MindsDB extras:

```bash
pip install mindsdb[valkey]
```

## Connection

Create a Valkey vector store connection in MindsDB:

```sql
CREATE DATABASE my_valkey
WITH ENGINE = 'valkey',
PARAMETERS = {
    "host": "localhost",        -- Valkey server hostname (default: localhost)
    "port": 6379,              -- Valkey server port (default: 6379)
    "password": "",            -- Authentication password (optional)
    "db": 0,                   -- Database number 0-15 (default: 0)
    "vector_dimension": 384,   -- Default embedding dimension (default: 384)
    "distance_metric": "COSINE", -- COSINE, L2, or IP (default: COSINE)
    "prefix": "doc:"           -- Key prefix for document hashes (default: "doc:")
};
```

## Usage

### Create a Table (Vector Index)

```sql
CREATE TABLE my_valkey.my_collection
(SELECT * FROM my_model
 WHERE content = 'sample text');
```

Or use the knowledge base pattern:

```sql
CREATE KNOWLEDGE BASE my_kb
USING
  VECTOR STORE = my_valkey,
  MODEL = my_embedding_model;
```

### Insert Data

```sql
INSERT INTO my_valkey.my_collection (id, content, embeddings, metadata)
VALUES ('doc1', 'Hello world', '[0.1, 0.2, ...]', '{"source": "web"}');
```

### Vector Similarity Search (KNN)

```sql
SELECT id, content, distance
FROM my_valkey.my_collection
WHERE search_vector = (SELECT embeddings FROM my_model WHERE content = 'query text')
LIMIT 5;
```

### Select by ID

```sql
SELECT id, content, metadata
FROM my_valkey.my_collection
WHERE id = 'doc1';
```

### Delete Documents

```sql
DELETE FROM my_valkey.my_collection
WHERE id = 'doc1';

DELETE FROM my_valkey.my_collection
WHERE id IN ('doc1', 'doc2', 'doc3');
```

### Drop Table (Index)

```sql
DROP TABLE my_valkey.my_collection;
```

## Configuration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `host` | str | `localhost` | Valkey server hostname |
| `port` | int | `6379` | Valkey server port |
| `password` | str | `None` | Authentication password |
| `db` | int | `0` | Database number (0-15) |
| `vector_dimension` | int | `384` | Default embedding dimension for new indexes |
| `distance_metric` | str | `COSINE` | Distance metric: `COSINE`, `L2`, or `IP` |
| `prefix` | str | `doc:` | Key prefix for document hash keys |

## Distance Metrics

| Metric | Description | Use Case |
|--------|-------------|----------|
| `COSINE` | Cosine similarity (1 - cos_sim) | Text embeddings, normalized vectors |
| `L2` | Euclidean distance | Image embeddings, spatial data |
| `IP` | Inner product (negative dot product) | Pre-normalized vectors, recommendation |

## Architecture

The handler uses:
- **valkey-glide** (async client) with a synchronous wrapper (`asyncio.run_until_complete`)
- **HASH-based storage**: Documents are stored as Redis hashes with the key pattern `{prefix}{table}:{id}`
- **FT.CREATE**: Creates HNSW vector indexes with configurable dimensions and distance metrics
- **FT.SEARCH**: Executes KNN vector similarity queries with optional pre-filtering

## Running Tests

```bash
# Unit tests (no Valkey required)
pytest mindsdb/integrations/handlers/valkey_handler/tests/test_valkey_handler.py -v -k "Unit"

# Integration tests (requires running Valkey with Search module)
VALKEY_HOST=localhost VALKEY_PORT=6379 pytest mindsdb/integrations/handlers/valkey_handler/tests/test_valkey_handler.py -v -k "Integration"
```

### Running Valkey for Tests

```bash
docker run -d --name valkey-test -p 6379:6379 valkey/valkey-bundle:9.1
```
