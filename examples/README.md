# System Design Examples

This section provides practice problems and designs for real-world systems. Each example includes key architectural decisions, trade-offs, and implementation considerations.

## 📖 Contents

| System Design Example                                            | Key Concepts                                      |
|------------------------------------------------------------------|---------------------------------------------------|
| [URL Shortening System](./01-url-shortening-system.md)           | Hashing, Database design, Caching, Load balancing |
| [OCR File Processing System](./02-ocr-file-processing-system.md) | OCR, Message queue, Auto-scaling, Storage         |

## 🍀 Future Examples (To be completed)

The following system designs are intended for future:

- **Design Uber**
  - Key concepts: Geospatial indexes, posteriori probability, Viterbi algorithm, Hidden Markov Model
  - References: [Designing Uber](https://highscalability.com/designing-uber/)

- **Design YouTube**
  - Key concepts: CDN, video encoding, content distribution, recommendation algorithms
  - References: [Design YouTube](https://bytebytego.com/courses/system-design-interview/design-youtube)

- **Design Distributed Rate Limiter**
  - Key concepts: Sliding window, token bucket, distributed coordination
  - References: [Sliding Window Rate Limiter](https://arpitbhayani.me/blogs/sliding-window-ratelimiter)

- **Design Distributed Counter**
  - Key concepts: Sharding, eventual consistency, conflict resolution
  - References: [Distributed Counter System Design](https://systemdesign.one/distributed-counter-system-design/)

- **Design Notification Service**
  - Key concepts: Message queues, push notifications, delivery guarantees
  - References: [Scalable Notification Service](https://blog.algomaster.io/p/design-a-scalable-notification-service)

## 📝 Thought Process

- **Clarify Requirements**: Ask about scale, features, and constraints
- **Use Back-of-the-Envelope**: Estimate capacity and resource requirements
- **Start Simple**: Begin with a basic design and iterate
- **Discuss Trade-offs**: Explain the pros and cons of your design choices

---

💡 **Tip**: Each system example may have a "Reference Materials" section with curated external resources for deeper exploration.

- <https://www.youtube.com/watch?v=jPKTo1iGQiE> - Design Youtube
- <https://www.youtube.com/watch?v=IUrQ5_g3XKs> - Design YouTube
- <https://www.youtube.com/watch?v=bUHFg8CZFws> - System Design Interview – Step By Step Guide
- <https://www.youtube.com/watch?v=XbkjEX-jgj0> - Top K Leaderboard
- <https://www.youtube.com/watch?v=olfaBgJrUBI> - Payment System
- <https://www.youtube.com/watch?v=rT4sS4l51PY> - Amazon Payment Gateway
- <https://www.youtube.com/watch?v=7MXV7RfNtv0> - Global Payment Processing
- <https://www.youtube.com/watch?v=m6DtqSb1BDM> - Build a robust Payments service using Idempotency Keys
- <https://www.youtube.com/watch?v=iUU4O1sWtJA> - Design Bitly
- <https://www.youtube.com/watch?v=qSJAvd5Mgio> - Design a URL Shortener (Bitly)
- <https://medium.com/algomaster-io/system-design-was-hard-until-i-learned-these-30-concepts-78042ff99cae> - System Design was HARD until I Learned these 30 Concepts
- <https://itnext.io/scaling-distributed-counters-designing-a-view-count-system-for-100k-rps-0567f6804900> - Scaling Distributed Counters: Designing a View Count System for 100K+ RPS
