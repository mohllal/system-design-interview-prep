# Microservices Architecture

Microservices architecture breaks down applications into small, independent services that communicate over well-defined APIs.

Each service is owned by a small team and can be developed, deployed, and scaled independently.

## Core Concept

Rather than building applications as single deployable units (monoliths), microservices architecture structures applications as collections of loosely coupled services.

Each service implements specific business capabilities and can be developed using different technologies, programming languages, and data storage systems.

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web App]
        MOBILE[Mobile App]
        API_CLIENT[API Client]
    end

    subgraph "Gateway Layer"
        GATEWAY[API Gateway]
    end

    subgraph "Service Layer"
        USER_SVC[User Service]
        ORDER_SVC[Order Service]
        PAYMENT_SVC[Payment Service]
        INVENTORY_SVC[Inventory Service]
        NOTIFICATION_SVC[Notification Service]
    end

    subgraph "Data Layer"
        USER_DB[(User DB)]
        ORDER_DB[(Order DB)]
        PAYMENT_DB[(Payment DB)]
        INVENTORY_DB[(Inventory DB)]
        NOTIFICATION_DB[(Notification DB)]
    end

    subgraph "Message Layer"
        QUEUE[Message Queue]
    end

    WEB --> GATEWAY
    MOBILE --> GATEWAY
    API_CLIENT --> GATEWAY

    GATEWAY --> USER_SVC
    GATEWAY --> ORDER_SVC
    GATEWAY --> PAYMENT_SVC
    GATEWAY --> INVENTORY_SVC

    USER_SVC --> USER_DB
    ORDER_SVC --> ORDER_DB
    PAYMENT_SVC --> PAYMENT_DB
    INVENTORY_SVC --> INVENTORY_DB
    NOTIFICATION_SVC --> NOTIFICATION_DB

    ORDER_SVC --> QUEUE
    PAYMENT_SVC --> QUEUE
    INVENTORY_SVC --> QUEUE
    QUEUE --> NOTIFICATION_SVC
```

## Key Characteristics

### Business Capability Alignment

Each microservice is organized around a specific business capability rather than technical layers.

This approach, inspired by Conway's Law, ensures that services align with business functions and team structures.

```mermaid
graph LR
    subgraph "Monolithic Layers"
        UI[UI Layer]
        BL[Business Layer]
        DL[Data Layer]
    end

    subgraph "Business Capabilities"
        subgraph "User Management"
            U_UI[User UI]
            U_BL[User Logic]
            U_DB[(User DB)]
            U_UI --> U_BL --> U_DB
        end

        subgraph "Order Processing"
            O_UI[Order UI]
            O_BL[Order Logic]
            O_DB[(Order DB)]
            O_UI --> O_BL --> O_DB
        end

        subgraph "Payment Processing"
            P_UI[Payment UI]
            P_BL[Payment Logic]
            P_DB[(Payment DB)]
            P_UI --> P_BL --> P_DB
        end
    end
```

### Decentralized Data Management

One of the most important principles is that each service manages its own database.

This prevents tight coupling through shared data stores and allows services to choose the most appropriate data storage technology for their needs.

## Communication Patterns

### Synchronous vs Asynchronous Communication

```mermaid
sequenceDiagram
    participant Client
    participant Order Service
    participant Inventory Service
    participant Payment Service

    Note over Client,Payment Service: Synchronous Communication
    Client->>Order Service: Create Order
    Order Service->>Inventory Service: Check Availability
    Inventory Service-->>Order Service: Available
    Order Service->>Payment Service: Process Payment
    Payment Service-->>Order Service: Payment Success
    Order Service-->>Client: Order Created

    Note over Client,Payment Service: Asynchronous Communication
    Client->>Order Service: Create Order
    Order Service-->>Client: Order Accepted
    Order Service->>Message Queue: Order Created Event
    Message Queue->>Inventory Service: Order Created Event
    Inventory Service->>Message Queue: Inventory Reserved Event
    Message Queue->>Payment Service: Inventory Reserved Event
    Payment Service->>Message Queue: Payment Processed Event
```

### Service Orchestration vs Choreography

**Orchestration Pattern:**

In orchestration, a central service coordinates the entire workflow, making sequential calls to other services and handling the overall transaction logic.

```mermaid
graph TD
    ORCHESTRATOR[Order Orchestrator]

    ORCHESTRATOR -->|"[1] Reserve Items"| INVENTORY[Inventory Service]
    ORCHESTRATOR -->|"[2] Process Payment"| PAYMENT[Payment Service]
    ORCHESTRATOR -->|"[3] Create Shipment"| SHIPPING[Shipping Service]
    ORCHESTRATOR -->|"[4] Send Notification"| NOTIFICATION[Notification Service]

    INVENTORY -->|Success/Failure| ORCHESTRATOR
    PAYMENT -->|Success/Failure| ORCHESTRATOR
    SHIPPING -->|Success/Failure| ORCHESTRATOR
```

**Choreography Pattern:**

In choreography, services react to events independently without central coordination. Each service publishes events when it completes its local transaction, and other services react to these events to continue the saga.

```mermaid
graph LR
    ORDER[Order Service]
    INVENTORY[Inventory Service]
    PAYMENT[Payment Service]
    SHIPPING[Shipping Service]
    NOTIFICATION[Notification Service]

    subgraph "Event Bus"
        QUEUE[Message Queue]
    end

    ORDER -->|Order Created| QUEUE
    QUEUE -->|Order Created| INVENTORY

    INVENTORY -->|Inventory Reserved| QUEUE
    QUEUE -->|Inventory Reserved| PAYMENT

    PAYMENT -->|Payment Processed| QUEUE
    QUEUE -->|Payment Processed| SHIPPING
    QUEUE -->|Payment Processed| NOTIFICATION
```

## Benefits and Challenges

### Benefits

- **Strong Module Boundaries:** Microservices reinforce modular structure
- **Independent Deployment:** Small services are easier to deploy, and since they are autonomous, are less likely to cause system failures when they go wrong
- **Technology Diversity:** Microservices can be developed in multiple languages, development frameworks and data-storage technologies

### Challenges

- **Distributed Systems:** Distributed systems are harder to program, since remote calls are slow and are always at risk of failure
- **Eventual Consistency:** Maintaining strong consistency is extremely difficult for a distributed system, which means every service has to manage eventual consistency
- **Operational Complexity:** Services are redeployed regularly which comes with extra operational overhead requiring robust automation, monitoring, and orchestration tools

## Microservice Granularity

Microservice granularity specifies the scope of business functionality in a service operation.

By definition a coarse-grained service operation has broader scope than a fine-grained service, although the terms are relative.

The former typically requires increased design complexity but can reduce the number of calls required to complete a task.

The guiding principle is that a microservice should be small enough to encapsulate a single business capability but large enough to manage its responsibilities independently.

## Common Anti-Patterns

### Distributed Monolith

A distributed monolith occurs when services are deployed separately but remain tightly coupled through synchronous calls and shared databases.

This anti-pattern combines the worst aspects of both monolithic and microservice architectures,  the complexity of distributed systems without the benefits of loose coupling.

```mermaid
graph TD
    A[Service A] -->|Synchronous Call| B[Service B]
    B -->|Synchronous Call| C[Service C]
    C -->|Synchronous Call| D[Service D]
    D -->|Synchronous Call| A

    A -.->|Shared Database| SHARED[(Shared DB)]
    B -.-> SHARED
    C -.-> SHARED
    D -.-> SHARED
```

### Chatty Services

Chatty services occur when a service makes excessive calls to other services to complete a single operation.

This creates performance bottlenecks, increases latency, and makes the system fragile due to cascading failures.

```mermaid
sequenceDiagram
    participant Client
    participant Service A
    participant Service B
    participant Service C
    participant Service D

    Note over Client,Service D: Anti-Pattern: Too Many Calls
    Client->>Service A: Request
    Service A->>Service B: Get Data 1
    Service A->>Service C: Get Data 2
    Service A->>Service D: Get Data 3
    Service A->>Service B: Get Data 4
    Service A->>Service C: Get Data 5
    Service A-->>Client: Response

    Note over Client,Service D: Better: Batch Operations
    Client->>Service A: Request
    Service A->>Service B: Batch Request
    Service B-->>Service A: Batch Response
    Service A-->>Client: Response
```

## Reference Materials

- [Microservices](https://martinfowler.com/articles/microservices.html)
- [Microservice Patterns](https://microservices.io/patterns/index.html)
- [Microservice Trade-Offs](https://martinfowler.com/articles/microservice-trade-offs.html)
- [Monolith First](https://martinfowler.com/bliki/MonolithFirst.html)
