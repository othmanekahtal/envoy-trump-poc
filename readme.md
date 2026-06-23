
# TRUMP POC WITH ENVOY
## 1. Architectural Strategy

The local Proof of Concept (POC) uses a **Local Split-Horizon DNS** strategy paired with an **Edge Reverse Proxy Layer**. This approach isolates and tests complex routing policies entirely on your machine before committing them to production infrastructure pipelines.



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
   ┌───────────────────┼───────────────────┐
   │ (Matches Rule 1)  │ (Matches Rule 2)  │ (Matches Rule 4)
   ▼                   ▼                   ▼

```

[ 200 Pass-Through ]   [ 301 Redirect ]    [ 200 Core App ]
my-account.xxxx.ma     xxxx.ma -> xxxx.com  xxxx.com


### Execution Strategy for Production Scaling
1. **Infrastructure Tiering:** In a production environment, Envoy handles edge routing policies, acting as the global entry point. Your application layer remains decoupled from top-level domain (TLD) and locale-string mutation logic.
2. **Deterministic Precedence:** Exceptions (such as subdomains or specific static API configurations) are evaluated at the top of the stack. Fallbacks and broader match criteria sit underneath.
3. **Canonical Host Normalization:** Regional domains execute a 301 redirect to consolidate global link equity and session tracking onto a unified host (`xxxx.com`), segmenting users explicitly via localized path structures.

---

## 2. Configuration Breakdown

### Core Route Evaluation Hierarchy
Envoy evaluates virtual hosts sequentially based on the `domains` array entries. The order dictates execution precedence:

| Route Sequence Block | Match Criterion | Target / Action | Business Case |
| :--- | :--- | :--- | :--- |
| **1. `my_account_exception`** | `my-account.xxxx.ma` | `route: target_app_cluster` | **Exception Rule:** Direct pass-through. Bypasses geo-redirection to preserve specialized application state. |
| **2. `ma_redirect_domain`** | `xxxx.ma` | `redirect: xxxx.com/ma/MA-fr` | **Geo-Mapping Rule:** Intercepts traffic to modify target domain and inject country/language codes. |
| **3. `fr_redirect_domain`** | `xxxx.fr` | `redirect: xxxx.com/fr/fr` | **Geo-Mapping Rule:** Handles regional traffic according to matching market requirements. |
| **4. `main_app_domain`** | `xxxx.com` | `route: target_app_cluster` | **Catch-All Hub:** Accepts and services pre-validated, correctly routed traffic. |

### Technical Directive Specifics
* **`host_redirect` & `path_redirect`**: Instructs Envoy to drop the original request attributes and issue an immediate downstream browser mutation. This minimizes backend load since un-mapped domains never traverse deeper into your application cluster network.
* **`MOVED_PERMANENTLY` (301)**: Informs clients and search crawlers that the localization mapping is immutable, forcing browsers to cache the resolution pathway locally for improved performance.
* **`host.docker.internal`**: A special DNS name used by the Docker container to route traffic back out to services hosted natively on the local machine or inside adjacent network namespaces without pinning rigid IP vectors.