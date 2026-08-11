# DataCamp HTTP & API Reconnaissance Lab

> **Security Research Lab | HTTP, Headers, Cookies, APIs & CORS**

## Overview

This lab documents practical HTTP reconnaissance against DataCamp's publicly accessible web infrastructure using `curl`.

The objective was to develop a repeatable methodology for:

* Inspecting HTTP response headers
* Following redirect chains
* Understanding cookies and session behavior
* Interacting with public APIs
* Inspecting request/response headers
* Testing HTTP content negotiation
* Identifying HTTP/2 and compression support
* Simulating browser requests
* Performing CORS preflight analysis
* Measuring HTTP performance
* Capturing and documenting HTTP responses

This lab focuses on **observation and understanding of HTTP behavior**, not exploitation.

---

## Scope & Ethics

Testing was performed against publicly accessible DataCamp endpoints and within the rules of the applicable bug bounty program.

No destructive testing, denial-of-service activity, brute forcing, credential attacks, or unauthorized access was performed.

Sensitive values such as authentication cookies, session identifiers, and tokens are excluded from this repository.

---

## Lab Environment

| Component        | Details                |
| ---------------- | ---------------------- |
| OS               | Kali Linux             |
| Primary Tool     | `curl`                 |
| Supporting Tools | `jq`, Python           |
| Target           | DataCamp               |
| Protocols        | HTTP/1.1, HTTP/2       |
| Focus            | Web/API reconnaissance |

---

# 1. Homepage Header Analysis

### Objective

Determine how the DataCamp homepage responds at the HTTP layer and identify relevant security and caching headers.

### Command

```bash
curl -I -L https://www.datacamp.com
```

### Evidence

See:

`evidence/01-homepage-headers.txt`

### Observations

The response was inspected for:

* HTTP protocol version
* Content type
* Redirect behavior
* Cookie-related headers
* Cache-control directives
* Transport security
* Other response metadata

### Security relevance

HTTP headers provide useful information about how an application handles transport security, caching, authentication state, and content delivery.

> **Important:** the presence or absence of a security header is not automatically a vulnerability. Impact and program scope must be considered.

---

# 2. Public Catalog API

### Objective

Identify the structure and behavior of a publicly accessible DataCamp catalog API.

### Command

```bash
curl -s "https://lms-catalog-api.datacamp.com/v1/catalog/live-courses?page=1&per_page=3" | python -m json.tool
```

### Evidence

See:

`evidence/02-catalog-api.json`

### Observations

The endpoint returned structured JSON containing course catalog information.

Fields examined included:

* Course title
* URL
* Content update information
* Technologies

### Security relevance

Public APIs should be examined to determine whether they expose information intended to remain private.

In this test, the endpoint was treated as a public catalog resource rather than assuming that publicly accessible data represents a vulnerability.

---

# 3. Verbose HTTP Analysis

### Objective

Inspect the complete request/response exchange.

### Command

```bash
curl -v "https://lms-catalog-api.datacamp.com/v1/catalog/live-courses?page=1&per_page=2"
```

### Key areas examined

```text
> Request headers
< Response headers
HTTP status
Content-Type
User-Agent
Accept
Content-Encoding
Request identifiers
```

### Security relevance

Verbose HTTP output provides a useful baseline for understanding application behavior before performing deeper API testing.

---

# 4. Content Negotiation

### Objective

Determine whether the application changes its response based on the client's requested representation.

### Tests

```bash
curl -I -H "Accept: application/json" https://www.datacamp.com

curl -I -H "Accept: text/html" https://www.datacamp.com

curl -I -H "Accept-Language: fr-FR" https://www.datacamp.com
```

### Observations

Responses were compared based on:

* Status code
* Content-Type
* Content-Language
* Redirect behavior
* Other response differences

### Security relevance

Content negotiation can become security-relevant when different representations expose different information or bypass security controls.

No security conclusion was drawn solely from differences in response formatting.

---

# 5. HTTP/2 & Compression

### Objective

Identify supported HTTP protocols and compression mechanisms.

### Commands

```bash
curl -I --http2 https://www.datacamp.com

curl -I -H "Accept-Encoding: br" https://www.datacamp.com

curl -I -H "Accept-Encoding: gzip" https://www.datacamp.com
```

### Observations

The responses were inspected for:

* HTTP protocol
* `Content-Encoding`
* Compression behavior

### Security relevance

Protocol and compression behavior are useful reconnaissance information and provide context for later HTTP security research.

---

# 6. Browser Request Simulation

### Objective

Compare a basic HTTP client request with a request containing common browser headers.

### Command

```bash
curl -v \
  -H "User-Agent: Mozilla/5.0" \
  -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8" \
  -H "Accept-Language: en-US,en;q=0.5" \
  -H "Accept-Encoding: gzip, deflate, br" \
  -H "DNT: 1" \
  -H "Sec-Fetch-Dest: document" \
  -H "Sec-Fetch-Mode: navigate" \
  -H "Sec-Fetch-Site: none" \
  -H "Sec-Fetch-User: ?1" \
  https://www.datacamp.com
```

### What I learned

This demonstrates that HTTP requests can be modified at the client level and that application behavior should be analyzed based on server-side controls rather than assuming that browser-supplied headers are trustworthy.

---

# 7. Cookie Observation

### Objective

Observe cookies issued during navigation without exposing sensitive values.

### Command

```bash
curl -I -L \
  -c datacamp_cookies.txt \
  -b datacamp_cookies.txt \
  https://www.datacamp.com
```

### Evidence

The cookie file was inspected locally.

Sensitive values were **not committed to GitHub**.

### Analysis

Cookie attributes were examined for:

```text
Name
Domain
Path
Expiration
Secure
HttpOnly
SameSite
```

### Security note

Cookie attributes were documented for learning purposes only.

The applicable DataCamp bug bounty policy explicitly excludes missing cookie flags, so this observation should not automatically be treated as a bounty finding.

---

# 8. CORS Preflight Analysis

### Objective

Understand how the API responds to a cross-origin preflight request.

### Command

```bash
curl -I -X OPTIONS \
  -H "Origin: https://example.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Authorization" \
  https://lms-catalog-api.datacamp.com/v1/catalog/live-courses
```

### Headers examined

```text
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Access-Control-Allow-Credentials
```

### Security relevance

CORS configuration becomes significant when an attacker-controlled origin can access sensitive authenticated responses.

A permissive CORS response on a public endpoint does not automatically demonstrate a vulnerability.

---

# 9. HTTP Performance Measurements

### Objective

Measure basic HTTP request timing.

### Command

```bash
curl -o /dev/null -s -w "
HTTP Code: %{http_code}
DNS Lookup: %{time_namelookup}s
TCP Connect: %{time_connect}s
TLS Handshake: %{time_appconnect}s
TTFB: %{time_starttransfer}s
Total: %{time_total}s
Size: %{size_download} bytes
" https://www.datacamp.com
```

### Metrics

| Metric        | Meaning                      |
| ------------- | ---------------------------- |
| DNS Lookup    | DNS resolution time          |
| TCP Connect   | TCP connection establishment |
| TLS Handshake | TLS negotiation              |
| TTFB          | Time to first byte           |
| Total         | Complete request duration    |
| Size          | Response size                |

---

# 10. Response Capture

### Objective

Save the HTTP response for offline inspection.

### Commands

```bash
curl -s https://www.datacamp.com -o datacamp_home.html

wc -c datacamp_home.html

file datacamp_home.html
```

### Analysis

The saved response can be examined for:

* HTML structure
* JavaScript references
* API endpoints
* Static resources
* Forms
* Metadata

This provides a bridge from basic HTTP reconnaissance into deeper application mapping.

---

# Key Lessons

This lab demonstrated that `curl` can be used as more than a simple HTTP downloader.

A structured workflow can move from:

```text
HTTP Request
      ↓
Response Headers
      ↓
Cookies
      ↓
API Discovery
      ↓
Request Manipulation
      ↓
CORS Analysis
      ↓
Application Mapping
      ↓
Security Testing
```

The most important lesson is that **an interesting technical behavior is not automatically a vulnerability**.

A security finding should be evaluated using:

```text
Behavior
   +
Security boundary
   +
Unauthorized impact
   +
Proof of concept
   +
Program scope
   =
Reportable finding
```

---

# Future Work

The next phase of this research can focus on authenticated API behavior and authorization testing using accounts controlled by the researcher.

Potential areas include:

* API endpoint mapping
* Object-level authorization
* BOLA/IDOR testing
* Cross-account access controls
* Enterprise tenant isolation
* Privilege boundaries
* Sensitive data exposure
* Authentication and authorization workflows

All future testing should remain within the program's defined scope and use researcher-controlled accounts.

---

## Disclaimer

This repository documents security research performed for educational and authorized testing purposes.

No credentials, session tokens, API keys, or private user information are intentionally published.

DataCamp's current bug bounty policy should always be consulted before interpreting an observation as a reportable vulnerability.

