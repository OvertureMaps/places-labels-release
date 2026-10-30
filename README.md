<p align="center">
  <a href="https://overturemaps.org">
    <img src="https://raw.githubusercontent.com/OvertureMaps/data/main/assets/overture-maps-foundation.png" alt="Overture Maps Foundation" width="300">
  </a>
</p>

<h1 align="center">Places Labels Release</h1>

<p align="center">
  Precomputed embedding, similarity, count-min sketch, core-word, and unique-name density lookups for Overture Maps Places matching.
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

The core-word lookups identify tokens that are common in a place's H3 neighborhood, such as a chain name or a street suffix, and strip them from names so a model can compare the distinctive core. They are built from a full Overture Places release.

The unique-name density lookups count how many places share the same name within an H3 resolution-5 cell, and bin the count. A name that is rare in its area is stronger evidence of a match. They are built from a full Overture Places release.

This repository contains only releases. It has no source code, and pull requests and issues are disabled.

## Get the data

Each release has 13 assets, listed in [Assets](#assets). Download them from the [Releases](https://github.com/OvertureMaps/places-labels-release/releases) page, or with the GitHub CLI, which fetches the latest release:

```bash
gh release download --repo OvertureMaps/places-labels-release
```

To pin a version, pass its tag:

```bash
gh release download <tag> --repo OvertureMaps/places-labels-release
```

To download only the CMS assets:

```bash
gh release download --repo OvertureMaps/places-labels-release --pattern 'cms_*.parquet'
```

To download only the unique-name density assets:

```bash
gh release download --repo OvertureMaps/places-labels-release --pattern 'unique_name_h3_*'
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
| `coreword_name_stripped.parquet` | Each labelled place's name with background tokens removed | Yes |
| `coreword_bg_filter.bin` | Bloom filter of background tokens by H3 cell | No |
| `coreword_bg_filter.json` | Metadata for `coreword_bg_filter.bin` | No |
| `unique_name_h3_count_bin.parquet` | Bin of same-name place count in the H3 cell, per labelled place | Yes |
| `unique_name_h3_bin_ge_2.bin` | Bloom filter of (cell, name) keys with at least 2 places | No |
| `unique_name_h3_bin_ge_2.json` | Metadata for `unique_name_h3_bin_ge_2.bin` | No |
| `unique_name_h3_bin_ge_10.bin` | Bloom filter of (cell, name) keys with at least 10 places | No |
| `unique_name_h3_bin_ge_10.json` | Metadata for `unique_name_h3_bin_ge_10.bin` | No |
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

### `coreword_name_stripped.parquet`

Each labelled place's name with background tokens removed. A token is background if it is frequent and name-diverse in its H3 neighborhood, measured at resolutions 9, 7, 5, and 3. One row per labelled place.

| Column | Type | Description |
|---|---|---|
| `id` | string | Overture place ID |
| `name_clean_stripped` | string | Normalized name with background tokens removed |

The key is `id`. Join on it to attach the stripped name to pair data, once for the candidate and once for the baseline.

### `coreword_bg_filter.bin` and `coreword_bg_filter.json`

A Spark Bloom filter of background token keys by H3 cell, for stripping names at serving time without the training lookup. `coreword_bg_filter.json` records the filter parameters, the key count, and the inputs it was built from (`places_release`, `normalizer_version`, the rule parameters). Tokens must be produced with the same normalizer version before probing the filter.

### `unique_name_h3_count_bin.parquet`

How many places share each labelled place's name within its H3 resolution-5 cell, binned. The name key is the lowercased, trimmed primary name. One row per labelled place.

| Column | Type | Description |
|---|---|---|
| `id` | string | Overture place ID |
| `unique_name_h3_cell_count_bin_side` | int | `0` if fewer than 2 places share the name in the cell, `1` for 2 to 9, `2` for 10 or more |

The key is `id`.

### `unique_name_h3_bin_ge_2.bin` and `unique_name_h3_bin_ge_10.bin`

Spark Bloom filters of `5:<cell>:<name key>` keys, for computing the same bin at serving time. Each filter holds the keys whose count is at least 2 or at least 10. A key missing from both filters bins to 0. Each `.bin` has a `.json` file of the same name that records the filter parameters, the key count, and the inputs it was built from (`places_release` and the bin rule).

H3 cells are computed with the DuckDB `h3` extension. Points within about 1e-9 degrees of a cell edge can land in a different cell on a different platform or H3 build. On Places release 2026-09-23.1, a Windows and a Linux build differed in 4 of 2.3 million `ge_2` keys.

The core-word and unique-name assets are rebuilt from the pinned Places release. A new labels release re-derives them without changing the Places release, so only `labels_version` in the metadata changes.

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
coreword = pd.read_parquet("coreword_name_stripped.parquet")
density = pd.read_parquet("unique_name_h3_count_bin.parquet")
```

Read the metadata recorded in each file:

```python
import pyarrow.parquet as pq

print(pq.read_schema("embedding_lookup.parquet").metadata)
print(pq.read_schema("cms_names.parquet").metadata)
```

The metadata includes the release version, plus the embedding model revision for the lookups and the Places release and normalizer version for the sketches and filters. The core-word and unique-name filters keep their metadata in the `.json` file beside each `.bin`. Use all lookups from the same release together.

## License

The release assets are derived from third-party data and are licensed under the terms in each release's `THIRD_PARTY_NOTICES.txt`. The MIT license in this repository covers only the repository's own files, such as this README.

## Contact

Questions and feedback: [Overture Maps Discussions](https://github.com/orgs/OvertureMaps/discussions).
