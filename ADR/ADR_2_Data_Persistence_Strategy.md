# ADR 2: Use Hibernate JPA with HSQLDB and PostgreSQL for Data Persistence

## Status
Accepted

## Context
Sismics Reader requires a data persistence layer to store users, feeds, articles, subscriptions, categories, authentication tokens, and system configuration. The application has approximately 13 database tables with complex relationships (user-article associations, feed-category hierarchies, role-based access control).

Key requirements include:
- Support for both lightweight development/testing databases and production-grade databases.
- Object-relational mapping for cleaner data access code.
- Connection pooling for production performance.
- Schema versioning for safe database migrations across releases.

Options considered:
- **Raw JDBC** — Maximum control but verbose and error-prone.
- **MyBatis** — SQL-centric ORM with XML mapping files.
- **Hibernate/JPA** — Full ORM with automatic mapping, criteria queries, and caching.
- **jOOQ** — Type-safe SQL builder, good for complex queries.

For the database engine:
- **H2 / HSQLDB** — Lightweight, embeddable, good for development.
- **PostgreSQL** — Production-grade, open-source RDBMS.
- **MySQL/MariaDB** — Popular but licensing considerations.
- **SQLite** — Too limited for multi-user web applications.

## Decision
We will use **Hibernate 4.2.21 (JPA 2.0)** as the ORM framework with **C3P0** connection pooling. The application will support **HSQLDB** for development/testing and **PostgreSQL** for production deployments. Database schema versioning will be managed through numbered SQL migration scripts (`dbupdate-NNN-0.sql`) applied incrementally at startup.

## Consequences

### Positive
- **Dual database support**: Developers can use lightweight HSQLDB for fast iteration while production uses battle-tested PostgreSQL.
- **ORM productivity**: Hibernate handles object-relational mapping, reducing boilerplate JDBC code and SQL string management.
- **Connection pooling**: C3P0 provides efficient connection management with configurable pool sizes (1-30 connections).
- **Schema versioning**: Incremental SQL migration scripts with a `DB_VERSION` config key ensure safe, ordered database upgrades across releases.
- **DAO pattern**: Clean separation between business logic and data access through dedicated DAO classes with criteria-based query builders.

### Negative
- **Hibernate complexity**: The ORM abstraction can produce inefficient SQL queries if not carefully managed (N+1 query problem, lazy loading pitfalls).
- **Custom migration system**: Using hand-rolled SQL migrations instead of established tools like Flyway or Liquibase increases maintenance burden and lacks rollback support.
- **Hibernate 4.x is outdated**: Hibernate 4.x has been superseded by 5.x and 6.x, missing newer features like improved type inference, better JPA 2.2 support, and optimized batch processing.
- **No rollback support**: The custom migration system only supports forward migrations, making it risky to roll back failed database changes.
