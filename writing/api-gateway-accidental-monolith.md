---
title: When your API gateway becomes an accidental monolith
date: 2026-06-27
description: On what a gateway should and shouldn't own, and the rule I use to keep it honest.
tags: [architecture, distributed-systems]
stack: [api-gateway]
---

An API gateway starts small. Route traffic, handle auth, maybe rate-limit. Clean, obvious responsibilities.

Then someone needs request transformation. The gateway already sits between clients and services, so it feels like the natural place. You add it. Six months later, the gateway validates business rules, aggregates responses from three services, and holds a retry budget that belongs in the calling service. The thing meant to route traffic is now making decisions.

This is the accidental monolith: not one giant codebase, but one component that accretes responsibility until it becomes the hardest thing in the system to change.

## The rule I use

A gateway should know *where* to send a request, not *what* the request means.

Put differently: if removing the gateway requires rewriting business logic, you've already lost. The gateway should be deletable and replaceable without touching the services behind it.

In practice, I apply a single test to each thing the gateway does:

> Does this logic change when business requirements change, or only when routing requirements change?

Auth middleware that enforces "this token must exist" is routing logic. It doesn't care about what the service does. It passes the test.

Auth middleware that enforces "this token must have the `premium` scope to call this endpoint" is starting to drift. The scope policy lives in the product, not the network layer.

Middleware that enriches a request with a user's subscription tier before forwarding it to a service is fully in product territory. That's aggregation logic, and it belongs in a service.

## Where aggregation actually lives

When clients need data from multiple services in one request, the instinct is to put the fan-out in the gateway. It's a reasonable instinct: the gateway already sees all the traffic, and it avoids an extra round trip from the client.

The problem is testing and ownership. The gateway is now coupled to the response shapes of every service it aggregates. When Service A changes a field name, the gateway breaks. Who owns the fix? Gateway team? Service A team? Both?

The better answer is a Backend for Frontend (BFF) service: a thin, purpose-built service that aggregates exactly what one client surface needs. It has clear ownership, is independently deployable, and keeps the gateway out of the business logic business.

## What a gateway should own

After fighting this a few times, my working list is short:

- TLS termination
- Request routing and load balancing
- Authentication token validation (not authorization policy)
- Rate limiting by identity or IP
- Request/response logging
- Circuit breaking at the network level

Everything else is a smell. Not always wrong, but worth naming clearly when you do it.

## The cost of getting it right

The tradeoff is real. Moving logic out of the gateway means more services, more deployment units, more things to operate. For a small team, the BFF pattern can feel like overkill.

The way I think about it: a gateway that's hard to change is a gateway that will slow down every engineer who touches a service behind it. That cost compounds. The BFF might feel expensive today, but the accidental monolith is always more expensive tomorrow.
