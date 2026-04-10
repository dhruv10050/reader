# ADR 7: Use OWASP HTML Sanitizer for Content Security

## Status
Accepted

## Context
Sismics Reader aggregates content from external RSS/Atom feeds, which may contain arbitrary HTML. This HTML is rendered in the web interface and Android app. Without proper sanitization, malicious feed authors could inject JavaScript, iframes, or other dangerous HTML elements, leading to Cross-Site Scripting (XSS) attacks, phishing overlays, or other security vulnerabilities.

Options considered:
- **Strip all HTML** — Safe but loses formatting, images, and links from articles, degrading the reading experience.
- **HTML whitelist with regex** — Error-prone and easily bypassed; regex-based HTML parsing is a known anti-pattern.
- **Jsoup Cleaner** — Java HTML parser with whitelist-based cleaning, good but less focused on security.
- **OWASP Java HTML Sanitizer** — Security-focused HTML sanitizer built by the OWASP project with a policy-based approach.
- **Bleach (Python)** — Not applicable to a Java application.

## Decision
We will use the **OWASP Java HTML Sanitizer** library to sanitize all article HTML content before storing it in the database. The `ArticleSanitizer` class applies a configurable sanitization policy that preserves safe HTML elements (text formatting, images, links) while stripping dangerous elements (scripts, iframes, event handlers, data URIs).

Additionally, **TagSoup** is used as a pre-processing step to parse and normalize malformed HTML from feeds before sanitization, ensuring that even broken HTML is properly handled rather than passed through unsanitized.

## Consequences

### Positive
- **Strong XSS protection**: OWASP HTML Sanitizer is specifically designed for security, with a policy-based whitelist approach that defaults to denying all elements.
- **Preserved reading experience**: Unlike stripping all HTML, the sanitizer preserves safe formatting (bold, italic, lists, images, links) while removing dangerous elements.
- **Malformed HTML handling**: TagSoup pre-processing ensures that broken or malformed feed HTML is normalized before sanitization, preventing parser bypass attacks.
- **OWASP backing**: The library is maintained by the OWASP community, benefiting from regular security reviews and updates.
- **Configurable policies**: The sanitization policy can be adjusted to allow or deny specific HTML elements and attributes as requirements evolve.

### Negative
- **Processing overhead**: Every article's HTML content is parsed and re-serialized during sanitization, adding CPU overhead during feed synchronization.
- **Potential content loss**: Aggressive sanitization may strip legitimate HTML elements used by some feeds (e.g., embedded videos, custom styling), reducing content fidelity.
- **Dependency on external library**: The sanitizer must be kept updated to address newly discovered bypass techniques and HTML5 attack vectors.
