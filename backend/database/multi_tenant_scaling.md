# Multi-Tenant Data at Scale

**Pitfalls of the **``** Pattern, Real-Life Scenarios, Tradeoffs, and Enterprise-Grade Designs (with Mongoose Examples)**

> This article is a generic, vendor-neutral overview you can share publicly.\
> It covers where `tenantId`-based designs tend to break down, how to avoid common traps, what tradeoffs exist, and what a robust enterprise-level architecture looks like. Includes **real-life scenarios** and **Mongoose code samples**.

---

## Executive Summary

- **Row-level multi-tenancy (a shared collection with a **``** field)** works for early stages but creates issues at scale.
- Risks include **leaky queries**, **index growth**, **tenant hot-spots**, **noisy neighbors**, and **compliance requirements**.
- Enterprise-grade approaches balance tradeoffs: **shared collections for cost efficiency**, **partitioning/sharding for scale**, and **dedicated databases for large or regulated tenants**.

---

## 1. The Baseline: The `tenantId` Row Pattern

**Definition:** All tenants share the same collections; every document has a `tenantId`. All queries must filter by tenant.

### Why teams start here

- 🚀 **Fast delivery:** only one schema/collection.
- 💰 **Low cost:** one database, minimal infra.
- 📊 **Easy analytics:** cross-tenant queries are trivial.

### Real-Life Example (Mongoose)

```js
const orderSchema = new Schema({
  tenantId: { type: String, index: true },
  product: String,
  status: String,
  createdAt: { type: Date, default: Date.now }
});

const Order = mongoose.model('Order', orderSchema);

// Query always needs tenantId filter
const orders = await Order.find({ tenantId, status: 'active' })
  .sort({ createdAt: -1 })
  .limit(50);
```

### Where it fails in practice

1. **Leaky Queries** – forget the tenant filter, and one tenant sees another’s orders.
2. **Index Bloat** – global indexes grow too large; queries slow down.
3. **Hot Tenants** – one enterprise customer generates 80% of the data, hurting others.
4. **Backups/Restore** – restoring only one tenant’s data is nearly impossible.
5. **Compliance** – “all tenants in one DB” fails for GDPR or HIPAA residency.

---

## 2. Hardening the `tenantId` Approach

### 2.1 Enforce Tenant Context

- Middleware injects tenant into all queries.
- Use **Mongoose plugins** to auto-append tenantId.

```js
function tenantPlugin(schema) {
  schema.pre(['find', 'findOne', 'updateOne', 'deleteOne'], function() {
    if (!this.getQuery().tenantId) {
      this.where({ tenantId: this.options.tenantContext });
    }
  });
}

orderSchema.plugin(tenantPlugin);
```

➡️ This prevents devs from forgetting tenant filters.

### 2.2 Indexing & Partitioning

- Compound index `(tenantId, status, createdAt)` speeds queries.
- **MongoDB sharding:** shard key `(tenantId, hashed)` to spread tenants.
- **Tradeoff:** global aggregations (all tenants) become more expensive.

### 2.3 Lifecycle & Archiving

- Archive old documents to S3 or a cold cluster.
- **Tradeoff:** historical queries are slower but hot collections stay lean.

### 2.4 Compute Isolation

- Use **separate queues** for heavy tenants.
- **Tradeoff:** more infra overhead but avoids noisy neighbors.

---

## 3. Alternatives and Tradeoffs

| Model                          | Strengths                              | Weaknesses                 | Real-Life Scenario                 |
| ------------------------------ | -------------------------------------- | -------------------------- | ---------------------------------- |
| Shared collection (`tenantId`) | Cheap, simple, cross-tenant analytics  | Risk of leaks, index bloat | SaaS MVP with 100+ small startups  |
| Database per tenant            | Hard isolation, per-tenant backups     | Expensive, ops heavy       | Enterprise SaaS with 10 banks      |
| Sharded shared collection      | Horizontal scale, bounded blast radius | Global queries harder      | Multi-region SaaS with 10k tenants |
| Hybrid tiered model            | Flexibility, promote big tenants       | Complexity in routing      | Mix of SMBs and a few whales       |

**Tradeoff rule of thumb:**

- Small tenants → shared collections.
- Medium tenants → shard or schema split.
- Large/regulated tenants → dedicated DB.

---

## 4. Enterprise-Level Blueprint

### Control Plane vs. Data Plane

- **Control Plane:** tenant registry, routing, billing, residency policy.
- **Data Plane:** sharded or per-tenant clusters.

```
[ Gateway ] → [ Control Plane ] → [ Cluster A (SMBs) ]
                                 → [ Cluster B (Enterprise) ]
```

### Real-Life Tradeoffs

- ✅ Easy scaling by moving “whales” to dedicated clusters.
- ❌ Higher cost if many tenants demand isolation.

---

## 5. Traps & Safer Patterns

| Trap                         | Real-Life Result                  | Safer Pattern                       |
| ---------------------------- | --------------------------------- | ----------------------------------- |
| Missing tenant filter        | Tenant A sees Tenant B’s invoices | Enforce filters with plugins or RLS |
| Index w/o tenantId           | Slow queries, collection scans    | Compound index `(tenantId, field)`  |
| One hot tenant in shared DB  | Everyone else slows down          | Promote tenant to dedicated DB      |
| Topic-per-tenant in Kafka    | Millions of topics, infra pain    | Shared topics with tenant routing   |
| Full DB restore for 1 tenant | Hours of downtime                 | Logical export/import by tenantId   |

---

## 6. Mongoose Implementation Patterns

### Tenant-Aware Base Model

```js
class TenantModel {
  constructor(model, tenantId) {
    this.model = model;
    this.tenantId = tenantId;
  }

  async find(query) {
    return this.model.find({ ...query, tenantId: this.tenantId });
  }

  async create(doc) {
    return this.model.create({ ...doc, tenantId: this.tenantId });
  }
}

const orderTenant = new TenantModel(Order, 'tenant123');
await orderTenant.create({ product: 'Widget', status: 'active' });
```

### Sharding with MongoDB

```js
sh.shardCollection("orders", { tenantId: "hashed" });
```

---

## 7. Migration & Rebalancing

**Scenario:** A startup tenant grows into an enterprise customer.

1. Provision new dedicated cluster.
2. Backfill tenant data.
3. Switch routing in control plane.
4. Archive old data in shared cluster.

**Tradeoff:** Simpler ops if everyone stays shared, but you risk one whale slowing down 10,000 SMB tenants.

---

## 8. Checklist

-

---

## Closing Thoughts

- The `tenantId` approach is great to start but dangerous to scale blindly.
- In real life, **small tenants are cheap in shared collections**, but **big tenants need isolation**.
- With **Mongoose plugins, sharding, and promotion strategies**, you can scale from MVP to enterprise SaaS without re-architecting from scratch.

