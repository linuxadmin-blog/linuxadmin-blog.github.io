---
title: "The Cap Theorem and Databases"
date: 2025-01-24
author: "Ugurcan Akkok"
tags: ["Distributed Systems", "Database"]
categories: ["Database"]
draft: false
---

When starting a new project or adding significant functionality to an existing one, you’ll likely face an important question: "Which database should I use?" With hundreds of database systems available—such as MySQL, Cassandra, Redis, Memcached, and PostgreSQL—choosing the right one can be overwhelming.

To make an informed decision, it's essential to understand how databases work and why they are designed the way they are. This is where the CAP theorem comes in. It provides a foundational framework for understanding the trade-offs between consistency, availability, and partition tolerance—key factors that influence how databases handle data in distributed systems.

# History on CAP Theorem

The CAP theorem was introduced by Eric Brewer in 1998 and later formally proven by Gilbert and Lynch in 2002. It states that in a distributed system, you can only guarantee two out of three key properties: Consistency, Availability, and Partition Tolerance. When a network partition occurs, a system must choose between maintaining consistency or availability, as achieving both simultaneously is impossible.

In 2010, the PACELC theorem expanded on CAP by introducing an additional trade-off: even when no network partition occurs, distributed systems must balance latency and consistency. This insight is particularly relevant to cloud applications, where low latency is often a priority, and network partitions are relatively rare.

# Consistency, Availability and Partitioning Tolerance

Distributed systems prioritize either consistency (CP systems) or availability (AP systems), depending on their design goals. Both approaches are widely used in real-world applications, each serving different needs.

A network partition occurs when communication between nodes is disrupted. This can result from high latency, packet loss, node failures, or software bugs that cause connectivity issues. During a partition, a distributed system must choose whether to remain consistent or available, as achieving both simultaneously is impossible.

- Consistency ensures that all reads and writes return the latest committed data. If a system cannot provide the most recent write, it must return an error. In weakly consistent systems, however, reads may return stale data if updates have not yet propagated to all nodes.

- Availability guarantees that the system responds to requests from non-failing nodes, even in the presence of network partitions. However, AP systems may serve outdated (stale) data because they prioritize responding to requests over maintaining strict consistency.

Many AP systems implement a strategy called eventual consistency to balance availability and consistency. Instead of blocking responses during a partition, they return the latest known data and synchronize updates over time. Once network issues are resolved, all nodes eventually converge to a consistent state. This ensures high availability at the cost of temporarily serving stale data.

# Databases and CAP

Databases are often designed to prioritize either Consistency and Partition Tolerance (CP) or Availability and Partition Tolerance (AP) based on their intended use cases.

- CP databases (e.g., MySQL, CockroachDB, PostgreSQL, and Google Spanner) prioritize strong consistency, ensuring that all nodes always reflect the latest committed data. However, during network partitions, these databases may sacrifice availability by rejecting requests until consistency can be guaranteed. These systems are commonly used in financial and banking applications, e-commerce platforms, and critical business applications, where data correctness is essential.

- AP databases (e.g., Cassandra, DynamoDB, and CouchDB) prioritize availability, ensuring that responses are always provided, even if some nodes have outdated data. These databases rely on eventual consistency, meaning updates propagate asynchronously across the system. They are well-suited for social media platforms, messaging applications, content delivery networks (CDNs), and caching services, where immediate availability is more important than strict consistency.

By understanding the trade-offs in CAP, developers can choose the right database based on their application's needs—whether it's strong consistency for accuracy or high availability for performance and scalability.


# Closing Thoughts

Understanding the CAP theorem is crucial for designing and selecting the right database for your application. By knowing the trade-offs between consistency, availability, and partition tolerance, developers can make informed decisions that align with their system's requirements. If you're interested in diving deeper, consider exploring real-world case studies of CAP trade-offs in databases like Google Spanner, Cassandra, or DynamoDB.