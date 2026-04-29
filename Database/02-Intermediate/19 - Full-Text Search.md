---
tags: [database, intermediate, postgresql, indexing]
aliases: [tsvector, tsquery, FTS, to_tsvector]
level: Intermediate
---

# Full-Text Search

> **One-liner**: Postgres has built-in linguistic full-text search — tokenize, stem, index with GIN, rank with `ts_rank`. Good enough to skip Elasticsearch for most apps.

---

## Quick Reference

| Type | Meaning |
|------|---------|
| `tsvector` | parsed, stemmed, weighted document |
| `tsquery` | search expression with `&`, `|`, `!`, `<->`, `:*` |
| `to_tsvector('english', text)` | parse text → tsvector |
| `to_tsquery('postgres & index')` | parse query string |
| `plainto_tsquery('postgres index')` | safer for user input — AND of words |
| `phraseto_tsquery('quick brown')` | phrase with positional operators |
| `websearch_to_tsquery('"postgres" -mysql')` | Google-like (PG 11+) |
| `@@` | match operator (`tsv @@ tsq`) |
| `ts_rank(tsv, tsq)` | rank by frequency |
| `ts_rank_cd(tsv, tsq)` | cover-density rank (positional) |
| `ts_headline(text, tsq)` | snippet with matches highlighted |

| Index | For |
|-------|-----|
| `GIN` on tsvector | the standard FTS index |
| `GIST` on tsvector | smaller, slower; rare |
| `pg_trgm` GIN | substring / fuzzy / ILIKE acceleration (separate concept) |

---

## Core Concept

Full-text search isn't substring matching. The DB **tokenizes** the document (splits into words), **normalizes** (lowercase, stems — "running" → "run"), drops **stopwords** ("the", "and"), and stores the result as a `tsvector` — a sorted list of unique stems with positions.

A query is a `tsquery` of stems combined with `&` (AND), `|` (OR), `!` (NOT), `<->` (followed by), and `:*` (prefix). The `@@` operator returns true if the document matches.

The whole pipeline is **language-aware** via configurations like `english`, `simple`, `russian`. Stemming and stopword lists differ per language. `simple` does no stemming — useful for codes, IDs, exact tokens.

`ts_rank` scores by frequency; `ts_rank_cd` adds **cover density** — rewarding matches close together. Combine with `ORDER BY rank DESC LIMIT`.

For substring or fuzzy search (typos, partials), use `pg_trgm` instead — different beast, see [[05 - Indexes Advanced]].

---

## Syntax & API

### Schema with a stored, generated tsvector
```sql
CREATE TABLE articles (
    id         INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    search_tsv TSVECTOR
        GENERATED ALWAYS AS (
            setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
            setweight(to_tsvector('english', coalesce(body,'')),  'B')
        ) STORED
);

CREATE INDEX idx_articles_search ON articles USING gin (search_tsv);
```

### Search and rank
```sql
SELECT id, title,
       ts_rank(search_tsv, q) AS rank
FROM articles, websearch_to_tsquery('english', 'postgres index') AS q
WHERE search_tsv @@ q
ORDER BY rank DESC
LIMIT 20;
```

### Highlighting / snippet
```sql
SELECT id,
       ts_headline('english', body,
           websearch_to_tsquery('english', 'postgres index'),
           'StartSel=<mark>, StopSel=</mark>, MaxFragments=2, MinWords=10'
       ) AS snippet
FROM articles
WHERE search_tsv @@ websearch_to_tsquery('english','postgres index');
```

### Query syntax
```sql
-- AND
SELECT * FROM articles WHERE search_tsv @@ to_tsquery('english','postgres & index');

-- OR
SELECT * FROM articles WHERE search_tsv @@ to_tsquery('english','postgres | mysql');

-- NOT
SELECT * FROM articles WHERE search_tsv @@ to_tsquery('english','postgres & !slow');

-- Phrase
SELECT * FROM articles WHERE search_tsv @@ phraseto_tsquery('english','quick brown fox');

-- Prefix (autocomplete)
SELECT * FROM articles WHERE search_tsv @@ to_tsquery('english','post:*');

-- User-friendly with quotes / minus (Google-like)
SELECT * FROM articles WHERE search_tsv @@ websearch_to_tsquery('english','"foreign key" -mysql');
```

### Inspect parsing
```sql
SELECT to_tsvector('english','The quick brown foxes jumped over lazy dogs.');
-- 'brown':3 'fox':4 'jump':5 'lazi':7 'dog':8 'quick':2

SELECT to_tsquery('english','foxes & jumping');
-- 'fox' & 'jump'
```

### Multiple weights for relevance
```sql
-- 'A' (highest) for title, 'B' for body, 'C' for tags
SELECT setweight(to_tsvector('english','PostgreSQL deep dive'), 'A') ||
       setweight(to_tsvector('english','indexes and partitions'),'B');
```

### Custom dictionary / config
```sql
-- Make 'sql' and 'postgresql' first-class (avoid stemming weirdness)
CREATE TEXT SEARCH CONFIGURATION my_eng (COPY = english);
ALTER TEXT SEARCH CONFIGURATION my_eng
    ALTER MAPPING FOR asciiword WITH simple;

SELECT to_tsvector('my_eng','postgresql versus mysql');
```

### Searching JSONB content
```sql
-- Index the concatenated text inside JSONB
CREATE INDEX idx_events_search
    ON events USING gin (
        to_tsvector('english', payload->>'description')
    );

SELECT id FROM events
WHERE to_tsvector('english', payload->>'description') @@ websearch_to_tsquery('error');
```

---

## Common Patterns

```sql
-- Pattern: trigram fallback for fuzzy / typo tolerance
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_articles_title_trgm
    ON articles USING gin (title gin_trgm_ops);

-- ILIKE substring + similarity ranking
SELECT id, title, similarity(title, 'postgrss') AS sim
FROM articles
WHERE title % 'postgrss'        -- trigram-similar
ORDER BY sim DESC LIMIT 10;
```

```sql
-- Pattern: combine FTS rank with business signals
SELECT id, title,
       ts_rank(search_tsv, q) * (1 + log(views + 1)) AS final_rank
FROM articles, websearch_to_tsquery('english', :query) q
WHERE search_tsv @@ q
ORDER BY final_rank DESC LIMIT 20;
```

```sql
-- Pattern: stored tsvector vs computed-on-the-fly
-- Stored (column + GIN index) is the right default for searchable tables.
-- Computed only useful for one-off ad-hoc reports.
```

---

## Gotchas & Tips

- **Use `websearch_to_tsquery` for user input** — it tolerates Google-style syntax and won't error out on stray characters like `to_tsquery` does.
- **Never feed `to_tsquery` raw user text** — `to_tsquery('foo bar')` errors (no operator); `plainto_tsquery` and `websearch_to_tsquery` are safe.
- **Stemming is per-language** — using `english` config on French text gives garbage. `simple` is a safe-ish multilingual fallback.
- **`STORED` generated column + GIN** is the canonical pattern — keeps the index up to date automatically without triggers.
- **`tsvector` GIN updates are slow** — heavy-write tables take a hit. Tune autovacuum and consider partial indexes if only some rows are searchable.
- **Result counts are not free** — Postgres needs to scan all matches to count. For paginated UIs, limit the count or use estimated counts.
- **Trigram (`pg_trgm`) is not FTS** — different problems. Use trigram for "did you mean" / partial / typo, FTS for "what's relevant for these words."
- **Phrase search needs lexeme positions** — `phraseto_tsquery` and `<->` distance work because tsvector includes positions.
- **Highlight quality** — `ts_headline` re-parses; expensive on long documents. Cache results when possible.
- **For massive corpora or analyzers beyond Postgres' built-ins**, switch to Elasticsearch ([[16 - Search Engines]]). Up to a few million docs, Postgres FTS keeps things simple.

---

## See Also

- [[05 - Indexes Advanced]]
- [[18 - JSON and JSONB]]
- [[06 - Query Optimization]]
- [[16 - Search Engines]]
