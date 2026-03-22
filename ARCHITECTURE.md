# Scalable GitHub Data Aggregation System

**Submitted by:** Himanshu Choudhary
**Program:** GSoC 2026 Pre-Task
**Project:** C2SI WebiU

---

## Architecture Diagram

<img width="1419" height="703" alt="image" src="https://github.com/user-attachments/assets/01bf38d7-fad7-4412-9eb9-7995f3741e03" />


---

## Overview

This document presents a scalable architecture for aggregating data from 300+ GitHub repositories and serving it efficiently to the WebiU platform. The design focuses on minimizing API usage, reducing response latency, and maintaining data freshness through event-driven updates.

The system follows a **cache-first, webhook-driven approach**, where data is served from Redis whenever possible, and GitHub API calls are made only when necessary.

---

## Design Goals

* Minimize unnecessary GitHub API calls
* Reduce response latency using caching
* Handle API rate limits efficiently
* Support scaling from hundreds to thousands of repositories
* Maintain data freshness using event-driven updates

These goals directly influence the use of Redis for caching and Webhooks for triggering updates.

---

## System Flow

1. The frontend sends a request to the backend
2. Backend checks Redis cache
3. If cache hit → response returned immediately
4. If cache miss:

   * Query MongoDB for stored data
   * If data is stale or missing → fetch from GitHub API
   * Update MongoDB and Redis
5. Backend returns response to frontend

Additionally:

* GitHub Webhooks trigger updates on repository changes
* This keeps data fresh without constant polling

---

## API Flow

### Frontend Request Flow

```
Frontend → Backend → Redis Check
                    → Cache HIT  → Return response
                    → Cache MISS → Query MongoDB
                                 → Fetch GitHub API
                                 → Update DB + Cache
                                 → Return response
```

### Webhook Flow

```
GitHub Event → Verify Signature → Invalidate Cache
            → Fetch Fresh Data → Update DB + Cache
```

This ensures fast responses while maintaining up-to-date data.

---

## Core Components

### Backend (Node.js / NestJS)

Central coordinator responsible for request handling, caching logic, webhook verification, and API communication.

### GitHub API (REST / GraphQL)

Primary data source. GraphQL is preferred for selective field fetching and reducing payload size.

### Redis (Upstash)

Shared caching layer used across all backend instances. Stores frequently accessed data with TTL-based expiry.

### MongoDB

Persistent storage for structured repository data such as metadata, contributors, and computed metrics.

### GitHub Webhooks

Event-driven update mechanism that triggers backend processing only when repository data changes.

---

## Caching Strategy

| Layer               | TTL          | Purpose                                   |
| ------------------- | ------------ | ----------------------------------------- |
| Redis (Upstash)     | 5 min – 1 hr | Shared cache across all instances         |
| Request-level cache | Per request  | Reduce redundant Redis calls for hot data |

Cache TTL is dynamically adjusted based on repository activity — frequently updated repositories use shorter TTLs, while inactive repositories use longer ones.

---

## Rate Limit Handling

| Strategy               | Effect                                          |
| ---------------------- | ----------------------------------------------- |
| GraphQL batch queries  | Fetch multiple repositories in a single request |
| Redis caching          | Avoid repeated API calls                        |
| Webhook-driven updates | API calls only when data changes                |
| ETag / Conditional GET | Use `If-None-Match` → 304 response if unchanged |
| Rate limit monitoring  | Pause non-critical requests when limit is low   |

Using ETags ensures that unchanged data does not consume API quota.

---

## Update Mechanism

### Primary — GitHub Webhooks

* Triggered on push, PR, and issue events
* Backend verifies webhook signature
* Cache is invalidated
* Fresh data is fetched and stored
* Cache is updated

Latency: **< 5 seconds**

---

### Fallback — Scheduled Sync

* Runs every 6 hours
* Handles missed webhook events
* Uses `pushed_at` timestamp to skip unchanged repositories

---

## Data Storage Strategy

### Stored in MongoDB

* Repository metadata (name, description, language, topics)
* Star and fork counts
* Contributor lists
* PR and issue counts

### Cached in Redis

* Live star counts
* Recent commits
* Real-time PR status

---

## Scalability Plan

| Scale                | Strategy                                                          |
| -------------------- | ----------------------------------------------------------------- |
| ~300 repositories    | Single backend instance with Redis and MongoDB                    |
| ~1,000 repositories  | Add read replicas, increase cache TTL                             |
| ~10,000 repositories | Multiple GitHub tokens, queue-based ingestion, horizontal scaling |

Key principles:

* Stateless backend enables horizontal scaling
* Redis absorbs most read traffic
* GraphQL batching reduces API calls

---

## Performance Optimization

* Redis caching reduces database load
* GraphQL minimizes payload size
* Pagination limits large responses (50 repos per page)
* Gzip compression reduces response size

---

## Failure Handling

| Failure              | Response                             |
| -------------------- | ------------------------------------ |
| GitHub API down      | Serve stale data from Redis          |
| Redis unavailable    | Query MongoDB directly               |
| Rate limit exhausted | Serve cached data and queue requests |
| Webhook failure      | Handled by scheduled sync            |

---

## Technology Justification

| Component   | Technology        | Reason                                         |
| ----------- | ----------------- | ---------------------------------------------- |
| Backend     | Node.js / NestJS  | Matches WebiU stack, strong TypeScript support |
| Cache       | Redis (Upstash)   | Shared, serverless-friendly caching            |
| Database    | MongoDB           | Already used in WebiU                          |
| API         | GitHub GraphQL v4 | Reduces over-fetching                          |
| Real-time   | GitHub Webhooks   | Eliminates polling                             |
| Compression | Gzip              | Improves performance                           |

---

## Design Decisions

- Redis is used instead of in-memory caching to ensure consistency across multiple instances
- Webhooks are preferred over polling to reduce unnecessary API calls
- GraphQL is used over REST for selective data fetching and reduced payload size
- MongoDB is used for flexibility in storing varying repository metadata

These choices are made to optimize performance, scalability, and API efficiency under real-world constraints.

---

## Conclusion

This architecture prioritizes simplicity while addressing real-world constraints such as rate limits, latency, and scalability. By combining caching, event-driven updates, and efficient API usage, the system ensures fast, reliable, and scalable data aggregation for the WebiU platform.
