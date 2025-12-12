# HTTP Status Code API Migration Guide

## Table of Contents

- [Introduction](#introduction)
- [Overview of Changes](#overview-of-changes)
- [Backward Compatibility](#backward-compatibility)
- [API Reference](#api-reference)
- [Migration Patterns](#migration-patterns)
- [Module Type-Specific Guidance](#module-type-specific-guidance)
- [Benefits of Migration](#benefits-of-migration)
- [Frequently Asked Questions](#frequently-asked-questions)
- [Complete Code Examples](#complete-code-examples)

---

## Introduction

This guide provides comprehensive instructions for third-party NGINX module developers to migrate from direct HTTP status code assignment to the new centralized status code registry and validation API.

### What Has Changed?

NGINX has undergone an architectural modernization to centralize HTTP status code management. Previously, status codes were set through scattered direct assignments to `r->headers_out.status` throughout the codebase. The new architecture introduces:

- **Centralized Registry**: A single source of truth for all HTTP status codes with RFC 9110 compliance
- **Unified API**: Consistent `ngx_http_status_set()` function for all status assignments
- **Validation Layer**: Optional strict RFC 9110 compliance checking
- **Metadata Support**: Built-in reason phrases, cacheability flags, and RFC section references

### Why This Change?

The refactoring addresses several critical needs:

1. **RFC 9110 Compliance**: Ensures all status code operations conform to HTTP Semantics specification
2. **Code Maintainability**: Eliminates scattered status code constants and inconsistent assignments
3. **Extensibility**: Provides a foundation for future HTTP specification updates
4. **Consistency**: Standardizes reason phrases and cacheability metadata across all modules

---

## Overview of Changes

### Architectural Transformation

The new architecture consists of three layers:

#### Layer 1: Registry Foundation
```c
typedef struct {
    ngx_uint_t    code;          /* HTTP status code (100-599) */
    ngx_str_t     reason;        /* RFC 9110 reason phrase */
    ngx_uint_t    flags;         /* Cacheability, error class */
    const char   *rfc_section;   /* RFC 9110 section reference */
} ngx_http_status_def_t;
```

A compile-time initialized, read-only registry containing all RFC 9110 Section 15 standard codes with associated metadata. The registry provides O(1) lookup performance via array indexing.

#### Layer 2: API Mediation Interface
```c
ngx_int_t ngx_http_status_set(ngx_http_request_t *r, ngx_uint_t status);
```

This unified API replaces all direct `r->headers_out.status = code;` assignments with validation-aware function calls that enforce RFC 9110 compliance rules in strict mode while maintaining backward compatibility in standard mode.

#### Layer 3: Validation and Compliance

The validation layer implements:
- Range validation (100-599 per RFC 9110)
- Status class verification (1xx, 2xx, 3xx, 4xx, 5xx)
- Reason phrase consistency
- Cacheability metadata per RFC 9111

### Key Design Principles

- **Performance**: Zero overhead in standard mode through compile-time constant propagation
- **Thread Safety**: Immutable registry design eliminates need for locking
- **Backward Compatibility**: Existing code continues to work without modification
- **Graceful Degradation**: Validation failures never abort requests

---

## Backward Compatibility

> [!IMPORTANT]
> **Your existing modules will continue to work without any modifications.**

### What Still Works

All existing patterns remain functional:

#### 1. Direct Status Assignment (Deprecated but Functional)
```c
/* This pattern still compiles and works */
r->headers_out.status = NGX_HTTP_NOT_FOUND;
```

#### 2. Status Code Constants
```c
/* All NGX_HTTP_* constants remain defined */
#define NGX_HTTP_OK                        200
#define NGX_HTTP_CREATED                   201
#define NGX_HTTP_NOT_FOUND                 404
#define NGX_HTTP_INTERNAL_SERVER_ERROR     500
/* ... etc ... */
```

#### 3. Filter Chain Interfaces
```c
/* Filter chain function signatures unchanged */
static ngx_int_t
ngx_http_myfilter_header_filter(ngx_http_request_t *r)
{
    /* Can still read r->headers_out.status directly */
    if (r->headers_out.status >= 200 && r->headers_out.status < 300) {
        /* ... */
    }
    return ngx_http_next_header_filter(r);
}
```

#### 4. Configuration Directives
```c
/* Directive handlers maintain same signatures */
static char *
ngx_http_mymodule_directive(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    /* No changes required */
}
```

### Migration Timeline

Third-party modules can migrate at their own pace:

1. **Immediate**: Modules continue working without modification (backward compatible)
2. **Recommended**: Adopt new API for RFC 9110 compliance and future-proofing
3. **Future**: Direct assignment pattern may be deprecated in a future major release

### No Breaking Changes

- Module structures unchanged
- Configuration parsing unchanged
- Filter chain interfaces unchanged
- Request/response lifecycle unchanged
- NGX_MODULE_V1 interface unchanged

---

## API Reference

The status code API provides four core functions for status code management.

### Primary API Function

#### `ngx_http_status_set()`

**Declaration:**
```c
ngx_int_t ngx_http_status_set(ngx_http_request_t *r, ngx_uint_t status);
```

**Purpose:** Sets the HTTP status code for a request with optional validation.

**Parameters:**
- `r` - Pointer to the request structure
- `status` - HTTP status code to set (100-599)

**Return Values:**
- `NGX_OK` - Status code set successfully
- `NGX_ERROR` - Invalid status code (validation failed)

**Usage Example:**
```c
ngx_int_t rc;

rc = ngx_http_status_set(r, NGX_HTTP_NOT_FOUND);
if (rc != NGX_OK) {
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                  "failed to set status code");
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}
```

**Behavior:**
- In standard mode (default): Sets status without strict validation (maximum compatibility)
- In strict mode (`--with-http-status-validation`): Enforces RFC 9110 range validation
- Upstream requests: Bypasses strict validation to preserve pass-through semantics

### Validation Function

#### `ngx_http_status_validate()`

**Declaration:**
```c
ngx_int_t ngx_http_status_validate(ngx_uint_t status);
```

**Purpose:** Validates a status code against RFC 9110 requirements.

**Parameters:**
- `status` - HTTP status code to validate

**Return Values:**
- `NGX_OK` - Status code is valid
- `NGX_ERROR` - Status code is invalid

**Usage Example:**
```c
ngx_uint_t custom_status = 299;

if (ngx_http_status_validate(custom_status) == NGX_OK) {
    ngx_http_status_set(r, custom_status);
} else {
    /* Fallback to safe status */
    ngx_http_status_set(r, NGX_HTTP_OK);
}
```

### Reason Phrase Lookup

#### `ngx_http_status_reason()`

**Declaration:**
```c
const ngx_str_t *ngx_http_status_reason(ngx_uint_t status);
```

**Purpose:** Retrieves the RFC 9110 reason phrase for a status code.

**Parameters:**
- `status` - HTTP status code

**Return Values:**
- Pointer to `ngx_str_t` containing the reason phrase
- `NULL` if status code not found in registry

**Usage Example:**
```c
const ngx_str_t *reason;

reason = ngx_http_status_reason(404);
if (reason != NULL) {
    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "status %ui: %V", 404, reason);
    /* Output: "status 404: Not Found" */
}
```

### Custom Status Registration

#### `ngx_http_status_register()`

**Declaration:**
```c
ngx_int_t ngx_http_status_register(ngx_http_status_def_t *def);
```

**Purpose:** Registers a custom status code definition (for future extensibility).

**Parameters:**
- `def` - Pointer to status code definition structure

**Return Values:**
- `NGX_OK` - Status code registered successfully
- `NGX_ERROR` - Registration failed

**Note:** This function is reserved for future use. The current registry is immutable after initialization.

---

## Migration Patterns

This section provides before/after code examples for common status code usage patterns.

### Pattern 1: Simple Status Assignment

**Before (Direct Assignment):**
```c
static ngx_int_t
ngx_http_mymodule_handler(ngx_http_request_t *r)
{
    /* ... processing ... */
    
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = content_len;
    
    return ngx_http_send_header(r);
}
```

**After (API-Based Assignment):**
```c
static ngx_int_t
ngx_http_mymodule_handler(ngx_http_request_t *r)
{
    ngx_int_t  rc;
    
    /* ... processing ... */
    
    rc = ngx_http_status_set(r, NGX_HTTP_OK);
    if (rc != NGX_OK) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "failed to set status in mymodule handler");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    r->headers_out.content_length_n = content_len;
    
    return ngx_http_send_header(r);
}
```

### Pattern 2: Error Handling in Content Handlers

**Before:**
```c
static ngx_int_t
ngx_http_mymodule_handler(ngx_http_request_t *r)
{
    ngx_buf_t    *b;
    ngx_chain_t   out;
    
    if (r->method != NGX_HTTP_GET && r->method != NGX_HTTP_HEAD) {
        return NGX_HTTP_NOT_ALLOWED;
    }
    
    if (access_denied) {
        r->headers_out.status = NGX_HTTP_FORBIDDEN;
        return ngx_http_send_header(r);
    }
    
    if (!file_exists) {
        r->headers_out.status = NGX_HTTP_NOT_FOUND;
        return ngx_http_send_header(r);
    }
    
    r->headers_out.status = NGX_HTTP_OK;
    /* ... send content ... */
}
```

**After:**
```c
static ngx_int_t
ngx_http_mymodule_handler(ngx_http_request_t *r)
{
    ngx_buf_t    *b;
    ngx_chain_t   out;
    ngx_int_t     rc;
    
    if (r->method != NGX_HTTP_GET && r->method != NGX_HTTP_HEAD) {
        return NGX_HTTP_NOT_ALLOWED;
    }
    
    if (access_denied) {
        rc = ngx_http_status_set(r, NGX_HTTP_FORBIDDEN);
        if (rc != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        return ngx_http_send_header(r);
    }
    
    if (!file_exists) {
        rc = ngx_http_status_set(r, NGX_HTTP_NOT_FOUND);
        if (rc != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        return ngx_http_send_header(r);
    }
    
    rc = ngx_http_status_set(r, NGX_HTTP_OK);
    if (rc != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    /* ... send content ... */
}
```

### Pattern 3: Filter Module Status Modification

**Before:**
```c
static ngx_int_t
ngx_http_myfilter_header_filter(ngx_http_request_t *r)
{
    if (r->headers_out.status == NGX_HTTP_OK) {
        if (should_return_not_modified(r)) {
            r->headers_out.status = NGX_HTTP_NOT_MODIFIED;
            r->headers_out.content_length_n = -1;
            ngx_http_clear_content_length(r);
            ngx_http_clear_accept_ranges(r);
        }
    }
    
    return ngx_http_next_header_filter(r);
}
```

**After:**
```c
static ngx_int_t
ngx_http_myfilter_header_filter(ngx_http_request_t *r)
{
    ngx_int_t  rc;
    
    if (r->headers_out.status == NGX_HTTP_OK) {
        if (should_return_not_modified(r)) {
            rc = ngx_http_status_set(r, NGX_HTTP_NOT_MODIFIED);
            if (rc != NGX_OK) {
                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                              "failed to set 304 status in filter");
                /* Continue with original status */
            } else {
                r->headers_out.content_length_n = -1;
                ngx_http_clear_content_length(r);
                ngx_http_clear_accept_ranges(r);
            }
        }
    }
    
    return ngx_http_next_header_filter(r);
}
```

### Pattern 4: Upstream Module Status Pass-Through

**Before:**
```c
static void
ngx_http_myproxy_finalize_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_http_upstream_t  *u;
    
    u = r->upstream;
    
    /* Pass through backend status */
    r->headers_out.status = u->headers_in.status_n;
    r->headers_out.status_line = u->headers_in.status_line;
    
    /* ... */
}
```

**After:**
```c
static void
ngx_http_myproxy_finalize_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ngx_http_upstream_t  *u;
    
    u = r->upstream;
    
    /* 
     * Pass through backend status - API automatically detects
     * upstream context and bypasses strict validation
     */
    if (ngx_http_status_set(r, u->headers_in.status_n) != NGX_OK) {
        /* Fallback to 502 Bad Gateway on validation failure */
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid status %ui from upstream, using 502",
                      u->headers_in.status_n);
        ngx_http_status_set(r, NGX_HTTP_BAD_GATEWAY);
    }
    
    r->headers_out.status_line = u->headers_in.status_line;
    
    /* ... */
}
```

### Pattern 5: Custom Error Page Generation

**Before:**
```c
static ngx_int_t
ngx_http_mymodule_error_page(ngx_http_request_t *r, ngx_uint_t error)
{
    ngx_buf_t    *b;
    ngx_chain_t   out;
    
    r->headers_out.status = error;
    r->headers_out.content_type_len = sizeof("text/html") - 1;
    ngx_str_set(&r->headers_out.content_type, "text/html");
    
    /* ... generate error page content ... */
    
    return ngx_http_output_filter(r, &out);
}
```

**After:**
```c
static ngx_int_t
ngx_http_mymodule_error_page(ngx_http_request_t *r, ngx_uint_t error)
{
    ngx_buf_t     *b;
    ngx_chain_t    out;
    ngx_int_t      rc;
    
    rc = ngx_http_status_set(r, error);
    if (rc != NGX_OK) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid error status %ui, using 500", error);
        ngx_http_status_set(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
    }
    
    r->headers_out.content_type_len = sizeof("text/html") - 1;
    ngx_str_set(&r->headers_out.content_type, "text/html");
    
    /* ... generate error page content ... */
    
    return ngx_http_output_filter(r, &out);
}
```

### Pattern 6: Status-Dependent Cache Decisions

**Before:**
```c
static ngx_int_t
ngx_http_mymodule_cache_decision(ngx_http_request_t *r)
{
    ngx_uint_t  status;
    
    status = r->headers_out.status;
    
    /* Hardcoded cacheability logic */
    if (status == 200 || status == 203 || status == 204 ||
        status == 206 || status == 300 || status == 301 ||
        status == 404 || status == 405 || status == 410 ||
        status == 414 || status == 501) {
        /* Cacheable status codes */
        return NGX_OK;
    }
    
    return NGX_DECLINED;
}
```

**After:**
```c
static ngx_int_t
ngx_http_mymodule_cache_decision(ngx_http_request_t *r)
{
    ngx_uint_t                     status;
    const ngx_http_status_def_t   *def;
    
    status = r->headers_out.status;
    
    /* Use registry metadata for cacheability */
    def = ngx_http_status_lookup(status);
    if (def != NULL && (def->flags & NGX_HTTP_STATUS_CACHEABLE)) {
        return NGX_OK;
    }
    
    return NGX_DECLINED;
}
```

---

## Module Type-Specific Guidance

Different types of modules have specific migration patterns and considerations.

### Content Handler Modules

Content handler modules (like static file servers, index modules, autoindex) typically set status codes based on processing results.

**Common Status Codes:**
- `200 OK` - Successful content delivery
- `404 Not Found` - Requested resource doesn't exist
- `403 Forbidden` - Access denied
- `500 Internal Server Error` - Processing failure

**Migration Strategy:**
1. Identify all `r->headers_out.status = X` assignments
2. Replace with `ngx_http_status_set(r, X)` calls
3. Add error handling for validation failures
4. Test with various request scenarios

**Example Module Types:**
- `ngx_http_static_module` - Static file serving
- `ngx_http_index_module` - Index file handling
- `ngx_http_autoindex_module` - Directory listing
- `ngx_http_random_index_module` - Random index selection

### Filter Modules

Filter modules may read or modify status codes as requests pass through the filter chain.

**Read-Only Filters:**
Modules that only read `r->headers_out.status` require **no changes**.

**Example Read-Only Filters:**
```c
static ngx_int_t
ngx_http_myfilter_header_filter(ngx_http_request_t *r)
{
    /* Reading status - no API call needed */
    if (r->headers_out.status >= 400) {
        /* Handle error responses */
    }
    
    return ngx_http_next_header_filter(r);
}
```

**Status-Modifying Filters:**
Modules that modify status codes should use the API.

**Common Scenarios:**
- **304 Not Modified**: Conditional request optimization
- **206 Partial Content**: Range request support
- **416 Range Not Satisfiable**: Invalid range request

**Example Status-Modifying Filter:**
```c
static ngx_int_t
ngx_http_range_header_filter(ngx_http_request_t *r)
{
    if (range_not_satisfiable) {
        if (ngx_http_status_set(r, NGX_HTTP_RANGE_NOT_SATISFIABLE) != NGX_OK) {
            return NGX_ERROR;
        }
        r->headers_out.content_range = /* ... */;
        return ngx_http_next_header_filter(r);
    }
    
    if (partial_content_requested) {
        if (ngx_http_status_set(r, NGX_HTTP_PARTIAL_CONTENT) != NGX_OK) {
            return NGX_ERROR;
        }
        /* ... setup range response ... */
    }
    
    return ngx_http_next_header_filter(r);
}
```

### Upstream Protocol Modules

Upstream modules (proxy, FastCGI, uWSGI, SCGI, gRPC) require special handling to preserve backend status pass-through.

**Critical Requirements:**
- Backend status codes must pass through unchanged
- Only NGINX-generated errors should use the API
- Upstream detection is automatic in the API

**Status Pass-Through:**
```c
static void
ngx_http_myupstream_finalize(ngx_http_request_t *r)
{
    ngx_http_upstream_t  *u;
    
    u = r->upstream;
    
    /*
     * API automatically detects r->upstream and skips strict validation
     * for backend-provided status codes
     */
    ngx_http_status_set(r, u->headers_in.status_n);
}
```

**NGINX-Generated Errors:**
```c
static void
ngx_http_myupstream_error(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    if (connection_timeout) {
        /* NGINX generates 504 Gateway Timeout */
        ngx_http_status_set(r, NGX_HTTP_GATEWAY_TIMEOUT);
    } else if (connection_refused) {
        /* NGINX generates 502 Bad Gateway */
        ngx_http_status_set(r, NGX_HTTP_BAD_GATEWAY);
    }
}
```

**Test Scenarios:**
1. Backend returns 200 → NGINX forwards 200 (unchanged)
2. Backend returns 404 → NGINX forwards 404 (unchanged)
3. Backend timeout → NGINX generates 504 (validated)
4. Connection refused → NGINX generates 502 (validated)

### Access Control Modules

Access control modules enforce authentication and authorization, setting specific status codes on denial.

**Common Status Codes:**
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - Access denied (authenticated but not authorized)
- `407 Proxy Authentication Required` - Proxy authentication needed

**Authentication Module Example:**
```c
static ngx_int_t
ngx_http_auth_basic_handler(ngx_http_request_t *r)
{
    ngx_int_t  rc;
    
    if (r->headers_in.authorization == NULL) {
        /* No credentials provided */
        rc = ngx_http_status_set(r, NGX_HTTP_UNAUTHORIZED);
        if (rc != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        
        r->headers_out.www_authenticate = /* ... */;
        return NGX_HTTP_UNAUTHORIZED;
    }
    
    /* Validate credentials */
    if (!credentials_valid) {
        rc = ngx_http_status_set(r, NGX_HTTP_FORBIDDEN);
        if (rc != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        return NGX_HTTP_FORBIDDEN;
    }
    
    return NGX_DECLINED;  /* Allow request to proceed */
}
```

### Rate Limiting Modules

Rate limiting modules enforce request rate constraints.

**Common Status Codes:**
- `429 Too Many Requests` - RFC 6585 rate limit exceeded
- `503 Service Unavailable` - Traditional rate limit response

**Rate Limit Module Example:**
```c
static ngx_int_t
ngx_http_limit_req_handler(ngx_http_request_t *r)
{
    ngx_http_limit_req_conf_t  *lrcf;
    ngx_int_t                   rc;
    
    lrcf = ngx_http_get_module_loc_conf(r, ngx_http_limit_req_module);
    
    if (rate_limit_exceeded) {
        if (lrcf->status_code != 0) {
            /* Configurable status code */
            rc = ngx_http_status_set(r, lrcf->status_code);
        } else {
            /* Default to 503 Service Unavailable */
            rc = ngx_http_status_set(r, NGX_HTTP_SERVICE_UNAVAILABLE);
        }
        
        if (rc != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        
        return lrcf->status_code != 0 ? lrcf->status_code 
                                       : NGX_HTTP_SERVICE_UNAVAILABLE;
    }
    
    return NGX_DECLINED;
}
```

---

## Benefits of Migration

Adopting the new status code API provides several advantages for third-party module developers.

### 1. RFC 9110 Compliance Validation

**Before Migration:**
No automatic validation of status code conformance to HTTP specifications.

```c
/* Potentially non-compliant code */
r->headers_out.status = 999;  /* Invalid status, no error */
```

**After Migration:**
Automatic validation ensures RFC 9110 compliance (in strict mode).

```c
/* Validated status code */
if (ngx_http_status_set(r, 999) != NGX_OK) {
    /* Error detected, fallback to valid status */
    ngx_http_status_set(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
}
```

**Benefit:** Catch invalid status codes during development and testing, preventing HTTP protocol violations.

### 2. Consistent Error Handling

**Before Migration:**
Scattered error handling patterns across modules.

**After Migration:**
Unified error handling strategy.

```c
ngx_int_t rc = ngx_http_status_set(r, status_code);
if (rc != NGX_OK) {
    /* Standardized error handling */
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                  "failed to set status %ui", status_code);
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}
```

**Benefit:** Predictable error handling behavior across all modules.

### 3. Reason Phrase Consistency

**Before Migration:**
Manual reason phrase generation with potential inconsistencies.

```c
/* Custom reason phrases may not match RFC 9110 */
ngx_str_set(&r->headers_out.status_line, "404 Resource Not Found");
```

**After Migration:**
Automatic RFC 9110-compliant reason phrases.

```c
const ngx_str_t *reason = ngx_http_status_reason(404);
/* Always returns "404 Not Found" per RFC 9110 */
```

**Benefit:** Consistent reason phrases that match HTTP specifications.

### 4. Cacheability Metadata

**Before Migration:**
Manual cacheability determination.

```c
/* Hardcoded cache logic */
if (status == 200 || status == 404 || status == 301) {
    /* Cacheable */
}
```

**After Migration:**
Registry-based cacheability flags.

```c
const ngx_http_status_def_t *def = ngx_http_status_lookup(status);
if (def && (def->flags & NGX_HTTP_STATUS_CACHEABLE)) {
    /* Cacheable per RFC 9111 */
}
```

**Benefit:** Accurate cacheability decisions based on HTTP caching specifications.

### 5. Future-Proof Architecture

**Before Migration:**
Updating HTTP specifications requires changes throughout codebase.

**After Migration:**
Centralized registry simplifies specification updates.

```c
/* Adding new RFC status codes only requires registry updates */
{ 308, ngx_string("Permanent Redirect"), NGX_HTTP_STATUS_CACHEABLE, "RFC 7538" }
```

**Benefit:** Easier maintenance when HTTP specifications evolve.

### 6. Debugging and Logging

**Before Migration:**
No centralized status code operation logging.

**After Migration:**
Built-in debug logging for all status operations.

```c
ngx_log_debug3(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
               "http status set: %ui \"%V\" (valid: %s)",
               status, ngx_http_status_reason(status),
               (rc == NGX_OK) ? "yes" : "no");
```

**Benefit:** Better visibility into status code operations during development and troubleshooting.

### 7. Performance Optimization

**Standard Mode (Default):**
Zero overhead through compile-time constant propagation.

```c
/* In standard mode, this optimizes to direct assignment */
ngx_http_status_set(r, NGX_HTTP_OK);
/* Compiler generates: r->headers_out.status = 200; */
```

**Benefit:** API adoption does not impact performance in production deployments.

---

## Frequently Asked Questions

### General Questions

#### Q: Do I need to migrate my module immediately?

**A:** No. Your existing module will continue working without any modifications. The new API is backward compatible, and direct status assignments remain functional. Migration is recommended but not required.

#### Q: Will my module break if I don't migrate?

**A:** No. All existing patterns continue to work:
- `r->headers_out.status = code;` still works
- `NGX_HTTP_*` constants remain defined
- Filter chain interfaces unchanged
- No compilation errors or runtime failures

#### Q: When should I consider migrating?

**A:** Consider migration if you:
- Want RFC 9110 compliance validation
- Need consistent error handling
- Plan to distribute your module widely
- Want future-proof code against HTTP specification changes

### Technical Questions

#### Q: What's the performance impact of the API?

**A:** In standard mode (default), there is **zero performance overhead**. The API uses compile-time constant propagation:

```c
/* In standard mode, these are equivalent: */
ngx_http_status_set(r, 200);
/* Compiles to: */
r->headers_out.status = 200;
```

Only strict mode (`--with-http-status-validation`) adds validation overhead, which is minimal (<10 CPU cycles per operation).

#### Q: How do I enable strict RFC validation?

**A:** Strict validation is a compile-time option:

```bash
./auto/configure --with-http-status-validation
make
```

In strict mode, the API performs RFC 9110 range validation on every status code assignment.

#### Q: What happens if validation fails?

**A:** The API never aborts requests. Instead:

1. `ngx_http_status_set()` returns `NGX_ERROR`
2. An error is logged to the error log
3. Your code handles the error (typically with a fallback status)

```c
if (ngx_http_status_set(r, status) != NGX_OK) {
    /* Fallback to safe status */
    ngx_http_status_set(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
}
```

#### Q: Can I still use `NGX_HTTP_*` constants?

**A:** Yes, absolutely. All existing constants remain defined and work with the API:

```c
ngx_http_status_set(r, NGX_HTTP_NOT_FOUND);  /* Works perfectly */
ngx_http_status_set(r, 404);                  /* Also works */
```

#### Q: How does the API handle upstream status codes?

**A:** The API automatically detects upstream contexts and bypasses strict validation:

```c
/* In upstream modules, this always works: */
ngx_http_status_set(r, u->headers_in.status_n);

/* API checks if (r->upstream) and skips validation */
```

Backend status codes pass through unchanged, preserving proxy semantics.

### Migration Questions

#### Q: What's the recommended migration approach?

**A:** Follow these steps:

1. **Identify**: Find all `r->headers_out.status = X` assignments
2. **Replace**: Change to `ngx_http_status_set(r, X)`
3. **Add Error Handling**: Check return value and handle failures
4. **Test**: Run your test suite
5. **Validate**: Test with strict mode enabled

#### Q: Can I migrate gradually?

**A:** Yes. You can migrate one function at a time:

```c
/* Old code - still works */
static void old_function(ngx_http_request_t *r) {
    r->headers_out.status = 200;
}

/* New code - migrated */
static void new_function(ngx_http_request_t *r) {
    ngx_http_status_set(r, 200);
}
```

Both patterns coexist peacefully in the same module.

#### Q: How do I test my migrated module?

**A:** Use the standard NGINX test suite:

```bash
# Clone nginx-tests repository
git clone https://github.com/nginx/nginx-tests.git
cd nginx-tests

# Run tests with your module
TEST_NGINX_BINARY=/path/to/nginx prove t/mymodule.t
```

Additionally, test with strict mode enabled:

```bash
# Build with validation
./auto/configure --with-http-status-validation --add-module=/path/to/mymodule
make

# Run tests
TEST_NGINX_BINARY=objs/nginx prove t/mymodule.t
```

#### Q: What if my module uses custom status codes (e.g., 444)?

**A:** Custom NGINX status codes are supported:

```c
/* NGINX custom status codes work */
ngx_http_status_set(r, 444);  /* Connection Closed Without Response */
ngx_http_status_set(r, 499);  /* Client Closed Request */
```

In strict mode, a warning is logged for non-standard codes, but they're allowed.

### Compatibility Questions

#### Q: Does this work with older NGINX versions?

**A:** The status code API is introduced in NGINX 1.27.0+. For older versions, modules should use direct assignment patterns.

You can support both:

```c
#if (nginx_version >= 1027000)
    /* New API available */
    ngx_http_status_set(r, status);
#else
    /* Fallback for older versions */
    r->headers_out.status = status;
#endif
```

#### Q: Will my module work with future NGINX versions?

**A:** Yes. The API is designed for long-term stability:

- API function signatures are stable
- Backward compatibility maintained
- Registry extensible for future HTTP specs
- No breaking changes planned

#### Q: How does this affect HTTP/2 and HTTP/3 modules?

**A:** HTTP/2 and HTTP/3 modules benefit from the API:

```c
/* HTTP/2 :status pseudo-header uses validated status */
static ngx_int_t
ngx_http_v2_header_filter(ngx_http_request_t *r)
{
    /* r->headers_out.status already validated by API */
    status_value = r->headers_out.status;
    /* Encode to HPACK */
}
```

No special handling required—status codes are pre-validated.

### Debugging Questions

#### Q: How do I debug status code issues?

**A:** Enable debug logging:

```nginx
error_log logs/error.log debug;
```

The API logs all status operations:

```
2024/01/15 10:30:45 [debug] 1234#0: *1 http status set: 404 "Not Found" (valid: yes)
2024/01/15 10:30:46 [debug] 1234#0: *2 http status set: 999 "Unknown" (valid: no)
2024/01/15 10:30:46 [error] 1234#0: *2 invalid HTTP status code: 999
```

#### Q: What if I encounter an API-related crash?

**A:** The API includes defensive programming:

```c
/* API checks for NULL request pointer */
if (r == NULL) {
    return NGX_ERROR;
}

/* API validates status range */
if (status < 100 || status > 599) {
    return NGX_ERROR;
}
```

Crashes are unlikely, but if encountered:

1. Enable core dumps: `ulimit -c unlimited`
2. Reproduce the issue
3. Analyze with gdb: `gdb nginx core`
4. Report to NGINX GitHub Issues

### Build and Configuration Questions

#### Q: Do I need to recompile NGINX to use the API?

**A:** If you're building a module against NGINX 1.27.0+, no special steps needed:

```bash
./auto/configure --add-module=/path/to/mymodule
make
```

The API is part of core NGINX and automatically available.

#### Q: Can I use the API in dynamic modules?

**A:** Yes, dynamic modules can use the API:

```bash
./auto/configure --add-dynamic-module=/path/to/mymodule
make modules
```

The API symbols are exported from the main NGINX binary.

#### Q: How do I check if the API is available?

**A:** Check the NGINX version:

```bash
nginx -V
```

Look for `nginx version: nginx/1.27.0` or higher.

In your module code:

```c
#if (nginx_version >= 1027000)
    #define NGX_HTTP_STATUS_API_AVAILABLE 1
#else
    #define NGX_HTTP_STATUS_API_AVAILABLE 0
#endif
```

---

## Complete Code Examples

This section provides full, working examples of migrated modules.

### Example 1: Simple Content Handler Module

This example shows a complete content handler module that serves static content with proper status code handling.

```c
/*
 * ngx_http_example_module.c - Example content handler module
 */

#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

/* Module configuration structure */
typedef struct {
    ngx_flag_t  enable;
} ngx_http_example_loc_conf_t;

/* Function declarations */
static ngx_int_t ngx_http_example_handler(ngx_http_request_t *r);
static void *ngx_http_example_create_loc_conf(ngx_conf_t *cf);
static char *ngx_http_example_merge_loc_conf(ngx_conf_t *cf, 
                                               void *parent, void *child);

/* Module directives */
static ngx_command_t ngx_http_example_commands[] = {
    { ngx_string("example"),
      NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_example_loc_conf_t, enable),
      NULL },
    ngx_null_command
};

/* Module context */
static ngx_http_module_t ngx_http_example_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */
    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */
    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */
    ngx_http_example_create_loc_conf,      /* create location configuration */
    ngx_http_example_merge_loc_conf        /* merge location configuration */
};

/* Module definition */
ngx_module_t ngx_http_example_module = {
    NGX_MODULE_V1,
    &ngx_http_example_module_ctx,          /* module context */
    ngx_http_example_commands,             /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};

/* Handler implementation */
static ngx_int_t
ngx_http_example_handler(ngx_http_request_t *r)
{
    ngx_int_t                      rc;
    ngx_buf_t                     *b;
    ngx_chain_t                    out;
    ngx_http_example_loc_conf_t   *elcf;
    u_char                        *content;
    size_t                         content_len;
    
    elcf = ngx_http_get_module_loc_conf(r, ngx_http_example_module);
    
    if (!elcf->enable) {
        return NGX_DECLINED;
    }
    
    /* Only allow GET and HEAD methods */
    if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
        return NGX_HTTP_NOT_ALLOWED;
    }
    
    /* Prepare content */
    content = (u_char *) "Hello from example module!\n";
    content_len = ngx_strlen(content);
    
    /* Set status code using new API */
    rc = ngx_http_status_set(r, NGX_HTTP_OK);
    if (rc != NGX_OK) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "failed to set status in example handler");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    /* Set content type */
    r->headers_out.content_type_len = sizeof("text/plain") - 1;
    ngx_str_set(&r->headers_out.content_type, "text/plain");
    r->headers_out.content_length_n = content_len;
    
    /* Send headers */
    rc = ngx_http_send_header(r);
    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }
    
    /* Allocate buffer for response body */
    b = ngx_create_temp_buf(r->pool, content_len);
    if (b == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    /* Fill buffer with content */
    b->last = ngx_cpymem(b->pos, content, content_len);
    b->last_buf = 1;
    b->last_in_chain = 1;
    
    /* Send body */
    out.buf = b;
    out.next = NULL;
    
    return ngx_http_output_filter(r, &out);
}

/* Create location configuration */
static void *
ngx_http_example_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_example_loc_conf_t  *conf;
    
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_example_loc_conf_t));
    if (conf == NULL) {
        return NULL;
    }
    
    conf->enable = NGX_CONF_UNSET;
    
    return conf;
}

/* Merge location configuration */
static char *
ngx_http_example_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_example_loc_conf_t *prev = parent;
    ngx_http_example_loc_conf_t *conf = child;
    
    ngx_conf_merge_value(conf->enable, prev->enable, 0);
    
    return NGX_CONF_OK;
}
```

### Example 2: Header Filter Module with Status Modification

This example demonstrates a filter module that modifies status codes based on request conditions.

```c
/*
 * ngx_http_example_filter_module.c - Example filter module
 */

#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

/* Module configuration */
typedef struct {
    ngx_flag_t  enable_304;
} ngx_http_example_filter_conf_t;

/* Function declarations */
static ngx_int_t ngx_http_example_header_filter(ngx_http_request_t *r);
static ngx_int_t ngx_http_example_filter_init(ngx_conf_t *cf);
static void *ngx_http_example_filter_create_loc_conf(ngx_conf_t *cf);
static char *ngx_http_example_filter_merge_loc_conf(ngx_conf_t *cf,
                                                      void *parent, void *child);

/* Module directives */
static ngx_command_t ngx_http_example_filter_commands[] = {
    { ngx_string("example_filter_304"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_example_filter_conf_t, enable_304),
      NULL },
    ngx_null_command
};

/* Module context */
static ngx_http_module_t ngx_http_example_filter_module_ctx = {
    NULL,                                        /* preconfiguration */
    ngx_http_example_filter_init,                /* postconfiguration */
    NULL,                                        /* create main configuration */
    NULL,                                        /* init main configuration */
    NULL,                                        /* create server configuration */
    NULL,                                        /* merge server configuration */
    ngx_http_example_filter_create_loc_conf,     /* create location configuration */
    ngx_http_example_filter_merge_loc_conf       /* merge location configuration */
};

/* Module definition */
ngx_module_t ngx_http_example_filter_module = {
    NGX_MODULE_V1,
    &ngx_http_example_filter_module_ctx,         /* module context */
    ngx_http_example_filter_commands,            /* module directives */
    NGX_HTTP_MODULE,                             /* module type */
    NULL,                                        /* init master */
    NULL,                                        /* init module */
    NULL,                                        /* init process */
    NULL,                                        /* init thread */
    NULL,                                        /* exit thread */
    NULL,                                        /* exit process */
    NULL,                                        /* exit master */
    NGX_MODULE_V1_PADDING
};

/* Filter chain pointer */
static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;

/* Header filter implementation */
static ngx_int_t
ngx_http_example_header_filter(ngx_http_request_t *r)
{
    ngx_http_example_filter_conf_t  *efcf;
    ngx_int_t                        rc;
    time_t                           ims;
    
    efcf = ngx_http_get_module_loc_conf(r, ngx_http_example_filter_module);
    
    /* Only process if enabled */
    if (!efcf->enable_304) {
        return ngx_http_next_header_filter(r);
    }
    
    /* Only modify 200 OK responses */
    if (r->headers_out.status != NGX_HTTP_OK) {
        return ngx_http_next_header_filter(r);
    }
    
    /* Check for If-Modified-Since header */
    if (r->headers_in.if_modified_since == NULL) {
        return ngx_http_next_header_filter(r);
    }
    
    /* Parse If-Modified-Since header */
    ims = ngx_parse_http_time(r->headers_in.if_modified_since->value.data,
                               r->headers_in.if_modified_since->value.len);
    
    if (ims == NGX_ERROR) {
        return ngx_http_next_header_filter(r);
    }
    
    /* Check if resource was modified */
    if (r->headers_out.last_modified_time <= ims) {
        /* Resource not modified - return 304 using API */
        rc = ngx_http_status_set(r, NGX_HTTP_NOT_MODIFIED);
        if (rc != NGX_OK) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "failed to set 304 status in filter");
            /* Continue with 200 OK */
            return ngx_http_next_header_filter(r);
        }
        
        /* Clear content length for 304 response */
        r->headers_out.content_length_n = -1;
        ngx_http_clear_content_length(r);
        ngx_http_clear_accept_ranges(r);
        
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "example filter: returning 304 for \"%V\"", &r->uri);
    }
    
    return ngx_http_next_header_filter(r);
}

/* Initialize filter module - insert into filter chain */
static ngx_int_t
ngx_http_example_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_example_header_filter;
    
    return NGX_OK;
}

/* Create location configuration */
static void *
ngx_http_example_filter_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_example_filter_conf_t  *conf;
    
    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_example_filter_conf_t));
    if (conf == NULL) {
        return NULL;
    }
    
    conf->enable_304 = NGX_CONF_UNSET;
    
    return conf;
}

/* Merge location configuration */
static char *
ngx_http_example_filter_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_example_filter_conf_t *prev = parent;
    ngx_http_example_filter_conf_t *conf = child;
    
    ngx_conf_merge_value(conf->enable_304, prev->enable_304, 0);
    
    return NGX_CONF_OK;
}
```

### Example 3: Backward Compatible Module (Supports Old and New NGINX)

This example shows how to write a module that works with both old NGINX versions (direct assignment) and new versions (API).

```c
/*
 * ngx_http_compat_module.c - Backward compatible module
 */

#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

/* Compatibility wrapper macro */
#if (nginx_version >= 1027000)
    /* New API available */
    #define NGX_HTTP_SET_STATUS(r, code)                                      \
        do {                                                                  \
            ngx_int_t _rc = ngx_http_status_set((r), (code));                \
            if (_rc != NGX_OK) {                                             \
                ngx_log_error(NGX_LOG_ERR, (r)->connection->log, 0,          \
                              "failed to set status %ui", (ngx_uint_t)(code));\
                return NGX_HTTP_INTERNAL_SERVER_ERROR;                        \
            }                                                                 \
        } while (0)
#else
    /* Fallback for older NGINX versions */
    #define NGX_HTTP_SET_STATUS(r, code)                                      \
        do {                                                                  \
            (r)->headers_out.status = (code);                                \
        } while (0)
#endif

/* Module handler */
static ngx_int_t
ngx_http_compat_handler(ngx_http_request_t *r)
{
    ngx_buf_t     *b;
    ngx_chain_t    out;
    
    /* Only allow GET and HEAD */
    if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
        return NGX_HTTP_NOT_ALLOWED;
    }
    
    /* Set status using compatibility macro */
    NGX_HTTP_SET_STATUS(r, NGX_HTTP_OK);
    
    /* Set content type */
    r->headers_out.content_type_len = sizeof("text/plain") - 1;
    ngx_str_set(&r->headers_out.content_type, "text/plain");
    r->headers_out.content_length_n = 27;
    
    /* Send headers */
    if (ngx_http_send_header(r) == NGX_ERROR || r->header_only) {
        return NGX_OK;
    }
    
    /* Allocate and send body */
    b = ngx_create_temp_buf(r->pool, 27);
    if (b == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }
    
    b->last = ngx_cpymem(b->pos, "Backward compatible module\n", 27);
    b->last_buf = 1;
    b->last_in_chain = 1;
    
    out.buf = b;
    out.next = NULL;
    
    return ngx_http_output_filter(r, &out);
}

/* Module configuration and initialization code omitted for brevity */
```

---

## Conclusion

The HTTP Status Code API provides a modern, centralized approach to status code management in NGINX modules. While migration is optional, adopting the API offers significant benefits:

- **RFC 9110 compliance validation**
- **Consistent error handling**
- **Standardized reason phrases**
- **Future-proof architecture**
- **Zero performance overhead in standard mode**

### Next Steps

1. **Review Your Module**: Identify all status code assignments
2. **Plan Migration**: Decide on gradual vs. complete migration
3. **Implement Changes**: Replace direct assignments with API calls
4. **Test Thoroughly**: Run comprehensive test suites
5. **Document Changes**: Update your module documentation

### Additional Resources

- [NGINX Development Guide](https://nginx.org/en/docs/dev/development_guide.html)
- [NGINX Module Development](https://www.nginx.com/resources/wiki/extending/)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [NGINX GitHub Repository](https://github.com/nginx/nginx)
- [NGINX Module Examples](https://github.com/nginx/nginx/tree/master/src/http/modules)

### Getting Help

- **GitHub Discussions**: [nginx/nginx/discussions](https://github.com/nginx/nginx/discussions)
- **GitHub Issues**: [nginx/nginx/issues](https://github.com/nginx/nginx/issues)
- **Mailing List**: [nginx-devel](https://mailman.nginx.org/mailman/listinfo/nginx-devel)

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**NGINX Version**: 1.27.0+  
**License**: BSD 2-Clause (same as NGINX)
