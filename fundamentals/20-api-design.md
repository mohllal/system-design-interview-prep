# API Design

API (Application Programming Interface) design is the process of defining how different software components communicate with each other through well-structured interfaces.

APIs serve as contracts between different parts of a system or between different systems entirely.

## REST API Design

Representational State Transfer (REST) is the most common architectural style for web APIs.

The four principles of REST are:

- **Uniform Interface**: Each resource is identified by a URL, and the four HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) are used to manipulate the resource. This makes the interface consistent and predictable
- **Stateless**: Each request contains all necessary information, and the server does not store any state between requests. This simplifies server design, improves scalability, and reliability, as any server can handle any request
- **Cacheable**: Resources should be explicitly or implicitly marked as cacheable or non-cacheable. The server can cache the response for a given request, and the client can cache the response for a given request
- **Client-Server**: The client and server are separate entities with distinct responsibilities. This separation allows for independent development and evolution of both components

### RESTful URL Design

RESTful URL design follows predictable patterns that make APIs intuitive to use. URLs should represent resources (nouns) rather than actions (verbs), and HTTP methods should indicate the operation being performed.

**Collection vs Individual Resource Patterns:**

- Collections use plural nouns: `/users`, `/posts`, `/orders`
- Individual resources append an identifier: `/users/123`, `/posts/456`
- Sub-resources follow hierarchical patterns: `/users/123/posts`, `/orders/456/items`

**What Makes URLs RESTful:**

1. **Resource-oriented**: URLs represent things (users, posts) not actions
2. **Hierarchical**: Related resources follow logical nesting patterns
3. **Consistent**: Same patterns work across different resource types
4. **Predictable**: Developers can guess URLs based on established patterns

**Anti-patterns to Avoid:**

- Action-based URLs: `/getUser/123`, `/createPost`, `/deleteOrder/456`
- Inconsistent naming: `/user/123` vs `/users/456/post` vs `/posts`
- Deep nesting: `/users/123/posts/456/comments/789/replies/012`
- Query parameters for resource identification: `/api?action=getUser&id=123`

## API Versioning Strategies

APIs evolve over time, and versioning ensures backward compatibility while allowing changes to happen in a controlled manner.

### Versioning Approaches

**1. URL Path Versioning**

- Format: `/v1/users`, `/v2/users`
- Pros: Clear and visible, easy to implement, cacheable
- Cons: URL proliferation, resource duplication

**2. Header Versioning**

- Format: `X-API-Version: v2`
- Pros: Clean URLs, flexible, multiple versions per request
- Cons: Less visible, caching complexity

**3. Query Parameter Versioning**

- Format: `/users?version=2`
- Pros: Simple implementation, visible in URLs
- Cons: Optional parameter issues, URL pollution

**4. Content Type Versioning**

- Format: `Accept: application/vnd.api+json;version=2` (use `Vary: Accept` header to indicate the version is part of the request cache key)
- Pros: Standards compliant, fine-grained control
- Cons: Tool compatibility issues, not widely supported, can be abused for caching

### Semantic Versioning

**API Version Structure:**

- **Major version** (v1, v2): Breaking changes that require client updates
- **Minor version** (v1.1, v2.1): New features that maintain backward compatibility  
- **Patch version** (v1.1.1, v2.1.3): Bug fixes and minor improvements

**Version Compatibility Rules:**

- Same major version: Backward compatible within minor/patch updates
- Different major version: Breaking changes, no compatibility guarantee
- Clients specify minimum required version, receive compatible or newer

### Calendar Versioning

Calendar versioning (CalVer) uses dates to version APIs, making it easy to understand when a version was released and how old it is.

**Format:**

- `YYYY.MM.DD` - Year, month, and day
- `YYYY.MM` - Year and month (for monthly releases)
- `YYYY.0M` - Year and zero-padded month (e.g., `2025.01`)

**Examples:**

- `2025.11.21` - Released on November 21, 2025
- `2025.03` - Released in March 2025
- `/v2025.11.21/users` - API endpoint using calendar versioning

**When to Use:**

- APIs with regular, time-based release cycles
- When version age is more important than feature compatibility
- Internal APIs or services where date-based tracking is preferred

**Pros:**

- Intuitive: Version age is immediately clear
- Predictable: Follows calendar schedule
- No version number inflation: Dates naturally increment
- Useful for time-sensitive features or deprecations

**Cons:**

- No semantic meaning: Version doesn't indicate breaking changes
- Less common than semantic versioning, may confuse developers
- Multiple releases per day require additional identifiers

### Hash Versioning

Hash versioning combines a date prefix with a source control commit hash, providing both temporal context and exact code identification.

**Format:**

- Date component: `YYYY.MM` or `YYYY.MM.DD` (year, zero-padded month, optional zero-padded day)
- Hash component: 10+ characters from the commit hash (typically `git rev-parse --short HEAD`)
- Combined: `YYYY.MM.<hash>` or `YYYY.MM.DD.<hash>`

**Examples:**

- `2020.01.67092445a1abc` - January 2020 release with commit hash `67092445a1abc`
- `2019.07.21.3731a8be0f1a8` - July 21, 2019 release with commit hash `3731a8be0f1a8`
- `/v2025.11.21.a1b2c3d4e5f6/users` - API endpoint using hash versioning

**When to Use:**

- APIs deployed directly from source control commits
- Continuous deployment environments
- Internal services requiring precise version tracking
- Debugging scenarios where commit hash correlation is valuable

**Pros:**

- Exact version identification: Hash maps directly to source code
- Traceability: Easy to find the exact commit for any version
- Prevents ambiguity: Hash uniquely identifies the codebase state
- Useful for debugging: Can correlate API behavior with specific commits

**Cons:**

- Not human-readable: Hash component is opaque
- No semantic meaning: Doesn't indicate breaking changes or features
- Requires tooling: Need access to source control to understand version
- Longer version strings: Can make URLs and headers unwieldy

### Deprecation Strategy

**Deprecation Process:**

1. **Track Deprecations:** Maintain a registry of deprecated endpoints with metadata including deprecation date, removal timeline, and alternative endpoints

2. **Add Warning Headers:** Include standardized HTTP headers in responses:
   - `Deprecation: true`: Indicates the endpoint is deprecated
   - `Sunset: v3.0.0`: Version when endpoint will be removed
   - `Link: </v2/users/{id}>; rel="successor-version"`: Points to replacement endpoint
   - `Warning: 299 - "This endpoint is deprecated and will be removed in v3.0.0"`: Human-readable warning (HTTP status code 299 is reserved for deprecation warnings)

**Best Practices:**

- Provide at least one major version notice before removal
- Log deprecation usage for monitoring adoption
- Gradually reduce functionality rather than immediate removal

## API Documentation

OpenAPI (formerly Swagger) provides a standardized way to document REST APIs, enabling automatic documentation generation, client SDK creation, and contract validation

## Contract-First Development

Defines the API specification before implementation, ensuring consistency between documentation and actual behavior

1. **Define API Contract**: Create OpenAPI specification with all endpoints, models, and responses
2. **Generate Documentation**: Use tools to create human-readable API docs from specification (e.g. Swagger UI, ReDoc)
3. **Create Mock Server**: Generate mock responses for testing and development (e.g. Mockoon, MockServer)
4. **Implement Validation**: Add request/response validation against contract (e.g. AJV, Zod)
5. **Generate Client SDKs**: Create client libraries from specification
6. **Continuous Testing**: Validate implementation matches contract in CI/CD pipeline

## Reference Materials

- [RESTful API Design Best Practices](https://restfulapi.net/)
- [Semantic Versioning](https://semver.org/)
- [Calendar Versioning](https://calver.org/)
- [Hash Versioning](https://miniscruff.github.io/hashver/)
- [API Versioning - A Deep Dive](https://newsletter.systemdesign.one/p/api-versioning)
