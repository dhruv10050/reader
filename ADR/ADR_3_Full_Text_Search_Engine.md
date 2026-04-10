# ADR 3: Use Apache Lucene for Full-Text Search

## Status
Accepted

## Context
Sismics Reader needs full-text search capabilities to allow users to search across article titles, descriptions, content, and creator metadata. Users expect fast, relevant search results with features like highlighting and grouping by feed.

Options considered:
- **Database full-text search (PostgreSQL tsvector / HSQLDB FTS)** — Simple to set up but database-dependent and limited in features.
- **Apache Lucene** — Mature, embedded Java search library with rich query capabilities.
- **Apache Solr** — Lucene-based search server with HTTP API, but adds operational complexity as a separate service.
- **Elasticsearch** — Distributed search engine, powerful but requires separate infrastructure and is overkill for a single-node RSS reader.

## Decision
We will use **Apache Lucene 4.2.0** as an embedded full-text search library, integrated directly into the application process. The search index will support two storage modes: **file-based** (persistent, for production) and **RAM-based** (in-memory, for development/testing). A custom `ReaderStandardAnalyzer` will handle language-aware tokenization.

Indexing will be managed asynchronously through the Guava EventBus: article creation, update, and deletion events trigger corresponding Lucene index operations via dedicated async listeners. A background `IndexingService` handles index optimization and full rebuild capabilities.

## Consequences

### Positive
- **No external dependencies**: Lucene runs embedded within the application JVM — no separate search server to deploy or manage.
- **Rich search features**: Supports complex query syntax (boolean, phrase, wildcard), result highlighting, grouping by feed, and relevance-based scoring.
- **Dual storage modes**: RAM-based indexing for fast development/testing cycles; file-based indexing for persistent production use.
- **Asynchronous indexing**: Article indexing runs in background threads via the EventBus, preventing search operations from blocking the main request processing.
- **Rebuild capability**: The full index can be rebuilt from the database if corrupted, providing self-healing recovery.

### Negative
- **Lucene 4.2 is significantly outdated**: This version (released 2013) lacks the performance improvements, new analyzers, and API enhancements available in current Lucene releases.
- **Single-node only**: Lucene is an embedded library without built-in distribution, limiting horizontal scalability if the article volume grows very large.
- **Index corruption risk**: File-based Lucene indexes can become corrupted on unclean shutdowns, though the rebuild capability mitigates this.
- **Memory overhead**: The RAM-based storage mode consumes JVM heap memory proportional to the index size, which can be significant with many articles.
