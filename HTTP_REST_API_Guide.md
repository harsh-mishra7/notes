# HTTP, REST APIs, and Web Communication

## Table of Contents
1. [REST API](#rest-api)
2. [HTTP and HTTPS Protocols](#http-and-https-protocols)
3. [HTTP Request Methods](#http-request-methods)
4. [HTTP Request and Response Structure](#http-request-and-response-structure)

---

## REST API

### What is REST?

REST stands for **Representational State Transfer**. It's an architectural style for designing networked applications, particularly web services. REST uses HTTP protocols to perform CRUD (Create, Read, Update, Delete) operations on resources.

### Key Principles of REST

1. **Client-Server Architecture**: Clear separation between client (consumer) and server (provider)
2. **Statelessness**: Each request contains all information needed; server doesn't store client context
3. **Uniform Interface**: Consistent way to communicate with resources
4. **Resource-Oriented**: Everything is a resource identified by URLs (URIs)
5. **Cacheable**: Responses can be cached to improve performance
6. **Layered System**: Client cannot directly tell if connected to end server

### REST vs SOAP

| Feature | REST | SOAP |
|---------|------|------|
| **Protocol** | HTTP (lightweight) | HTTP/SMTP/FTP (overhead) |
| **Message Format** | JSON, XML, plain text | XML only |
| **Performance** | Faster | Slower |
| **Ease of Use** | Simple | Complex |
| **Standards** | Flexible | Strict standards |

### RESTful API Example

```
GET /api/users/123          → Fetch user with ID 123
POST /api/users             → Create a new user
PUT /api/users/123          → Update user with ID 123
DELETE /api/users/123       → Delete user with ID 123
```

---

## HTTP and HTTPS Protocols

### HTTP (HyperText Transfer Protocol)

HTTP is an **application-layer protocol** used for transferring data over the web. It's based on a **request-response model**.

**Characteristics:**
- **Stateless**: Server doesn't maintain client state between requests
- **Plain Text**: Data transmitted in readable text format
- **Port**: Default port 80
- **Connection**: Can be persistent or non-persistent

### HTTPS (HyperText Transfer Protocol Secure)

HTTPS is the **secure version** of HTTP, encrypted using **SSL/TLS protocols**.

**Characteristics:**
- **Encrypted**: Data is encrypted before transmission
- **Port**: Default port 443
- **SSL/TLS Certificate**: Required for HTTPS communication
- **Authentication**: Verifies the identity of the server
- **Data Integrity**: Prevents tampering with data in transit

### HTTP vs HTTPS Comparison

| Aspect | HTTP | HTTPS |
|--------|------|-------|
| **Security** | No encryption | Encrypted with SSL/TLS |
| **Port** | 80 | 443 |
| **Speed** | Slightly faster | Slightly slower (due to encryption) |
| **Data Visibility** | Visible in transit | Hidden |
| **Certificate** | Not required | Required (CA signed) |
| **SEO** | Lower ranking | Better ranking by search engines |
| **Use Case** | Non-sensitive data | Sensitive data (passwords, payments) |

### Why HTTPS?

1. **Privacy**: Encrypts data so it can't be intercepted
2. **Authentication**: Verifies server identity via certificates
3. **Integrity**: Ensures data isn't modified in transit
4. **Compliance**: Required for PCI-DSS, HIPAA, GDPR compliance

---

## HTTP Request Methods

HTTP methods define the action to be performed on a resource. The primary methods are:

### 1. GET
- **Purpose**: Retrieve data from server
- **Body**: No request body
- **Idempotent**: Yes (multiple calls have same effect)
- **Cacheable**: Yes
- **Safe**: Yes (doesn't modify server state)

```
GET /api/users/123
```

### 2. POST
- **Purpose**: Create new resource
- **Body**: Yes, contains data to create
- **Idempotent**: No (multiple calls create multiple resources)
- **Cacheable**: No
- **Safe**: No (modifies server state)

```
POST /api/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

### 3. PUT
- **Purpose**: Update entire resource
- **Body**: Yes, contains replacement data
- **Idempotent**: Yes (multiple calls result in same state)
- **Cacheable**: No
- **Safe**: No (modifies server state)

```
PUT /api/users/123
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```

### 4. PATCH
- **Purpose**: Partially update resource
- **Body**: Yes, contains partial data
- **Idempotent**: Can be (depends on implementation)
- **Cacheable**: No
- **Safe**: No (modifies server state)

```
PATCH /api/users/123
Content-Type: application/json

{
  "email": "newemail@example.com"
}
```

### 5. DELETE
- **Purpose**: Remove resource
- **Body**: Typically no
- **Idempotent**: Yes (deleting again has same effect)
- **Cacheable**: No
- **Safe**: No (modifies server state)

```
DELETE /api/users/123
```

### 6. HEAD
- **Purpose**: Same as GET but returns only headers (no body)
- **Body**: No
- **Idempotent**: Yes
- **Cacheable**: Yes
- **Safe**: Yes

```
HEAD /api/users/123
```

### 7. OPTIONS
- **Purpose**: Describe communication options for the resource
- **Body**: No
- **Use**: CORS preflight requests
- **Safe**: Yes

```
OPTIONS /api/users
```

### Method Safety & Idempotency

| Method | Safe | Idempotent |
|--------|------|------------|
| GET | Yes | Yes |
| HEAD | Yes | Yes |
| OPTIONS | Yes | Yes |
| POST | No | No |
| PUT | No | Yes |
| PATCH | No | Maybe |
| DELETE | No | Yes |

---

## HTTP Request and Response Structure

### HTTP Request Structure

An HTTP request consists of:

#### 1. Request Line
```
METHOD URL HTTP_VERSION

Example:
GET /api/users/123 HTTP/1.1
```

Components:
- **METHOD**: HTTP method (GET, POST, etc.)
- **URL**: Resource path (can include query parameters)
- **HTTP_VERSION**: HTTP version (1.0, 1.1, 2.0, 3.0)

#### 2. Headers
```
Key: Value

Example:
Host: api.example.com
Content-Type: application/json
Authorization: Bearer token123
User-Agent: Mozilla/5.0
Accept: application/json
```

**Common Headers:**
- `Host`: Domain of the server
- `Content-Type`: Format of request body (application/json, text/html, etc.)
- `Content-Length`: Size of request body in bytes
- `Authorization`: Authentication credentials
- `User-Agent`: Client information
- `Accept`: Acceptable response formats
- `Referer`: URL of the page making the request
- `Cookie`: Session/tracking data
- `Cache-Control`: Caching instructions

#### 3. Blank Line
A blank line separates headers from body

#### 4. Body (Optional)
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}
```

Not required for GET, HEAD, DELETE requests. Required for POST, PUT, PATCH.

### Complete HTTP Request Example

```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 56
Authorization: Bearer eyJhbGc...
User-Agent: Mozilla/5.0
Accept: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

---

### HTTP Response Structure

An HTTP response consists of:

#### 1. Status Line
```
HTTP_VERSION STATUS_CODE STATUS_MESSAGE

Example:
HTTP/1.1 200 OK
```

Components:
- **HTTP_VERSION**: HTTP version
- **STATUS_CODE**: Numeric code (3 digits)
- **STATUS_MESSAGE**: Human-readable message

#### 2. Headers
```
Key: Value

Example:
Content-Type: application/json
Content-Length: 234
Server: nginx/1.14.0
Date: Wed, 17 Sep 2026 10:30:00 GMT
Cache-Control: public, max-age=3600
```

**Common Response Headers:**
- `Content-Type`: Format of response body
- `Content-Length`: Size of response body
- `Server`: Server software information
- `Date`: When response was generated
- `Set-Cookie`: Store cookies in client
- `Cache-Control`: How to cache response
- `Access-Control-Allow-Origin`: CORS permission
- `Location`: Redirect URL (3xx responses)

#### 3. Blank Line
Separates headers from body

#### 4. Body
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "createdAt": "2026-09-17T10:30:00Z"
}
```

### Complete HTTP Response Example

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 120
Server: nginx/1.14.0
Date: Wed, 17 Sep 2026 10:30:00 GMT
Cache-Control: public, max-age=3600

{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "createdAt": "2026-09-17T10:30:00Z"
}
```

---

### HTTP Status Codes

Status codes are grouped into 5 categories:

#### 1xx - Informational
- `100 Continue`: Server ready to receive body
- `101 Switching Protocols`: Changing protocol (WebSocket)

#### 2xx - Success
- `200 OK`: Request successful
- `201 Created`: Resource created successfully
- `202 Accepted`: Request accepted for processing
- `204 No Content`: Successful but no content in response

#### 3xx - Redirection
- `301 Moved Permanently`: Resource moved permanently
- `302 Found`: Temporary redirect
- `304 Not Modified`: Cached response still valid
- `307 Temporary Redirect`: Temporary redirect (preserves method)

#### 4xx - Client Error
- `400 Bad Request`: Malformed request syntax
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Authenticated but not authorized
- `404 Not Found`: Resource not found
- `409 Conflict`: Request conflicts with current state
- `422 Unprocessable Entity`: Request validation failed
- `429 Too Many Requests`: Rate limit exceeded

#### 5xx - Server Error
- `500 Internal Server Error`: Server error
- `501 Not Implemented`: Feature not supported
- `502 Bad Gateway`: Invalid response from upstream
- `503 Service Unavailable`: Server temporarily unavailable
- `504 Gateway Timeout`: Upstream server timeout

---

## Request-Response Flow Diagram

```
CLIENT                                    SERVER

  |                                          |
  |------ HTTP Request (Method, URL) ------->|
  |                                          |
  |          (with headers & body)           |
  |                                          |
  |                              [Processing]|
  |                                          |
  |<----- HTTP Response (Status Code) -------|
  |                                          |
  |          (with headers & body)           |
  |                                          |
```

---

## Quick Reference

### When to use each method?

```
Create:  POST   /api/users
Read:    GET    /api/users/123
Update:  PUT    /api/users/123  (full update)
Update:  PATCH  /api/users/123  (partial update)
Delete:  DELETE /api/users/123
```

### Essential Headers

```
Request:
  Content-Type: application/json
  Authorization: Bearer token
  Accept: application/json

Response:
  Content-Type: application/json
  Cache-Control: max-age=3600
```

### Common Status Codes

```
2xx - Success (200, 201, 204)
4xx - Client Error (400, 401, 403, 404)
5xx - Server Error (500, 503)
```

---

## Summary

| Topic | Key Point |
|-------|-----------|
| **REST** | Architecture using HTTP for CRUD on resources |
| **HTTP** | Stateless protocol for web communication (port 80) |
| **HTTPS** | Secure HTTP with SSL/TLS encryption (port 443) |
| **Methods** | GET, POST, PUT, PATCH, DELETE for different operations |
| **Request** | Line + Headers + Blank Line + Body |
| **Response** | Status Line + Headers + Blank Line + Body |
