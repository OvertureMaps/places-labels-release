# places-labels-release

Precomputed embedding and similarity lookup tables for Overture Maps Places matching, published as GitHub Releases.

The lookups hold text-embedding features for a curated set of labelled Places pairs, each pair being a candidate place and a baseline place. Places matching models use them as inputs for training and evaluation, so they can read precomputed values instead of running the embedding model themselves. The underlying labelled pairs aren't published, which keeps them useful as unbiased training and evaluation data.

This repository contains only releases. It has no source code, and pull requests and issues are disabled.

## Get the data

Each release has three assets. Download them from the [Releases](https://github.com/OvertureMaps/places-labels-release/releases) page, or with the GitHub CLI:

```bash
gh release download v1.2.3 --repo OvertureMaps/places-labels-release
```

Asset names are the same in every release. The version is in the release tag and in each parquet file's metadata.

Releases are immutable. A published release's tag and assets never change, so pinning a version always returns the same files.

## Assets

### `embedding_lookup.parquet`

Embeddings of the place names and taxonomy values that appear in the labelled pairs. One row per unique normalized string, so a string shared by many places is stored once.

| Column | Type | Description |
|---|---|---|
| `text_hash` | string | Hash of the normalized text |
| `field_type` | string | `name` or `taxonomy` |
| `embedding` | int8[768] | Embedding vector for the string |

The key is (`text_hash`, `field_type`).

### `cosine_similarity_lookup.parquet`

Precomputed similarity for each labelled pair of Overture places. Use it to get name and taxonomy similarity for a pair without computing embeddings.

| Column | Type | Description |
|---|---|---|
| `id` | string | Overture place ID of the candidate |
| `base_id` | string | Overture place ID of the baseline |
| `name_cosine_similarity` | float | Cosine similarity of the two names' embeddings |
| `taxonomy_cosine_similarity` | float | Cosine similarity of the two taxonomy values' embeddings |

The key is (`id`, `base_id`). Join on these columns to attach similarity features to pair data.

### `THIRD_PARTY_NOTICES.txt`

Attribution, license notices, and full license texts for the data providers behind the labels. Redistribute this file with any derived data.

## Usage

DuckDB:

```sql
SELECT id, base_id, name_cosine_similarity
FROM read_parquet('cosine_similarity_lookup.parquet')
WHERE name_cosine_similarity > 0.9;
```

DuckDB can also read a release asset directly over HTTPS:

```sql
SELECT *
FROM read_parquet('https://github.com/OvertureMaps/places-labels-release/releases/download/v1.2.3/embedding_lookup.parquet')
LIMIT 10;
```

Python:

```python
import pandas as pd

similarity = pd.read_parquet("cosine_similarity_lookup.parquet")
embeddings = pd.read_parquet("embedding_lookup.parquet")
```

Read the metadata recorded in each file:

```python
import pyarrow.parquet as pq

print(pq.read_schema("embedding_lookup.parquet").metadata)
```

The metadata includes the embedding model revision and the release version. Use both lookup files from the same release together.

## License

The release assets are derived from third-party data and are licensed under the terms in each release's `THIRD_PARTY_NOTICES.txt`. The MIT license in this repository covers only the repository's own files, such as this README.

## Contact

Questions and feedback: [Overture Maps Discussions](https://github.com/orgs/OvertureMaps/discussions).
