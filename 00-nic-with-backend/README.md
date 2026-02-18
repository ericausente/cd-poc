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

