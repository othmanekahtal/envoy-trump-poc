
# TRUMP POC WITH ENVOY
## 1. Architectural Strategy

The local Proof of Concept (POC) uses a **Local Split-Horizon DNS** strategy paired with an **Edge Reverse Proxy Layer**. This approach isolates and tests complex routing policies entirely on your machine before committing them to production infrastructure pipelines.

![Architecture Diagram](archi.png)

```
              ┌─────────────────┐
              │   Client/cURL   │
              └────────┬────────┘
                       │ (HTTP Port 80)
                       ▼
              ┌───────────────────────┐
              │ Envoy Proxy Container │
              └────────┬──────────────┘
                       │
   ┌───────────────────┼───────────────────┬───────────────────┐
   │ (Matches Rule 1)  │ (Matches Rule 2)  │ (Matches Rule 4)  │ (Matches Rule 4a)
   ▼                   ▼                   ▼                   ▼

```

[ 200 Pass-Through ]   [ 301 Redirect ]    [ 200 Core App ]    [ 200 Proxied ]
my-account.xxxx.ma     xxxx.ma -> xxxx.com  xxxx.com            xxxx.com/MA-fr/portal
                        xxxx.ma/portal ->                       -> wesite-portal.com
                        xxxx.com/MA-fr/portal                   (Node.js Hello World)


### Execution Strategy for Production Scaling
1. **Infrastructure Tiering:** In a production environment, Envoy handles edge routing policies, acting as the global entry point. Your application layer remains decoupled from top-level domain (TLD) and locale-string mutation logic.
2. **Deterministic Precedence:** Exceptions (such as subdomains, portal paths, or specific static API configurations) are evaluated at the top of the stack. Fallbacks and broader match criteria sit underneath.
3. **Canonical Host Normalization:** Regional domains execute a 301 redirect to consolidate global link equity and session tracking onto a unified host (`xxxx.com`), segmenting users explicitly via localized path structures.
4. **Legacy System Masking:** A dedicated `/portal` experience is surfaced under the canonical `xxxx.com` host while the actual content is transparently proxied from a separate legacy backend (`wesite-portal.com`), so the browser's address bar never reveals the real origin.

---

## 2. Configuration Breakdown

### Core Route Evaluation Hierarchy
Envoy evaluates virtual hosts sequentially based on the `domains` array entries. Within a virtual host, routes are evaluated top-to-bottom and the first prefix match wins. The order dictates execution precedence:

| Route Sequence Block | Match Criterion | Target / Action | Business Case |
| :--- | :--- | :--- | :--- |
| **1. `my_account_exception`** | `my-account.xxxx.ma` | `route: target_app_cluster` | **Exception Rule:** Direct pass-through. Bypasses geo-redirection to preserve specialized application state. |
| **2a. `ma_redirect_domain` / `/portal`** | `xxxx.ma/portal` (prefix) | `redirect: xxxx.com/MA-fr/portal` | **Portal Rule:** Sends portal traffic to its dedicated path on the canonical host, evaluated before the broader geo-redirect below. |
| **2b. `ma_redirect_domain` / `/`** | `xxxx.ma` (catch-all) | `redirect: xxxx.com/ma/MA-fr` | **Geo-Mapping Rule:** Intercepts remaining traffic to modify target domain and inject country/language codes. |
| **3. `fr_redirect_domain`** | `xxxx.fr` | `redirect: xxxx.com/fr/fr` | **Geo-Mapping Rule:** Handles regional traffic according to matching market requirements. |
| **4a. `main_app_domain` / `/MA-fr/portal`** | `xxxx.com/MA-fr/portal` (prefix) | `route: wesite_portal_cluster` + `host_rewrite_literal` | **Legacy Proxy Rule:** Transparently forwards the request to the legacy `wesite-portal.com` backend. The browser URL never changes. |
| **4b. `main_app_domain` / `/`** | `xxxx.com` (catch-all) | `route: target_app_cluster` | **Catch-All Hub:** Accepts and services pre-validated, correctly routed traffic. |

### Technical Directive Specifics
* **`host_redirect` & `path_redirect`**: Instructs Envoy to drop the original request attributes and issue an immediate downstream browser mutation. This minimizes backend load since un-mapped domains never traverse deeper into your application cluster network.
* **`MOVED_PERMANENTLY` (301)**: Informs clients and search crawlers that the localization mapping is immutable, forcing browsers to cache the resolution pathway locally for improved performance. Because of this caching, changes to a redirect target may require a hard-refresh/private window to observe locally.
* **`host.docker.internal`**: A special DNS name used by the Docker container to route traffic back out to services hosted natively on the local machine or inside adjacent network namespaces without pinning rigid IP vectors.
* **`host_rewrite_literal`**: Used on the `/MA-fr/portal` route so the legacy backend receives a `Host: wesite-portal.com` header even though Envoy is physically connecting to `host.docker.internal:8080`. This mocks hitting the real legacy domain without needing actual DNS for it.
* **Envoy does not hot-reload `envoy.yaml`**: after editing the file you must run `docker compose restart envoy` for changes to take effect.

---

## 3. The Legacy Portal Backend (`wesite-portal.com`)

To simulate the legacy system referenced by rule 4a, this POC ships a minimal Node.js server (`node-app/`) that returns `Hello World` on port `8080`. It is built and run as the `wesite-portal` service in `docker-compose.yaml`, published on host port `8080`.

Add the following to your local `/etc/hosts` (note: `/etc/hosts` does not support port suffixes — only map the hostname to an IP):

```
127.0.0.1 wesite-portal.com
```



This lets you hit the legacy backend directly at `http://wesite-portal.com:8080` for comparison, while Envoy reaches it internally via `host.docker.internal:8080`.

---

## 4. Local Testing

With the stack up (`docker compose up -d --build`) and `xxxx.ma`, `xxxx.fr`, `xxxx.com`, `my-account.xxxx.ma` mapped to `127.0.0.1` in `/etc/hosts`:

```bash
curl -I http://my-account.xxxx.ma        # 200, pass-through
curl -I http://xxxx.fr                   # 301 -> xxxx.com/fr/fr
curl -I http://xxxx.ma                   # 301 -> xxxx.com/ma/MA-fr
curl -I http://xxxx.ma/portal            # 301 -> xxxx.com/MA-fr/portal
curl http://xxxx.com/MA-fr/portal        # 200, "Hello World" (proxied from wesite-portal.com)
```
