# AI Usage Report - Sismics Reader Architecture Documentation

## Overview
This report documents the process of generating architectural diagrams (C1, C2, C3) and Architectural Decision Records (ADRs) for the Sismics Reader open-source RSS/Atom feed aggregator.

---

## 1. C1 - System Context Diagram

### How it was generated
- **Tool**: PlantUML with C4-PlantUML macros (`!include <C4/C4_Context>`)
- **Approach**: The repository was explored using automated codebase analysis tools to understand the system's external interfaces and actors. The PlantUML source was authored manually based on the analysis, then rendered to PNG using the PlantUML CLI (`java -jar plantuml.jar -tpng c1.puml`) with Graphviz for layout.

### What the initial generated version contained
The initial version identified:
- Two actor types: Reader User and Administrator
- The central Sismics Reader system
- Three external systems: RSS/Atom Feed Sources, Favicon Servers, and OPML Sources
- Relationships showing web browser access, mobile app access, feed fetching, favicon downloading, and OPML import

### What changes were made
- Refined the description of the Sismics Reader system to emphasize its key capabilities (subscribe, organize into categories, read, full-text search)
- Added the Android App relationship as a separate connection from the user to distinguish mobile from web access
- Changed OPML Sources relationship direction to show it as an inbound data flow (file upload) rather than an outbound fetch
- Added protocol annotations (HTTPS, HTTP/HTTPS, OPML/JSON File Upload) to all relationship arrows for clarity

### Why those changes were necessary
- The initial version underspecified the mobile access pattern, which is a distinct deployment target (Android app) separate from the web SPA
- Protocol annotations are essential in C4 context diagrams to communicate how systems interact at a high level
- The OPML relationship direction was corrected because OPML files are uploaded by users, not fetched by the system from external sources

---

## 2. C2 - Container Diagram

### How it was generated
- **Tool**: PlantUML with C4-PlantUML macros (`!include <C4/C4_Container>`)
- **Approach**: Each Maven module in the repository was analyzed to identify deployable containers. The `pom.xml` files, web.xml configuration, and source code structure were examined to determine technology stacks and inter-container communication patterns.

### What the initial generated version contained
The initial version identified 8 containers within the system boundary:
- Web Application (AngularJS SPA)
- REST API Server (Jersey/Jetty)
- Android App
- Desktop Agent (Swing/embedded Jetty)
- Relational Database (HSQLDB/PostgreSQL)
- Lucene Search Index
- Feed Synchronization Service
- Async Event Bus (Guava EventBus)

### What changes were made
- Separated the Event Bus as a distinct container rather than embedding it within the REST API server, because it serves as an independent communication backbone connecting multiple components
- Added the Feed Synchronization Service as a distinct container because it runs as a background `AbstractScheduledService` with its own lifecycle, separate from the request-processing REST API
- Clarified that the Desktop Agent both embeds and serves the REST API (in-process relationship) rather than calling it over HTTP
- Added database access relationship from the Feed Synchronization Service directly, since it writes articles and feed metadata independently of the REST API

### Why those changes were necessary
- The Event Bus is architecturally significant — it decouples feed synchronization from Lucene indexing and other async operations; showing it as a separate container communicates this design decision
- The Feed Sync Service has a distinct runtime lifecycle (scheduled execution) and shouldn't be conflated with the request-driven REST API
- The Desktop Agent's relationship with the REST API is in-process (embedded WAR), not over HTTP — this is a materially different deployment topology

---

## 3. C3 - Component Diagram

### How it was generated
- **Tool**: PlantUML with C4-PlantUML macros (`!include <C4/C4_Component>`)
- **Approach**: The Java source code in `reader-core` and `reader-web` modules was analyzed in detail — all REST resource classes, DAO classes, service classes, event listeners, and utility classes were cataloged. The component diagram focuses on the REST API Server container, decomposing it into its internal components.

### What the initial generated version contained
The initial version mapped all major components:
- 8 REST Resource components (UserResource, SubscriptionResource, ArticleResource, CategoryResource, StarredResource, SearchResource, AppResource, JobResource)
- Security filters (TokenBasedSecurityFilter, HeaderBasedSecurityFilter)
- 7 JPA DAO components
- 1 Lucene DAO component
- 2 Background services (FeedService, IndexingService)
- Feed parsing components (RssReader, OpmlReader)
- Utility components (ArticleSanitizer, ReaderHttpClient)
- Event Bus & Listeners component
- AppContext singleton

### What changes were made
- Added ValidationUtil as a separate component to show input validation as a cross-cutting concern
- Added the HTML Sanitizer (ArticleSanitizer) as a component connected to the FeedService, since sanitization happens during feed sync, not during API response rendering
- Connected the Event Bus to the HTTP Client for favicon downloads (FaviconUpdateRequestedEvent triggers HTTP download)
- Added the AppContext component to show lifecycle management of services and the event bus
- Grouped JPA DAOs to show they all connect to the database via JDBC/JPA, reducing visual clutter while maintaining accuracy

### Why those changes were necessary
- The ArticleSanitizer's connection point matters architecturally — it processes content during ingestion (feed sync), not during delivery (API response), which affects where security controls are applied
- The AppContext is the application's lifecycle manager and a critical architectural component that initializes and coordinates services, the event bus, and thread pools
- Showing favicon download as an event-driven HTTP operation (EventBus → HttpClient → Favicon Servers) accurately represents the async, decoupled nature of this operation

---

## 4. Architectural Decision Records (ADRs)

### How they were generated
- **Tool**: Manual authoring in Markdown format following the standard ADR template (Status, Context, Decision, Consequences)
- **Approach**: The codebase was analyzed to identify the most architecturally significant technology and design decisions. For each ADR, the relevant source code, configuration files, and dependency declarations were examined to understand the rationale behind each decision.

### What the initial generated versions contained
7 ADRs were created covering the major architectural decisions:

1. **ADR 1 - RESTful API Framework**: Jersey 1.19 (JAX-RS) selection for REST API
2. **ADR 2 - Data Persistence Strategy**: Hibernate JPA with HSQLDB/PostgreSQL and custom migrations
3. **ADR 3 - Full-Text Search Engine**: Apache Lucene 4.2 for embedded search
4. **ADR 4 - Authentication Mechanism**: Custom token-based stateless authentication with jBCrypt
5. **ADR 5 - Async Event Architecture**: Guava EventBus for decoupled async operations
6. **ADR 6 - Multi-Platform Distribution**: Six distribution formats (standalone, Debian, Red Hat, Mac, Windows, Docker)
7. **ADR 7 - Content Security Sanitization**: OWASP HTML Sanitizer for XSS prevention

### What changes were made
- Added specific version numbers and configuration details (e.g., C3P0 pool sizes, Hibernate dialect settings) to ground the ADRs in the actual codebase rather than generic descriptions
- Expanded the "Negative" consequences section for each ADR to include version-specific concerns (e.g., Jersey 1.x being legacy, Hibernate 4.x missing newer features, Lucene 4.2 being outdated)
- Added the comparison of alternatives considered for each decision to document why other options were rejected
- Included the dual-filter authentication pipeline detail (TokenBased + HeaderBased) in ADR 4, as this is a unique architectural choice
- Added the DeadEventListener detail in ADR 5 as it represents an important debugging/observability pattern

### Why those changes were necessary
- ADRs should be grounded in the specific codebase, not generic technology descriptions — version numbers and configuration details make them actionable for future developers
- Documenting alternatives considered is a key part of the ADR format that helps future architects understand the decision space
- Negative consequences are often underspecified in initial drafts but are the most valuable part of an ADR for future decision-makers evaluating whether to change the architecture
- The version-specific concerns (legacy Jersey 1.x, outdated Lucene 4.2) are critical for a maintenance-focused project to understand its technical debt

---

## Tools Used Summary

| Artifact | Authoring Tool | Rendering Tool | Output Format |
|----------|---------------|----------------|---------------|
| C1 Diagram | PlantUML (C4 macros) | PlantUML CLI + Graphviz | PNG |
| C2 Diagram | PlantUML (C4 macros) | PlantUML CLI + Graphviz | PNG |
| C3 Diagram | PlantUML (C4 macros) | PlantUML CLI + Graphviz | PNG |
| ADRs (7) | Markdown (standard ADR template) | N/A | .md files |

All PlantUML source files (`.puml`) are included alongside the PNG images for editability.
