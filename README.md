# Focal Point · Case Study

> **Multi-tenant ERP SaaS for Paraguayan SMBs, with conversational AI agents over WhatsApp and government-grade electronic invoicing.**

> *This is a public case study. The source code is private. Architecture diagrams, technical decisions, and tradeoffs are described in enough detail to evaluate the engineering work without exposing proprietary code.*

---

## TL;DR

Focal Point is a production-oriented ERP + ecommerce SaaS that consolidates inventory, sales, CRM, HR, finance, electronic invoicing, a per-tenant storefront with online payments, and AI-powered customer interactions into a single multi-tenant platform. It was built to give Paraguayan SMBs an integrated alternative to the typical mix of spreadsheets, legacy desktop ERPs, disconnected WhatsApp conversations, and ad-hoc Tiendanube-style storefronts, while remaining compliant with SIFEN (the national e-invoicing system).

**Stack:** Django 5.x · DRF · Channels · Next.js · PostgreSQL 17 (schema-per-tenant via `django-tenants`) · Valkey/Redis · LangGraph + OpenAI · Meta WhatsApp Cloud API · PagoPar · AWS (EC2, RDS, ElastiCache, S3, CloudFront, SES) · Terraform · GitHub Actions · Sentry + Prometheus + Grafana.

**Status:** Production system. Core ERP modules, per-tenant ecommerce storefront, and the WhatsApp AI agent are live, running on a single AWS region for active tenants in retail and services.

**Role:** Co-founder & Founding Engineer. I co-founded Focal Point with my partners and led engineering end-to-end.

**My tenure:** Dec 2023 – Mar 2026. The company continues to operate; this case study covers the system as I built and ran it.

---

## 1. Business problem

Paraguayan SMBs operate under a specific set of constraints that off-the-shelf ERPs handle poorly:

- **SIFEN compliance is mandatory.** Since 2023, the Paraguayan tax authority (SET) requires electronic invoicing through SIFEN. Non-compliant invoices are not legally valid. International ERPs either don't support SIFEN at all or charge enterprise pricing for the integration.
- **WhatsApp is the dominant communication channel.** Customers expect to place orders, ask about products, and follow up on deliveries through WhatsApp rather than email or a customer portal. Sales teams already operate on WhatsApp; the data just isn't captured anywhere.
- **Existing tools are fragmented.** A typical SMB stack looks like: a desktop accounting tool from the 2000s, Excel for inventory, WhatsApp for sales, a separate billing tool for SIFEN, and no shared source of truth.
- **Price sensitivity is high.** Enterprise ERPs (SAP, Oracle, Odoo Enterprise) are out of reach. Self-hosted Odoo requires technical staff most SMBs don't have.

The opportunity: a multi-tenant SaaS that bundles the operational basics (inventory, sales, CRM, HR, finance, reporting), is SIFEN-native, and treats WhatsApp as a first-class interface, with AI agents that can answer customer questions and capture orders without human intervention.

---

## 2. Constraints

The constraints shaped the architecture more than the requirements did:

- **Strict tenant isolation.** Tenants are competing businesses; cross-tenant data leakage is an existential risk.
- **SIFEN integration is non-negotiable.** Invoices must be signed, transmitted, and reconciled with SET on a per-tenant basis (each tenant has its own RUC and signing certificate).
- **Cost ceiling.** Infrastructure must run on a single small EC2 instance and `db.t4g.micro` RDS in early stages, with a clear path to scale horizontally without rewriting the data layer.
- **AI cost discipline.** Every WhatsApp message could trigger an LLM call. Model choice and prompt structure had to be cost-aware from day one.
- **Solo-friendly operations.** The system needed to be operable by a small team: reproducible infrastructure, structured logs, real observability, and zero manual deployment steps.

---

## 3. System architecture

### 3.1 Multi-tenancy: schema-per-tenant

Tenant isolation is implemented via PostgreSQL schemas using `django-tenants`. Each tenant gets its own schema; the `public` schema holds shared models (tenant registry, plans, billing).

```
PostgreSQL
├── public                  ← tenant registry, plans, platform billing
├── tenant_acme             ← Acme S.A. — products, sales, invoices
├── tenant_globex           ← Globex SRL — products, sales, invoices
└── tenant_initech          ← Initech SAS — products, sales, invoices
```

Tenants are routed by subdomain (`<tenant>.<platform>`). A `TenantMainMiddleware` resolves the subdomain to a schema and sets the active connection schema for the request.

**Why schema-per-tenant instead of row-level (`tenant_id` on every table)?**

- **Stronger isolation by default.** A bug in a query filter cannot leak data across tenants: the connection is physically scoped to a schema.
- **Per-tenant operations are clean.** Backups, restores, exports, deletions, and migrations can be performed on a single tenant without touching the others.
- **Compliance posture is easier to explain** to customers worried about their competitors using the same system.

**Tradeoffs accepted:**

- **Migrations are slower.** A schema migration runs once per tenant. At hundreds of tenants this becomes minutes; at thousands it becomes a problem. Mitigation: migrations are run in parallel batches, and breaking changes are gated behind feature flags.
- **Connection pooling is more complex.** Each schema effectively requires its own search path. Mitigation: middleware sets `search_path` per request; no persistent per-tenant connections.
- **Cross-tenant analytics requires a separate path.** Solved via a dedicated reporting layer in the `public` schema fed by per-tenant aggregates.

### 3.2 Application layers

Rather than one monolithic API, the backend is organized into four explicit layers, each with its own auth model, URL prefix, and middleware chain:

| Layer | Domain | Auth | API prefix | Purpose |
|---|---|---|---|---|
| **ADMIN** | `admin.<platform>` | Sessions (`BackofficeUser`) | `/api/v1/admin/` | Platform staff: tenant provisioning, billing, analytics |
| **ERP** | `<tenant>.<platform>` | JWT cookie (`User`) | `/api/v1/erp/` | Tenant employees: full ERP functionality |
| **SITE** | `<tenant>.<platform>` | Anonymous (optional account) | `/api/v1/site/` | Public storefront: catalog, cart, anonymous checkout, order tracking |
| **PUBLIC** | `<platform>` | None | `/` | Webhooks (SIFEN, PagoPar, WhatsApp), OAuth |

Layers are enforced by an `AppLayerMiddleware` + `LayerAuthMiddleware` pair: the layer is determined from the subdomain, and the auth backend / permission classes are selected accordingly. A request to the wrong layer for the wrong subdomain is rejected before it reaches a view.

**Why separate layers instead of one API with role-based permissions?**

- The four contexts have **fundamentally different security models** (session vs JWT vs anonymous vs none). Mixing them in one auth pipeline invited bugs.
- The **blast radius of a permission mistake is contained**. A bug in storefront code can't accidentally expose ERP endpoints, because they share no URL prefix.
- **Documentation and testing are simpler**: each layer gets its own OpenAPI schema and its own permission test suite.

### 3.3 Request flow

```
Request
  └─> TenantMainMiddleware       resolve subdomain → schema
      └─> AppLayerMiddleware     determine layer (ADMIN/ERP/SITE/PUBLIC)
          └─> LayerAuthMiddleware  apply layer-appropriate auth
              └─> ContextMiddleware  attach user/tenant/branch to thread-local
                  └─> View
                      └─> Serializer
                          └─> Service (business logic)
                              └─> BranchAwareManager (filters by active branch)
                                  └─> Model
```

`BranchAwareManager` is a custom Django manager that auto-filters querysets by the active `Branch` (most ERP entities, like stock, sales and employees, belong to a branch). This keeps view code branch-agnostic and prevents the "I forgot to filter by branch" class of bugs.

---

## 4. Key technical decisions

### 4.1 Why Django over FastAPI or Node?

- **The admin and the ORM pay for themselves.** A multi-module ERP has hundreds of models and dozens of internal tools. The Django admin handles the long tail of one-off operational needs without a line of frontend code.
- **`django-tenants` is the most mature multi-schema multi-tenancy implementation** in any ecosystem we evaluated.
- **DRF + drf-spectacular** gives us OpenAPI schemas per layer essentially for free.
- **Trade-off:** Django's async story is weaker than FastAPI's. Mitigated by using **Django Channels for WebSockets** (real-time chat updates) and **Celery for background work** (SIFEN submissions, heavy reports, AI agent runs).

### 4.2 Why EC2 + Terraform over ECS or Kubernetes?

For the current scale (single instance, single region, single-digit active tenants), EC2 with systemd-managed services is the right level of complexity:

- **Operational surface is small.** One instance, one Nginx, one Gunicorn, one Uvicorn (Channels), Celery worker, Celery beat. All managed by systemd. Logs go to Vector → CloudWatch.
- **Deploys are boring.** GitHub Actions handles deploys: `git pull` + `systemctl restart`. No image registry, no container orchestrator, no surprises.
- **Terraform handles everything else.** 8+ modules (network, security groups, RDS, EC2, ALB, S3, ElastiCache, CloudFront, ASG, monitoring) provision the full stack reproducibly.
- **The migration path to ECS is preserved.** The Django app is already container-friendly; moving to ECS is a packaging change rather than a rewrite.

What we'd do differently at scale: introduce ECS Fargate as soon as we need a second instance for HA, and move the database migration step out of the deploy script into a separate gated job.

### 4.3 AI agents: LangGraph over a single prompt

The WhatsApp agent isn't a single LLM call: it's a small graph with explicit nodes for intent classification, retrieval over the tenant's catalog and knowledge base, action execution (creating quotes, checking stock), and response generation.

- **Channel:** Meta WhatsApp Cloud API. The original integration shipped on Twilio and was later migrated to Meta's API directly.
- **LangGraph** gives us a real state machine, persisted between turns. The agent can hand off, ask clarifying questions, and resume mid-workflow.
- **Memory is per-tenant per-conversation**, stored in Redis with explicit TTLs.
- **Cost discipline:** cost-aware model routing with a deterministic fallback chain for high-volume cheap nodes. Each call is logged with token counts and estimated USD cost, surfaced in the platform admin dashboard.
- **Safety:** the agent can read most of the tenant's data but can only write through a small allowlist of tools (create quote, request human handoff). Anything financial requires human confirmation in the ERP.

### 4.4 SIFEN: integration as a separate domain

Electronic invoicing is its own bounded context inside `apps/integrations/sifen/`:

- Tenants upload their SET-issued signing certificate; it's stored encrypted in S3.
- Invoice creation in `apps/sales/` emits a domain event; the SIFEN integration consumes it, signs the document, submits it, and reconciles the response asynchronously.
- Failures are surfaced both in the ERP UI (per invoice) and as platform-level alerts (when SET's API itself is degraded).

Keeping SIFEN out of the sales code path was important: SET's APIs go down, change schemas, and have undocumented behavior. The integration layer absorbs that volatility.

### 4.5 Ecommerce storefront: same backend, new surface

What started as a bare-bones `SITE` layer evolved into a full per-tenant storefront: every tenant gets a public catalog and checkout at their subdomain, sharing the same product, inventory, and pricing data as the ERP.

- **One source of truth.** The catalog the storefront serves is the same `apps/inventory/` data the ERP edits. No sync job, no eventual consistency. Stock decremented at checkout reflects immediately in the ERP's inventory view.
- **Anonymous-first checkout.** Most LATAM ecommerce shoppers don't want an account, so the storefront supports anonymous orders end-to-end. Optional account creation lives on top, outside the critical path.
- **PagoPar for payments.** Integrated via webhook into the existing `apps/integrations/` pattern (same shape as SIFEN). Payment status drives order state via domain events; the AI agent and the ERP both see updates without polling.
- **Order → invoice flow is unified.** An ecommerce order and a sales-team order both hit the same `apps/sales/` service, which means SIFEN invoicing, stock reservation, and customer records work identically regardless of channel.
- **Frontend stack:** Next.js (SSR for SEO + sub-second TTFB on product pages), reusing the design tokens and component library from the ERP frontend.

**What the layered architecture bought us:** building ecommerce didn't require new auth, billing, or invoicing infrastructure. The `SITE` layer was already isolated in the middleware chain, anonymous-by-default, and pointed at the same tenant schemas. Most of the work was product surface instead of platform plumbing.

### 4.6 Observability from day one

Wired in from the first deploy:

- **Sentry** for exceptions, with tenant context attached to every event.
- **Silk** for SQL profiling in development and staging.
- **Prometheus + Grafana** for request rates, latency percentiles, Celery queue depth, AI cost per tenant.
- **Vector → CloudWatch** for structured log shipping.

This pays off operationally every week. The most common cause of a production issue is a tenant-specific data shape we didn't anticipate; tenant-tagged error reports turn an hour of investigation into a five-minute fix.

---

## 5. Infrastructure (production)

```
                        Internet
                           │
                       CloudFront
                           │
                       ALB (HTTPS)
                           │
                ┌──────────┴────────────┐
                │  EC2 application instance   │
                │                             │
                │  Nginx ──┬─ Gunicorn   (ERP / admin / SITE API)
                │          ├─ Uvicorn    (Channels / WebSockets)
                │          └─ Next.js    (ERP + storefront frontends)
                │  Celery worker
                │  Celery beat
                └─────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   RDS PostgreSQL    ElastiCache         S3 + SES
   (small instance)  Valkey              (media, certificates,
                     (serverless)         transactional email)
```

All provisioned via Terraform (`infra/modules/`). Environments (`production/`, `staging/`) share modules; only variables differ.

---

## 6. Current status

- **In production** with active tenants in retail and services.
- **Live modules:** inventory, sales, CRM, finance basics, electronic invoicing (SIFEN), per-tenant ecommerce storefront with PagoPar checkout, WhatsApp AI agent, platform backoffice.
- **Operational baseline:** all SIFEN invoices reconciled asynchronously and surfaced per-tenant; AI cost tracked per call and visible in the platform admin dashboard; tenant-scoped error reporting via Sentry with per-tenant tagging.

---

## 7. What we'd do differently

Honest reflection: these are the decisions we'd revisit:

- **The deploy runner was initially co-located with the application instance**; moved off to a dedicated host to remove the coupling between deploy and runtime health.
- **No automatic migrations on deploy** is a deliberate choice (safer for multi-schema migrations) but requires a manual step that's easy to forget. A gated migration job in CI would close the gap.
- **The agent's tool surface grew organically**; it would benefit from a more formal tool registry with per-tool permissions and rate limits, rather than ad-hoc allowlists.
- **The storefront launched against a backend tuned for ERP write patterns**; anonymous read traffic exposed the lack of a read-replica strategy. Mitigated for now with CloudFront caching of product pages and per-tenant cached responses, but a dedicated read-replica path is on the roadmap.

---

## 8. Why this project matters for what I do

Focal Point is the project where backend, infrastructure, AI, and product judgment all had to come together. The interesting work wasn't any one piece: it was deciding *which problems deserved sophistication and which deserved boring solutions*. Multi-tenancy, SIFEN, and AI cost control got real architectural attention. Deploys, monitoring, and the storefront got the simplest thing that works.

That balance is what I bring to engineering work: enough seniority to know when to over-engineer, and enough discipline to know when not to.

---

## Contact

Franco Cazal · Co-founder & Founding Engineer, Focal Point (Dec 2023 – Mar 2026)

- Portfolio · [francocazal.com](https://francocazal.com)
- Email · [contact@francocazal.com](mailto:contact@francocazal.com)
- LinkedIn · [linkedin.com/in/franco-cazal](https://www.linkedin.com/in/franco-cazal)
- GitHub · [@FrancoCazal](https://github.com/FrancoCazal)
