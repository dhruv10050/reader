# ADR 5: Use Guava EventBus for Asynchronous Event-Driven Architecture

## Status
Accepted

## Context
Sismics Reader requires decoupled communication between components for operations such as article indexing in Lucene after feed synchronization, favicon downloading when a feed is discovered, and bulk subscription import processing. These operations are I/O-intensive and should not block the main request-processing threads.

Options considered:
- **Direct method calls** — Simple but creates tight coupling between components and blocks the calling thread.
- **Java ExecutorService with Runnables** — Manual thread management without a publish/subscribe model.
- **JMS (Java Message Service)** — Enterprise messaging with queues/topics, but requires a message broker (ActiveMQ, RabbitMQ).
- **Guava EventBus** — Lightweight, in-process publish/subscribe event bus with synchronous and asynchronous variants.
- **Spring Application Events** — Integrated into Spring Framework, but adds Spring dependency.

## Decision
We will use **Google Guava's EventBus** (both synchronous and asynchronous variants) for in-process event-driven communication. The `AppContext` singleton holds the EventBus instances and manages listener registration.

Key event flows:
- **ArticleCreatedAsyncEvent** → `ArticleCreatedAsyncListener` → Lucene index creation
- **ArticleUpdatedAsyncEvent** → `ArticleUpdatedAsyncListener` → Lucene index update
- **ArticleDeletedAsyncEvent** → `ArticleDeletedAsyncListener` → Lucene index deletion
- **FaviconUpdateRequestedEvent** → `FaviconUpdateRequestedAsyncListener` → Favicon download
- **SubscriptionImportedEvent** → `SubscriptionImportAsyncListener` → Bulk OPML import processing
- **RebuildIndexAsyncEvent** → `RebuildIndexAsyncListener` → Full Lucene index rebuild

A synchronous `DeadEventListener` catches unhandled events for debugging.

## Consequences

### Positive
- **Loose coupling**: Publishers (FeedService, REST resources) don't need direct references to listeners (Lucene indexing, favicon downloading), improving modularity.
- **Non-blocking operations**: Async listeners execute in a thread pool, preventing I/O-intensive operations (HTTP requests, Lucene writes) from blocking API responses.
- **Simple API**: Guava EventBus provides a clean `@Subscribe` annotation-based API that is easy to understand and extend.
- **No external infrastructure**: Unlike JMS, the EventBus runs entirely in-process without requiring a separate message broker service.
- **Dead event detection**: The `DeadEventListener` catches events with no registered listeners, helping identify misconfigured event flows during development.

### Negative
- **In-process only**: Events are lost if the application crashes before listeners complete processing; no persistence or replay capability unlike JMS.
- **No guaranteed delivery**: Failed listener execution (e.g., Lucene write failure) silently drops the event with only logging; no retry mechanism.
- **Limited scalability**: The in-process EventBus doesn't support distributed event processing across multiple application instances.
- **Guava EventBus is deprecated-in-spirit**: Google recommends reactive streams or other patterns for new projects; EventBus is maintained but no longer actively enhanced.
- **Debugging difficulty**: Asynchronous event dispatch can make it harder to trace the flow of operations through the system compared to direct method calls.
