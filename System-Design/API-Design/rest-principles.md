# REST Principles

**RE**presentational **S**tate **T**ransfer — an architectural style for distributed systems defined by Roy Fielding
in his 2000 dissertation. REST is the dominant paradigm for web APIs and one of the most asked topics on backend
interviews.

## REST Constraints (Fielding's 6 Principles)

A truly RESTful API satisfies **all six** architectural constraints:

| # | Constraint                | Description                                                          |
|---|---------------------------|----------------------------------------------------------------------|
| 1 | **Client–Server**         | Separation of concerns: client handles UI, server handles data/logic |
| 2 | **Stateless**             | Each request contains **all** information needed — no server-side session |
| 3 | **Cacheable**             | Responses must declare if they are cacheable to improve performance  |
| 4 | **Uniform Interface**     | Standardized way to interact with resources (see below)             |
| 5 | **Layered System**        | Client doesn't know if it talks to the end server or intermediary (proxy, gateway, CDN) |
| 6 | **Code on Demand** *(optional)* | Server can send executable code to the client (e.g., JavaScript) |

### Statelessness — Why It Matters

```
❌ Stateful:  Server stores session → "Add to cart" → server remembers cart
              Problem: sticky sessions, hard to scale, server crash loses state

✅ Stateless: Every request carries all context (auth token, cart ID, etc.)
              Server treats each request independently → easy to scale horizontally
```

The server does NOT store client state between requests. Authentication is handled via tokens (JWT, OAuth2) sent with
every request, not via server-side sessions.

## Uniform Interface — The Core of REST

The uniform interface has four sub-constraints:

### 1. Resource Identification (URIs)

Everything is a **resource**, identified by a **URI**:

```
/users              → collection of users
/users/42           → specific user
/users/42/orders    → orders belonging to user 42
/users/42/orders/7  → specific order of user 42
```

**URI Design Best Practices:**

| Rule                        | ✅ Good                      | ❌ Bad                        |
|-----------------------------|------------------------------|-------------------------------|
| Use nouns, not verbs        | `GET /users`                 | `GET /getUsers`               |
| Plurals for collections     | `/users`, `/orders`          | `/user`, `/order`             |
| Hierarchical nesting        | `/users/42/orders`           | `/getUserOrders?userId=42`    |
| Lowercase, hyphens          | `/user-profiles`             | `/UserProfiles`, `/user_profiles` |
| No trailing slash           | `/users`                     | `/users/`                     |
| No file extensions          | `/users/42`                  | `/users/42.json`              |
| Max 2-3 levels of nesting   | `/users/42/orders`           | `/users/42/orders/7/items/3/reviews` |

### 2. Resource Manipulation Through Representations

Clients interact with **representations** of resources (JSON, XML), not the resources themselves. The server sends a
representation; the client sends a representation to modify the resource.

```http
GET /users/42
Accept: application/json

→ Response:
{
    "id": 42,
    "name": "Alice",
    "email": "alice@example.com"
}
```

The JSON is a **representation** of the user resource, not the user itself (which lives in the database).

### 3. Self-Descriptive Messages

Each message contains enough information to understand how to process it:

```http
HTTP/1.1 200 OK
Content-Type: application/json       ← how to parse the body
Cache-Control: max-age=3600          ← how long to cache
ETag: "abc123"                       ← version for conditional requests
```

### 4. HATEOAS (Hypermedia as the Engine of Application State)

Responses include **links** to related actions/resources — the client discovers the API by following links, not by
hardcoding URLs:

```json
{
    "id": 42,
    "name": "Alice",
    "status": "active",
    "_links": {
        "self":    { "href": "/users/42" },
        "orders":  { "href": "/users/42/orders" },
        "deactivate": { "href": "/users/42/deactivate", "method": "POST" }
    }
}
```

**In practice:** Most APIs call themselves "RESTful" but don't implement HATEOAS. This is called **REST Level 2** in
the Richardson Maturity Model. True REST (Level 3) includes HATEOAS.

### Richardson Maturity Model

| Level | Description                      | Example                                      |
|-------|----------------------------------|----------------------------------------------|
| 0     | One URI, one verb (RPC over HTTP)| `POST /api` with action in body               |
| 1     | Multiple URIs (resources)        | `/users`, `/orders` but only `POST`           |
| 2     | HTTP verbs + status codes        | `GET /users`, `POST /users`, `204 No Content` |
| 3     | HATEOAS (hypermedia links)       | Responses contain links to related resources  |

Most production APIs aim for **Level 2**.

## HTTP Methods

| Method    | CRUD    | Idempotent | Safe | Request Body | Typical Use                          |
|-----------|---------|:---:|:---:|:---:|----------------------------------------------|
| `GET`     | Read    | ✅  | ✅  | ❌  | Retrieve resource(s)                          |
| `POST`    | Create  | ❌  | ❌  | ✅  | Create a new resource                         |
| `PUT`     | Replace | ✅  | ❌  | ✅  | Replace entire resource                       |
| `PATCH`   | Update  | ❌* | ❌  | ✅  | Partial update of a resource                  |
| `DELETE`  | Delete  | ✅  | ❌  | ❌  | Delete a resource                             |
| `HEAD`    | —       | ✅  | ✅  | ❌  | Same as GET but no body (check existence)     |
| `OPTIONS` | —       | ✅  | ✅  | ❌  | Discover allowed methods (CORS preflight)     |

*`PATCH` can be idempotent depending on implementation (`{ "name": "Alice" }` is idempotent; `{ "op": "increment" }`
is not).

### Idempotent vs Safe

- **Safe** — no side effects. The request doesn't modify the server state. `GET`, `HEAD`, `OPTIONS`.
- **Idempotent** — making the same request N times has the same effect as making it once. `GET`, `PUT`, `DELETE`,
  `HEAD`, `OPTIONS`. `POST` is NOT idempotent (each call may create a new resource).

```
GET  /users/42       → always returns user 42 (safe + idempotent)
PUT  /users/42       → replaces user 42 (not safe, but idempotent — same result each time)
POST /users          → creates a new user (not safe, NOT idempotent — creates duplicate)
DELETE /users/42     → deletes user 42 (not safe, but idempotent — already deleted = same result)
```

### `PUT` vs `PATCH`

```http
PUT /users/42
{
    "name": "Alice",
    "email": "alice@example.com",
    "role": "admin"
}
→ Replaces the ENTIRE resource. Missing fields are set to null/default.

PATCH /users/42
{
    "role": "admin"
}
→ Updates ONLY the specified fields. Other fields remain unchanged.
```

**Rule:** Use `PUT` when the client sends the complete resource. Use `PATCH` for partial updates (more common in
practice).

## HTTP Status Codes

### Success (2xx)

| Code | Name         | When to use                                               |
|------|--------------|-----------------------------------------------------------|
| 200  | OK           | Successful GET, PUT, PATCH with response body             |
| 201  | Created      | Successful POST — new resource created                    |
| 204  | No Content   | Successful DELETE or PUT/PATCH with no response body      |

### Client Errors (4xx)

| Code | Name                  | When to use                                          |
|------|-----------------------|------------------------------------------------------|
| 400  | Bad Request           | Malformed syntax, invalid JSON, validation errors    |
| 401  | Unauthorized          | Not authenticated — missing or invalid token         |
| 403  | Forbidden             | Authenticated but not authorized for this resource   |
| 404  | Not Found             | Resource doesn't exist                               |
| 405  | Method Not Allowed    | HTTP method not supported for this endpoint          |
| 409  | Conflict              | Resource conflict (duplicate, state conflict)        |
| 422  | Unprocessable Entity  | Valid syntax but semantic errors (validation failure) |
| 429  | Too Many Requests     | Rate limit exceeded                                  |

### Server Errors (5xx)

| Code | Name                  | When to use                                          |
|------|-----------------------|------------------------------------------------------|
| 500  | Internal Server Error | Unexpected server error (catch-all)                  |
| 502  | Bad Gateway           | Upstream server returned invalid response            |
| 503  | Service Unavailable   | Server overloaded or in maintenance                  |
| 504  | Gateway Timeout       | Upstream server didn't respond in time               |

### 401 vs 403

- **401 Unauthorized** — "Who are you?" (identity unknown, need to authenticate).
- **403 Forbidden** — "I know who you are, but you can't do this." (authenticated but insufficient permissions).

## Content Negotiation

Client and server agree on the representation format via headers:

```http
// Client says: "I want JSON"
GET /users/42
Accept: application/json

// Client says: "I'm sending JSON"
POST /users
Content-Type: application/json
```

## Pagination

### Offset-Based (Common)

```http
GET /users?page=2&size=20

→ Response:
{
    "content": [...],
    "page": 2,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
}
```

**Pros:** Simple, random access to any page.
**Cons:** Inconsistent results if data changes between pages; `OFFSET` is slow on large tables.

### Cursor-Based (Better for Large/Real-time Data)

```http
GET /users?cursor=eyJpZCI6NDJ9&limit=20

→ Response:
{
    "data": [...],
    "nextCursor": "eyJpZCI6NjJ9",
    "hasMore": true
}
```

**Pros:** Consistent results, performant (index-based `WHERE id > cursor`).
**Cons:** No random access, only forward/backward.

## Filtering, Sorting, Field Selection

```http
# Filtering
GET /users?role=admin&status=active

# Sorting
GET /users?sort=name,asc&sort=createdAt,desc

# Field selection (sparse fieldsets) — reduce payload
GET /users?fields=id,name,email

# Search
GET /users?q=alice
```

## Versioning

| Strategy              | Example                               | Pros / Cons                           |
|-----------------------|---------------------------------------|---------------------------------------|
| **URI path**          | `/api/v1/users`                       | Simple, explicit. Breaks URI purity   |
| **Query parameter**   | `/api/users?version=1`                | Easy to default. Clutters params      |
| **Header**            | `Accept: application/vnd.api.v1+json` | Clean URIs. Hard to test in browser   |

**Most common in practice:** URI path (`/api/v1/...`). Simple and explicit.

## Error Response Format (RFC 9457 — Problem Details)

Standardized error response format:

```json
{
    "type": "https://api.example.com/errors/validation",
    "title": "Validation Error",
    "status": 422,
    "detail": "Email format is invalid",
    "instance": "/users",
    "errors": [
        { "field": "email", "message": "must be a valid email address" },
        { "field": "name", "message": "must not be blank" }
    ]
}
```

## REST vs GraphQL vs gRPC

| Aspect          | REST                    | GraphQL                  | gRPC                       |
|-----------------|-------------------------|--------------------------|----------------------------|
| Protocol        | HTTP/1.1 or HTTP/2      | HTTP (single endpoint)   | HTTP/2                     |
| Data format     | JSON (typically)        | JSON                     | Protocol Buffers (binary)  |
| Schema          | OpenAPI / Swagger       | SDL (strongly typed)     | .proto (strongly typed)    |
| Over-fetching   | Common                  | ❌ Client picks fields   | ❌ Schema-defined          |
| Under-fetching  | Common (multiple calls) | ❌ Single query           | ❌ Service-level calls     |
| Caching         | HTTP caching built-in   | Harder (POST-based)      | No built-in HTTP caching   |
| Browser support | Native (fetch)          | Native (fetch)           | Requires grpc-web proxy    |
| Best for        | Public APIs, CRUD       | Complex/nested data, BFF | Internal service-to-service|

## Security Checklist

- ✅ Always use **HTTPS** (TLS) — never send tokens or data over plain HTTP.
- ✅ **Authenticate** with OAuth2 / JWT — not API keys in query params.
- ✅ **Authorize** at the resource level — don't rely on obscured URLs.
- ✅ **Rate limit** — protect against abuse and DDoS.
- ✅ **Validate input** — reject unexpected fields, enforce types and constraints.
- ✅ **CORS** — configure `Access-Control-Allow-Origin` restrictively.
- ✅ **Don't expose internals** — no stack traces, DB errors, or internal IDs in error responses.

## Common Interview Questions

1. **What is REST?** — An architectural style with six constraints: client-server, stateless, cacheable, uniform
   interface, layered system, code on demand (optional). Resources are identified by URIs and manipulated via HTTP
   methods.
2. **What does stateless mean?** — The server does not store client state between requests. Each request carries
   all necessary context (auth token, parameters). Enables horizontal scaling.
3. **PUT vs PATCH?** — `PUT` replaces the entire resource (must send all fields). `PATCH` updates only the specified
   fields. Both update, but `PUT` is idempotent by definition; `PATCH` may or may not be.
4. **What is idempotency? Which methods are idempotent?** — Same request made N times has the same effect as once.
   `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS` are idempotent. `POST` and `PATCH` are generally not.
5. **401 vs 403?** — 401: not authenticated (unknown identity). 403: authenticated but not authorized (insufficient
   permissions).
6. **How do you version a REST API?** — URI path (`/v1/`), query param (`?version=1`), or Accept header. URI path
   is most common. Avoid breaking changes — add fields, don't remove them.
7. **What is HATEOAS?** — Responses include links to related resources and actions. The client navigates the API
   by following links, not by hardcoding URLs. Level 3 of the Richardson Maturity Model.
8. **REST vs GraphQL?** — REST: multiple endpoints, HTTP caching, simpler. GraphQL: single endpoint, client picks
   exact fields needed (no over/under-fetching), better for complex nested data.

## Related

- [Spring REST API](../../Java/rest-api/spring-rest-api.md) — Spring MVC implementation
- [Microservices Architecture](../../Java/spring/cloud/microservices-architecture.md) — inter-service communication

## Resources

- Roy Fielding — [Architectural Styles and the Design of Network-Based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html)
