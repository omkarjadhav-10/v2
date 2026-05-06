# Authentication & Authorization Documentation
## CEP Online Portal V2 - Spring Boot Microservices

> **📄 Quick Summary**
> This document provides a complete reference for authentication and authorization in the CEP Online Portal V2 system — a Java Spring Boot microservices architecture using stateless JWT-based authentication with dual identity provider support (Auth0 + Azure AD).
> 
> **Sections:** Fundamentals · Architecture · Request Flow · Security Implementation · Components · Risk Analysis · Best Practices

---

## Table of Contents

1. [Fundamental Terminology](#1-fundamental-terminology)
2. [High-Level System Overview](#2-high-level-system-overview)
3. [Authentication Architecture](#3-authentication-architecture)
4. [Authorization Architecture](#4-authorization-architecture)
5. [Complete Request Lifecycle](#5-complete-request-lifecycle)
6. [Key Components Reference](#6-key-components-reference)
7. [Security Configuration](#7-security-configuration)
8. [Error Handling](#8-error-handling)
9. [Security Analysis & Risks](#9-security-analysis--risks)
10. [Improvement Recommendations](#10-improvement-recommendations)
11. [Quick Reference for Developers](#11-quick-reference-for-developers)

---

## 1. Fundamental Terminology

### Core Concepts

#### Authentication
The process of **verifying who a user is**. In this system, authentication answers: "Is this user who they claim to be?"

**Example:** When you present a JWT token, the system validates the token's signature, expiry, and issuer to authenticate you.

#### Authorization
The process of **determining what an authenticated user can do**. In this system, authorization answers: "Does this user have permission to access this resource?"

**Example:** Even after authentication, a user might not have permission to view orders from a specific institution.

---

### Authentication-Related Terms

#### JWT (JSON Web Token)
A compact, URL-safe token format for securely transmitting information between parties. Structure: `header.payload.signature`

**Example Token:**
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhdXRoMHwxMjM0NTYiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20iLCJwZXJtaXNzaW9ucyI6WyJyZWFkOm9yZGVyQ3JlYXRvciJdfQ.signature
```

**JWT Parts:**
- **Header:** Metadata (algorithm, token type)
- **Payload:** Claims (user data, permissions, expiry)
- **Signature:** Cryptographic signature for verification

#### Bearer Token
An access token sent in the HTTP `Authorization` header:
```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

The word "Bearer" indicates that whoever *bears* (possesses) this token is authorized.

#### Stateless Authentication
Authentication where the server does not store session data. Every request must carry a valid token. Benefits:
- **Scalability:** No session storage needed
- **Microservices-friendly:** Each service validates independently
- **No session management:** No session expiry or invalidation complexity

#### Identity Provider (IdP)
An external service that authenticates users and issues tokens. This system supports two IdPs:
- **Auth0:** Primary identity provider
- **Azure Active Directory (Azure AD):** Secondary/fallback provider

#### Claims
Pieces of information asserted about a subject (user), stored in the JWT payload.

**Common Claims:**
```json
{
  "sub": "auth0|123456",                    // Subject (user ID)
  "email": "user@example.com",              // Email address
  "permissions": ["read:all"],              // Permissions array
  "institutionCodes": ["INST001", "INST002"], // Custom claim
  "exp": 1735689600,                        // Expiry timestamp
  "aud": "https://api.example.com"          // Audience
}
```

#### JWKS (JSON Web Key Set)
A set of public keys used to verify JWT signatures. The system fetches JWKS from Auth0/Azure AD dynamically.

**JWKS Endpoint Example:**
```
https://dev-example.auth0.com/.well-known/jwks.json
```

#### Token Validation
The process of verifying that a JWT is:
1. **Cryptographically valid** (signature matches)
2. **Not expired** (current time < `exp` claim)
3. **From trusted issuer** (matches configured issuer)
4. **For this audience** (matches configured audience)
5. **Currently valid** (current time >= `nbf` claim)
6. **Not too old** (within 24-hour max age in this system)

---

### Authorization-Related Terms

#### Permissions
String-based capabilities that define what actions a user can perform. Stored in JWT claims.

**Permission Examples in This System:**
- `read:all` - Admin permission, bypasses institution restrictions
- `update:orderSubmitter` - Can create AND submit orders
- `read:orderSubmitter` - Can submit orders
- `read:orderCreator` - Can create/view orders but NOT submit

#### Roles
Collections of permissions. In this system, roles exist in the JWT but permissions are the primary authorization mechanism.

#### Institution Codes
A custom authorization dimension specific to this system. Users are assigned institution codes in their JWT, restricting which institution's data they can access.

**Example:**
```json
{
  "institutionCodes": ["INST001", "INST002"]
}
```

This user can only access orders/data from institutions INST001 and INST002.

#### SpEL (Spring Expression Language)
A powerful expression language used in `@PreAuthorize` and `@PostAuthorize` annotations for fine-grained authorization.

**Example:**
```java
@PreAuthorize("@institutionSecurity.canAccessOrder(#orderId)")
public OrderDTO getOrder(Long orderId) { ... }
```

#### Security Context
A thread-local holder for the current user's authentication information in Spring Security.

```java
// Retrieve current user's authentication
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

---

### Security Patterns & Concepts

#### Multi-Tenant Token Validation
This system supports multiple identity providers (Auth0 tenants + Azure AD). When a token arrives:
1. Try validating against each Auth0 tenant (in parallel)
2. If all Auth0 validations fail, try Azure AD
3. First successful validation wins

#### API Key Authentication
Alternative authentication for **service-to-service** communication. Instead of JWT:
```http
Authorization: api-key abc123xyz789
```

API keys are stored in database, validated, and cached in memory.

#### Clock Skew Tolerance
Network latency and clock differences between servers can cause timing issues. This system allows **60 seconds** of clock skew when validating `exp`, `nbf`, and `iat` claims.

#### OncePerRequestFilter
A Spring filter that guarantees execution **exactly once per request**, even if the request is forwarded or included.

---

### HTTP Status Codes

#### 401 Unauthorized
Authentication failed. User is not authenticated (no valid token or invalid token).

**When it happens:**
- Missing token
- Expired token
- Invalid signature
- Token from untrusted issuer

#### 403 Forbidden
Authorization failed. User is authenticated but lacks permission.

**When it happens:**
- Insufficient permissions
- Wrong institution code
- Access to unauthorized resource

---

## 2. High-Level System Overview

### Architecture Type
**Microservices architecture** using stateless JWT-based authentication.

### Key Characteristics
1. **Stateless:** No server-side session storage
2. **Distributed:** 9 microservices, each validates tokens independently
3. **Multi-tenant IdP:** Supports Auth0 (primary) + Azure AD (fallback)
4. **Shared library:** All auth logic centralized in `cep-common` library
5. **Two-tier authorization:** Permission-based + Institution-based

### Authentication Methods
```
┌─────────────────────────────────────────────────────────┐
│                    Client Request                        │
└────────────────────┬────────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
     ┌────▼─────┐        ┌─────▼────┐
     │   JWT    │        │ API Key  │
     │  (Users) │        │(Services)│
     └────┬─────┘        └─────┬────┘
          │                     │
          │   Authorization     │
          │      Header         │
          │                     │
     Bearer {token}      api-key {key}
```

### System Components
- **9 Microservices:** Each imports `cep-common` for auth
- **Shared Library:** `cep-common` contains all auth/authz logic
- **Identity Providers:** Auth0 (multiple tenants) + Azure AD
- **Database:** Stores API keys, user permissions (indirect via JWT)

---

## 3. Authentication Architecture

### Overview
Authentication is the **first security gate**. Every request passes through `JwtAuthenticationFilter` which validates the token before the request reaches any controller.

---

### Authentication Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Request                            │
│               (Authorization: Bearer {token})                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │JwtAuthenticationFilter│ ◄─── OncePerRequestFilter
                  │  (Entry Point)        │      (runs on EVERY request)
                  └──────────┬────────────┘
                             │
                    ┌────────┴─────────┐
                    │ Public Endpoint? │
                    └────────┬─────────┘
                             │
                  ┌──────────┴───────────┐
                  │                      │
                YES                     NO
                  │                      │
                  ▼                      ▼
          ┌─────────────┐      ┌──────────────────┐
          │ Skip Auth   │      │ Extract Token    │
          │ Continue    │      │ from Header      │
          └─────────────┘      └────────┬─────────┘
                                         │
                              ┌──────────┴──────────┐
                              │ Token Type?          │
                              └──────────┬──────────┘
                                         │
                       ┌─────────────────┼─────────────────┐
                       │                 │                 │
                  Bearer {token}    api-key {key}      (missing)
                       │                 │                 │
                       ▼                 ▼                 ▼
          ┌────────────────────┐  ┌──────────────┐  ┌──────────┐
          │MultiTenantToken    │  │ApiKeyService │  │  Return  │
          │Validator           │  │Impl          │  │   401    │
          └────────┬───────────┘  └──────┬───────┘  └──────────┘
                   │                     │
         ┌─────────┴──────────┐          │
         │                    │          │
    ┌────▼──────┐      ┌──────▼──────┐  │
    │  Auth0    │      │  Azure AD   │  │
    │ Tenants   │      │  Fallback   │  │
    │(parallel) │      │             │  │
    └────┬──────┘      └──────┬──────┘  │
         │                    │          │
         └─────────┬──────────┘          │
                   │                     │
                   ▼                     ▼
          ┌─────────────────┐   ┌───────────────┐
          │JwtClaimsValidator│   │DB Validation  │
          │- Signature       │   │- isActive?    │
          │- Expiry          │   │- Service match│
          │- Audience        │   │(cached 60min) │
          │- Issuer          │   └───────┬───────┘
          │- Max Age (24h)   │           │
          └────────┬─────────┘           │
                   │                     │
                   ▼                     ▼
          ┌─────────────────┐   ┌───────────────────┐
          │JwtSigning       │   │ApiKeyAuth         │
          │KeyResolver      │   │Token              │
          │(fetch JWKS)     │   └─────────┬─────────┘
          └────────┬────────┘             │
                   │                      │
                   ▼                      │
          ┌─────────────────┐             │
          │Extract Claims:  │             │
          │- username       │             │
          │- permissions    │             │
          │- roles          │             │
          │- institutionCodes            │
          └────────┬────────┘             │
                   │                      │
                   └──────────┬───────────┘
                              │
                              ▼
                  ┌───────────────────────┐
                  │SecurityContextHolder  │
                  │.setAuthentication()   │
                  └───────────┬───────────┘
                              │
                              ▼
                  ┌───────────────────────┐
                  │ Request continues to  │
                  │    Controller         │
                  └───────────────────────┘
```

---

### JWT Validation Chain (Detailed)

#### Step 1: Multi-Tenant Strategy
```java
// MultiTenantTokenValidator.java
public Authentication validateToken(String token) {
    // Try Auth0 tenants in parallel
    List<CompletableFuture<Authentication>> auth0Futures = 
        auth0Tenants.stream()
            .map(tenant -> validateWithAuth0(token, tenant))
            .collect(Collectors.toList());
    
    // Check if any Auth0 validation succeeded
    for (CompletableFuture<Authentication> future : auth0Futures) {
        if (future.succeeded()) return future.get();
    }
    
    // Fallback to Azure AD
    return validateWithAzureAD(token);
}
```

#### Step 2: Claims Validation
```java
// JwtClaimsValidator.java
public Claims validateAndExtractClaims(String token, Config config) {
    JwtParser parser = Jwts.parserBuilder()
        .setSigningKeyResolver(signingKeyResolver)  // Fetch public key
        .setAllowedClockSkewSeconds(60)             // 60s tolerance
        .build();
    
    Jws<Claims> jws = parser.parseClaimsJws(token);
    Claims claims = jws.getBody();
    
    // Validate audience
    validateAudience(claims.getAudience(), config.getAudience());
    
    // Validate expiry
    if (claims.getExpiration().before(new Date())) {
        throw new TokenExpiredException();
    }
    
    // Validate max age (24 hours)
    Date issuedAt = claims.getIssuedAt();
    if (issuedAt.before(now.minus(24, HOURS))) {
        throw new TokenTooOldException();
    }
    
    return claims;
}
```

#### Step 3: Signing Key Resolution
```java
// JwtSigningKeyResolver.java
public Key resolveSigningKey(JwsHeader header, Claims claims) {
    String keyId = header.getKeyId();  // "kid" from JWT header
    
    // Fetch JWKS from IdP
    JsonNode jwks = fetchJwks(jwksUrl);  // Auth0 or Azure AD URL
    
    // Find matching public key
    for (JsonNode key : jwks.get("keys")) {
        if (key.get("kid").asText().equals(keyId)) {
            return constructPublicKey(key);
        }
    }
    
    throw new SigningKeyNotFoundException();
}
```

---

### Token Claims Extraction

After successful validation, the system extracts and stores claims:

```java
// JwtAuthenticationDetails.java
public class JwtAuthenticationDetails {
    private Claims claims;              // All JWT claims
    private String identityProvider;    // "Auth0" or "AzureAD"
    
    public String getUsername() {
        // Try multiple claim names (IdP-dependent)
        return claims.get("email", String.class) 
            ?: claims.get("unique_name", String.class)
            ?: claims.get("preferred_username", String.class);
    }
    
    public List<String> getPermissions() {
        return claims.get("permissions", List.class) ?: [];
    }
    
    public List<String> getInstitutionCodes() {
        String customClaimKey = "https://orders-qa.baylorgenetics.com/institutionCodes";
        return claims.get(customClaimKey, List.class) ?: [];
    }
}
```

---

### API Key Authentication (Service-to-Service)

For internal microservice communication:

```
┌─────────────────────────────────────────────┐
│   Service A → Service B                     │
│   Authorization: api-key abc123xyz789       │
└────────────────┬────────────────────────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │ ApiKeyServiceImpl    │
      └──────────┬───────────┘
                 │
          ┌──────┴──────┐
          │ Check Cache │
          └──────┬──────┘
                 │
         ┌───────┴────────┐
         │                │
     Cache Hit        Cache Miss
         │                │
         ▼                ▼
    ┌─────────┐     ┌──────────┐
    │ Return  │     │ Query DB │
    │ Result  │     │ api_keys │
    └─────────┘     └─────┬────┘
                          │
                   ┌──────┴───────┐
                   │ Validate:    │
                   │ - isActive?  │
                   │ - service OK?│
                   └──────┬───────┘
                          │
                   ┌──────┴───────┐
                   │ Cache Result │
                   │ (60 min TTL) │
                   └──────────────┘
```

#### API Key Entity
```java
@Entity
@Table(name = "api_keys", schema = "cep_core")
public class ApiKey {
    @Id
    private Long id;
    
    private String apiKey;           // The actual key value
    private String serviceName;      // Which service owns this key
    private boolean isActive;        // Can be revoked
    private LocalDateTime lastUsedAt; // Audit trail
    
    public boolean isAuthorizedForService(String targetService) {
        return this.serviceName.equals(targetService);
    }
}
```

---

### Token Validation Rules

| Rule | Validation | Failure Error Code |
|------|------------|-------------------|
| **Signature** | Verified using JWKS public key from IdP | AUTH_1007 |
| **Expiry (`exp`)** | Current time must be < exp | AUTH_1002 |
| **Not Before (`nbf`)** | Current time must be >= nbf | AUTH_1004 |
| **Issued At (`iat`)** | Token must be < 24 hours old | AUTH_1005 |
| **Audience (`aud`)** | Must match configured audience | AUTH_1003 |
| **Subject (`sub`)** | Must be present | AUTH_1006 |
| **Clock Skew** | 60-second tolerance for timing checks | N/A |

---

## 4. Authorization Architecture

### Overview
Authorization determines **what an authenticated user can do**. This system uses a **two-tier authorization model**:

1. **Permission-based:** What actions can the user perform?
2. **Institution-based:** Which institution's data can the user access?

---

### Two-Tier Authorization Model

```
┌─────────────────────────────────────────────────────────────┐
│              Authenticated User Request                      │
│              (SecurityContext populated)                     │
└───────────────────────┬─────────────────────────────────────┘
                        │
           ┌────────────┴────────────┐
           │                         │
    ┌──────▼─────────┐      ┌───────▼────────┐
    │ Permission     │      │ Institution    │
    │ Check          │      │ Code Check     │
    └──────┬─────────┘      └───────┬────────┘
           │                         │
    Does user have         Does user belong to
    required permission?   the right institution?
           │                         │
           │                         │
    ┌──────▼─────────┐      ┌───────▼────────┐
    │ SecurityUtil   │      │ Institution    │
    │.hasPermission()│      │ Security       │
    │                │      │ Evaluator      │
    └──────┬─────────┘      └───────┬────────┘
           │                         │
           └──────────┬──────────────┘
                      │
              ┌───────▼────────┐
              │  Both Pass?    │
              └───────┬────────┘
                      │
              ┌───────┴────────┐
              │                │
            YES              NO
              │                │
              ▼                ▼
      ┌──────────────┐  ┌──────────┐
      │ Allow Access │  │ Return   │
      │ Continue     │  │ 403      │
      └──────────────┘  └──────────┘
```

---

### Permission-Based Authorization

#### Permission Model

**Permissions are hierarchical strings** stored in JWT claims:

| Permission | Level | Capabilities |
|------------|-------|-------------|
| `read:all` | **Admin** | Full access, bypasses institution restrictions |
| `update:orderSubmitter` | **Write** | Can create AND submit orders |
| `read:orderSubmitter` | **Submit** | Can submit orders (but not create) |
| `read:orderCreator` | **Read** | Can create/view orders but NOT submit |

#### Default Permission Fallback

**⚠️ Security Concern:** If a user has NO permissions in their JWT:

```java
// SecurityUtil.java
public static boolean hasPermission(String requiredPermission) {
    List<String> permissions = getPermissionsFromToken();
    
    if (permissions.isEmpty()) {
        // DANGEROUS: Defaults to granting write permission!
        return requiredPermission.equals("update:orderSubmitter");
    }
    
    return permissions.contains(requiredPermission);
}
```

This **fail-open** behavior grants write access by default, which violates security best practices.

#### Permission Check Examples

```java
// Controller method with permission check
@PreAuthorize("@securityUtil.hasPermission('read:orderCreator')")
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    return orderService.findById(id);
}

// Programmatic permission check
if (SecurityUtil.hasPermission("update:orderSubmitter")) {
    orderService.submitOrder(orderId);
}
```

---

### Institution-Based Authorization

#### Institution Code Model

Users are assigned **institution codes** in their JWT, which restricts data access:

```json
{
  "https://orders-qa.baylorgenetics.com/institutionCodes": [
    "INST001",
    "INST002",
    "INST003"
  ]
}
```

This user can ONLY access data from institutions INST001, INST002, and INST003.

#### Admin Bypass

Users with `read:all` permission **bypass** institution code checks entirely.

```java
// InstitutionSecurityEvaluator.java
public boolean canAccessOrder(Long orderId) {
    // Admin bypass
    if (SecurityUtil.hasPermission("read:all")) {
        return true;  // No institution check needed
    }
    
    // Regular users: check institution match
    Order order = orderRepository.findById(orderId);
    List<String> userInstitutions = SecurityUtil.getInstitutionCodesFromToken();
    
    return userInstitutions.contains(order.getHospitalCode());
}
```

---

### Method Security Annotations

#### @PreAuthorize (Before Method Execution)

Checks authorization **before** the method runs:

```java
@PreAuthorize("@institutionSecurity.canAccessOrder(#orderId)")
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long orderId) {
    return orderService.findById(orderId);
}
```

**Flow:**
1. Request arrives
2. `@PreAuthorize` expression evaluates
3. If true → method executes
4. If false → throws `AccessDeniedException` (403)

#### @PostAuthorize (After Method Execution)

Checks authorization **after** the method runs, based on the return value:

```java
@PostAuthorize("@institutionSecurity.canAccessOrderResponse(returnObject)")
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    return orderService.findById(id);
}
```

**⚠️ Performance Concern:** The DB query executes fully before authorization check. If denied, processing cost and audit trail are already incurred.

**Flow:**
1. Method executes
2. Result is computed
3. `@PostAuthorize` expression evaluates on `returnObject`
4. If true → return result
5. If false → throws `AccessDeniedException` (403), discards result

---

### InstitutionSecurityEvaluator (SpEL Bean)

This bean provides authorization methods used in `@PreAuthorize` expressions:

```java
@Component("institutionSecurity")
public class InstitutionSecurityEvaluator {
    
    /**
     * Check if user can access an order by ID
     */
    public boolean canAccessOrder(Long orderId) {
        if (SecurityUtil.hasPermission("read:all")) {
            return true;  // Admin bypass
        }
        
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found"));
        
        List<String> userInstitutions = SecurityUtil.getInstitutionCodesFromToken();
        
        if (!userInstitutions.contains(order.getHospitalCode())) {
            throw new InvalidInstitutionCodeException(
                "User not authorized for institution: " + order.getHospitalCode()
            );
        }
        
        return true;
    }
    
    /**
     * Check if user can access an order from response DTO
     */
    public boolean canAccessOrderResponse(Object response) {
        if (!(response instanceof InstitutionAware)) {
            return true;  // No institution check needed
        }
        
        InstitutionAware dto = (InstitutionAware) response;
        String institutionCode = dto.getInstitutionCode();
        
        List<String> userInstitutions = SecurityUtil.getInstitutionCodesFromToken();
        
        return userInstitutions.contains(institutionCode);
    }
}
```

---

### SecurityUtil (Authorization Helper)

Central utility for extracting authorization data from `SecurityContext`:

```java
public class SecurityUtil {
    
    /**
     * Extract username from current authentication
     */
    public static String getCurrentUsername() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        if (auth instanceof UsernamePasswordAuthenticationToken) {
            JwtAuthenticationDetails details = (JwtAuthenticationDetails) auth.getDetails();
            return details.getUsername();
        }
        
        return auth.getName();
    }
    
    /**
     * Get permissions from JWT claims
     */
    public static List<String> getPermissionsFromToken() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        JwtAuthenticationDetails details = (JwtAuthenticationDetails) auth.getDetails();
        
        return details.getPermissions();
    }
    
    /**
     * Check if user has specific permission
     */
    public static boolean hasPermission(String permission) {
        List<String> permissions = getPermissionsFromToken();
        
        // DANGEROUS DEFAULT: grants write if no permissions
        if (permissions.isEmpty()) {
            return permission.equals("update:orderSubmitter");
        }
        
        return permissions.contains(permission);
    }
    
    /**
     * Get institution codes from JWT claims
     */
    public static List<String> getInstitutionCodesFromToken() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        JwtAuthenticationDetails details = (JwtAuthenticationDetails) auth.getDetails();
        
        String claimKey = "https://orders-qa.baylorgenetics.com/institutionCodes";
        return details.getClaims().get(claimKey, List.class);
    }
}
```

---

## 5. Complete Request Lifecycle

### Step-by-Step Request Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                     HTTP Request Arrives                                │
│          GET /api/orders/123                                            │
│          Authorization: Bearer eyJhbGci...                              │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 1: Filter Layer                                                    │
│ Component: JwtAuthenticationFilter                                      │
├──────────────────────────────────────────────────────────────────────────┤
│ - Intercepts EVERY request before controller                            │
│ - Checks if endpoint is public                                          │
│ - If public → skip to Step 8                                            │
│ - If protected → continue to Step 2                                     │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 2: Token Extraction                                                │
│ Component: JwtAuthenticationFilter                                      │
├──────────────────────────────────────────────────────────────────────────┤
│ - Read Authorization header                                             │
│ - Extract token value                                                   │
│ - If missing → Step 10 (401 error)                                      │
│ - If present → Step 3                                                   │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 3: Token Type Identification                                       │
│ Component: JwtAuthenticationFilter                                      │
├──────────────────────────────────────────────────────────────────────────┤
│ - Bearer token? → Step 4 (JWT validation)                               │
│ - API key? → Step 11 (API key validation)                               │
│ - Unknown format? → Step 10 (401 error)                                 │
└───────────────────────────┬────────────────────────────────────────────┘
                            │ (JWT path)
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 4: Multi-Tenant Validation                                         │
│ Component: MultiTenantTokenValidator                                    │
├──────────────────────────────────────────────────────────────────────────┤
│ - Try Auth0 tenants (all in parallel via CompletableFuture)            │
│ - If any Auth0 succeeds → Step 5                                        │
│ - If all Auth0 fail → try Azure AD                                      │
│ - If Azure AD succeeds → Step 5                                         │
│ - If all fail → Step 10 (401 error)                                     │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 5: Claims Validation                                               │
│ Component: JwtClaimsValidator                                           │
├──────────────────────────────────────────────────────────────────────────┤
│ - Fetch public key via JwtSigningKeyResolver (JWKS endpoint)            │
│ - Verify signature                                                      │
│ - Check expiry (exp claim)                                              │
│ - Check not-before (nbf claim)                                          │
│ - Check token age (iat claim, max 24 hours)                             │
│ - Validate audience (aud claim)                                         │
│ - Validate issuer (iss claim)                                           │
│ - Allow 60-second clock skew                                            │
│ - If any check fails → Step 10 (401 error with specific code)          │
│ - If all pass → Step 6                                                  │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 6: Extract User Identity & Permissions                             │
│ Component: JwtAuthenticationDetails                                     │
├──────────────────────────────────────────────────────────────────────────┤
│ - Extract username (email / unique_name / preferred_username)           │
│ - Extract permissions array from claims                                 │
│ - Extract roles array from claims                                       │
│ - Extract institutionCodes from custom claim                            │
│ - Store all claims in JwtAuthenticationDetails object                   │
│ - Store identity provider name (Auth0 or AzureAD)                       │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 7: Set Security Context                                            │
│ Component: SecurityContextHolder                                        │
├──────────────────────────────────────────────────────────────────────────┤
│ - Create UsernamePasswordAuthenticationToken                            │
│ - Attach JwtAuthenticationDetails                                       │
│ - Store in SecurityContextHolder (thread-local)                         │
│ - Authentication is now available to downstream components              │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 8: Pre-Authorization Check (if present)                            │
│ Component: @PreAuthorize annotation                                     │
├──────────────────────────────────────────────────────────────────────────┤
│ Example: @PreAuthorize("@institutionSecurity.canAccessOrder(#orderId)")│
│                                                                          │
│ - Evaluate SpEL expression before method execution                      │
│ - InstitutionSecurityEvaluator.canAccessOrder() is called               │
│   - If user has read:all → return true (admin bypass)                   │
│   - Otherwise: fetch order from DB, check institution code              │
│   - Compare order's institution vs user's institutionCodes              │
│ - If expression returns false → Step 10 (403 error)                     │
│ - If expression returns true → Step 9                                   │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 9: Controller & Service Execution                                  │
│ Components: Controller → Service → Repository                           │
├──────────────────────────────────────────────────────────────────────────┤
│ - Controller method executes                                            │
│ - Business logic runs                                                   │
│ - Database queries execute                                              │
│ - Response DTO is built                                                 │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 10: Post-Authorization Check (if present)                          │
│ Component: @PostAuthorize annotation                                    │
├──────────────────────────────────────────────────────────────────────────┤
│ Example: @PostAuthorize("@institutionSecurity.canAccessOrderResponse    │
│                          (returnObject)")                               │
│                                                                          │
│ - Evaluate SpEL expression on return value                              │
│ - Check if returnObject implements InstitutionAware                     │
│ - Extract institution code from response                                │
│ - Compare vs user's institutionCodes                                    │
│ - If mismatch → discard result, return 403 error                        │
│ - If match → return response to client                                  │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│ STEP 11: Response Sent                                                  │
├──────────────────────────────────────────────────────────────────────────┤
│ - HTTP 200 OK with response body                                        │
│ OR                                                                       │
│ - HTTP 401 Unauthorized (auth failure)                                  │
│ OR                                                                       │
│ - HTTP 403 Forbidden (authz failure)                                    │
└──────────────────────────────────────────────────────────────────────────┘


┌──────────────────────────────────────────────────────────────────────────┐
│ ALTERNATE PATH: API Key Authentication (Step 11)                         │
│ Component: ApiKeyServiceImpl                                             │
├──────────────────────────────────────────────────────────────────────────┤
│ - Check Caffeine cache for API key                                       │
│ - Cache hit? → use cached validation result                              │
│ - Cache miss? → query database (api_keys table)                          │
│   - Check isActive flag                                                  │
│   - Check serviceName matches target service                             │
│   - Update lastUsedAt timestamp                                          │
│   - Cache result (60-minute TTL)                                         │
│ - If valid → create ApiKeyAuthenticationToken, set SecurityContext       │
│ - If invalid → return 401                                                │
│ - Continue to Step 8 (authorization)                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### Request Lifecycle Table

| Step | Component | Action | Success → | Failure → |
|------|-----------|--------|-----------|-----------|
| 1 | JwtAuthenticationFilter | Intercept request | Step 2 | N/A |
| 2 | JwtAuthenticationFilter | Extract Authorization header | Step 3 | 401 (AUTH_1001) |
| 3 | JwtAuthenticationFilter | Identify token type | Step 4 or 11 | 401 (AUTH_1008) |
| 4 | MultiTenantTokenValidator | Try Auth0 + Azure AD | Step 5 | 401 (varies) |
| 5 | JwtClaimsValidator | Validate JWT signature & claims | Step 6 | 401 (AUTH_1002-1007) |
| 6 | JwtAuthenticationDetails | Extract user data & permissions | Step 7 | N/A |
| 7 | SecurityContextHolder | Store authentication | Step 8 | N/A |
| 8 | @PreAuthorize (if present) | Check permissions/institution | Step 9 | 403 |
| 9 | Controller/Service | Execute business logic | Step 10 | 500 (if error) |
| 10 | @PostAuthorize (if present) | Verify response institution | Step 11 | 403 |
| 11 | HTTP Response | Return result | End | N/A |
| Alt | ApiKeyServiceImpl | Validate API key (cached) | Step 7 | 401 |

---

## 6. Key Components Reference

### Core Security Components

#### JwtAuthenticationFilter
**Location:** `shared/cep-common/src/main/.../security/authentication/JwtAuthenticationFilter.java`

**Type:** `OncePerRequestFilter` (Spring Security)

**Purpose:** Entry point for all authentication. Runs once per request.

**Key Methods:**
```java
@Override
protected void doFilterInternal(
    HttpServletRequest request,
    HttpServletResponse response,
    FilterChain filterChain
) throws ServletException, IOException {
    
    // Check if public endpoint
    if (shouldNotFilter(request)) {
        filterChain.doFilter(request, response);
        return;
    }
    
    // Extract Authorization header
    String authHeader = request.getHeader("Authorization");
    
    if (authHeader == null) {
        handleMissingAuthentication(response);
        return;
    }
    
    // Route to JWT or API key authentication
    if (authHeader.startsWith("Bearer ")) {
        handleJwtAuthentication(authHeader, request, response);
    } else if (authHeader.startsWith("api-key ")) {
        handleApiKeyAuthentication(authHeader, request, response);
    } else {
        handleInvalidAuthentication(response);
    }
    
    filterChain.doFilter(request, response);
}

@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    String path = request.getRequestURI();
    return Constants.PUBLIC_ENDPOINTS.stream()
        .anyMatch(path::startsWith);
}
```

**Responsibilities:**
- Intercept every HTTP request
- Check if endpoint is public
- Extract and validate tokens
- Set SecurityContext
- Handle authentication errors

---

#### MultiTenantTokenValidator
**Location:** `shared/cep-common/src/main/.../security/authentication/MultiTenantTokenValidator.java`

**Purpose:** Orchestrates validation across multiple identity providers.

**Key Methods:**
```java
public Authentication validateToken(String token) {
    // Try all Auth0 tenants in parallel
    List<CompletableFuture<Authentication>> auth0Futures = 
        auth0Configurations.stream()
            .map(config -> CompletableFuture.supplyAsync(() -> 
                validateWithTenant(token, config)
            ))
            .collect(Collectors.toList());
    
    // Check Auth0 results
    for (CompletableFuture<Authentication> future : auth0Futures) {
        try {
            Authentication result = future.get(2, TimeUnit.SECONDS);
            if (result != null) {
                return result;  // First success wins
            }
        } catch (Exception e) {
            // Continue to next tenant
        }
    }
    
    // Fallback to Azure AD
    return validateWithAzureAD(token);
}

private Authentication validateWithTenant(String token, Auth0TenantConfiguration config) {
    try {
        Claims claims = jwtClaimsValidator.validateAndExtractClaims(token, config);
        return buildAuthentication(claims, "Auth0");
    } catch (JwtException e) {
        return null;  // Validation failed
    }
}
```

**Responsibilities:**
- Try each Auth0 tenant (asynchronously)
- Fallback to Azure AD if all Auth0 fail
- Return first successful authentication
- Handle validation exceptions

---

#### JwtClaimsValidator
**Location:** `shared/cep-common/src/main/.../security/authentication/JwtClaimsValidator.java`

**Purpose:** Low-level JWT parsing and validation.

**Key Methods:**
```java
public Claims validateAndExtractClaims(String token, IdPConfiguration config) 
    throws JwtException {
    
    JwtParser parser = Jwts.parserBuilder()
        .setSigningKeyResolver(jwtSigningKeyResolver)
        .setAllowedClockSkewSeconds(ALLOWED_CLOCK_SKEW_SECONDS)  // 60
        .build();
    
    try {
        Jws<Claims> jws = parser.parseClaimsJws(token);
        Claims claims = jws.getBody();
        
        // Validate audience
        validateAudience(claims, config.getAudience());
        
        // Validate max age (24 hours)
        validateTokenAge(claims);
        
        // Validate subject exists
        if (claims.getSubject() == null) {
            throw new MissingSubjectException();
        }
        
        return claims;
        
    } catch (ExpiredJwtException e) {
        throw new TokenExpiredException("Token expired", e);
    } catch (SignatureException e) {
        throw new InvalidSignatureException("Invalid signature", e);
    } catch (MalformedJwtException e) {
        throw new MalformedTokenException("Malformed token", e);
    }
}

private void validateTokenAge(Claims claims) {
    Date issuedAt = claims.getIssuedAt();
    Instant maxAge = Instant.now().minus(TOKEN_MAX_AGE_HOURS, ChronoUnit.HOURS);
    
    if (issuedAt.toInstant().isBefore(maxAge)) {
        throw new TokenTooOldException("Token older than " + TOKEN_MAX_AGE_HOURS + " hours");
    }
}
```

**Validation Rules:**
- **Signature:** JWKS public key from IdP
- **Expiry (`exp`):** Must be in future
- **Not Before (`nbf`):** Must be in past
- **Issued At (`iat`):** Must be within 24 hours
- **Audience (`aud`):** Must match configuration
- **Subject (`sub`):** Must be present
- **Clock Skew:** 60-second tolerance

---

#### JwtSigningKeyResolver
**Location:** `shared/cep-common/src/main/.../security/resolver/JwtSigningKeyResolver.java`

**Purpose:** Dynamically fetch public keys from IdP JWKS endpoints.

**Key Methods:**
```java
@Override
public Key resolveSigningKey(JwsHeader header, Claims claims) {
    String keyId = header.getKeyId();  // "kid" from JWT header
    
    // Construct JWKS URL based on issuer
    String issuer = claims.getIssuer();
    String jwksUrl = buildJwksUrl(issuer);
    
    // Fetch JWKS from IdP
    JsonNode jwks = fetchJwks(jwksUrl);
    
    // Find matching key by kid
    for (JsonNode key : jwks.get("keys")) {
        if (keyId.equals(key.get("kid").asText())) {
            return buildPublicKey(key);
        }
    }
    
    throw new SigningKeyNotFoundException("No key found for kid: " + keyId);
}

private String buildJwksUrl(String issuer) {
    if (issuer.contains("auth0.com")) {
        return issuer + ".well-known/jwks.json";
    } else if (issuer.contains("microsoft")) {
        return issuer + "/discovery/v2.0/keys";
    }
    throw new UnsupportedIssuerException(issuer);
}

private PublicKey buildPublicKey(JsonNode keyNode) {
    String n = keyNode.get("n").asText();  // Modulus
    String e = keyNode.get("e").asText();  // Exponent
    
    byte[] nBytes = Base64.getUrlDecoder().decode(n);
    byte[] eBytes = Base64.getUrlDecoder().decode(e);
    
    BigInteger modulus = new BigInteger(1, nBytes);
    BigInteger exponent = new BigInteger(1, eBytes);
    
    RSAPublicKeySpec spec = new RSAPublicKeySpec(modulus, exponent);
    KeyFactory factory = KeyFactory.getInstance("RSA");
    
    return factory.generatePublic(spec);
}
```

**JWKS Endpoints:**
- **Auth0:** `https://{domain}.auth0.com/.well-known/jwks.json`
- **Azure AD:** `https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys`

---

#### JwtAuthenticationDetails
**Location:** `shared/cep-common/src/main/.../security/authentication/JwtAuthenticationDetails.java`

**Purpose:** Store and provide access to JWT claims throughout request lifecycle.

**Key Fields:**
```java
public class JwtAuthenticationDetails {
    private Claims claims;              // All JWT claims
    private String identityProvider;    // "Auth0" or "AzureAD"
    
    // Extract username from multiple possible claim names
    public String getUsername() {
        return Optional.ofNullable(claims.get("email", String.class))
            .orElseGet(() -> Optional.ofNullable(claims.get("unique_name", String.class))
            .orElseGet(() -> claims.get("preferred_username", String.class)));
    }
    
    // Get permissions array
    public List<String> getPermissions() {
        return claims.get("permissions", List.class);
    }
    
    // Get roles array
    public List<String> getRoles() {
        return claims.get("roles", List.class);
    }
    
    // Get institution codes from custom claim
    public List<String> getInstitutionCodes() {
        String claimKey = "https://orders-qa.baylorgenetics.com/institutionCodes";
        return claims.get(claimKey, List.class);
    }
    
    // Raw claims access
    public Claims getClaims() {
        return claims;
    }
    
    // Identity provider
    public String getIdentityProvider() {
        return identityProvider;
    }
}
```

---

#### ApiKeyServiceImpl
**Location:** `shared/cep-common/src/main/.../service/impl/ApiKeyServiceImpl.java`

**Purpose:** Validate and cache API keys for service-to-service auth.

**Key Methods:**
```java
@Service
public class ApiKeyServiceImpl implements ApiKeyService {
    
    @Autowired
    private ApiKeyRepository apiKeyRepository;
    
    @Autowired
    @Qualifier("apiKeyCacheManager")
    private CacheManager cacheManager;
    
    @Override
    public ApiKey validateKey(String keyValue, String targetService) {
        // Check cache first
        Cache cache = cacheManager.getCache("apiKeys");
        String cacheKey = keyValue + ":" + targetService;
        
        ApiKey cached = cache.get(cacheKey, ApiKey.class);
        if (cached != null) {
            return cached;
        }
        
        // Query database
        ApiKey apiKey = apiKeyRepository.findByApiKey(keyValue)
            .orElseThrow(() -> new InvalidApiKeyException("Invalid API key"));
        
        // Validate
        if (!apiKey.isActive()) {
            throw new InactiveApiKeyException("API key is inactive");
        }
        
        if (!apiKey.isAuthorizedForService(targetService)) {
            throw new UnauthorizedServiceException(
                "API key not authorized for service: " + targetService
            );
        }
        
        // Update last used timestamp
        apiKey.setLastUsedAt(LocalDateTime.now());
        apiKeyRepository.save(apiKey);
        
        // Cache result (60-minute TTL)
        cache.put(cacheKey, apiKey);
        
        return apiKey;
    }
    
    @Override
    @CacheEvict(cacheNames = "apiKeys", allEntries = true)
    public void evictAllApiKeyCache() {
        // Manual cache eviction (e.g., on key revocation)
    }
}
```

**Cache Configuration:**
```java
@Configuration
public class CacheConfig {
    
    @Bean("apiKeyCacheManager")
    public CacheManager apiKeyCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("apiKeys");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(60, TimeUnit.MINUTES)
            .recordStats());
        return cacheManager;
    }
}
```

---

#### InstitutionSecurityEvaluator
**Location:** `shared/cep-common/src/main/.../security/evaluator/InstitutionSecurityEvaluator.java`

**Purpose:** SpEL bean for institution-based authorization checks.

**Key Methods:**
```java
@Component("institutionSecurity")
public class InstitutionSecurityEvaluator {
    
    @Autowired
    private OrderRepository orderRepository;
    
    /**
     * Pre-authorization: Check before fetching order
     */
    public boolean canAccessOrder(Long orderId) {
        // Admin bypass
        if (SecurityUtil.hasPermission("read:all")) {
            return true;
        }
        
        // Fetch order
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found: " + orderId));
        
        // Check institution match
        List<String> userInstitutions = SecurityUtil.getInstitutionCodesFromToken();
        
        if (!userInstitutions.contains(order.getHospitalCode())) {
            throw new InvalidInstitutionCodeException(
                "User not authorized for institution: " + order.getHospitalCode()
            );
        }
        
        return true;
    }
    
    /**
     * Post-authorization: Check after returning response
     */
    public boolean canAccessOrderResponse(Object returnObject) {
        // Skip if not institution-aware
        if (!(returnObject instanceof InstitutionAware)) {
            return true;
        }
        
        // Admin bypass
        if (SecurityUtil.hasPermission("read:all")) {
            return true;
        }
        
        // Extract institution from response
        InstitutionAware dto = (InstitutionAware) returnObject;
        String institutionCode = dto.getInstitutionCode();
        
        // Check match
        List<String> userInstitutions = SecurityUtil.getInstitutionCodesFromToken();
        
        return userInstitutions.contains(institutionCode);
    }
}
```

---

### Configuration Components

#### BaseSecurityConfiguration
**Location:** `shared/cep-common/src/main/.../security/configuration/BaseSecurityConfiguration.java`

**Purpose:** Shared Spring Security configuration for all microservices.

**Key Configuration:**
```java
@Configuration
public class BaseSecurityConfiguration {
    
    @Bean
    public SecurityFilterChain filterChain(
        HttpSecurity http,
        JwtAuthenticationFilter jwtFilter
    ) throws Exception {
        
        http
            // Disable CSRF (stateless JWT auth)
            .csrf(csrf -> csrf.disable())
            
            // Stateless session management
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            
            // Authorization rules
            .authorizeHttpRequests(authz -> authz
                // Public endpoints
                .requestMatchers(Constants.PUBLIC_ENDPOINTS.toArray(new String[0]))
                    .permitAll()
                
                // Everything else requires authentication
                .anyRequest().authenticated()
            )
            
            // Add JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            
            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authenticationEntryPoint())
                .accessDeniedHandler(accessDeniedHandler())
            );
        
        return http.build();
    }
    
    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return (request, response, authException) -> {
            AuthenticationResponseHandler.handleAuthenticationException(
                response, 
                authException
            );
        };
    }
    
    @Bean
    public AccessDeniedHandler accessDeniedHandler() {
        return (request, response, accessDeniedException) -> {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.setContentType("application/json");
            
            ErrorResponse error = ErrorResponse.builder()
                .errorCode("AUTH_2001")
                .message("Access denied")
                .build();
            
            response.getWriter().write(objectMapper.writeValueAsString(error));
        };
    }
}
```

**Key Settings:**
- **CSRF:** Disabled (stateless authentication)
- **Session:** Stateless (no server-side sessions)
- **Public Endpoints:** Permit without authentication
- **Protected Endpoints:** Require authentication
- **JWT Filter:** Runs before standard auth filters

---

#### Auth0TenantConfiguration
**Location:** `shared/cep-common/src/main/.../security/configuration/Auth0TenantConfiguration.java`

**Purpose:** Configuration model for Auth0 tenants.

```java
@Configuration
@ConfigurationProperties(prefix = "auth0")
public class Auth0TenantConfiguration {
    
    private List<TenantConfig> tenants;
    
    @Data
    public static class TenantConfig {
        private String domain;          // e.g., "dev-example.auth0.com"
        private String audience;        // e.g., "https://api.example.com"
        private String issuer;          // e.g., "https://dev-example.auth0.com/"
    }
    
    // Getters/setters
}
```

**application.yml Example:**
```yaml
auth0:
  tenants:
    - domain: dev-tenant1.auth0.com
      audience: https://api.example.com
      issuer: https://dev-tenant1.auth0.com/
    - domain: dev-tenant2.auth0.com
      audience: https://api.example.com
      issuer: https://dev-tenant2.auth0.com/
```

---

#### AzureActiveDirectoryConfiguration
**Location:** `shared/cep-common/src/main/.../security/configuration/AzureActiveDirectoryConfiguration.java`

**Purpose:** Configuration model for Azure AD.

```java
@Configuration
@ConfigurationProperties(prefix = "azure.activedirectory")
public class AzureActiveDirectoryConfiguration {
    
    private String tenantId;
    private String clientId;
    private String audience;
    private String issuer;
    
    public String getJwksUri() {
        return String.format(
            "https://login.microsoftonline.com/%s/discovery/v2.0/keys",
            tenantId
        );
    }
    
    // Getters/setters
}
```

**application.yml Example:**
```yaml
azure:
  activedirectory:
    tenant-id: 12345678-1234-1234-1234-123456789abc
    client-id: 87654321-4321-4321-4321-987654321xyz
    audience: api://example-api
    issuer: https://login.microsoftonline.com/12345678-1234-1234-1234-123456789abc/v2.0
```

---

## 7. Security Configuration

### Public Endpoints

**Location:** `shared/cep-common/src/main/.../utils/Constants.java`

```java
public class Constants {
    
    /**
     * Endpoints that bypass authentication
     */
    public static final List<String> PUBLIC_ENDPOINTS = Arrays.asList(
        "/api/auth/**",           // Auth endpoints
        "/actuator/health",       // Health check
        "/actuator/info",         // Info endpoint
        "/swagger-ui/**",         // Swagger UI
        "/v3/api-docs/**",        // OpenAPI docs
        "/swagger-resources/**",  // Swagger resources
        "/webjars/**",            // Webjars for Swagger
        "/favicon.ico"            // Favicon
    );
}
```

**Note:** These endpoints skip `JwtAuthenticationFilter` entirely via `shouldNotFilter()`.

---

### CORS Configuration

**Location:** API Gateway service

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        config.setAllowedOrigins(Arrays.asList(
            "https://app.example.com",
            "http://localhost:3000"  // Dev
        ));
        
        config.setAllowedMethods(Arrays.asList(
            "GET", "POST", "PUT", "DELETE", "OPTIONS"
        ));
        
        config.setAllowedHeaders(Arrays.asList(
            "Authorization",
            "Content-Type",
            "X-Requested-With"
        ));
        
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = 
            new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsWebFilter(source);
    }
}
```

---

## 8. Error Handling

### Error Response Structure

All authentication/authorization errors return structured JSON:

```json
{
  "errorCode": "AUTH_1002",
  "message": "Token has expired",
  "timestamp": "2026-05-05T10:30:45.123Z",
  "path": "/api/orders/123"
}
```

---

### Authentication Error Codes

| Error Code | Meaning | HTTP Status | Cause |
|------------|---------|-------------|-------|
| **AUTH_1001** | Missing Authentication | 401 | No Authorization header |
| **AUTH_1002** | Token Expired | 401 | `exp` claim in past |
| **AUTH_1003** | Invalid Audience | 401 | `aud` claim mismatch |
| **AUTH_1004** | Token Not Yet Valid | 401 | `nbf` claim in future |
| **AUTH_1005** | Token Too Old | 401 | `iat` older than 24 hours |
| **AUTH_1006** | Missing Subject | 401 | No `sub` claim |
| **AUTH_1007** | Invalid Signature | 401 | Signature verification failed |
| **AUTH_1008** | Unknown Auth Error | 401 | Unexpected validation failure |

---

### Authorization Error Codes

| Error Code | Meaning | HTTP Status | Cause |
|------------|---------|-------------|-------|
| **AUTH_2001** | Access Denied | 403 | Generic authorization failure |
| **InvalidInstitutionCodeException** | Wrong Institution | 403 | User's institution doesn't match resource |
| **InsufficientPermissionsException** | Missing Permission | 403 | User lacks required permission |

---

### AuthenticationResponseHandler

**Location:** `shared/cep-common/src/main/.../security/authentication/AuthenticationResponseHandler.java`

```java
public class AuthenticationResponseHandler {
    
    public static void handleMissingAuthentication(HttpServletResponse response) 
        throws IOException {
        sendErrorResponse(response, HttpServletResponse.SC_UNAUTHORIZED, 
            "AUTH_1001", "Missing authentication token");
    }
    
    public static void handleAuthenticationException(
        HttpServletResponse response,
        AuthenticationException exception
    ) throws IOException {
        
        String errorCode = determineErrorCode(exception);
        String message = exception.getMessage();
        
        sendErrorResponse(response, HttpServletResponse.SC_UNAUTHORIZED, 
            errorCode, message);
    }
    
    private static String determineErrorCode(AuthenticationException exception) {
        if (exception instanceof TokenExpiredException) {
            return "AUTH_1002";
        } else if (exception instanceof InvalidAudienceException) {
            return "AUTH_1003";
        } else if (exception instanceof TokenNotYetValidException) {
            return "AUTH_1004";
        } else if (exception instanceof TokenTooOldException) {
            return "AUTH_1005";
        } else if (exception instanceof MissingSubjectException) {
            return "AUTH_1006";
        } else if (exception instanceof SigningKeyNotFoundException) {
            return "AUTH_1007";
        } else {
            return "AUTH_1008";
        }
    }
    
    private static void sendErrorResponse(
        HttpServletResponse response,
        int status,
        String errorCode,
        String message
    ) throws IOException {
        
        response.setStatus(status);
        response.setContentType("application/json");
        
        ErrorResponse error = ErrorResponse.builder()
            .errorCode(errorCode)
            .message(message)
            .timestamp(Instant.now())
            .path(getCurrentRequestPath())
            .build();
        
        ObjectMapper mapper = new ObjectMapper();
        response.getWriter().write(mapper.writeValueAsString(error));
    }
}
```

---

### GlobalExceptionHandler

**Location:** `shared/cep-common/src/main/.../exception/GlobalExceptionHandler.java`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    /**
     * Handle authorization exceptions
     */
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
        AccessDeniedException ex,
        HttpServletRequest request
    ) {
        ErrorResponse error = ErrorResponse.builder()
            .errorCode("AUTH_2001")
            .message("Access denied: " + ex.getMessage())
            .timestamp(Instant.now())
            .path(request.getRequestURI())
            .build();
        
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(error);
    }
    
    /**
     * Handle invalid institution code
     */
    @ExceptionHandler(InvalidInstitutionCodeException.class)
    public ResponseEntity<ErrorResponse> handleInvalidInstitution(
        InvalidInstitutionCodeException ex,
        HttpServletRequest request
    ) {
        ErrorResponse error = ErrorResponse.builder()
            .errorCode("INVALID_INSTITUTION")
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .path(request.getRequestURI())
            .build();
        
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(error);
    }
    
    /**
     * Handle insufficient permissions
     */
    @ExceptionHandler(InsufficientPermissionsException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientPermissions(
        InsufficientPermissionsException ex,
        HttpServletRequest request
    ) {
        ErrorResponse error = ErrorResponse.builder()
            .errorCode("INSUFFICIENT_PERMISSIONS")
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .path(request.getRequestURI())
            .build();
        
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(error);
    }
    
    /**
     * Handle all authentication exceptions
     */
    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthentication(
        AuthenticationException ex,
        HttpServletRequest request
    ) {
        String errorCode = determineAuthErrorCode(ex);
        
        ErrorResponse error = ErrorResponse.builder()
            .errorCode(errorCode)
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .path(request.getRequestURI())
            .build();
        
        return ResponseEntity
            .status(HttpStatus.UNAUTHORIZED)
            .body(error);
    }
}
```

---

## 9. Security Analysis & Risks

### Critical Risks

#### 1. Default Permission Fallback (High Severity)

**Issue:** Users with no permissions in JWT default to `update:orderSubmitter` (write access).

**Location:** `SecurityUtil.hasPermission()`

```java
public static boolean hasPermission(String permission) {
    List<String> permissions = getPermissionsFromToken();
    
    // DANGEROUS: Fail-open behavior
    if (permissions.isEmpty()) {
        return permission.equals("update:orderSubmitter");
    }
    
    return permissions.contains(permission);
}
```

**Impact:**
- Misconfigured users get write access by default
- Violates principle of least privilege
- Fail-open instead of fail-closed

**Recommendation:**
```java
public static boolean hasPermission(String permission) {
    List<String> permissions = getPermissionsFromToken();
    
    // FIX: Fail-closed behavior
    if (permissions.isEmpty()) {
        log.warn("User has no permissions configured");
        return false;  // Deny by default
    }
    
    return permissions.contains(permission);
}
```

---

#### 2. Institution Codes Not Verified Against Database (Medium Severity)

**Issue:** Institution codes are extracted directly from JWT claims without server-side verification.

**Location:** `SecurityUtil.getInstitutionCodesFromToken()`

```java
public static List<String> getInstitutionCodesFromToken() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    JwtAuthenticationDetails details = (JwtAuthenticationDetails) auth.getDetails();
    
    // Trusts JWT claim entirely
    return details.getInstitutionCodes();
}
```

**Impact:**
- If Auth0/Azure AD incorrectly populates institution codes, users access wrong data
- No cross-check against user database
- Trust boundary is entirely at IdP

**Recommendation:**
Add a database validation layer:
```java
public static List<String> getInstitutionCodesFromToken() {
    List<String> claimInstitutions = extractFromJwt();
    
    // Verify against database
    String userId = getCurrentUserId();
    List<String> dbInstitutions = userRepository.getInstitutionCodes(userId);
    
    // Return intersection (only valid ones)
    return claimInstitutions.stream()
        .filter(dbInstitutions::contains)
        .collect(Collectors.toList());
}
```

---

#### 3. Async JWT Validation Without Dedicated Thread Pool (Medium Severity)

**Issue:** `MultiTenantTokenValidator` uses `CompletableFuture.supplyAsync()` without specifying an executor.

**Location:** `MultiTenantTokenValidator.validateToken()`

```java
List<CompletableFuture<Authentication>> auth0Futures = 
    auth0Configurations.stream()
        .map(config -> CompletableFuture.supplyAsync(() -> 
            validateWithTenant(token, config)  // Runs on ForkJoinPool.commonPool()
        ))
        .collect(Collectors.toList());
```

**Impact:**
- Uses `ForkJoinPool.commonPool()` by default
- Under load, JWT validation can starve other async tasks
- I/O-bound work (JWKS fetching) on CPU-focused pool

**Recommendation:**
```java
@Configuration
public class AsyncConfig {
    @Bean("jwtValidationExecutor")
    public Executor jwtValidationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("jwt-validation-");
        executor.initialize();
        return executor;
    }
}

// Usage
@Autowired
@Qualifier("jwtValidationExecutor")
private Executor jwtExecutor;

List<CompletableFuture<Authentication>> auth0Futures = 
    auth0Configurations.stream()
        .map(config -> CompletableFuture.supplyAsync(() -> 
            validateWithTenant(token, config), jwtExecutor  // Dedicated pool
        ))
        .collect(Collectors.toList());
```

---

### Medium Risks

#### 4. API Key Cache TTL Too Long (Medium Severity)

**Issue:** API keys cached for 60 minutes. Revoked keys remain valid in memory.

**Location:** `ApiKeyServiceImpl` cache configuration

**Impact:**
- Revoked keys work for up to 60 minutes
- No event-driven cache eviction

**Recommendation:**
1. Reduce TTL to 5-10 minutes
2. Implement event-driven eviction:
```java
@Service
public class ApiKeyRevocationService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void revokeKey(Long keyId) {
        apiKeyRepository.deactivate(keyId);
        
        // Publish eviction event
        eventPublisher.publishEvent(new ApiKeyRevokedEvent(keyId));
    }
}

@Component
public class ApiKeyRevokedListener {
    
    @Autowired
    private ApiKeyServiceImpl apiKeyService;
    
    @EventListener
    public void onApiKeyRevoked(ApiKeyRevokedEvent event) {
        apiKeyService.evictAllApiKeyCache();
    }
}
```

---

#### 5. No JWKS Key Caching (Medium Severity)

**Issue:** Every token validation may fetch JWKS from Auth0/Azure AD.

**Location:** `JwtSigningKeyResolver.resolveSigningKey()`

**Impact:**
- Dependent on external IdP availability
- Increased latency per request
- Potential rate limiting from IdP

**Recommendation:**
```java
@Component
public class JwtSigningKeyResolver extends SigningKeyResolverAdapter {
    
    // Cache JWKS for 5 minutes
    private final Cache<String, JsonNode> jwksCache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .maximumSize(10)
        .build();
    
    @Override
    public Key resolveSigningKey(JwsHeader header, Claims claims) {
        String issuer = claims.getIssuer();
        String jwksUrl = buildJwksUrl(issuer);
        
        // Check cache first
        JsonNode jwks = jwksCache.get(jwksUrl, url -> fetchJwks(url));
        
        // Find key by kid
        String keyId = header.getKeyId();
        for (JsonNode key : jwks.get("keys")) {
            if (keyId.equals(key.get("kid").asText())) {
                return buildPublicKey(key);
            }
        }
        
        throw new SigningKeyNotFoundException(keyId);
    }
}
```

---

#### 6. @PostAuthorize Processes Query Before Denying Access (Low-Medium Severity)

**Issue:** Methods with `@PostAuthorize` execute fully before authorization check.

**Example:**
```java
@PostAuthorize("@institutionSecurity.canAccessOrderResponse(returnObject)")
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    return orderService.findById(id);  // DB query runs first
}
```

**Impact:**
- DB query executes even if access will be denied
- Processing cost incurred
- Audit trail shows access attempt

**Recommendation:**
Prefer `@PreAuthorize` where possible:
```java
@PreAuthorize("@institutionSecurity.canAccessOrder(#id)")
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    return orderService.findById(id);  // Only runs if authorized
}
```

---

### Low Risks

#### 7. Public Endpoints Hardcoded (Low Severity)

**Issue:** `Constants.PUBLIC_ENDPOINTS` is a compile-time constant.

**Impact:**
- Requires code change + redeployment to modify
- Not externally configurable

**Recommendation:**
Move to `application.yml`:
```yaml
security:
  public-endpoints:
    - /api/auth/**
    - /actuator/health
    - /swagger-ui/**
```

```java
@ConfigurationProperties(prefix = "security")
public class SecurityProperties {
    private List<String> publicEndpoints;
    // Getters/setters
}
```

---

#### 8. cep-kit-service Has Custom Security Config (Low Severity)

**Issue:** `cep-kit-service` doesn't extend `BaseSecurityConfiguration`, has manual config.

**Impact:**
- Changes to base config don't automatically apply
- Maintenance burden

**Recommendation:**
Refactor to extend base config and add exceptions:
```java
@Configuration
public class KitServiceSecurityConfig extends BaseSecurityConfiguration {
    
    @Override
    protected void configureAdditionalPublicEndpoints(HttpSecurity http) {
        http.authorizeHttpRequests(authz -> authz
            .requestMatchers("/webhooks/salesforce/**").permitAll()
            .requestMatchers("/webhooks/biotouch/**").permitAll()
        );
    }
}
```

---

## 10. Improvement Recommendations

### High Priority

1. **Fix Default Permission Fallback**
   - Change `SecurityUtil.hasPermission()` to deny-by-default
   - Users with no permissions should be blocked, not granted write access
   
2. **Add Dedicated Thread Pool for JWT Validation**
   - Create bounded `ExecutorService` for `CompletableFuture` operations
   - Prevent starvation of `ForkJoinPool.commonPool()`

3. **Implement JWKS Caching**
   - Cache JWKS responses locally (5-minute TTL)
   - Reduce dependency on external IdP availability
   - Improve request latency

4. **Reduce API Key Cache TTL**
   - Decrease from 60 minutes to 5-10 minutes
   - Implement event-driven cache eviction on key revocation

---

### Medium Priority

5. **Validate Institution Codes Against Database**
   - Cross-check JWT institution codes with user database
   - Add a validation layer after token authentication

6. **Convert @PostAuthorize to @PreAuthorize Where Possible**
   - Minimize processing before authorization denial
   - Reduce unnecessary DB queries

7. **Add Rate Limiting on Failed Auth Attempts**
   - Implement circuit breaker or rate limiter
   - Protect against brute-force token attempts

8. **Make Public Endpoints Configurable**
   - Move to `application.yml` from hardcoded constant
   - Enable runtime configuration changes

---

### Low Priority

9. **Standardize Security Config Across All Services**
   - Ensure all services extend `BaseSecurityConfiguration`
   - Remove custom security configs like in `cep-kit-service`

10. **Add Comprehensive Audit Logging**
    - Log all authentication attempts (success/failure)
    - Log authorization denials with user + resource context
    - Enable security incident investigation

11. **Implement Token Introspection Endpoint (Optional)**
    - Allow frontend to validate tokens before expiry
    - Provide token metadata without full auth flow

---

## 11. Quick Reference for Developers

### 10-Point Summary for New Developers

1. **Stateless JWT Authentication**
   - No server sessions
   - Every request carries a Bearer token
   - SecurityContext populated per-request and discarded

2. **Shared Auth Library**
   - All 9 microservices import `cep-common`
   - Auth logic centralized in one place
   - `BaseSecurityConfiguration` applied by all services

3. **Dual Identity Providers**
   - Auth0 (primary, multiple tenants)
   - Azure AD (fallback)
   - `MultiTenantTokenValidator` tries each in sequence

4. **Filter-Based Validation**
   - `JwtAuthenticationFilter` intercepts every request
   - Validates JWT cryptographically (JWKS + claims)
   - Sets principal before controller

5. **24-Hour Max Token Age**
   - Even if `exp` is far future, tokens >24h old rejected
   - Error code: `AUTH_1005`

6. **Two Authorization Axes**
   - **Permissions:** What actions can user perform? (e.g., `read:all`)
   - **Institution Codes:** Which institution's data can user access?

7. **SpEL-Based Authorization**
   - Controllers use `@PreAuthorize` with SpEL expressions
   - `InstitutionSecurityEvaluator` bean does DB lookup + JWT comparison

8. **API Key Auth for Services**
   - Internal services use `Authorization: api-key {secret}`
   - Keys stored in DB, cached 60 minutes

9. **Specific Error Codes**
   - Auth failures return JSON with codes `AUTH_1001` through `AUTH_1008`
   - Enables precise debugging

10. **No Refresh Token Logic**
    - Token refresh handled by frontend/IdP
    - Backend only validates tokens, never issues them

---

### Common Development Tasks

#### Add a New Public Endpoint
```java
// Constants.java
public static final List<String> PUBLIC_ENDPOINTS = Arrays.asList(
    "/api/auth/**",
    "/actuator/health",
    "/your-new-endpoint/**"  // Add here
);
```

#### Check User Permission in Code
```java
if (SecurityUtil.hasPermission("update:orderSubmitter")) {
    // User can submit orders
    orderService.submitOrder(orderId);
} else {
    throw new InsufficientPermissionsException("Cannot submit orders");
}
```

#### Add Institution-Based Authorization to Controller
```java
@PreAuthorize("@institutionSecurity.canAccessOrder(#orderId)")
@GetMapping("/orders/{orderId}")
public OrderDTO getOrder(@PathVariable Long orderId) {
    return orderService.findById(orderId);
}
```

#### Get Current User Information
```java
// Get username
String username = SecurityUtil.getCurrentUsername();

// Get permissions
List<String> permissions = SecurityUtil.getPermissionsFromToken();

// Get institution codes
List<String> institutions = SecurityUtil.getInstitutionCodesFromToken();

// Check if admin
boolean isAdmin = SecurityUtil.hasPermission("read:all");
```

#### Create New API Key (Service-to-Service)
```sql
-- Insert into database
INSERT INTO cep_core.api_keys (api_key, service_name, is_active, created_at)
VALUES ('your-secret-key-here', 'service-name', true, NOW());
```

```java
// Use in HTTP client
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "api-key your-secret-key-here");
```

---

### Testing Authentication Locally

#### 1. Get a Valid JWT Token
```bash
# From Auth0
curl --request POST \
  --url 'https://YOUR_DOMAIN.auth0.com/oauth/token' \
  --header 'content-type: application/json' \
  --data '{
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET",
    "audience": "https://api.example.com",
    "grant_type": "client_credentials"
  }'
```

#### 2. Use Token in Request
```bash
curl -H "Authorization: Bearer eyJhbGci..." \
  http://localhost:8080/api/orders/123
```

#### 3. Test Public Endpoint (No Token)
```bash
curl http://localhost:8080/actuator/health
# Should work without Authorization header
```

#### 4. Test Invalid Token
```bash
curl -H "Authorization: Bearer invalid-token" \
  http://localhost:8080/api/orders/123
# Should return 401 with error code
```

---

### Troubleshooting Common Issues

#### Issue: "Missing authentication token" (AUTH_1001)
**Cause:** No `Authorization` header sent

**Fix:** Add header: `Authorization: Bearer {your-token}`

---

#### Issue: "Token has expired" (AUTH_1002)
**Cause:** Token `exp` claim is in the past

**Fix:** Generate a new token from Auth0/Azure AD

---

#### Issue: "Invalid audience" (AUTH_1003)
**Cause:** Token `aud` claim doesn't match configured audience

**Fix:** Check `application.yml` audience configuration matches token audience

---

#### Issue: "Access denied" (403)
**Cause:** User authenticated but lacks permission or institution access

**Fix:** 
1. Check user's permissions in JWT
2. Check user's institution codes
3. Verify required permission in `@PreAuthorize`

---

#### Issue: API key authentication not working
**Cause:** 
- Key not in database
- Key inactive
- Service name mismatch
- Cache contains old data

**Fix:**
1. Verify key exists: `SELECT * FROM cep_core.api_keys WHERE api_key = 'your-key';`
2. Check `is_active = true`
3. Check `service_name` matches
4. Clear cache: Call `ApiKeyServiceImpl.evictAllApiKeyCache()`

---

### File Structure Reference

```
shared/cep-common/
└── src/main/java/.../
    ├── security/
    │   ├── authentication/
    │   │   ├── JwtAuthenticationFilter.java          # Entry point
    │   │   ├── MultiTenantTokenValidator.java        # Multi-IdP validation
    │   │   ├── JwtClaimsValidator.java               # Token validation logic
    │   │   ├── JwtAuthenticationDetails.java         # Claims storage
    │   │   ├── ApiKeyAuthenticationToken.java        # API key auth
    │   │   └── AuthenticationResponseHandler.java    # Error responses
    │   ├── configuration/
    │   │   ├── BaseSecurityConfiguration.java        # Shared security config
    │   │   ├── Auth0TenantConfiguration.java         # Auth0 config
    │   │   └── AzureActiveDirectoryConfiguration.java # Azure AD config
    │   ├── resolver/
    │   │   └── JwtSigningKeyResolver.java            # JWKS fetcher
    │   ├── evaluator/
    │   │   └── InstitutionSecurityEvaluator.java     # SpEL authorization
    │   └── utils/
    │       ├── SecurityUtil.java                      # Auth helpers
    │       └── Constants.java                         # Public endpoints
    ├── service/
    │   └── impl/
    │       └── ApiKeyServiceImpl.java                # API key validation
    ├── entity/
    │   └── ApiKey.java                               # API key entity
    └── exception/
        └── GlobalExceptionHandler.java               # Error handling
```

---

### Important Configuration Files

#### application.yml (per service)
```yaml
# Auth0 configuration
auth0:
  tenants:
    - domain: dev-tenant1.auth0.com
      audience: https://api.example.com
      issuer: https://dev-tenant1.auth0.com/

# Azure AD configuration
azure:
  activedirectory:
    tenant-id: 12345678-1234-1234-1234-123456789abc
    client-id: 87654321-4321-4321-4321-987654321xyz
    audience: api://example-api

# Security
security:
  public-endpoints:
    - /api/auth/**
    - /actuator/health
```

---

## Appendix: Glossary

| Term | Definition |
|------|------------|
| **Authentication** | Verifying identity (who you are) |
| **Authorization** | Verifying permissions (what you can do) |
| **JWT** | JSON Web Token - secure token format |
| **Bearer Token** | Access token carried in Authorization header |
| **Claims** | Information asserted about a subject in JWT |
| **JWKS** | JSON Web Key Set - public keys for JWT verification |
| **IdP** | Identity Provider (Auth0, Azure AD) |
| **SpEL** | Spring Expression Language |
| **CSRF** | Cross-Site Request Forgery |
| **CORS** | Cross-Origin Resource Sharing |
| **TTL** | Time To Live (cache expiration) |
| **Stateless** | No server-side session storage |

---

## Document Version

- **Version:** 1.0
- **Date:** May 5, 2026
- **Author:** Claude AI
- **Based On:** CEP Online Portal V2 codebase analysis
- **Last Updated:** May 5, 2026

---

**End of Documentation**
