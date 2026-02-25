# 🚦 NGINX Plus Dynamic Rate Limiting (No Lua / No njs / No Internal Locations)

This lab demonstrates production-ready dynamic rate limiting using:

-   NGINX Plus
-   Keyval (runtime database)
-   Map directive
-   limit_req (static zones, dynamic keys)
-   Mergeable Ingress (master + single minion `/mw/`)
-   optyional Zone Sync (shared counters across pods)
-   No Lua
-   No njs
-   No internal gear locations

------------------------------------------------------------------------

# Architecture Overview

We dynamically classify endpoints using a key-value database:

POST\|/mw/.../fetchy → high\
POST\|/mw/.../opty → low\
POST\|/mw/.../eventy → veryhigh

Instead of dynamically changing rate values (not supported by
limit_req), we:

1.  Define static rate zones
2.  Use keyval to assign a tier (low, high, veryhigh)
3.  Use map to ensure only one limiter key is active
4.  Apply all limiters in one location
5.  Only the selected zone enforces

------------------------------------------------------------------------

# What "Bucket" Means

A bucket is the per-key limiter state stored in the shared memory zone.

Example key:

POST\|/mw/dummy-management/v1/offers/fetchy

That key has its own rate counter inside the zone.

With Zone Sync enabled, buckets are synchronized across NGINX pods.

------------------------------------------------------------------------

# Namespace

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rate-lab
```

------------------------------------------------------------------------

# httpbin Backend

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: rate-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - name: httpbin
        image: kennethreitz/httpbin
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin-svc
  namespace: rate-lab
spec:
  selector:
    app: httpbin
  ports:
  - name: http
    port: 80
    targetPort: 80
```

------------------------------------------------------------------------

# NIC Global ConfigMap (http-snippet)

``` yaml
data:
  zone-sync: "true"

  http-snippet: |
    keyval_zone zone=ep_tier:10m state=/var/lib/nginx/state/ep_tier.state;

    map "$request_method|$uri" $ep_key {
      default "$request_method|$uri";
    }

    keyval $ep_key $tier zone=ep_tier;

    map $tier $tier_eff {
      low      low;
      high     high;
      veryhigh veryhigh;
      default  low;
    }

    map $tier_eff $rl_key_low {
      low     $ep_key;
      default "";
    }

    map $tier_eff $rl_key_high {
      high    $ep_key;
      default "";
    }

    map $tier_eff $rl_key_veryhigh {
      veryhigh $ep_key;
      default  "";
    }

    limit_req_zone $rl_key_low      zone=limitlow:10m      rate=5r/s    sync;
    limit_req_zone $rl_key_high     zone=limithigh:10m     rate=10r/s   sync;
    limit_req_zone $rl_key_veryhigh zone=limitveryhigh:10m rate=15r/s   sync;
```

------------------------------------------------------------------------

# Master Ingress

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-master
  namespace: rate-lab
  annotations:
    nginx.org/mergeable-ingress-type: "master"
spec:
  ingressClassName: nginx
  rules:
  - host: cdgratelimit.kushikimi.xyz
    http: {}
```

------------------------------------------------------------------------

# Minion Ingress (/mw/)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mw-minion
  namespace: rate-lab
  annotations:
    nginx.org/mergeable-ingress-type: "minion"
    nginx.org/location-snippets: |
      add_header X-EP-Key $ep_key always;
      add_header X-Tier $tier_eff always;

      limit_req zone=limitlow      burst=20 nodelay;
      limit_req zone=limithigh     burst=20 nodelay;
      limit_req zone=limitveryhigh burst=20 nodelay;

    nginx.org/rewrites: "serviceName=httpbin-svc rewrite=/anything"
spec:
  ingressClassName: nginx
  rules:
  - host: cdgratelimit.kushikimi.xyz
    http:
      paths:
      - path: /mw/
        pathType: Prefix
        backend:
          service:
            name: httpbin-svc
            port:
              number: 80
```

------------------------------------------------------------------------

# Populate Keyval DB

``` bash
curl -k \
  -H 'Content-Type: application/json' \
  -H 'Expect:' \
  -X PATCH \
  --data-binary '{
    "POST|/mw/dummy-management/dummymanagement/v1/offers/fetchy": "high",
    "POST|/mw/dummy-management/dummymanagement/v4/offers/opty": "low",
    "POST|/mw/dummy-management/dummymanagement/v1/eventy": "veryhigh"
  }' \
  http://cdgratelimit.kushikimi.xyz/_nginx_api/6/http/keyvals/ep_tier
```

------------------------------------------------------------------------

# Testing

``` bash
hey -z 10s -q 50 -m POST \
  http://cdgratelimit.kushikimi.xyz/mw/dummy-management/dummymanagement/v1/offers/fetchy
```

Check headers:

X-Tier: high\
X-EP-Key: POST\|...

------------------------------------------------------------------------

# Production Notes

-   Secure the API endpoint
-   Mount /var/lib/nginx/state/ for persistence
-   Normalize URIs if dynamic IDs are used
-   Tune burst values appropriately

------------------------------------------------------------------------
