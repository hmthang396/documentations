# Domain-Driven Design – Context Map

## What is a Context Map?

A **Context Map** is a visual representation that describes:

* The **existing Bounded Contexts** within a large system
* The **relationships between them** (integration patterns)
* The **direction of dependency (upstream/downstream)**: who influences whom, which teams can evolve independently, and which teams are impacted by others

It helps answer key **strategic questions**:

* How is the system currently divided?
* How do teams depend on each other (technically and organizationally)?
* Where are the risks (e.g. tight coupling, “polluted” domain models)?
* How should integration be designed to avoid a **Big Ball of Mud**?

A Context Map is **not just a technical diagram**.
It also reflects **organizational and team relationships**, making it especially valuable for **microservices architectures** and **legacy modernization initiatives**.

---

## Common Integration Patterns

*(from Eric Evans and the DDD community)*

These patterns are classified by **upstream (U – influencer)** and **downstream (D – influenced)** relationships.

| Pattern                     | Short Description                                                    | When to Use                                                            | Upstream / Downstream    |
| --------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------ |
| Partnership                 | Two contexts/teams collaborate closely and coordinate changes        | When both sides must evolve together (e.g., a large shared feature)    | Symmetric                |
| Shared Kernel               | A small shared part of the model/codebase; changes require agreement | When there is real overlap but independence is still desired elsewhere | Symmetric                |
| Customer/Supplier           | Upstream evolves to meet downstream needs                            | Upstream is stronger, downstream depends but has a voice               | U: Supplier, D: Customer |
| Conformist                  | Downstream fully accepts upstream’s model without translation        | Upstream is dominant (e.g., third-party), downstream cannot adapt it   | Strong U                 |
| Anti-Corruption Layer (ACL) | Downstream translates upstream models to protect itself              | Upstream is legacy or harmful; downstream must stay clean              | D protects itself        |
| Open Host Service (OHS)     | Upstream exposes a clear, standardized API/service                   | Many downstreams need to integrate                                     | Open U                   |
| Published Language          | Standardized, published schema/language (e.g., OpenAPI, JSON Schema) | External or multi-team integration                                     | U publishes              |
| Separate Ways               | No integration at all; full independence                             | Features are unrelated; avoid unnecessary coupling                     | None                     |
| Big Ball of Mud             | A messy, tangled area marked to prevent spread                       | Legacy chaos that must be isolated                                     | Warning                  |

---

## 1. Shared Kernel

**Shared Kernel** is a Context Mapping pattern where two or more Bounded Contexts share a **small, carefully defined subset of the domain model** (the “kernel”), which may include:

* Entities
* Value Objects
* Ubiquitous Language
* Even code implementations

This shared part **cannot be changed independently**.
Any modification requires **discussion and agreement** between the owning teams.

### Goals

* Avoid duplication when there is genuine domain overlap
* Maintain consistency in the shared core
* Still allow independence outside the kernel

This is a **symmetric pattern**: there is no clear upstream or downstream; the teams are equal partners.

### Real-World Example

In a large e-commerce system:

* **Sales Context**: manages sales, orders, promotions
* **Catalog Context**: manages products, categories, attributes

Both need a basic concept of **Product**:

* ProductId
* Name
* SKU
* Price

→ Create a Shared Kernel containing:

* `Product` class (Id, Name, SKU)
* `Money` value object
* Shared repository interfaces

---

## 2. Customer / Supplier

**Customer/Supplier** describes a clear dependency relationship between two Bounded Contexts (and their teams):

* **Upstream (U) – Supplier**: provides models, services, or data
* **Downstream (D) – Customer**: depends on the upstream to fulfill its business responsibilities

Unlike Shared Kernel or Partnership, this relationship is **asymmetric**.

The downstream has real needs, and the upstream **should evolve to support those needs**, like a good supplier serving its customer.

On a Context Map, the arrow usually goes **U → D** (upstream influences downstream).

### When to Use Customer/Supplier

* Downstream strongly depends on upstream to deliver its business capability
* Upstream is more powerful or represents a core domain
* Teams can collaborate well: downstream expresses needs, upstream prioritizes support
* You want to avoid complex translation layers (ACL) or forced conformity (Conformist)

### When NOT to Use

* Upstream ignores downstream needs → leads to bottlenecks
* Team relationships are tense → Conformist or ACL may be safer

### Example

In an e-commerce system:

* **Supplier (Upstream)**: Catalog Context (product management team)
* **Customer (Downstream)**: Sales Context (orders and checkout team)

Sales team:

> “We need a `DiscountPrice` field on Product to calculate promotions.”

Catalog team prioritizes and implements it because Sales is an important customer.

---

## 3. Conformist

**Conformist** is a pattern where the downstream context **fully accepts the upstream model** without any translation (no ACL).

* **Upstream (U)**: dominant, unwilling or unable to change
* **Downstream (D)**: must conform, even if the model does not fit its own ubiquitous language
* Arrow: **U → D**

This is a **pragmatic pattern** used when the downstream is weaker (organizationally, politically, or technically).

### Difference from Customer/Supplier

* Customer/Supplier: upstream actively supports downstream
* Conformist: upstream does not care; downstream adapts

### When to Use Conformist

* Upstream is a third-party system (e.g., Stripe, SAP)
* Upstream is a powerful core team that won’t adapt
* Downstream lacks time/resources to build an ACL
* Upstream model is “good enough”

Avoid it if the upstream model is truly harmful → use ACL instead.

### Example

* **Upstream**: Pricing Context (legacy pricing engine)
* **Downstream**: Sales Context

Pricing team refuses to change APIs.
Sales team conforms and uses Pricing’s model directly—even if field names don’t match Sales’ ubiquitous language.

Classic example: integrating with PayPal—you conform to their API and terminology.

---

## 4. Anti-Corruption Layer (ACL)

An **Anti-Corruption Layer (ACL)** is a **translation layer** built by the downstream to protect its domain model from being “corrupted” by upstream models.

“Corruption” means:

* Poor-quality legacy models
* Different ubiquitous language
* Anemic or unstable schemas
* Frequent upstream changes

ACL acts as a **protective wall**:

* Translates inbound models into clean downstream models
* Optionally translates outbound calls as well

Context Map flow: **U → ACL → D**

### Difference from Conformist

* Conformist accepts corruption
* ACL prevents it and keeps the downstream model expressive and clean

### When to Use ACL

* Upstream is legacy or third-party
* Models do not fit downstream language
* Upstream changes frequently
* Downstream is a core domain and must remain clean

### Example

A modern e-commerce system:

* **Downstream**: Sales Context (modern, core domain)
* **Upstream**: Legacy ERP Inventory Context using `ItemMasterRecord`

Without ACL → Sales code becomes polluted
With ACL:

* ACL receives legacy data
* Translates `ItemMasterRecord` → `Product`
* Applies transformation rules (e.g., derive Price from multiple legacy fields)
* Sales works only with its own clean `Product`

### Pros & Cons

**Pros**

* Protects domain purity
* Strong decoupling
* Ideal for legacy modernization

**Cons**

* Extra complexity
* Mapping maintenance required
* Overkill if upstream model is already good

### Typical ACL Components

* **Facade**: Simplified interface over complex external systems
* **Adapter**: Converts external interfaces into expected ones
* **Translator**: Core mapping logic (e.g., legacy status → domain enum)

---

## 5. Separate Ways

**Separate Ways** describes a situation where Bounded Contexts are **completely independent**:

* No dependencies
* No shared models
* No APIs or domain events
* Each context evolves independently

On a Context Map, these contexts appear unconnected or explicitly labeled “Separate Ways”.

This pattern provides **maximum autonomy** and is recommended when integration adds more risk than value.

### When to Use Separate Ways

* Contexts belong to entirely different subdomains
* Integration would create unnecessary coupling
* Supporting or generic domains unrelated to the core
* Large organizations aiming for team autonomy

### Example

In a large e-commerce company:

* **Sales Context**: orders, payments, shipping
* **HR Context**: recruitment, payroll
* **Monitoring Context**: internal logs and metrics

HR and Monitoring do not integrate with Sales → Separate Ways.

I’ve applied this successfully in fintech projects by isolating **Compliance Reporting** from core transaction systems.

---

## 6. Open Host Service (OHS)

### What is Open Host Service?

**Open Host Service (OHS)** is a pattern where an upstream context exposes a **clear, stable, well-defined service/API** for multiple downstream contexts.

* Upstream becomes an “open host”
* Interfaces may be REST, GraphQL, gRPC, messaging, etc.
* Often combined with **Published Language**
* Context Map: **U → many D**

### Goals

* Enable effective integration with loose coupling
* Allow downstreams to consume services without knowing internal models
* Support scale and evolution

### Published Language – the Companion of OHS

OHS works best with **Published Language (PL)**:

* A standardized, documented schema or language
* Versioned and publicly shared
* Examples: OpenAPI, JSON Schema, Protobuf, OpenID Connect

Without PL, OHS degrades into unstable, ad-hoc APIs.

### When to Use OHS

* Upstream is a core or platform domain
* Many downstream consumers
* Microservices or SOA environments
* Need long-term evolvability

### Example

In e-commerce:

* **Upstream**: Catalog Context
* **Downstreams**: Sales, Recommendation, Search

Catalog exposes:

* REST API `/products/{id}`
* Published Language with standardized JSON schema

Downstreams integrate via API or domain events without knowing internal Catalog models.

I’ve used OHS in fintech systems where an **Identity Context** exposed a GraphQL OHS based on OpenID Connect standards, enabling 10+ services to integrate safely.

### Pros & Cons

**Pros**

* Easy integration for many consumers
* Reduced coupling
* Supports versioning and evolution

**Cons**

* Requires strong API design discipline
* Governance and documentation overhead
* Overkill for small systems

---
