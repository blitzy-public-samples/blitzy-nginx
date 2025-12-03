# NGINX HTTP Status Code API - Deployment Documentation

## Table of Contents

1. [Overview](#overview)
2. [Pre-Deployment Checklist](#pre-deployment-checklist)
3. [Build Configuration Options](#build-configuration-options)
4. [Deployment Procedure](#deployment-procedure)
5. [Rollback Instructions](#rollback-instructions)
6. [Validation Checkpoints](#validation-checkpoints)
7. [Configuration File Reference](#configuration-file-reference)
8. [Troubleshooting](#troubleshooting)

---

## Overview

This deployment introduces a centralized HTTP status code management API to NGINX, implementing RFC 9110 HTTP Semantics compliance. The refactoring modernizes status code handling by transitioning from scattered constant-based assignments to a registry-based API system.

### Key Changes Summary

| Component | Change Type | Impact |
|-----------|-------------|--------|
| `src/http/ngx_http.h` | API Declarations | New functions: `ngx_http_status_set()`, `ngx_http_status_validate()`, `ngx_http_status_reason()` |
| `src/http/ngx_http_request.h` | Status Flags | Added: `NGX_HTTP_STATUS_CACHEABLE`, `NGX_HTTP_STATUS_CLIENT_ERROR`, `NGX_HTTP_STATUS_SERVER_ERROR`, `NGX_HTTP_STATUS_INFORMATIONAL` |
| `src/http/ngx_http_request.c` | Registry Implementation | Static const registry with 60+ RFC 9110 status codes |
| `src/http/modules/*` | Module Updates | Updated to use `ngx_http_status_set()` API |
| `auto/options` | Build System | New flag: `--with-http_status_validation` |

### Backward Compatibility

**IMPORTANT:** This deployment maintains 100% backward compatibility:
- All existing `nginx.conf` configurations work unchanged
- All existing directives (`error_page`, `return`, `try_files`, etc.) function identically
- Third-party modules continue working without modification
- Graceful upgrade from older NGINX versions is supported

---

## Pre-Deployment Checklist

Before deploying, verify the following requirements:

### System Requirements

- [ ] Operating System: Linux 2.6+ kernel, FreeBSD 10+, or macOS 10+
- [ ] Compiler: GCC 4.8+ or Clang 3.4+
- [ ] Libraries: PCRE (8.x or 10.x), zlib (1.1.3+), OpenSSL (1.0.2+)
- [ ] Build tools: make (3.81+), perl (5.6+)

### Pre-flight Checks

```bash
# Verify current NGINX version
/path/to/current/nginx -v

# Backup current configuration
cp -r /usr/local/nginx/conf /usr/local/nginx/conf.backup

# Backup current binary
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.old

# Verify disk space (minimum 100MB for build)
df -h /usr/local/nginx
```

### Configuration Backup

```bash
# Create timestamped backup
BACKUP_DIR="/var/backups/nginx/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"
cp -r /usr/local/nginx/conf "$BACKUP_DIR/"
cp /usr/local/nginx/sbin/nginx "$BACKUP_DIR/"
echo "Backup created: $BACKUP_DIR"
```

---

## Build Configuration Options

### Standard Mode (Default - Recommended for Production)

Standard mode provides maximum backward compatibility with zero performance overhead:

```bash
./auto/configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    [other existing flags]

make -j$(nproc)
make install
```

In standard mode:
- Status code API is available via inline macro (zero overhead)
- Basic range validation (100-599) is performed
- All modules use the unified API internally
- No strict RFC 9110 enforcement

### Strict Validation Mode (Optional - For RFC Compliance Testing)

Strict mode enables comprehensive RFC 9110 validation:

```bash
./auto/configure \
    --prefix=/usr/local/nginx \
    --with-http_status_validation \
    --with-http_ssl_module \
    --with-http_v2_module \
    [other existing flags]

make -j$(nproc)
make install
```

In strict validation mode:
- Full RFC 9110 compliance checking enabled
- Reserved status code 306 is rejected
- Non-standard NGINX codes (444, 494-499) generate warnings in logs
- Upstream status pass-through is preserved (no validation for backend responses)
- Additional debug logging for status operations

**Performance Note:** Strict mode may add minimal overhead (~1-2%). Use for staging/testing environments or when RFC compliance is mandatory.

### Build Verification

After building, verify the binary:

```bash
# Check version and build flags
./objs/nginx -V

# Verify configuration syntax
./objs/nginx -t -c /path/to/nginx.conf

# Test configuration with new binary
./objs/nginx -T -c /path/to/nginx.conf
```

---

## Deployment Procedure

### Method 1: Standard Upgrade (Recommended)

This method ensures zero downtime using NGINX's built-in graceful upgrade:

#### Step 1: Build New Binary

```bash
cd /path/to/nginx-source
./auto/configure --prefix=/usr/local/nginx [your options]
make -j$(nproc)
```

#### Step 2: Test New Binary

```bash
# Test configuration with new binary
./objs/nginx -t -c /usr/local/nginx/conf/nginx.conf

# If test fails, DO NOT proceed - check error messages
```

#### Step 3: Deploy New Binary

```bash
# Copy new binary (old binary is preserved as nginx.old)
cp ./objs/nginx /usr/local/nginx/sbin/nginx.new
```

#### Step 4: Perform Hot Upgrade

```bash
# Get current master process PID
NGINX_PID=$(cat /usr/local/nginx/logs/nginx.pid)

# Replace binary
mv /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.old
mv /usr/local/nginx/sbin/nginx.new /usr/local/nginx/sbin/nginx

# Signal master to start new workers with new binary
kill -USR2 $NGINX_PID

# Wait for new master process to start
sleep 2

# Verify both old and new workers are running
ps aux | grep nginx

# Gracefully shutdown old workers
kill -WINCH $NGINX_PID
```

#### Step 5: Complete Upgrade

```bash
# After verifying new workers are handling traffic correctly:
# Quit old master process (if .oldbin file exists)
if [ -f /usr/local/nginx/logs/nginx.pid.oldbin ]; then
    kill -QUIT $(cat /usr/local/nginx/logs/nginx.pid.oldbin)
fi
```

### Method 2: Restart Deployment (Causes Brief Downtime)

Use this method for simpler deployments where brief downtime is acceptable:

```bash
# Stop NGINX
/usr/local/nginx/sbin/nginx -s stop

# Replace binary
cp ./objs/nginx /usr/local/nginx/sbin/nginx

# Start NGINX
/usr/local/nginx/sbin/nginx
```

### Method 3: Containerized Deployment

For Docker/Kubernetes environments:

```dockerfile
# Dockerfile
FROM nginx:1.29.3 as builder

# Copy patched source
COPY ./nginx-source /usr/src/nginx

WORKDIR /usr/src/nginx
RUN ./auto/configure \
    --prefix=/etc/nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    && make -j$(nproc)

FROM nginx:1.29.3
COPY --from=builder /usr/src/nginx/objs/nginx /usr/sbin/nginx
```

---

## Rollback Instructions

### Immediate Rollback (During Hot Upgrade)

If issues are detected during hot upgrade:

```bash
# Return to old binary
kill -HUP $(cat /usr/local/nginx/logs/nginx.pid.oldbin)

# Quit new master
kill -QUIT $(cat /usr/local/nginx/logs/nginx.pid)
```

### Rollback After Completed Deployment

```bash
# Stop current NGINX
/usr/local/nginx/sbin/nginx -s stop

# Restore old binary
cp /usr/local/nginx/sbin/nginx.old /usr/local/nginx/sbin/nginx

# Restore configuration (if changed)
cp -r /var/backups/nginx/[timestamp]/conf/* /usr/local/nginx/conf/

# Start NGINX with old binary
/usr/local/nginx/sbin/nginx
```

### Emergency Rollback Script

Save this script for emergency use:

```bash
#!/bin/bash
# emergency_rollback.sh

set -e

NGINX_PREFIX="/usr/local/nginx"
BACKUP_DIR="/var/backups/nginx/latest"

echo "=== NGINX Emergency Rollback ==="

# Stop current NGINX
echo "Stopping NGINX..."
$NGINX_PREFIX/sbin/nginx -s stop 2>/dev/null || true

# Restore backup
if [ -f "$NGINX_PREFIX/sbin/nginx.old" ]; then
    echo "Restoring previous binary..."
    cp "$NGINX_PREFIX/sbin/nginx.old" "$NGINX_PREFIX/sbin/nginx"
elif [ -f "$BACKUP_DIR/nginx" ]; then
    echo "Restoring from backup..."
    cp "$BACKUP_DIR/nginx" "$NGINX_PREFIX/sbin/nginx"
else
    echo "ERROR: No backup binary found!"
    exit 1
fi

# Start NGINX
echo "Starting NGINX..."
$NGINX_PREFIX/sbin/nginx

# Verify
$NGINX_PREFIX/sbin/nginx -v
echo "=== Rollback Complete ==="
```

---

## Validation Checkpoints

### Checkpoint 1: Build Verification

```bash
# Verify compilation succeeded
./objs/nginx -v
# Expected: nginx version: nginx/1.29.3

# Check for compilation warnings (should be zero)
make 2>&1 | grep -i "warning" | wc -l
# Expected: 0

# Verify binary links correctly
ldd ./objs/nginx
# Should show all required libraries without "not found" errors
```

### Checkpoint 2: Configuration Validation

```bash
# Test configuration syntax
./objs/nginx -t -c /usr/local/nginx/conf/nginx.conf
# Expected: syntax is ok, test is successful

# Dump full configuration
./objs/nginx -T -c /usr/local/nginx/conf/nginx.conf | head -50
# Verify configuration parses correctly
```

### Checkpoint 3: Status Code Behavior Verification

Create a test configuration:

```nginx
# /tmp/status_test.conf
daemon off;
worker_processes 1;
error_log /tmp/status_test.log debug;
pid /tmp/status_test.pid;

events { worker_connections 64; }

http {
    server {
        listen 8765;
        
        # Test various status codes
        location /test200 { return 200 "OK"; }
        location /test201 { return 201 "Created"; }
        location /test301 { return 301 /redirected; }
        location /test302 { return 302 /found; }
        location /test304 { return 304; }
        location /test400 { return 400 "Bad Request"; }
        location /test401 { return 401 "Unauthorized"; }
        location /test403 { return 403 "Forbidden"; }
        location /test404 { return 404 "Not Found"; }
        location /test429 { return 429 "Too Many Requests"; }
        location /test500 { return 500 "Internal Error"; }
        location /test502 { return 502 "Bad Gateway"; }
        location /test503 { return 503 "Service Unavailable"; }
        
        # Test error_page directive
        error_page 404 /custom_404.html;
        location = /custom_404.html { return 200 "Custom 404"; }
    }
}
```

Run validation tests:

```bash
#!/bin/bash
# status_code_test.sh

NGINX_BIN="./objs/nginx"
CONFIG="/tmp/status_test.conf"

# Start test server
$NGINX_BIN -c $CONFIG &
NGINX_PID=$!
sleep 1

# Test all status codes
declare -A STATUS_CODES=(
    ["/test200"]=200
    ["/test201"]=201
    ["/test301"]=301
    ["/test302"]=302
    ["/test400"]=400
    ["/test401"]=401
    ["/test403"]=403
    ["/test404"]=404
    ["/test429"]=429
    ["/test500"]=500
    ["/test502"]=502
    ["/test503"]=503
)

FAILED=0
for path in "${!STATUS_CODES[@]}"; do
    expected="${STATUS_CODES[$path]}"
    actual=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:8765$path")
    
    if [ "$actual" == "$expected" ]; then
        echo "✓ $path returned $actual"
    else
        echo "✗ $path expected $expected, got $actual"
        FAILED=$((FAILED + 1))
    fi
done

# Cleanup
kill $NGINX_PID 2>/dev/null

if [ $FAILED -eq 0 ]; then
    echo "=== All status code tests passed ==="
    exit 0
else
    echo "=== $FAILED tests failed ==="
    exit 1
fi
```

### Checkpoint 4: Upstream Pass-Through Verification

For proxy deployments, verify upstream status codes pass through correctly:

```bash
# Test proxy pass-through (requires upstream server)
curl -I http://localhost/proxy-endpoint
# Verify: Status code matches upstream response
```

### Checkpoint 5: Performance Verification

```bash
# Install wrk if not available
# apt-get install wrk || brew install wrk

# Baseline measurement
wrk -t4 -c100 -d30s http://localhost:8765/test200 > baseline.txt

# Compare with previous measurements
# Latency increase should be < 2%
```

### Checkpoint 6: Error Log Verification

```bash
# Check for status code related errors
grep -i "status\|invalid.*code" /usr/local/nginx/logs/error.log

# In strict mode, verify warnings are logged for non-standard codes
grep "non-standard HTTP status code" /tmp/status_test.log
```

---

## Configuration File Reference

### No Changes Required

The following directives work identically after deployment:

```nginx
# error_page directive - unchanged
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

# return directive - unchanged
location /redirect {
    return 301 https://example.com$request_uri;
}

# try_files directive - unchanged
location / {
    try_files $uri $uri/ =404;
}

# proxy_pass - unchanged
location /api {
    proxy_pass http://backend;
}

# proxy_intercept_errors - unchanged
location /api {
    proxy_pass http://backend;
    proxy_intercept_errors on;
    error_page 502 503 504 /custom_error.html;
}
```

### Debug Logging (Optional)

To enable detailed status code logging for troubleshooting:

```nginx
# Set error_log to debug level
error_log /var/log/nginx/error.log debug;

# In http block, enable debug for specific connections
# (for production, use debug_connection for specific IPs only)
events {
    debug_connection 192.168.1.100;
}
```

---

## Troubleshooting

### Issue: Configuration Test Fails After Upgrade

**Symptoms:**
```
nginx: [emerg] unknown directive "..."
```

**Resolution:**
1. Verify build options match previous installation
2. Check that all required modules are compiled in
3. Compare `nginx -V` output between old and new binaries

### Issue: Status Code Validation Errors in Strict Mode

**Symptoms:**
```
invalid HTTP status code: 999
```

**Resolution:**
- Verify status codes in `return` directives are in valid RFC 9110 range (100-599)
- Reserved code 306 is not allowed
- If using custom codes, consider standard mode instead

### Issue: Upstream Status Codes Rejected

**Symptoms:**
Backend responses fail with 502 errors in strict mode.

**Resolution:**
Upstream status codes should pass through without validation. If issues occur:
1. Verify `r->upstream` is set correctly in proxy modules
2. Check error logs for validation failures
3. Consider using standard mode for proxy-heavy deployments

### Issue: Performance Regression After Deployment

**Symptoms:**
Increased latency in status code responses.

**Resolution:**
1. Verify using standard mode (not strict validation)
2. Run `perf` analysis on status code hot paths
3. Consider recompiling with optimization flags

### Issue: Hot Upgrade Fails

**Symptoms:**
```
kill: (12345): No such process
```

**Resolution:**
1. Verify NGINX master PID file location
2. Check process is running: `ps aux | grep nginx`
3. Use restart deployment method if hot upgrade fails

### Contact and Support

For issues related to this deployment:
- Check NGINX documentation: https://nginx.org/en/docs/
- Review RFC 9110 HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Consult CONTRIBUTING.md for development guidelines

---

## Deployment Scenarios Matrix

| Scenario | Recommended Method | Downtime | Risk Level |
|----------|-------------------|----------|------------|
| Production - High Availability | Hot Upgrade | Zero | Low |
| Production - Maintenance Window | Restart | Brief | Low |
| Staging/Testing | Restart | Acceptable | Very Low |
| Containerized (K8s) | Rolling Update | Zero | Low |
| Development | Direct Replace | Acceptable | None |

---

## Quick Reference Commands

```bash
# Build (standard mode)
./auto/configure --prefix=/usr/local/nginx && make -j$(nproc)

# Build (strict mode)
./auto/configure --prefix=/usr/local/nginx --with-http_status_validation && make -j$(nproc)

# Test configuration
./objs/nginx -t -c /usr/local/nginx/conf/nginx.conf

# Hot upgrade
kill -USR2 $(cat /usr/local/nginx/logs/nginx.pid)

# Graceful shutdown of old workers
kill -WINCH $(cat /usr/local/nginx/logs/nginx.pid)

# Immediate rollback during hot upgrade
kill -QUIT $(cat /usr/local/nginx/logs/nginx.pid)
kill -HUP $(cat /usr/local/nginx/logs/nginx.pid.oldbin)

# Full restart
/usr/local/nginx/sbin/nginx -s stop && /usr/local/nginx/sbin/nginx

# Check version
/usr/local/nginx/sbin/nginx -v
```

---

*Document Version: 1.0*  
*Last Updated: December 2025*  
*Applies to: NGINX 1.29.3 with HTTP Status Code API*
