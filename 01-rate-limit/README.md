# NGINX Plus Ingress -- Dynamic Rate Limiting with Profiles (Small / Medium / Large)

## Overview

This document explains how to implement **dynamic request rate limiting
(TPS)** using:

-   NGINX Plus Ingress Controller
-   Mergeable Ingress (Master / Minion)
-   Predefined rate profiles (5 / 10 / 15 / 25 TPS)
-   NGINX Plus Key-Value Store (keyval)
-   Runtime profile switching using the NGINX Plus API (no reload
    required)

------------------------------------------------------------------------

# Why We Use Profiles Instead of Dynamically Changing `rate=`

NGINX architecture defines `limit_req_zone rate=...` at **configuration
load time**.

The `rate=` parameter: - Cannot be a variable - Cannot be modified at
runtime - Requires reload if changed directly

Therefore, the correct runtime-safe strategy is:

1.  Predefine multiple rate zones (5 / 10 / 15 / 25 TPS).
2.  Dynamically select which zone to use via a `map` directive.
3.  Store the active profile in NGINX Plus keyval.
4.  Change profile using API without reload.

This approach provides:

-   No NGINX reload
-   Operational safety (bounded values only)
-   Governance control (approved TPS gears)
-   Immediate runtime switching

------------------------------------------------------------------------

# Architecture

## Default Behavior (Customer Requirement)

If no endpoint-specific limit is configured:

-   Master Ingress enforces default **2 TPS** globally.

## Per-Application Behavior

Minion Ingress overrides rate limit using:

-   Predefined rate zones
-   keyval-driven profile
-   `limit_req zone=$mapped_zone`

------------------------------------------------------------------------

# Step 1 -- Enable Snippets

In NIC ConfigMap:

``` yaml
data:
  allow-snippet-annotations: "true"
```

------------------------------------------------------------------------

# Step 2 -- Add Rate Control in NIC ConfigMap

Add to `http-snippets`:

``` nginx
# Define key-value storage
keyval_zone zone=profiles:1m state=/var/lib/nginx/state/profiles.json;

# Bind key into variable
keyval $export_profile profiles app_export;

# Static rate zones
limit_req_zone "export" zone=export_5tps:10m  rate=5r/s;
limit_req_zone "export" zone=export_10tps:10m rate=10r/s;
limit_req_zone "export" zone=export_15tps:10m rate=15r/s;
limit_req_zone "export" zone=export_25tps:10m rate=25r/s;

# Map profile to zone
map $export_profile $export_zone {
    default export_5tps;
    5       export_5tps;
    10      export_10tps;
    15      export_15tps;
    25      export_25tps;
}
```

------------------------------------------------------------------------

# Step 3 -- Master Ingress (Default 2 TPS)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-master
  namespace: rate-lab
  annotations:
    nginx.org/mergeable-ingress-type: "master"
    nginx.org/limit-req-rate: "2r/s"
    nginx.org/limit-req-key: "global"
    nginx.org/limit-req-zone-size: "20M"
spec:
  ingressClassName: nginx
  rules:
  - host: api.kushikimi.xyz
```

Master contains NO paths.

------------------------------------------------------------------------

# Step 4 -- Minion Ingress for /export

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-export
  namespace: rate-lab
  annotations:
    nginx.org/mergeable-ingress-type: "minion"
    nginx.org/location-snippets: |
      limit_req zone=$export_zone burst=20 nodelay;
spec:
  ingressClassName: nginx
  rules:
  - host: api.kushikimi.xyz
    http:
      paths:
      - path: /export
        pathType: Prefix
        backend:
          service:
            name: export-svc
            port:
              number: 80
```

------------------------------------------------------------------------

# Step 5 -- Runtime Profile Switching

## Set to 5 TPS

``` bash
curl -X PATCH http://<nginx-plus-ip>:8080/api/9/http/keyvals/profiles -d '{"key":"app_export","value":"5"}'
```

## Set to 15 TPS

``` bash
curl -X PATCH http://<nginx-plus-ip>:8080/api/9/http/keyvals/profiles -d '{"key":"app_export","value":"15"}'
```

## Verify

``` bash
curl http://<nginx-plus-ip>:8080/api/9/http/keyvals/profiles/app_export
```

------------------------------------------------------------------------

# Testing

``` bash
hey -z 10s -c 50 http://api.kushikimi.xyz/export
```

Switch profile while test runs and observe 429 behavior change
instantly.

------------------------------------------------------------------------

# Production Considerations

## Horizontal Scaling

Rate limiting is per Ingress pod.

If 3 replicas: - 10 TPS per pod = \~30 TPS total

For cluster-consistent global limits: - Redis-backed counters are
recommended.

## Default Enforcement

Master ensures 2 TPS baseline even if endpoint policy missing.

------------------------------------------------------------------------

# Summary

We implemented:

-   Default global 2 TPS
-   Per-app dynamic rate profiles
-   Small/Medium/Large gears
-   No reload switching
-   NGINX Plus API integration
-   Mergeable Ingress compatibility

This is production-grade API traffic governance using NGINX Plus.
