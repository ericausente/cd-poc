We are doing here in this POC two flows:
1. client_credentials (Basic)
2. UMA ticket (Bearer permission check)

So in Keycloak you need:

A) Realm
Realm name: mw-dev-realm (or change NGINX to match)

B) Confidential Client for Client-Credentials
This is what your Basic header represents.
```
Create a client:
Client ID: mw-test-client (example)
Client type: Confidential
Enable: Service Accounts
Client Authentication: ON

Allowed grant types:
✅ Client credentials
```

Then you’ll use:
```
echo -n 'mw-test-client:<client-secret>' | base64
```

and pass it as Authorization: Basic <base64>

C) Resource Server Client for UMA (audience)

Your UMA request uses:
audience=mw-resource-owner

So you need a client:
```
Client ID: mw-resource-owner
Authorization: ✅ Enabled (Keycloak “Authorization Services”)
```

D) Create a Protected Resource + Scopes

You are sending permissions like:
/headers#GET (or /mw/campaign-management/...#GET)

So you need to create resources that match your lab URIs.

In Keycloak (in mw-resource-owner client):
Authorization → Resources → Create:
Resource Name: httpbin-headers
URI: /headers

(Optionally set URI matching / wildcard if you prefer)

Authorization → Scopes:
Create scope: GET (or read)
Attach scope to resource.

E) Create Policy + Permission

Minimal permissive setup (so you can test “success path” first):

Policy: Allow All (or Role-based)
Easiest: “User-based” or “Role-based” policy

Permission:
Type: Resource-based

Resource: httpbin-headers

Scopes: GET

Apply Policy: Allow All / role policy

This makes UMA succeed for the requested permission.

Step 3 — Validate Keycloak manually (before NGINX)

1) client_credentials
```
curl -sk --request POST 'https://keycloakaz.kushikimi.xyz/realms/mw-dev-realm/protocol/openid-connect/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'Authorization: Basic <BASE64(client:secret)>' \
  --data-urlencode 'grant_type=client_credentials' | jq .
```

2) UMA ticket (permission check)

You need a Bearer token that Keycloak accepts for UMA.
For lab simplicity, you can use a user token or service account token depending on how you configured policies.
```
curl -sk --request POST 'https://keycloakaz.kushikimi.xyz/realms/mw-dev-realm/protocol/openid-connect/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'Authorization: Bearer <ACCESS_TOKEN>' \
  --data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:uma-ticket' \
  --data-urlencode 'permission=/headers#GET' \
  --data-urlencode 'audience=mw-resource-owner' \
  --data-urlencode 'permission_resource_format=uri' \
  --data-urlencode 'permission_resource_matching_uri=true' | jq .
```


If this returns 200 with an access_token, your Keycloak side is correct.
