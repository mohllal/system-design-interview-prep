---
title: "HTTP versions evolution"
concepts:
  - http-1.1
  - http-2
  - http-3
  - multiplexing
  - header-compression
  - head-of-line-blocking
  - quic
  - connection-migration
related:
  - fundamentals/02-network-protocols.md
  - fundamentals/03-latency-and-throughput.md
  - fundamentals/05-rest-api.md
  - fundamentals/11-caching.md
  - fundamentals/34-cdn.md
---

# HTTP versions evolution

HTTP's request-response semantics have barely changed since 1996: the same methods, status codes, and headers work across every version. What has changed is the transport underneath, and every change has had the same goal — spend fewer round trips, and stop letting one slow response block the others.

The [round-trip budget](02-network-protocols.md) is the thing to keep in mind while reading this note. Each version attacks a different part of it.

## Version timeline

```mermaid
timeline
    title HTTP evolution
    1991 : HTTP/0.9 - Simple document retrieval
    1996 : HTTP/1.0 - Headers, status codes, methods
    1997 : HTTP/1.1 - Persistent connections, Host header, chunked transfer
    2015 : HTTP/2 - Multiplexing, header compression, binary framing
    2022 : HTTP/3 - HTTP over QUIC (UDP), faster recovery, connection migration
```

## Version comparison

| Aspect                | HTTP/1.1                                                             | HTTP/2                                           | HTTP/3                                     |
| --------------------- | -------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------ |
| Transport             | TCP                                                                  | TCP                                              | QUIC over UDP                              |
| Wire format           | Plain text                                                           | Binary frames                                    | Binary frames                              |
| Concurrency           | One request at a time per connection; browsers open 6 or so per host | Many multiplexed streams on one connection       | Many independent streams on one connection |
| Header overhead       | Full headers on every request                                        | Compressed with HPACK                            | Compressed with QPACK                      |
| Head-of-line blocking | At the application layer                                             | Removed at the application layer, remains at TCP | Removed at both layers                     |
| Encryption            | Optional                                                             | Optional in the spec, required by browsers       | Always on (TLS 1.3 built into QUIC)        |
| Setup round trips     | TCP + TLS (2 with TLS 1.3)                                           | TCP + TLS (2 with TLS 1.3)                       | 1, or 0 on resumption                      |

## HTTP/0.9

What it introduced:

- A single method, `GET`
- A response body only: no headers, no status codes
- One request per TCP connection

Why it matters: it established the [client-server](01-client-server.md) request-response shape the web still uses. It is a historical footnote otherwise — with no headers there is no content negotiation, no caching metadata, and no way to signal errors.

## HTTP/1.0

What it introduced:

- Multiple methods (`GET`, `POST`, `HEAD`)
- Request and response headers
- Status codes (for example `200`, `404`)
- Media type handling via `Content-Type`

Why it matters: headers turned HTTP from a document-fetch protocol into a general-purpose application protocol. It was still slow in practice, because each request opened and closed its own TCP connection, paying the handshake cost every time.

## HTTP/1.1

What it introduced:

- Persistent connections (`keep-alive` by default)
- The `Host` header, enabling virtual hosting (many domains on one IP)
- Chunked transfer encoding, for responses whose length is not known upfront
- Conditional requests and better caching (`ETag`, `If-Modified-Since`)
- Range requests for partial content

Why it matters: it was the default web protocol for nearly two decades and remains a valid fallback everywhere. Its limitation is that a connection carries one request at a time, so concurrency has to come from opening more connections.

### Persistent connections

Client and server reuse one TCP connection for many requests instead of reconnecting each time. This removes the TCP and TLS handshakes from every request but the first, which on a 50 ms link saves roughly 100 ms per request.

### Chunked transfer encoding

The server streams a response in pieces when the total size is not known when the response starts — for example, rendering report rows while the backend is still computing them. The user sees the first bytes sooner, which improves perceived latency even though total time is unchanged.

### Head-of-line blocking

HTTP/1.1 allows pipelining — sending several requests without waiting for each response — but responses must come back in request order, so one slow response stalls everything queued behind it. This is **application-layer head-of-line (HOL) blocking**, and it is why pipelining is effectively unused: browsers instead open around six connections per host and dispatch requests across them.

That workaround shaped a generation of front-end practice: domain sharding to get more parallel connections, plus sprite sheets and file concatenation to reduce the number of requests. HTTP/2 made all of it unnecessary, and domain sharding actively harmful.

## HTTP/2

What it introduced:

- Binary framing instead of plain text
- Multiplexing many streams over a single TCP connection
- Header compression (HPACK)
- Stream prioritization and server push (push saw little adoption and has since been removed from browsers)

Why it matters: it removes application-layer HOL blocking and most per-request overhead, which is a large win for pages with many small assets. It does not help a single large download, and it inherits TCP's own HOL blocking.

### Binary framing

Messages are encoded as compact binary frames rather than text lines. Framing is what makes everything else possible: frames carry a stream identifier, so frames from different requests can be interleaved on one connection and reassembled at the other end. It is also faster and less ambiguous to parse than text.

### Multiplexing

Many request-response streams share one connection concurrently and their frames interleave, so a slow response no longer blocks the others at the application layer. Unlike pipelining, responses can complete in any order.

The practical consequence is that request count stops being the thing to optimize. One connection to one host, with 100 requests in flight, is now the fast path — which is exactly why domain sharding became counterproductive.

### Header compression with HPACK

Requests to the same host repeat nearly identical headers: `Cookie`, `User-Agent`, `Accept`, auth tokens. HPACK keeps a shared table of previously sent headers on both ends, so repeats are sent as small index references. For chatty API traffic where headers can outweigh the body, this is a significant bandwidth saving.

### TCP head-of-line blocking

HTTP/2 removed HOL blocking at the application layer but not at the transport layer. All streams ride one TCP connection, and TCP delivers a single ordered byte stream: if one packet is lost, the kernel holds back every byte behind it — including bytes belonging to unrelated streams — until the retransmission arrives.

On a clean network this never shows up. On a lossy mobile network it produces exactly the tail latency spikes that [percentiles](03-latency-and-throughput.md) expose, and it is the specific problem HTTP/3 was designed to solve.

## HTTP/3

What it introduced:

- HTTP semantics over QUIC, a transport built on UDP
- Independent streams, so loss on one does not stall the others
- TLS 1.3 integrated into the transport handshake
- Connection migration across network changes
- QPACK header compression, an HPACK variant that tolerates out-of-order stream delivery

Why it matters: it improves p99 latency on lossy and mobile networks, where HTTP/2's TCP HOL blocking hurts most. Median latency on a good connection changes little, so the honest framing is that HTTP/3 fixes the tail, not the average.

### QUIC and stream independence

QUIC implements reliability, ordering, and congestion control in user space on top of UDP, and it does so per stream rather than per connection. A lost packet stalls only the stream whose data it carried; unrelated streams keep being delivered. This is the same multiplexing idea as HTTP/2, finally applied at the layer where the blocking actually happened.

Running in user space also means QUIC can be updated with the application rather than waiting on kernel and middlebox upgrades — the practical reason TCP itself could not be fixed this way.

### Fewer setup round trips

QUIC merges the transport and TLS handshakes into one exchange, so a new connection is established in one round trip instead of the two that TCP plus TLS 1.3 require, and a resumed connection can send data in zero round trips. Referring back to the [round-trip budget](02-network-protocols.md), that removes one of the four round trips from a cold page load and both of them from a warm one.

The 0-RTT path has a caveat worth naming: early data is replayable, so it must only carry idempotent requests.

### Connection migration

A QUIC connection is identified by a connection ID rather than the source IP and port, so it survives a client changing networks (Wi-Fi to cellular, or a NAT rebinding). With TCP the connection breaks and everything has to be re-established. This matters most for mobile apps and long-lived streams.

### Operational cost

HTTP/3 is not free to adopt: UDP is blocked or rate-limited on some corporate networks, load balancers and firewalls need explicit QUIC support, encrypted transport headers reduce what network monitoring can see, and CPU cost per byte is higher than kernel TCP.

Clients handle this with `Alt-Svc`: the server advertises HTTP/3 availability over an HTTP/2 response, the client tries QUIC, and falls back to TCP if it fails.

## How a version is chosen

Nothing in HTTP negotiates the version over multiple round trips. HTTP/1.1 and HTTP/2 are selected during the TLS handshake via ALPN (see [network protocols](02-network-protocols.md)), so the choice costs nothing extra. HTTP/3 is discovered separately: the client learns about it from an `Alt-Svc` header or an HTTPS DNS record, then uses it on subsequent connections.

In practice a system runs all three at once. The CDN or edge proxy terminates whatever the client supports and usually speaks a single simpler protocol to the origin, so the origin rarely needs to care.

## Interview talking points

- HTTP/1.1 to HTTP/2 is an efficiency win: multiplexing and header compression remove per-request overhead.
- HTTP/2 to HTTP/3 is a tail-latency and mobility win: QUIC removes transport HOL blocking and saves a setup round trip.
- Application semantics are identical across versions, so an upgrade is a transport decision, not an API change.
- Say which percentile improves. HTTP/3's benefit is concentrated in p99 on lossy networks, not in the median.
- Version support is usually terminated at the CDN or edge, so the origin often stays on the simplest option.

## Reference materials

- [RFC 1945 - HTTP/1.0](https://www.rfc-editor.org/rfc/rfc1945)
- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9112 - HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112)
- [RFC 9113 - HTTP/2](https://www.rfc-editor.org/rfc/rfc9113)
- [RFC 9114 - HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000)
- [RFC 9204 - QPACK: Field Compression for HTTP/3](https://www.rfc-editor.org/rfc/rfc9204)
