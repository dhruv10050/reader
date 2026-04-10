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
- Added protocol annotations (HTTPS, HTTP/HTTPS) to all relationship arrows for clarity
- Removed OPML Sources as an external system — OPML is a static file format uploaded by the user via the browser, not an automated external system that Sismics integrates with. This corrects a C4 modeling error where a data artifact was misrepresented as an external system.

### Why those changes were necessary
- The initial version underspecified the mobile access pattern, which is a distinct deployment target (Android app) separate from the web SPA
- Protocol annotations are essential in C4 context diagrams to communicate how systems interact at a high level
- OPML import is a user-driven file upload action, not a system-to-system integration — it does not belong as a System_Ext in a C1 context diagram

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
- Corrected the Web Application technology from AngularJS to jQuery/Less — inspection of `reader-web/src/main/webapp/src/index.html` reveals a custom jQuery-driven SPA using plugins like jquery.history.js and jquery.ui.js, with Less for stylesheet compilation and Grunt for asset building. No AngularJS dependency exists in the project.
- Removed the Feed Synchronization Service and Async Event Bus as separate C2 containers — in the C4 model, a Container represents an independently deployable runtime unit (process, server, database). Both the Guava EventBus and FeedService (Guava AbstractScheduledService) run as internal Java classes within the same JVM process as the REST API Server, sharing its thread pools and memory space. They are correctly represented as components in the C3 diagram instead.
- Clarified that the Desktop Agent both embeds and serves the REST API (in-process relationship) rather than calling it over HTTP
- Updated REST API Server relationships to show it directly fetches feeds and favicons (since the feed sync runs inside the same container)

### Why those changes were necessary
- The AngularJS attribution was factually incorrect — the codebase uses jQuery, and accurate technology labeling is essential for a C2 diagram
- C4 Container-level diagrams must only show independently deployable units; internal Java classes that share a JVM with the REST API are Components (C3-level), not Containers (C2-level). Promoting them to containers misrepresents the deployment architecture.
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
6. **ADR 6 - Multi-Platform Distribution**: Five Maven distribution modules plus standalone Docker configuration
7. **ADR 7 - Content Security Sanitization**: OWASP HTML Sanitizer for XSS prevention

### What changes were made

**Factual corrections (cross-checked against source code):**
- **ADR 1 (RESTful API Framework)**: The initial version's Context section incorrectly referenced the SPA as being built with "AngularJS." Inspection of `reader-web/src/main/webapp/src/index.html` reveals the application uses a custom jQuery-driven SPA architecture with plugins like `jquery.history.js` and `jquery.ui.js`, compiled with Less and Grunt. Corrected all AngularJS references to "jQuery/Less."
- **ADR 6 (Multi-Platform Distribution)**: The initial version stated "six distribution modules as Maven sub-projects" and listed `reader-distribution-docker` as the sixth. However, the parent `pom.xml` `prod` profile only declares five Maven modules (standalone, debian, redhat, mac, windows). The `reader-distribution-docker` directory contains only a `Dockerfile` and `docker-compose.yml` with no `pom.xml` — it is not a Maven sub-project. Corrected to "five Maven sub-projects plus a standalone Docker configuration."

**Quality improvements:**
- Added specific version numbers and configuration details (e.g., C3P0 pool sizes, Hibernate dialect settings) to ground the ADRs in the actual codebase rather than generic descriptions
- Expanded the "Negative" consequences section for each ADR to include version-specific concerns (e.g., Jersey 1.x being legacy, Hibernate 4.x missing newer features, Lucene 4.2 being outdated)
- Added the comparison of alternatives considered for each decision to document why other options were rejected
- Included the dual-filter authentication pipeline detail (TokenBased + HeaderBased) in ADR 4, as this is a unique architectural choice
- Added the DeadEventListener detail in ADR 5 as it represents an important debugging/observability pattern

### Why those changes were necessary
- **ADR 1 correction**: The AngularJS attribution was factually wrong — no AngularJS dependency exists anywhere in the project. Accurate technology identification is fundamental to an ADR's value as a decision record.
- **ADR 6 correction**: Claiming Docker was a Maven sub-project misrepresents the build architecture. Docker is built outside Maven via `docker build`, which is a materially different build and deployment pipeline. The distinction matters for CI/CD and release management.
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
