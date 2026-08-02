# Client-Server Model

The client-server model is a computing architecture where clients (requesters) and servers (providers) communicate over a network to share resources and services.

**Core characteristics:**

- **Role separation**: Client handles user interaction; server handles data and business logic
- **Request-response interaction**: Clients initiate communication, servers respond
- **Independent scaling**: Clients and servers can scale and evolve separately

## Basic Building Blocks

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

## State Management

### Stateless Servers

Each request carries all required context (for example, auth token and parameters).

- ✅ Easier horizontal scaling behind a load balancer
- ✅ Simpler failover and replacement
- ❌ Repeated metadata in requests

### Stateful Servers

Server keeps session or workflow state in memory.

- ✅ Can simplify some interactions
- ❌ Harder to scale and fail over
- ❌ Often needs sticky sessions or shared session storage

## Interview Lens

When discussing a client-server design, clarify:

- Who is the client, and what are request patterns (RPS, burstiness)?
- Where state lives (client, server, shared store)?
- How failures are handled (timeouts, retries, failover)?
- How the server tier and data tier scale independently?
