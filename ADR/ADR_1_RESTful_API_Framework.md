# ADR 1: Use Java with Jersey (JAX-RS) for RESTful API

## Status
Accepted

## Context
Sismics Reader needs a web framework to expose its functionality as RESTful API endpoints. The application must support multiple client types including a web-based single-page application (jQuery/Less), an Android mobile app, and a desktop agent. The chosen framework should integrate well with the Java ecosystem, support JSON serialization, and provide a clean resource-oriented API design.

Several options were considered:
- **Spring MVC** — Full-featured but heavy; brings a large dependency footprint and an opinionated application structure.
- **Jersey (JAX-RS Reference Implementation)** — Lightweight, standards-based REST framework focused on building RESTful web services.
- **Apache CXF** — Enterprise-grade but more complex configuration.
- **Play Framework** — Reactive and modern but requires a different development paradigm.

## Decision
We will use **Jersey 1.19.4** (JAX-RS 1.1 reference implementation) as the REST framework, deployed as a servlet within a standard Java Servlet Container (Jetty 9.4).

## Consequences

### Positive
- **Standards-based**: Follows the JAX-RS specification, making the API design portable and well-understood by Java developers.
- **Lightweight**: Minimal overhead compared to full-stack frameworks like Spring MVC, keeping the application footprint small.
- **Clean resource mapping**: Annotation-based routing (`@Path`, `@GET`, `@POST`, etc.) leads to readable and maintainable resource classes.
- **Multi-client support**: The stateless REST API naturally supports web, mobile, and desktop clients with the same endpoints.
- **Servlet compatibility**: Deploys as a standard WAR file to any Java Servlet container, enabling flexible deployment options (embedded Jetty, standalone Tomcat, etc.).

### Negative
- **Jersey 1.x is significantly outdated**: Jersey 1.x is multiple major versions behind the current JAX-RS specification, which limits access to newer features like async request processing, improved dependency injection, and modern Java support.
- **No built-in dependency injection**: Unlike Spring, Jersey 1.x requires manual wiring or integration with a DI framework, leading to more boilerplate in resource classes.
- **Limited middleware ecosystem**: Compared to Spring Boot, Jersey has fewer out-of-the-box integrations for security, monitoring, and configuration management.
