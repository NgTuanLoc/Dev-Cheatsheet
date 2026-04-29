---
tags: [database, advanced, nosql]
aliases: [Elasticsearch, OpenSearch, Lucene, Inverted Index, Analyzer]
level: Advanced
---

# Search Engines

> **One-liner**: Search engines (Elasticsearch, OpenSearch, Solr) build inverted indexes over text; reach for them when full-text relevance, multi-field ranking, faceting, and analytics outgrow Postgres FTS.

---

## Quick Reference

| Engine | Backed by | Use |
|--------|-----------|-----|
| **Elasticsearch** | Apache Lucene | de-facto standard; JSON DSL; license change to Elastic v2 |
| **OpenSearch** | fork of ES 7.10 | Apache 2.0; mostly drop-in compatible |
| **Apache Solr** | Lucene | older, schema-first, strong faceting |
| **Meilisearch / Typesense** | Rust/C++ | typo-tolerant, simple API, "instant search" |
| **Postgres FTS** | built-in | small to medium corpus; see [[19 - Full-Text Search]] |
| **MeiliSearch / Algolia** | hosted | turnkey site search |

| Concept | Meaning |
|---------|---------|
| **Index** | a logical collection of documents (≈ table) |
| **Document** | JSON object indexed by `_id` (≈ row) |
| **Mapping** | schema (field types, analyzers) |
| **Inverted index** | term → posting list (doc IDs containing it) |
| **Analyzer** | tokenizer + filters (lowercase, stem, stop) |
| **Shard** | physical partition of an index |
| **Replica** | copy of a shard for HA + read scale |
| **Relevance score** | per-doc score (TF/IDF, BM25) |
| **Aggregation** | OLAP-style group/metric on indexed data |

---

## Core Concept

An **inverted index** flips the table: instead of `doc → words`, it's `word → list-of-docs-containing-it`. Searching is then a fast set intersection.

The pipeline before indexing:
1. **Tokenize** — split text into tokens
2. **Lowercase / normalize**
3. **Filter** — remove stopwords, stem, apply synonyms
4. **Index** — store tokens → doc IDs with positions for phrase queries

Querying applies the **same analyzer** to the input, then scores matching docs (BM25 by default in modern Lucene).

Search engines also do:
- **Faceting** — counts per category for filter UIs ("brand: Dell (32), HP (18)")
- **Aggregations** — date histograms, stats, percentiles
- **Highlighting** — surround matched terms with `<mark>`
- **Suggesters** — autocomplete, "did you mean"
- **Geo / vector / range** queries

When Postgres FTS hits limits (relevance tuning is awkward, faceting performance, multi-language analyzers), graduate to Elasticsearch / OpenSearch. Mirror data via CDC ([[13 - ETL and CDC]]).

---

## Syntax & API

### Create an index with mapping
```bash
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "english_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stop", "english_stemmer"]
        }
      },
      "filter": {
        "english_stop":     { "type": "stop",     "stopwords": "_english_" },
        "english_stemmer":  { "type": "stemmer",  "language": "english" }
      }
    }
  },
  "mappings": {
    "properties": {
      "name":     { "type": "text",    "analyzer": "english_custom" },
      "brand":    { "type": "keyword" },
      "category": { "type": "keyword" },
      "price":    { "type": "scaled_float", "scaling_factor": 100 },
      "tags":     { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}
```

### Index a document
```bash
POST /products/_doc/1
{ "name": "Dell XPS 15 Laptop",
  "brand": "Dell", "category": "Laptops",
  "price": 1499.99, "tags": ["work","portable"], "created_at": "2026-04-01" }
```

### Search
```bash
GET /products/_search
{
  "query": {
    "bool": {
      "must":   [{ "match": { "name": "laptop fast" } }],
      "filter": [
        { "term":  { "brand": "Dell" } },
        { "range": { "price": { "lte": 2000 } } }
      ]
    }
  },
  "highlight": { "fields": { "name": {} } },
  "size": 20
}
```

`must` contributes to the score; `filter` is a yes/no test (not scored, often cached).

### Aggregations (analytics)
```bash
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": { "field": "brand", "size": 10 },
      "aggs": { "avg_price": { "avg": { "field": "price" } } }
    },
    "monthly": {
      "date_histogram": { "field": "created_at", "calendar_interval": "month" }
    }
  }
}
```

### Bulk indexing (preferred over single-doc)
```bash
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "Laptop A", "brand": "Dell", "price": 999 }
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "Laptop B", "brand": "HP",   "price": 899 }
```

### Update by query / delete
```bash
POST /products/_update_by_query
{
  "query":  { "term": { "brand": "Dell" } },
  "script": { "source": "ctx._source.discounted = true" }
}

POST /products/_delete_by_query
{ "query": { "range": { "created_at": { "lt": "2020-01-01" } } } }
```

### .NET client (NEST / Elastic.Clients.Elasticsearch)
```csharp
using Elastic.Clients.Elasticsearch;

var client = new ElasticsearchClient(new Uri("http://localhost:9200"));

// Index
await client.IndexAsync(new Product { Id = 1, Name = "Laptop", Brand = "Dell" },
    i => i.Index("products"));

// Search
var resp = await client.SearchAsync<Product>(s => s
    .Index("products")
    .Query(q => q
        .Bool(b => b
            .Must(m => m.Match(t => t.Field(p => p.Name).Query("laptop fast")))
            .Filter(f => f.Term(t => t.Field(p => p.Brand).Value("Dell"))))));

foreach (var hit in resp.Hits) Console.WriteLine(hit.Source!.Name);
```

### Reindex (rebuild index)
```bash
POST /_reindex
{
  "source": { "index": "products" },
  "dest":   { "index": "products_v2" }
}

# Then alias swap (zero-downtime cutover)
POST /_aliases
{ "actions": [
    { "remove": { "index": "products",    "alias": "products_search" }},
    { "add":    { "index": "products_v2", "alias": "products_search" }}
]}
```

---

## Common Patterns

```text
Pattern: Postgres-as-truth + Elasticsearch-as-search
- App writes to Postgres
- CDC (Debezium) emits row events to Kafka
- A worker consumes events, transforms to ES documents
- Search UI reads from ES; reads from Postgres for full record / writes
```

```text
Pattern: alias-based zero-downtime reindex
- Always query through an alias (products_search)
- Build new index → reindex → atomic alias swap → drop old
- Schema/mapping changes need a new index, not in-place ALTER
```

```text
Pattern: hot-warm-cold tiered storage
- Hot index: SSDs, recent data, high replicas
- Warm index: spinning disk, older
- Cold index: object storage (searchable snapshots in ES Enterprise)
- Index lifecycle management (ILM) automates the tier movement
```

```bash
# Pattern: faceted UI counts
GET /products/_search
{
  "size": 0,
  "query": { "match": { "name": "laptop" } },
  "aggs": {
    "brands":     { "terms": { "field": "brand" } },
    "categories": { "terms": { "field": "category" } },
    "price_buckets": { "range": {
        "field": "price",
        "ranges": [{ "to": 500 }, { "from": 500, "to": 1500 }, { "from": 1500 }]
    }}
  }
}
```

---

## Gotchas & Tips

- **Eventual consistency** — search lags writes by milliseconds–seconds. Don't read-your-writes from search.
- **Mappings are mostly immutable** — adding fields is fine; changing a field's type or analyzer requires reindex + alias swap.
- **`text` vs `keyword`** — `text` is analyzed (full-text search); `keyword` is exact (filter, sort, agg). A field can be both via subfields: `name` (text) + `name.raw` (keyword).
- **Ignored fields silently fail** — strict mappings (`dynamic: strict`) reject unknown fields; loose mappings (`dynamic: true`) auto-create them and explode mapping size.
- **Shard count is set at index-creation** — too few = scaling cap; too many = wasted overhead. Aim for 10–50 GB per shard.
- **Refresh interval trades latency for ingest** — default 1s makes new docs visible quickly but burns CPU on heavy writes. Bump to 30s or `-1` (manual refresh) during bulk loads.
- **`source: false` for fields you only filter/aggregate** — saves storage, but blocks reading the original.
- **Aggregations are heavy** — date histograms over 100M docs are real CPU. Cache aggressively.
- **Don't use ES as primary store** — it's not ACID. Postgres or another OLTP is the source of truth; ES is a derived view.
- **License change for Elasticsearch 7.11+** — moved to SSPL/ELv2; OpenSearch is the AGPL-friendly fork.
- **Score isn't comparable across indices** — the same query gets different scores in different indexes (different IDF). Normalize if you must.
- **Use `_msearch`** for parallel queries — one HTTP round-trip, multiple queries.

---

## See Also

- [[19 - Full-Text Search]]
- [[20 - NoSQL Fundamentals]]
- [[13 - ETL and CDC]]
- [[18 - Vector Databases]]
