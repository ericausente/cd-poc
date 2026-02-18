## CelcomDigi Auth Lab (NIC + NJS + Keycloak)

### What it does
- Protects httpbin behind NGINX Ingress
- Uses `auth_request` to call an internal NJS script
- NJS chooses:
  - Basic -> Keycloak client_credentials
  - Bearer -> Keycloak UMA ticket permission check
- NJS returns only 204 (allow) or 401/403 (deny)

Test
```
curl -i https://cdg-auth-lab.kushikimi.xyz/headers -H "Authorization: Bearer <TOKEN>"
curl -i https://cdg-auth-lab.kushikimi.xyz/headers -H "Authorization: Basic <BASE64(client:secret)>"
```

```
# curl -i http://cdg-auth-lab.kushikimi.xyz/headers   -H "Authorization: Basic <redacted>"
HTTP/1.1 200 OK
Server: nginx/1.29.3
Date: Wed, 18 Feb 2026 09:23:11 GMT
Content-Type: application/json
Content-Length: 1540
Connection: keep-alive
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

{
  "headers": {
    "Accept": "*/*",
    "Authorization": "Bearer <redacted>",
    "Connection": "close",
    "Host": "cdg-auth-lab.kushikimi.xyz",
    "User-Agent": "curl/8.0.1",
    "X-Forwarded-Host": "cdg-auth-lab.kushikimi.xyz"
  }
}
```
