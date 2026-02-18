# AKS + NGINX Plus Ingress – Structured Logging with PV (Production-Style Lab)

This document describes how to configure **NGINX Plus Ingress Controller
(NIC)** on **AKS** to:

-   Write structured JSON access logs
-   Persist logs to a Persistent Volume (Azure Files)
-   Include mandatory `X-Request-ID`
-   Capture Keycloak auth subrequest failures
-   Prepare logs for Fluentd → Elasticsearch → Kibana pipeline

------------------------------------------------------------------------

# 🎯 Design Principles

This implementation:

-   Does NOT log to container stdout
-   Writes logs to `/var/log/dte/nginx/`
-   Uses Azure Files (RWX) so logs survive pod restarts
-   Enables `log_subrequest on;` so Keycloak auth failures are visible
-   Keeps payload logging limited to IDP failures (safe, bounded)

Kibana filtering should be done via JSON fields — NOT by directory
splitting.

------------------------------------------------------------------------

# 1️⃣ Storage – Azure Files PVC (RWX)

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-logs-pvc
  namespace: nginx-ingress
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 50Gi
```

Apply:

    kubectl apply -f 20-nginx-logs-pvc.yaml

------------------------------------------------------------------------

# 2️⃣ NIC ConfigMap – Structured JSON Logging

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-plus-nginx-ingress
  namespace: nginx-ingress
data:
  http-snippets: |
    underscores_in_headers on;
    subrequest_output_buffer_size 64k;

    map $http_x_request_id $req_id {
      default $http_x_request_id;
      ""      $request_id;
    }

    log_subrequest on;

    log_format cdg_json escape=json
      '{'
        '"ts":"$time_iso8601",'
        '"req_id":"$req_id",'
        '"remote_addr":"$remote_addr",'
        '"xff":"$http_x_forwarded_for",'
        '"host":"$host",'
        '"method":"$request_method",'
        '"uri":"$uri",'
        '"args":"$args",'
        '"status":$status,'
        '"bytes_sent":$bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_addr":"$upstream_addr",'
        '"upstream_status":"$upstream_status",'
        '"upstream_response_time":"$upstream_response_time",'
        '"kc_status":"$kc_status",'
        '"auth_detail":"$kc_detail",'
        '"kc_body":"$kc_body",'
        '"namespace":"$resource_namespace",'
        '"ingress":"$resource_name",'
        '"service":"$service"'
      '}';

    access_log /var/log/dte/nginx/access.json cdg_json buffer=256k flush=1s;
    error_log  /var/log/dte/nginx/error.log info;
```

Apply:

    kubectl apply -f 21-nic-configmap-logging.yaml

------------------------------------------------------------------------

# 3️⃣ NIC Deployment – Mount PV at /var/log/dte

Key parts only (volume + mount):

``` yaml
volumes:
- name: nginx-logs
  persistentVolumeClaim:
    claimName: nginx-logs-pvc

containers:
- name: nginx-ingress
  volumeMounts:
  - name: nginx-logs
    mountPath: /var/log/dte
```

Logs will be written to:

    /var/log/dte/nginx/access.json

------------------------------------------------------------------------

# 4️⃣ Ingress Snippet – Capture Auth Debug Metadata

``` nginx
set $permission "$lower_uri#$request_method";

proxy_set_header X-Request-ID $req_id;
add_header X-Request-ID $req_id always;

auth_request /_oauth2_token_introspection;

auth_request_set $final_auth $sent_http_token_forward;
auth_request_set $kc_status $sent_http_x_auth_status;
auth_request_set $kc_detail $sent_http_x_auth_detail;
auth_request_set $kc_body   $sent_http_x_auth_body;

proxy_set_header Authorization $final_auth;
```

------------------------------------------------------------------------

# 5️⃣ Validation

Trigger traffic:

    curl http://cdg-auth-lab.kushikimi.xyz/headers -H "Authorization: Basic ..."

Check logs:

    kubectl -n nginx-ingress exec -it deploy/nginx-plus-nginx-ingress-controller --   sh -lc 'tail -n 5 /var/log/dte/nginx/access.json'

You should see structured JSON lines including:

-   req_id
-   status
-   request_time
-   upstream_status
-   kc_status (Keycloak response)
-   kc_body (truncated IDP failure payload)

------------------------------------------------------------------------

# 6️⃣ Kibana Strategy

Fluentd should tail:

    /var/log/dte/nginx/access.json

Filtering should be based on JSON fields:

-   req_id
-   status
-   kc_status
-   namespace
-   service

Directory splitting by pod is NOT recommended. Use JSON metadata
filtering instead.

------------------------------------------------------------------------

# 🔐 Notes on Payload Logging

Full API payload logging for all non-2xx responses is NOT recommended at
ingress layer due to performance risk.

This implementation safely logs:

-   Keycloak error payloads (small, bounded)
-   Structured metadata for troubleshooting

If deeper payload inspection is required, implement sampling + sidecar
logging.

------------------------------------------------------------------------

# ✅ Summary

This setup provides:

-   Production-grade structured JSON logs
-   Persistent storage via Azure Files
-   Correlated tracing using X-Request-ID
-   Visibility into NJS + Keycloak failures
-   Ready for Fluentd → Elasticsearch → Kibana integration
