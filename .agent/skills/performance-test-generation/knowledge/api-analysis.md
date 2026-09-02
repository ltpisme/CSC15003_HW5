# REST API Specification Analysis & Flow Modeling

## 1. Principles of REST API Performance Analysis
When extracting performance test requirements from an API specification, analyze the following dimensions:

### A. Endpoint Classification by Workload Characteristic
1. **Read-Heavy Endpoints:**
   - Methods: `GET`.
   - Characteristics: High query volume, read from database/cache, low disk I/O, potential bandwidth consumer.
   - Example: Listing catalogs, searching products, viewing public profiles.
2. **Auth-Heavy / CPU-Heavy Endpoints:**
   - Methods: `POST /login`, `POST /register`, `POST /forgot-password`.
   - Characteristics: Involves password hashing (bcrypt/argon2), JWT signing/verification, token validation, rate-limiting, and security lockouts.
3. **Transactional / Write-Heavy Endpoints:**
   - Methods: `POST`, `PUT`, `PATCH`, `DELETE`.
   - Characteristics: Database mutation (`INSERT`/`UPDATE`), transaction locks, state changes, in-memory updates.
   - Example: Adding to cart, checkout, placing orders, updating inventory.

---

## 2. Dependency & Data Flow Modeling

### A. Constructing Valid E2E Chains
A realistic performance test must link dependent requests into cohesive journeys:
```text
[Identity Provisioning] ──► [Authentication] ──► [Resource Discovery] ──► [Mutation/Action] ──► [State Verification]
```

### B. Dynamic Data Extraction & Correlation
* **Session/JWT Correlation:**
  - Sampler: `POST /api/login`
  - Extractor: JSONPath `$.token` $\to$ Store as `${jwt_token}`
  - Downstream Injection: HTTP Header Manager `Authorization: Bearer ${jwt_token}`.
* **Resource ID Correlation:**
  - Sampler: `GET /api/products`
  - Extractor: JSONPath `$.items[0].id` $\to$ Store as `${product_id}`
  - Downstream Injection: `POST /api/cart` payload `{"productId": "${product_id}", "quantity": 1}`.

---

## 3. Test Data Parameterization Strategy
1. **Static Pre-seeded Data:** For existing catalog items, categories, or shared credentials.
2. **Dynamic Runtime Generated Data:** When unique entities are required per thread to prevent collision or duplicate key constraint errors:
   - Pattern: `${prefix}_${__threadNum}_${__counter(FALSE,)}@domain.com`
3. **CSV Isolation:** Ensure relative CSV paths are portable across execution environments (`data/test_data.csv`).
