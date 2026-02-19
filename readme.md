# Traefik Header-Based Routing Test Setup

This repository contains a set of Kubernetes manifests designed to test Traefik's header-based routing capabilities.

## Components

The `ingress-test.yaml` file defines the following resources:

### 1. IngressRoute (`header-routing-test`)

Defines three distinct routing rules to test different matching behaviors:

| Route Name | Path | Headers Required | Description |
|------------|------|------------------|-------------|
| **No Header** | `/whoami-noheader` | None | Baseline test. Routes traffic without checking for any specific headers. |
| **Simple Header** | `/whoami-simpleheader` | `X-Tenant: simpleheader` | Exact match test. Traffic is only routed if the `X-Tenant` header exactly matches `simpleheader`. |
| **Regex Header** | `/whoami-regexheader` | `X-Tenant: .*` | Regex match test. Traffic is routed if the `X-Tenant` header exists (matches any value). |

**Note:** All routes use the `whoami-stripprefix` middleware and point to the same `whoami` service.

### 2. Backend Service

*   **Deployment**: A single replica of `traefik/whoami`. This service simply echoes back the request details (headers, path, IP, etc.), allowing us to verify that traffic reached the destination.
*   **Service**: Exposes the deployment on port 8080.
*   **Middleware**: A `StripPrefix` middleware configured to strip `/whoami`.

### 3. Automated Validation (CronJobs)

Three CronJobs are included to automatically generate traffic for validation. They run every minute (`*/1 * * * *`).

*   **`curl-noheader`**:
    *   POSTs to `/whoami-noheader`
    *   Sends header: `X-Tenant: simpleheader` (Ignored by the rule)
*   **`curl-simpleheader`**:
    *   POSTs to `/whoami-simpleheader`
    *   Sends header: `X-Tenant: simpleheader` (Should match)
*   **`curl-regexheader`**:
    *   POSTs to `/whoami-regexheader`
    *   Sends header: `X-Tenant: simpleheader` (Should match the regex `.*`)

## Usage

1. **Apply the manifests:**
   ```bash
   kubectl apply -f ingress-test.yaml
   ```

2. **Verify the Deployment:**
   Ensure the `whoami` pod is up and running:
   ```bash
   kubectl get pods -l app=whoami
   ```

3. **Check Validation Results:**
   To verify if routing is working, check the logs of the CronJobs. If a job fails, it means `curl` received a non-200 response (likely 404 if the route didn't match).

   ```bash
   # List the jobs
   kubectl get cronjobs

   # Check logs for a specific job (e.g., the simple header test)
   kubectl logs jobs/curl-simpleheader-<job-id>
   # Or find the latest pod for the job:
   kubectl logs -l application=curl-simpleheader --tail=20
   ```

## Troubleshooting

If the CronJobs are failing (showing error status or non-200 codes):

1.  **Check Traefik Logs:** Ensure Traefik picked up the `IngressRoute` configuration without errors.
2.  **Validate Headers:** Ensure the `X-Tenant` header is passing through any upstream load balancers or potential firewalls.
3.  **Path Matching:** Verify that the `whoami-stripprefix` middleware configuration aligns with your expected path rewriting logic.


