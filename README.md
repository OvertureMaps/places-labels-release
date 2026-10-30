<p align="center">
  <a href="https://overturemaps.org">
    <img src="https://raw.githubusercontent.com/OvertureMaps/data/main/assets/overture-maps-foundation.png" alt="Overture Maps Foundation" width="300">
  </a>
</p>

<h1 align="center">Places Labels Release</h1>

<p align="center">
  Precomputed embedding, similarity, and count-min sketch lookups for Overture Maps Places matching.
</p>

<p align="center">
  <a href="https://github.com/OvertureMaps/places-labels-release/releases/latest"><img src="https://img.shields.io/github/v/release/OvertureMaps/places-labels-release?label=release&color=5c6b7a" alt="Latest release"></a>
</p>

<p align="center">
  <a href="https://github.com/OvertureMaps/places-labels-release/releases">Releases</a> ·
  <a href="https://docs.overturemaps.org">Docs</a> ·
  <a href="https://github.com/orgs/OvertureMaps/discussions">Discussions</a>
</p>

Lookup tables for training and evaluating Places matching models, published as GitHub Releases.

## Contents <!-- omit in toc -->

- [Overview](#overview)
- [Get the data](#get-the-data)
- [Assets](#assets)
- [Usage](#usage)
- [License](#license)
- [Contact](#contact)

## Overview

The embedding and similarity lookups hold text-embedding features for a curated set of labelled Places pairs, each pair being a candidate place and a baseline place. Places matching models use them as inputs for training and evaluation, so they can read precomputed values instead of running the embedding model themselves. The underlying labelled pairs aren't published, which keeps them useful as unbiased training and evaluation data.

The count-min sketch (CMS) lookups summarize how often name and address tokens occur across a full Overture Places release. They do not depend on the labelled pairs.

This repository contains only releases. It has no source code, and pull requests and issues are disabled.

## Get the data

Each release has five assets. Download them from the [Releases](https://github.com/OvertureMaps/places-labels-release/releases) page, or with the GitHub CLI:

```bash
gh release download v1.3.0 --repo OvertureMaps/places-labels-release
```

To download only the CMS assets:

```bash
gh release download v1.3.0 --repo OvertureMaps/places-labels-release --pattern 'cms_*.parquet'
```

Asset names are the same in every release. The version is in the release tag and in each parquet file's metadata.

Releases are immutable. A published release's tag and assets never change, so pinning a version always returns the same files.

## Assets

| Asset | Contents | Depends on labelled pairs |
|---|---|---|
| `embedding_lookup.parquet` | Embeddings of place names and taxonomy values | Yes |
| `cosine_similarity_lookup.parquet` | Name and taxonomy similarity per labelled pair | Yes |
| `cms_names.parquet` | Count-min sketch of name token frequencies | No |
| `cms_address.parquet` | Count-min sketch of address token frequencies | No |
| `THIRD_PARTY_NOTICES.txt` | Attribution and license texts | Not applicable |

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

### `cms_names.parquet` and `cms_address.parquet`

Each file holds one row: a serialized [count-min sketch](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch) of token document frequencies, built from every place in an Overture Places release. `cms_names.parquet` covers the primary name, and `cms_address.parquet` covers the first address's freeform text. Use them to estimate how common a token is, for example to weight rare tokens higher when comparing names or addresses.

| Column | Type | Description |
|---|---|---|
| `cms_bytes` | binary | Serialized count-min sketch, in the Apache Spark `CountMinSketch` binary format |
| `num_docs` | int64 | Number of places the sketch was built from |

Each sketch counts the number of places that contain a token, once per place. Text is normalized and tokenized before counting, so tokens must be produced with the same normalizer version (recorded as `normalizer_version` in the file metadata) before querying the sketch.

Sketches are approximate: a count can be over-estimated, never under-estimated. The file metadata records the sketch parameters (`cms_eps`, `cms_confidence`, `cms_seed`) and the inputs the sketch was built from (`places_release`, `normalizer_version`).

The CMS assets are rebuilt only when the Places release or the normalizer version changes. Otherwise a new labels release re-publishes the previous sketches unchanged, with only `labels_version` updated in the metadata.

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
FROM read_parquet('https://github.com/OvertureMaps/places-labels-release/releases/download/v1.3.0/embedding_lookup.parquet')
LIMIT 10;
```

Python:

```python
import pandas as pd

similarity = pd.read_parquet("cosine_similarity_lookup.parquet")
embeddings = pd.read_parquet("embedding_lookup.parquet")
cms_names = pd.read_parquet("cms_names.parquet")
```

Read the metadata recorded in each file:

```python
import pyarrow.parquet as pq

print(pq.read_schema("embedding_lookup.parquet").metadata)
print(pq.read_schema("cms_names.parquet").metadata)
```

The metadata includes the release version, plus the embedding model revision for the lookups and the Places release and normalizer version for the sketches. Use the embedding and similarity lookups from the same release together.

## License

The release assets are derived from third-party data and are licensed under the terms in each release's `THIRD_PARTY_NOTICES.txt`. The MIT license in this repository covers only the repository's own files, such as this README.

## Contact

Questions and feedback: [Overture Maps Discussions](https://github.com/orgs/OvertureMaps/discussions).
