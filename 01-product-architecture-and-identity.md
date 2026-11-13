# Harbor Shop

Product, architecture, and the identity service.

This is the first chapter of the Harbor notes. It explains what the shop is, how the services fit together, and every identity idea a DevOps engineer should be able to explain before the service is built.

The shop is built **without Keycloak**. A later part of this same document explains the same login **with Keycloak**, so the two designs can be compared. Keycloak is not installed and is not on the build path yet.

The pictures live in the `images` folder next to this file.

---

## 1. What this product is

Harbor Shop is an e-commerce application: one online store that sells physical goods to customers.

It is a single-brand shop. Harbor itself is the seller. It is not a marketplace where many sellers list products, and it is not a bank. A payment partner takes the money. Harbor stores the result: paid, failed, or refunded.

Version one sells a short list of household goods, priced in rupees, and ships them to an address the customer types at checkout.

| Product | Price | Stock at the start |
|---|---:|---:|
| Copper kettle, 1.5 L | Rs 2,499 | 40 |
| Desk lamp, white | Rs 1,299 | 25 |
| Cotton bedsheet, queen | Rs 1,899 | 12 |

Three people use the shop.

| Person | Job in the story | What they are allowed to do |
|---|---|---|
| Priya | Customer, `priya@example.com` | Search, keep a cart, pay, see her own orders |
| Arun | Support, `arun@harbor.example` | Look up an order and refund it |
| Meera | Admin, `meera@harbor.example` | Assign roles and manage catalog data |

![Harbor Shop is one online store. Priya buys, Arun refunds, Meera administers.](images/01-product.svg)

Priya's normal visit:

1. She opens the site and searches for a kettle.
2. She registers or logs in.
3. She puts the copper kettle in the cart.
4. She checks out and pays Rs 2,499.
5. She receives a receipt for order 1001.
6. If the kettle arrives damaged, Arun refunds order 1001.

Pages in version one: home, product list, product page, search, cart, checkout, login, my orders, and a support page for refunds.

Included in version one:

- Email and password login, issued by Harbor's own identity service
- Three roles: `CUSTOMER`, `SUPPORT`, `ADMIN`
- Browse, search, cart, checkout
- A practice payment provider, so no real card number is used
- A receipt written to a log if no mail server is configured yet
- Docker Compose on a laptop

Waiting until that core runs: a second factor, login through a company directory, a real payment company, Kubernetes, and more than one shop. Keycloak is in that later group. It is documented below so the idea is clear, and it is not part of the first build.

---

## 2. How the shop is put together

A **microservice** is a small program with one job, its own data, and its own deploy. Catalog can ship on Tuesday while payments ships on Thursday.

A **monolith** is one program that contains all of those jobs. Many companies still run one. Harbor uses separate services because the point of this project is to learn how to run many of them.

The browser talks only to the API gateway. The gateway routes the call. Orders do not open the inventory database. Orders call the inventory service, or they publish an event.

![Each Harbor service and the data store that belongs to it.](images/02-architecture.svg)

| Service | Language | Data store | Why this pairing |
|---|---|---|---|
| Identity | Java 21, Spring Boot 3 | PostgreSQL | Accounts need transactions, constraints, and a mature security library. Java is the usual corporate choice for that. |
| Catalog | Go | PostgreSQL | A small, fast product API. The product table is stable and relational. |
| Inventory | Go | PostgreSQL | Stock updates are short transactions with row locks. Same kind of program as catalog, separate deploy so a sale can scale stock alone. |
| Cart | TypeScript on Node.js | Redis | A cart is temporary JSON and can expire. |
| Orders | C# on .NET | PostgreSQL | The business record of a purchase. .NET is common for this kind of workflow in enterprises. |
| Payments | Python, FastAPI | PostgreSQL | Partner APIs and callbacks. The payment result stays in a relational table. |
| Search | Python | OpenSearch | Typo-tolerant product search. |
| Notifications | TypeScript worker | Kafka in, mail out | A background worker. It has no product database of its own. |

Platform pieces around those services:

| Piece | Role |
|---|---|
| API gateway (NGINX or Kong) | The only public door. TLS, routing, rate limits. |
| Kafka | The event log. "Order 1001 was placed" is a message other services can read. |
| React storefront | The website. It is a client, not a microservice. |
| OpenTelemetry, Prometheus, Grafana, Loki | Metrics, logs, and traces once the shop is running. |

### Why the databases differ

PostgreSQL is the default when the row must survive and must be correct: a user, a price, a stock count, an order, a payment result.

Redis is the default when the data is temporary and can be rebuilt or lost without corrupting the business: Priya's cart, a login rate-limit counter, a short denylist of logged-out access tokens.

OpenSearch is the default when the shop must search text. A SQL `LIKE` search does not handle typos and ranking well.

Kafka is the default when a service should announce a fact and continue, while another service reacts later.

Each service has **its own** database. Identity tables are not a schema inside the orders database. Sharing one database makes deploys look easy and then couples every service to every table change.

### Words that apply to every service

**Stateless** means any copy of the service can take the next request. Priya's session is not kept only in the memory of one machine. The durable state is in PostgreSQL, Redis, or Kafka. Then a restart, or a second copy, still works.

**Config** is the database host, token lifetime, and public URL. Those are environment variables.

**Secrets** are the database password, the token signing key, and the payments client secret. They stay out of Git and out of the image.

**Liveness** asks "is the process up?" **Readiness** asks "should we send traffic yet?" Readiness fails when that service's database is down. Liveness should stay healthy in that case, so the platform does not kill a process that can recover when the database returns.

**A sync call** waits for an answer. Login, cart, and checkout use HTTP because a person is waiting.

**An event** announces something that already happened. Notifications and search listen for `OrderPlaced` and `ProductChanged`. They can be a few seconds behind. That delay is **eventual consistency**: the system becomes correct shortly, not in the same millisecond.

**A timeout** caps how long orders will wait for payments. **A retry** is safe only when repeating the call does not do the work twice. **An idempotency key** such as `order-1001` makes the second charge return the first result.

### The purchase, once she is logged in

![Checkout of order 1001 across cart, orders, inventory, payments, and mail.](images/14-checkout.svg)

If payment is declined, orders cancel 1001 and inventory puts the kettle back. That undo is a **compensating action**. The whole cross-service story is a **saga**. One database transaction cannot cover orders, inventory, and payments, because they are three databases.

The identity service is involved at the start of this path and then stays quiet. Checkout does not ask Priya for her password again.

### Order we will build in

1. Identity, its PostgreSQL, one Dockerfile, health URLs.
2. Catalog, a second runtime, its own database.
3. Cart and Redis.
4. Inventory, then orders.
5. Payments, with the idempotency key.
6. Kafka, then notifications and search.
7. The gateway, TLS, and rate limits.
8. Kubernetes only after the same flows already work in Docker Compose.

---

## 3. Identity without Keycloak

This is the service we will build.

Identity answers, for the whole shop, "who is calling?" It stores the account, checks the password, issues tokens, and records roles. It does not store the cart, the order, or the shipping address.

| Data | Where it lives |
|---|---|
| Email, password hash, roles, refresh tokens, login audit | Identity's PostgreSQL |
| Shipping address and display preferences | A customer-profile table in the shop, keyed by the same user id |
| Orders | Orders database, keyed by that user id |
| Cart | Redis, keyed by that user id |

Priya's user id in every example below is `8f3c2a10-priya`.

### 3.1 The words

**Identity** is the record of a person or a program: Priya, Arun, or the orders service.

**An account** is that record plus the credential used to open it.

**A principal** is whoever just proved who they are on this request. Sometimes that is Priya. Sometimes it is the orders service.

**Authentication** is the proof of who it is.

**Authorization** is the decision of what they may do.

**A credential** is the proof itself: a password, a refresh token, or a client secret.

**A session** is server-side memory that a login already succeeded.

**A token** is a proof the caller can show on the next request, so she does not send the password again.

![Authentication asks who this is. Authorization asks what they may do.](images/03-authn-authz.svg)

Worked example. Priya calls `POST /orders` for the kettle.

- Authentication succeeds. The token really is Priya.
- Authorization succeeds. Her role is `CUSTOMER`.
- The order is created.

Arun calls `POST /orders/1001/refund`.

- Authentication succeeds. The token really is Arun.
- Authorization succeeds because his role is `SUPPORT`.

Priya calls that same refund URL with her own valid token.

- Authentication succeeds.
- Authorization fails. The response is **403**.

A missing or expired token is **401**. The two status codes are different incidents.

| Status | Meaning in Harbor | What you check first |
|---|---|---|
| 401 | The caller is not authenticated | Missing header, expired token, bad signature, wrong issuer |
| 403 | The caller is authenticated and is not allowed | Role, or she asked for someone else's order |
| 400 | The request body is invalid | Missing email, empty cart |
| 500 | The service failed | Database down, unhandled error, bug |

### 3.2 Register

`POST /register`

```json
{
  "email": "priya@example.com",
  "password": "Kettle#2026"
}
```

Identity does the following.

1. Rejects a duplicate email with 409.
2. Rejects a weak password with 400. Version one asks for at least 12 characters, mixed letters and a number.
3. Creates a random salt for this user only.
4. Hashes the password with Argon2id.
5. Inserts the user with status `ACTIVE` and the role `CUSTOMER`.
6. Writes an audit row `REGISTER_OK`.
7. Returns the new user id. It does not return the password or the hash.

Meera's admin screen creates Arun in the same table and assigns `SUPPORT`. His first password is temporary, and the first successful login requires him to replace it. That flag is `must_change_password`.

### 3.3 How the password is stored

A **hash** is a one-way result. Identity can check a guess. It cannot open the hash and read the original password back.

A **salt** is random data mixed in before hashing, different for every user. If Priya and another customer both choose `Kettle#2026`, the stored hashes still differ. A stolen table cannot be matched against one precomputed list of common passwords.

**Argon2id** is the algorithm Harbor uses. It is slow and memory-heavy on purpose, which makes mass guessing expensive. **bcrypt** is the older algorithm you will see in existing companies. Both are appropriate password hashes.

A fast general hash such as plain SHA-256 is a poor password store. An attacker who copies the table can test billions of guesses per second.

![Priya's password is hashed with a salt. The database stores the hash, not Kettle#2026.](images/04-password-hash.svg)

Example row after register, with a shortened hash:

| Column | Value |
|---|---|
| id | `8f3c2a10-priya` |
| email | `priya@example.com` |
| password_hash | `$argon2id$v=19$m=65536,t=3,p=1$...` |
| status | `ACTIVE` |
| created_at | `2026-11-13T09:00:00Z` |

The hash string from Argon2id already contains the salt and the settings. A separate salt column is optional when the library stores the salt inside that string. Either way, the readable password is never stored.

Login hashes what she types today and compares. The comparison must take the same time for a match and a mismatch, so an attacker cannot use timing as a hint. Libraries do this. Hand-written string compare is the wrong place to be clever.

Failed attempts are slowed. Version one waits a short, fixed extra time and writes `LOGIN_FAIL`. It does not lock the account after three tries. An attacker who knows Priya's email could lock her out on purpose. Slowing the guesses is the safer default. Meera can still set status to `DISABLED`, which is a deliberate admin action.

The application log records `LOGIN_FAIL` and the user id when the email exists. It does not record the password, the hash, or the token.

### 3.4 Login

`POST /login`

```json
{
  "email": "priya@example.com",
  "password": "Kettle#2026"
}
```

![Harbor identity checks the password and returns two tokens.](images/05-login-flow.svg)

On success the body looks like this.

```json
{
  "access_token": "<jwt>",
  "expires_in": 900,
  "refresh_token": "<random-secret>",
  "token_type": "Bearer"
}
```

`expires_in` is 900 seconds, which is 15 minutes.

On failure the body is the same for an unknown email and a wrong password:

```json
{ "error": "invalid_credentials" }
```

HTTP status 401. The audit row differs internally (`LOGIN_FAIL`), and the public message stays the same so the endpoint is not a directory of who has an account.

A `DISABLED` user also receives 401. Support can tell her the account is disabled through the support process. The login API stays quiet.

### 3.5 Session or token

A classic website creates a session. The server stores `session abc = Priya` in Redis and sends a cookie:

```http
Set-Cookie: SESSION=abc; HttpOnly; Secure; SameSite=Lax
```

| Cookie flag | What it does |
|---|---|
| HttpOnly | Browser JavaScript cannot read the cookie |
| Secure | The browser sends it only on HTTPS |
| SameSite=Lax | Another website cannot casually attach this cookie to its own request |

That design is sound for one application. Harbor's catalog, orders, and payments are separate programs. They would all need that shared session store, and they would all need to understand its format.

Harbor uses a **bearer token** instead. After login, calls look like this:

```http
GET /orders HTTP/1.1
Host: shop.harbor.local
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImhhcmJvci1rZXktMSJ9.eyJzdWIiOiI4ZjNjMmExMC1wcml5YSJ9.signature
```

Whoever holds that access token is treated as Priya until it expires. It travels only over HTTPS. Access logs are configured to drop the `Authorization` header.

The browser keeps the access token in memory and the refresh token in a place the shop's JavaScript can use only to call `/refresh`. A longer-lived note for production: many companies put the refresh token in an `HttpOnly` cookie so page scripts cannot read it. Version one may keep both tokens in memory to make the learning path obvious. The DevOps point is the same either way: the refresh token is more valuable than the access token, and it has one job, which is talking to identity.

### 3.6 The access token is a JWT

JWT means JSON Web Token. It is three parts separated by dots: header, payload, signature. The header and payload are JSON, encoded so they can sit in a header. They are not encrypted.

![Header, payload, and signature of Priya's access token.](images/06-jwt.svg)

Header:

```json
{
  "alg": "RS256",
  "kid": "harbor-key-1"
}
```

`alg` is the signature algorithm. RS256 means identity signs with an RSA private key and services check with the RSA public key. `kid` is the key id, so a service can choose the right public key after a rotation.

Payload for Priya:

```json
{
  "iss": "https://identity.harbor.local",
  "sub": "8f3c2a10-priya",
  "email": "priya@example.com",
  "roles": ["CUSTOMER"],
  "iat": 1763020500,
  "exp": 1763021400
}
```

| Claim | Meaning here |
|---|---|
| iss | Issuer. Must be exactly `https://identity.harbor.local`. |
| sub | Subject. Priya's stable user id. Other services store this. |
| email | Convenience for the website header. |
| roles | What authorization will read. |
| iat | Issued-at time, Unix seconds. |
| exp | Expiry. Fifteen minutes after `iat` in this example. |

A service accepts the token only when all of these are true:

1. The signature matches a current Harbor public key.
2. `iss` is `https://identity.harbor.local`.
3. `exp` is still in the future, with a leeway of about 30 seconds.
4. `alg` is RS256. The service's library is told the expected algorithm. It does not trust whatever string is inside the token.

The payload is readable by anyone who copies the token. Shipping address, phone number, and payment data stay out of it.

**Clock skew.** If the orders server is five minutes ahead, it will reject a brand-new token as expired. Lab machines and cluster nodes need the correct time. A 30-second leeway covers small drift. Minutes of drift are an incident, not a setting to widen forever.

**Key rotation.** Identity keeps a private key and publishes the public key at:

```text
GET https://identity.harbor.local/.well-known/jwks.json
```

That document is a JWKS, a JSON Web Key Set. Services cache it. Rotation creates `harbor-key-2`, new tokens use it, and `harbor-key-1` stays in the JWKS until every token signed by key 1 has expired. Deleting key 1 in a hurry makes every still-valid token fail at once.

The private key is a secret. The JWKS is public. Back up the private key with the same care as the database, or plan to force every user to log in again after a restore that lost the key.

### 3.7 Refresh, logout, and the denylist

![The access token calls the shop. The refresh token calls only identity.](images/07-two-tokens.svg)

| | Access token | Refresh token |
|---|---|---|
| Lifetime in this lab | 15 minutes | 7 days |
| Format | Signed JWT | Long random string, not a JWT |
| Sent to | Gateway and shop services | Identity `POST /refresh` only |
| Stored server-side | No. The signature is the proof. | Hash only, in `refresh_tokens` |
| Stolen copy | Useful until `exp` | Useful until it is used or revoked |

`POST /refresh` with the refresh token:

1. Hash the presented token and find the row.
2. Reject it if it is missing, expired, or `revoked`.
3. Mark that row revoked.
4. Insert a new refresh-token hash.
5. Return a new access token and the new refresh token.

That one-time use is **rotation**. If a stolen old refresh token appears after Priya has already rotated, identity revokes every refresh token for her user id and writes `REFRESH_REUSE`. She must log in with her password again. One stolen token should not become a quiet second session.

`POST /logout` revokes the refresh token. The access token cannot be unsigned. It remains valid until `exp`. Identity also writes its `jti` (a unique token id claim) into Redis with a TTL equal to the remaining lifetime. The gateway checks that denylist. After 15 minutes the Redis key expires on its own, because the access token is dead anyway.

Logout from one browser does not need to delete the user. `DISABLED` is the admin action that stops future logins.

### 3.8 Roles

Harbor uses **role-based access control**. A person has roles. An endpoint requires a role.

![Priya, Arun, and Meera against the same order.](images/08-rbac.svg)

| Action | Who passes |
|---|---|
| `GET /orders` | `CUSTOMER`, and only rows with her `sub` |
| `POST /orders` | `CUSTOMER` |
| `GET /orders/1001` | The owner, or `SUPPORT`, or `ADMIN` |
| `POST /orders/1001/refund` | `SUPPORT` or `ADMIN` |
| `POST /admin/users/{id}/roles` | `ADMIN` |
| `GET /products` | Public. No token required. |

**Ownership** is a second check. Role `CUSTOMER` does not mean "any order." Order 1002 belongs to someone else. Priya's token on `GET /orders/1002` is a 403 even though the signature is valid. Arun's `SUPPORT` token may read 1002 because support is the job.

**Scopes** are a related OAuth word. A scope is a permission on this particular token, such as `orders.read`. A role describes the person. A scope describes the token. Harbor version one puts roles in the token and does not add a second permission system. You should still recognize a scope when you see one in another company's token.

**ABAC**, attribute-based access control, adds conditions: "support may refund when the amount is under Rs 5,000." That is a later design. The version-one rule is the role table above.

Roles are copied into the access token at login. If Meera removes `SUPPORT` from Arun, his current access token still says `SUPPORT` until it expires. That is one reason the lifetime is 15 minutes. Refresh must load roles from the database again, not copy them from the old access token.

### 3.9 The two doors on a user request

![The gateway checks the token. Orders checks the role and the owner.](images/09-api-check.svg)

The gateway is the front door:

- Header present
- Signature valid
- Issuer matches
- Not expired
- Not on the logout denylist

Orders is the second door:

- Role allowed for this route
- For a customer, `order.user_id` equals `sub`

A health check is neither door. `GET /health/live` and `GET /health/ready` must succeed with no token. A probe that needs a user login will mark the service down whenever the test user fails.

### 3.10 The orders service has its own identity

When Priya checks out, orders calls payments. That call is the program acting as itself.

![Orders uses a client secret. Payments sees the service, and the order row still names Priya.](images/10-service-token.svg)

Identity has a `service_clients` table:

| client_id | secret hash | allowed call |
|---|---|---|
| harbor-orders | hash of a long random secret | payments charge and inventory reserve |
| harbor-payments | hash of a different secret | none in version one. It only receives calls. |

`POST /token` with the client id and secret returns a 5-minute JWT:

```json
{
  "iss": "https://identity.harbor.local",
  "sub": "harbor-orders",
  "roles": ["SERVICE"],
  "exp": 1763020800
}
```

Payments accepts a charge from `sub` `harbor-orders`. It rejects Priya's `CUSTOMER` token on that internal URL. Her user id is already a column on order 1001, so the receipt still has her name on it.

The client secret is created once, shown once, and stored as a hash. The running orders container receives the secret from the secret store. Replacing it means issuing a second secret, deploying orders with the new one, then disabling the old hash. That overlap is the rotation window.

### 3.11 Tables

![Users, roles, refresh tokens, and the audit log in identity's database.](images/11-data-model.svg)

`user_roles` links users to roles so one person can hold two roles later. Version one gives each story character a single role.

`refresh_tokens` stores a hash. If this table leaks, the attacker still does not have a usable refresh token.

`audit_log` is append-only for the lab. Rows are not updated to hide a failure.

`service_clients` holds machine credentials, separate from human passwords, so a password-hash change never touches a service secret.

Customer shipping data is **not** in this database. A profile row elsewhere looks like this:

| user_id | city | pin |
|---|---|---|
| `8f3c2a10-priya` | Hyderabad | 500081 |

Disabling Priya in identity stops login. It does not delete order 1001. The shop keeps the order so support can still see the purchase. Deleting the account and deleting the history are different decisions.

### 3.12 Endpoints in version one

| Method and path | Who | Result |
|---|---|---|
| `POST /register` | Public | Creates a customer |
| `POST /login` | Public | Access token and refresh token |
| `POST /refresh` | Holds a refresh token | New pair, old refresh token revoked |
| `POST /logout` | Holds an access token and a refresh token | Refresh revoked, access id denylisted |
| `GET /me` | Any logged-in user | Id, email, roles |
| `POST /admin/users` | `ADMIN` | Creates a staff user |
| `POST /admin/users/{id}/roles` | `ADMIN` | Replaces roles |
| `POST /admin/users/{id}/disable` | `ADMIN` | Sets `DISABLED` and revokes refresh tokens |
| `POST /token` | A service with a client secret | Short service token |
| `GET /.well-known/jwks.json` | Public | Public keys |
| `GET /health/live` | Platform | Process is up |
| `GET /health/ready` | Platform | PostgreSQL is reachable |

Rate limits sit on the gateway for `/register`, `/login`, `/refresh`, and `/token`. Identity stays simple. A flood of logins is absorbed before it reaches the hash function.

### 3.13 What you operate

| Item | Lab value | Why it matters |
|---|---|---|
| Listen port | 8080 | One process, one port |
| Public issuer | `https://identity.harbor.local` | Must match `iss` exactly, including `https` |
| Access lifetime | 15 minutes | Bounds a stolen token and a stale role |
| Refresh lifetime | 7 days | Bounds a stolen refresh token |
| Service-token lifetime | 5 minutes | Machine calls are frequent and easy to renew |
| Database | Its own PostgreSQL | Backup this database or you lose every account |
| Schema change | Flyway migration before traffic | A new column exists before the new code needs it |
| Private key | Secret, file or store | Loss forces everyone to log in again if you generate a new key |
| Replicas | Two is fine | No login state in process memory. Refresh rows are in PostgreSQL. Denylist is in Redis, shared. |
| Ready check | Fails if PostgreSQL is down | The gateway stops sending logins there |
| Logs | stdout, JSON, with a request id | The password and both tokens are absent |

Migrations are a step in deploy, not a manual click after the outage starts. Expand the database first (add a column), deploy the code, then contract later (remove an old column) once no running version reads it.

Backup the database and test a restore. A backup that has never been restored is an untested file. Restoring identity without the signing key invalidates every access token, which is acceptable if you planned for a fresh login, and surprising if you did not.

Personal data in a ticket or a chat paste is an incident of its own. A token pasted into a public issue tracker is a live key until `exp`, and the refresh token is worse. Revoke that user's refresh tokens.

### 3.14 Failures you should be able to name

| What Priya sees | Likely cause | Where you look |
|---|---|---|
| Login spins, then 503 | Identity is not ready, or the gateway cannot reach it | Ready check, database connections |
| Login returns 401 for a known good password | Wrong environment database, or the account is `DISABLED` | User row, which database the pod uses |
| Login works, every API returns 401 | Issuer URL or public key does not match the token | `iss`, JWKS, clock |
| APIs work for 15 minutes, then all 401 | Refresh is broken | `refresh_tokens`, gateway route to `/refresh` |
| One refund returns 403 | The token is valid and the role is not `SUPPORT` | Role rows, and whether this token was issued before the role change |
| Priya can open order 1002 | Ownership check is missing | Orders code and the test that uses two customers |
| After a deploy, random 401s | New replica has a different signing key | One shared key, or a JWKS that lists both during rotation |

Search can be down and login still works. Identity down means new logins fail. Existing access tokens keep working until they expire, because services check the signature locally. Those are two different outages. Say which one you are in before changing anything.

### 3.15 The story in one pass

1. Priya registers. Identity stores an Argon2id hash and the role `CUSTOMER`.
2. She logs in. She receives a 15-minute access token and a 7-day refresh token.
3. She searches for "kettle." Catalog does not need her token if the product list is public.
4. She adds the copper kettle. Cart writes Redis under `8f3c2a10-priya`.
5. She checks out. The gateway checks the signature. Orders checks `CUSTOMER` and writes order 1001 with her user id and Rs 2,499.
6. Orders uses the client secret to get a service token, reserves stock, and asks payments to capture Rs 2,499 with idempotency key `order-1001`.
7. A retry of that capture returns the first result.
8. Orders publishes `OrderPlaced`. Notifications send the receipt.
9. Sixteen minutes later her access token is dead. The website refreshes it and continues.
10. Arun refunds 1001 with a `SUPPORT` token. Priya's token on that URL receives 403.

---

## 4. The same login with Keycloak

This section is the comparison. **We do not install Keycloak for the current build.** The left-hand design in the picture is the one the later build prompt will follow.

![Without Keycloak, Harbor identity issues tokens. With Keycloak, Keycloak issues them. Shop services still only see a JWT.](images/12-with-without-keycloak.svg)

Keycloak is an open-source identity product. It speaks OAuth 2.0, OpenID Connect, and SAML. Companies run it when they want a ready-made login server. Okta and Microsoft Entra ID are the hosted products people compare it with. The words below are the same words those products use.

### 4.1 What moves, and what stays

| Concern | Without Keycloak, this build | With Keycloak, later |
|---|---|---|
| Who checks the password | Harbor identity, Java | Keycloak |
| Who stores the password hash | Identity's PostgreSQL | Keycloak's PostgreSQL |
| Who signs the access token | Harbor identity | Keycloak |
| What orders validate | JWT signature, issuer, expiry, role | The same checks, against Keycloak's issuer and keys |
| Shipping address | Shop profile table | Still the shop profile table |
| Order 1001 | Orders database | Still the orders database |
| Staff single sign-on from a company directory | Not in version one | A Keycloak strength. Add it there, not in homemade password code. |

Orders do not grow a special Keycloak mode. They already depend on a signed token. Only the issuer URL and the JWKS URL change.

### 4.2 The login Priya would see

![The browser logs in at Keycloak, then the shop calls orders with the same kind of access token.](images/13-keycloak-flow.svg)

This is the authorization-code flow with PKCE, which is the normal browser login in companies.

1. Priya clicks login.
2. The browser goes to Keycloak. She types her password on Keycloak's host, for example `https://auth.harbor.local`, not into a form the shop invented.
3. Keycloak redirects back with a short-lived code.
4. The website exchanges that code, plus a one-time PKCE proof, for an access token, a refresh token, and an ID token.
5. Orders receive the access token and run the same signature, issuer, expiry, and role checks.

**PKCE** is a secret the website creates at the start and proves at the end. It stops some other program from stealing the short code on the redirect and swapping it for Priya's tokens.

The shop application never receives her password. That separation is the main reason a company buys an identity product.

There is an older flow where the website posts her email and password to Keycloak itself. Keycloak calls it direct access grants. Leave it off. The browser redirect above is the flow to recognize.

### 4.3 Objects you would operate inside Keycloak

A **realm** is a walled space: its own users, clients, roles, and keys. Keycloak always has a realm named `master`. That realm is for administering Keycloak. Harbor's people would live in a realm named `harbor`. Building the shop inside `master` mixes admin power with customers.

A **client** is an application Keycloak knows about.

| Client | Type | Why |
|---|---|---|
| `harbor-web` | Public | The browser. It cannot keep a secret. PKCE protects the login. The redirect URL is an allow-list, such as `https://shop.harbor.local/callback`. |
| `harbor-orders` | Confidential | A server. It has a client secret and can ask for a service token. |

Public means "this app cannot hide a secret." Confidential means "this app runs on a server and can."

A **realm role** is `CUSTOMER`, `SUPPORT`, or `ADMIN`. A **group** such as `support-team` can hold the role `SUPPORT`. Adding Arun to the group gives him the role on his next token. A **client role** is narrower, attached to one client. Realm roles are enough to mirror Harbor version one.

**Mappers** decide which fields enter the token. Harbor needs `sub`, email, and roles. Extra profile fields make the token large and more likely to land in a log.

An **authentication flow** is the ordered list of steps: username, password, and later a one-time code. Adding a second factor is a change in that flow. The shop's order API stays as it is.

**Required actions** are one-off demands such as "replace the temporary password." If the mail server settings are wrong, Arun's invite never arrives and the shop looks broken. SMTP for Keycloak is an operations setting.

**Brute-force detection** slows repeated failures. Pair it with the gateway rate limit. Prefer slowdown over a lockout that an attacker can trigger against Priya.

### 4.4 Three tokens, company directories, and keys

Keycloak often returns three objects.

| Token | Use |
|---|---|
| Access token | Call Harbor APIs. Short-lived. |
| Refresh token | Ask Keycloak for a new access token. |
| ID token | Let the website say "hello, Priya." It is not the credential for `GET /orders`. |

An API that accepts the ID token in place of the access token is a review comment worth making.

**Identity brokering** means Keycloak trusts another login system. "Continue with Google" or "Continue with the company Entra ID" is brokering. Keycloak still issues the Harbor token. Orders still validate Harbor's issuer.

**User federation** means Keycloak reads employees from LDAP or Active Directory. The directory remains the source of staff. Customers usually stay as users inside the realm. You will meet federation on internal corporate platforms.

**Sessions** inside Keycloak are separate from the access-token lifetime. She can have a Keycloak session while the website refreshes access tokens. Keep the access token near 15 minutes even in a demo. A 24-hour access token hides the refresh flow and makes a stolen token useful all day.

Each realm has signing keys. The public half is on a JWKS URL. Rotation publishes the new key and keeps the old public key until old tokens expire. The `kid` header selects the key. The private keys live in Keycloak's database, so that database backup is also the key backup.

The **admin console** at `/admin` creates realms, clients, and users. It belongs on a restricted network. The **account console** is what Priya uses to change her password. The install-time admin password comes from a secret. A lab left on `admin` / `admin` and reachable from other machines is a real exposure, even if the shop data is fake.

### 4.5 What a DevOps engineer would run

| Piece | What good looks like |
|---|---|
| Database | Keycloak's own PostgreSQL, backed up, not shared with orders |
| Hostname | A fixed `https://auth.harbor.local`. Every token's issuer uses it. |
| Proxy | TLS ends at the gateway, and Keycloak is told the original scheme was HTTPS. A mistake here breaks redirects and cookies. |
| Time | Host clock is correct |
| Health | Ready check includes the database |
| Replicas | One node in a lab. Two nodes need the shared database and a shared session cache. |
| Upgrade | Back up first. Keycloak upgrades change the database schema. |
| Restore | Database restore plus a realm export of clients and roles |

The hostname is the setting that causes the long outages. Tokens say `iss: https://auth.harbor.local/realms/harbor`. If a service later reaches Keycloak at `http://keycloak:8080` and compares issuers, every call fails. Pick the public URL and use it everywhere.

When Keycloak is down, new logins fail. Services can keep accepting existing tokens while their cached JWKS is still valid. That is the same split as in section 3.14.

### 4.6 How the Priya story would read

1. She clicks login and the browser moves to realm `harbor`, client `harbor-web`.
2. Keycloak stores the password hash in its database and gives her realm role `CUSTOMER`.
3. She returns to the shop with an access token whose `sub` is Keycloak's id for her.
4. The profile table uses that `sub` for her Hyderabad address.
5. Checkout is unchanged: gateway, orders, service token, payment, event.
6. Arun is in the group `support-team`, so his token includes `SUPPORT`.

Nothing in catalog, cart, inventory, payments, search, or notifications needs Keycloak libraries if they already validate a JWT. That is why this document can explain Keycloak now and leave it out of the build.

---

## 5. What you should be able to say without notes

- Harbor Shop is a single-store e-commerce app. Priya buys, Arun refunds, Meera administers.
- Each service has its own data store. PostgreSQL for facts that must be correct, Redis for carts and short denylists, OpenSearch for text, Kafka for events.
- Authentication is who she is. Authorization is what she may do. 401 and 403 are different.
- The password is stored as an Argon2id hash with a per-user salt.
- The access token is a 15-minute JWT. The refresh token is a rotating secret stored as a hash.
- The gateway checks the signature. The service checks the role and, for customers, the owner of the row.
- Orders call payments with a service credential, not with Priya's token.
- The current build's issuer is Harbor identity. Keycloak would replace that issuer later. The shop services would keep checking a JWT.

---

## 6. Next chapter

The next note can go into one of these, at the same depth:

- Catalog and inventory: tables, stock reservation, and what you operate for two Go services
- The runtime contract every service must expose: port, health, config, logs, and a correlation id
- Orders, the saga, and the idempotency key on payment

The build prompt for Cursor comes after those chapters are as concrete as this one.
