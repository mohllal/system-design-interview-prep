---
title: "Client-server model"
concepts:
  - client-server-architecture
  - stateless-servers
  - stateful-servers
  - sticky-sessions
  - horizontal-scaling
  - load-balancing
related:
  - fundamentals/02-network-protocols.md
  - fundamentals/10-scalability.md
  - fundamentals/13-load-balancing.md
  - fundamentals/14-resilience.md
---

# Client-server model

The client-server model is a computing architecture where clients (requesters) and servers (providers) communicate over a network to share resources and services.

**Core characteristics:**

- **Role separation**: Client handles user interaction; server handles data and business logic
- **Request-response interaction**: Clients initiate communication, servers respond
- **Independent scaling**: Clients and servers can scale and evolve separately

## Basic building blocks

### Client

- Initiates requests
- Renders user experience
- Handles local state and retries

### Server

- Exposes APIs/services
- Executes business logic
- Reads/writes persistent data

### Network

- Connects clients and servers over IP
- Uses ports to route traffic to the correct process
- Adds latency, packet loss, and possible failures

## State management

### Stateless servers

Each request carries all required context (for example, auth token and parameters).

Pros:

- Easier horizontal scaling behind a load balancer
- Simpler failover and replacement

Cons:

- Repeated metadata in requests

### Stateful servers

Server keeps session or workflow state in memory.

Pros:

- Can simplify some interactions

Cons:

- Harder to scale and fail over
- Often needs sticky sessions or shared session storage

## Interview talking points

- Who is the client, and what are the request patterns (RPS, burstiness)?
- Where does state live (client, server, or shared store)?
- How are failures handled (timeouts, retries, failover)?
- How do the server tier and data tier scale independently?
