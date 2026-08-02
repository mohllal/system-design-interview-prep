# Network Protocols

Network protocols define how devices communicate across networks.

## TCP/IP Network Model

The TCP/IP model provides a layered approach to network communication, with each layer serving specific functions.

```mermaid
graph TD
    A[Application Layer<br/>HTTP, HTTPS, FTP, SMTP, DNS] --> B[Transport Layer<br/>TCP, UDP]
    B --> C[Internet Layer<br/>IP, ICMP, ARP]
    C --> D[Link Layer<br/>Ethernet, Wi-Fi, PPP]
```

### Layers

**1. Application Layer**

- Provides network services to applications
- Handles user interfaces and data formatting
- **Protocols**: HTTP/HTTPS, FTP, SMTP, DNS, SSH

**2. Transport Layer**

- Ensures reliable end-to-end communication
- Handles flow control, error detection, and recovery
- **Protocols**: TCP (reliable), UDP (fast)

**3. Internet Layer**

- Routes packets across networks
- Handles addressing and fragmentation
- **Protocols**: IP (IPv4/IPv6), ICMP, ARP

**4. Link Layer**

- Manages physical network connections
- Handles frame transmission and error detection
- **Protocols**: Ethernet, Wi-Fi, PPP

## Data Encapsulation Process

Each layer adds its own header (and sometimes trailer) to the data from the layer above.

```mermaid
graph TD
    A["Application Data<br/>(HTTP Request)"] --> B["TCP Segment<br/>[TCP Header | Data]"]
    B --> C["IP Packet<br/>[IP Header | TCP Segment]"]
    C --> D["Ethernet Frame<br/>[Ethernet Header | IP Packet | Trailer]"]
```

### Encapsulation Steps

**1. Application Layer**

- Data: HTTP request, email message, file content
- Format: Protocol-specific (JSON, HTML, binary)

**2. Transport Layer**

- **TCP Segments**: Include sequence numbers, acknowledgment numbers, window size
- **UDP Datagrams**: Include source/destination ports, length, checksum
- Adds: Port information, flow control, error detection

**3. Internet Layer**

- **IP Packets**: Include source/destination IP addresses, TTL, protocol type
- Adds: Routing information, fragmentation control, quality of service

**4. Link Layer**

- **Frames**: Include source/destination MAC addresses, frame type, CRC
- Adds: Physical addressing, error detection, frame boundaries

## What Happens When You Type `example.com` in Your Browser?

```mermaid
sequenceDiagram
    participant B as Browser
    participant DNS as DNS Server
    participant S as Web Server
    
    B->>DNS: 1. DNS Query (example.com)
    DNS->>B: 2. IP Address (192.0.2.146)
    
    Note over B,S: 3. TCP Connection (3-way handshake)
    B->>S: SYN
    S->>B: SYN-ACK
    B->>S: ACK
    
    Note over B,S: 4. TLS Handshake (if HTTPS)
    B->>S: ClientHello (cipher suites, TLS version, key share, SNI, ALPN)
    S->>B: ServerHello + Certificate + key agreement parameters

    Note over B,S: Shared secrets are derived and secure session keys are created
    S->>B: Finished (encrypted)
    B->>S: Finished (encrypted)
    
    Note over B,S: 5. HTTP Request/Response
    B->>S: HTTP Request
    S->>B: HTTP Response
    
    Note over B,S: 6. Connection Close
```

### Detailed Steps

**1. DNS Resolution**

- Browser checks cache (browser → OS → router → ISP)
- DNS query for A record (IPv4) or AAAA record (IPv6)

**2. TCP Connection Establishment**

- Three-way handshake (SYN → SYN-ACK → ACK)
- Establishes reliable communication channel

**3. TLS Handshake (HTTPS only)**

- Certificate verification and cipher negotiation
- Establishes encrypted communication (TLS 1.3 in modern deployments)

**4. HTTP Request**

- Includes headers (Host, User-Agent, Accept, etc.)
- May include cookies, authentication tokens

**5. Server Processing**

- Route handling, business logic execution
- Database queries, cache lookups

**6. HTTP Response**

- Status code, headers, body content
- Caching directives, content encoding

**7. Response Processing**

- Parse headers (redirects, caching, compression)
- Decompress content (gzip, brotli)
- Render content (HTML parsing, CSS, JavaScript)

**8. Connection Management**

- HTTP/1.1: Keep-alive or close
- HTTP/2: Multiplexed streams
- HTTP/3: QUIC protocol over UDP

## Reference Materials

- [RFC 9293 - Transmission Control Protocol (TCP)](https://www.rfc-editor.org/rfc/rfc9293)
- [RFC 768 - User Datagram Protocol (UDP)](https://www.rfc-editor.org/rfc/rfc768)
- [RFC 791 - Internet Protocol (IPv4)](https://www.rfc-editor.org/rfc/rfc791)
- [RFC 8200 - Internet Protocol, Version 6 (IPv6)](https://www.rfc-editor.org/rfc/rfc8200)
- [RFC 1034 - Domain Names: Concepts and Facilities](https://www.rfc-editor.org/rfc/rfc1034)
- [RFC 1035 - Domain Names: Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
- [RFC 8446 - The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9114 - HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000)
