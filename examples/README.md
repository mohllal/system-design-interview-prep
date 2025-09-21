# System Design Examples

This section provides practice problems and designs for real-world systems. Each example includes key architectural decisions, trade-offs, and implementation considerations.

## About These Examples

These notes serve as a refresher and collection of key points for each system design. They are not comprehensive but focus on the most important aspects for interview preparation.

For deeper understanding, refer to the external references provided in each example.

## 📚 Completed Examples

| Topic                                                            | Key Concepts                                      |
|------------------------------------------------------------------|---------------------------------------------------|
| [URL Shortening System](./01-url-shortening-system.md)           | Hashing, Database Design, Caching, Load Balancing |
| [OCR File Processing System](./02-ocr-file-processing-system.md) | OCR, Message Queue, Auto-scaling, Storage         |

## 🍀 Future Examples

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

## 💡 Tips

- **Clarify Requirements**: Ask about scale, features, and constraints
- **Start Simple**: Begin with a basic design and iterate
- **Use Back-of-the-Envelope**: Estimate capacity and resource requirements
- **Discuss Trade-offs**: Explain the pros and cons of your design choices
