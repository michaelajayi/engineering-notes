---
title: Use PostgreSQL for transactional data instead of DynamoDB
date: 2026-05-15
status: accepted
description: Why strong consistency and relational integrity won over DynamoDB for financial records.
tags: [databases, architecture]
stack: [postgresql, dynamodb]
---

## Context

The platform needed a primary data store for core transactional data: user accounts, payment records, and job state. Existing DynamoDB infrastructure was already in use for other workloads, which created pressure to default to it here too.

DynamoDB is a reasonable choice when access patterns are known upfront, data is denormalized, and strong consistency isn't required everywhere. For financial records, none of those conditions hold cleanly.

## Decision

Use PostgreSQL (via RDS) for all transactional data. DynamoDB remains appropriate for high-throughput lookup tables and caching layers where eventual consistency is acceptable.

## Reasoning

**Transactions.** Payment records require multi-table atomicity. DynamoDB transactions exist but are limited to 25 items and add latency. PostgreSQL transactions are first-class.

**Query flexibility.** Financial reporting requires ad-hoc aggregations across dimensions that can't be fully predicted at schema design time. A relational model lets you write the query when you need it. With DynamoDB, every new access pattern requires a new index designed upfront.

**Referential integrity.** Foreign key constraints at the database level catch bugs that application-layer validation misses. This matters more in a payment domain than most.

**Operational familiarity.** PostgreSQL was the stronger operational ground for this team. Choosing a well-understood tool for the most critical data store reduces incident risk.

## Consequences

- RDS requires more operational care than DynamoDB (backups, failover, connection pooling). Managed RDS and PgBouncer mitigate this.
- Read scaling requires explicit read replicas; DynamoDB scales reads automatically. Acceptable at current volume.
- DynamoDB remains in use for non-transactional workloads: session tokens, rate limit counters, feature flags.

## Status

Accepted. In production since May 2026.
