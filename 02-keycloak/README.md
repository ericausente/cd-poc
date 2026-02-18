# NGINX + NJS + Keycloak Authorization POC

## Overview

This Proof of Concept demonstrates NGINX Ingress Controller using
**auth_request + NJS** to perform authorization against Keycloak using
two OAuth2 flows:

1.  **Client Credentials (Basic)**
2.  **UMA Ticket (Bearer Permission Check)**

NGINX does not expose Keycloak directly.\
All validation happens server-side via NJS.

------------------------------------------------------------------------

# Architecture Flow

    Client → NGINX → auth_request → NJS
          → Keycloak Token Endpoint
              → 200 → Allow (204)
              → Non‑200 → Deny (401/403)

------------------------------------------------------------------------

# Keycloak Configuration

⚠️ This Keycloak instance is exposed under `/auth`.

All token endpoints must use:

    https://keycloakaz.kushikimi.xyz/auth/realms/<realm>/protocol/openid-connect/token

Confirm using discovery:

``` bash
curl -sk https://keycloakaz.kushikimi.xyz/auth/realms/mw-dev-realm/.well-known/openid-configuration | jq -r .token_endpoint
```

------------------------------------------------------------------------

# A) Create Realm

Realm name:

    mw-dev-realm

------------------------------------------------------------------------

# B) Client for Client Credentials Flow (Basic)

Create client:

  Setting                    Value
  -------------------------- ----------------
  Client ID                  mw-test-client
  Client Type                Confidential
  Client Authentication      ON
  Service Accounts Enabled   ON
  Standard Flow              OFF
  Implicit Flow              OFF
  Direct Access Grants       OFF

No redirect URIs required.

### Generate Basic Header

``` bash
echo -n 'mw-test-client:<client-secret>' | base64
```

Use as:

    Authorization: Basic <base64>

------------------------------------------------------------------------

# C) Resource Server Client for UMA

Create client:

  Setting                 Value
  ----------------------- -------------------
  Client ID               mw-resource-owner
  Client Type             Confidential
  Authorization Enabled   ON

------------------------------------------------------------------------

# D) Create Resource + Scope

Inside:

    Clients → mw-resource-owner → Authorization

### Resource

    Name: httpbin-headers
    URI: /headers

### Scope

    GET

Attach scope to resource.

------------------------------------------------------------------------

# E) Create Policy + Permission

### Policy

Create a Role-based or User-based policy (Allow All for lab).

### Permission

Type: Resource-based\
Resource: httpbin-headers\
Scope: GET\
Apply policy created above.

------------------------------------------------------------------------

# Manual Validation Before NGINX

## 1) Client Credentials Test

``` bash
curl -sk -X POST   'https://keycloakaz.kushikimi.xyz/auth/realms/mw-dev-realm/protocol/openid-connect/token'   -H 'Content-Type: application/x-www-form-urlencoded'   -H 'Authorization: Basic <BASE64(client:secret)>'   --data-urlencode 'grant_type=client_credentials' | jq .
```

Expected:

``` json
{
  "access_token": "...",
  "expires_in": 300,
  "token_type": "Bearer"
}
```

------------------------------------------------------------------------

## 2) UMA Ticket Test

``` bash
curl -sk -X POST   'https://keycloakaz.kushikimi.xyz/auth/realms/mw-dev-realm/protocol/openid-connect/token'   -H 'Content-Type: application/x-www-form-urlencoded'   -H 'Authorization: Bearer <ACCESS_TOKEN>'   --data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:uma-ticket'   --data-urlencode 'permission=/headers#GET'   --data-urlencode 'audience=mw-resource-owner'   --data-urlencode 'permission_resource_format=uri'   --data-urlencode 'permission_resource_matching_uri=true' | jq .
```

If this returns 200 with `access_token`, UMA configuration is correct.

------------------------------------------------------------------------

# NGINX Validation

``` bash
curl -i https://cdg-auth-lab.kushikimi.xyz/headers   -H "Authorization: Basic <BASE64>"
```

or

``` bash
curl -i https://cdg-auth-lab.kushikimi.xyz/headers   -H "Authorization: Bearer <TOKEN>"
```

------------------------------------------------------------------------

# Common Pitfalls

1.  Missing `/auth` base path
2.  Wrong realm name
3.  Authorization not enabled on resource client
4.  Permission string mismatch (`/headers#GET`)
5.  Wrong client secret

------------------------------------------------------------------------

# Summary

This POC validates:

• NGINX auth_request pattern\
• NJS dynamic flow switching\
• Keycloak client_credentials\
• Keycloak UMA Authorization Services\
• Permission-based backend access control

------------------------------------------------------------------------
