# Bounded Context in Domain-Driven Design (DDD)

## 1. What Is a Bounded Context?

A **Bounded Context** is an **explicit boundary** within which:

* A **specific Ubiquitous Language** applies
* Domain models have **clear and unambiguous meaning**
* Rules, assumptions, and definitions are consistent

> **Same word, different meaning → different Bounded Contexts**

Bounded Context is **not a technical boundary first** (not a database, not a microservice),
it is a **language and meaning boundary**.

---

## 2. Why Bounded Context Exists

Without Bounded Context:

* Teams argue about definitions
* Models become bloated and inconsistent
* One change breaks unrelated parts of the system
* Business logic becomes “if-else hell”

Bounded Context solves this by **allowing the same term to exist with different meanings — safely**.

---

## 3. Core Rule of Bounded Context

> **Inside a Bounded Context, language must be consistent.
> Across Bounded Contexts, language may differ.**

Violating this rule leads to **semantic coupling**, the most dangerous kind of coupling.

---

## 4. Concrete Example: “Order”

### Business Reality

The word **Order** is overloaded.

### Split into Bounded Contexts

#### Sales Context

* Order = customer purchase intent
* Focus: pricing, promotions, checkout

```text
Order
OrderItem
Discount
Cart
Checkout
```

#### Shipping Context

* Order = package to be delivered
* Focus: logistics, delivery status

```text
Shipment
Package
DeliveryRoute
Carrier
TrackingNumber
```

#### Accounting Context

* Order = financial obligation
* Focus: invoices, payments, tax

```text
Invoice
Payment
LedgerEntry
TaxCalculation
```

Each context has:

* Its **own models**
* Its **own database (ideally)**
* Its **own Ubiquitous Language**

---

## 5. Bounded Context vs Microservices

### Important Clarification

❌ **Bounded Context ≠ Microservice**
✅ **Microservice should align to a Bounded Context**

| Concept         | Purpose                       |
| --------------- | ----------------------------- |
| Bounded Context | Language & model boundary     |
| Microservice    | Deployment & runtime boundary |

You can:

* Start with **multiple bounded contexts in one monolith**
* Later extract them into microservices

This is called **DDD-first, architecture-second**.

---

## 6. How to Identify Bounded Contexts

### 6.1 Listen for Language Breaks

Signals:

* “In this case, order means…”
* “That’s not the same order”
* “Depends on who you ask”

These are **context boundaries trying to emerge**.

---

### 6.2 Look for Rule Conflicts

Example:

* Sales allows cancellation anytime
* Shipping forbids cancellation after pickup

Same entity → conflicting rules → **split contexts**.

---

### 6.3 Organizational Boundaries

Often (not always):

* One team = one bounded context
* Different KPIs = different context

Conway’s Law applies whether you like it or not.

---

## 7. Evolution of Bounded Contexts

Bounded Contexts are **discovered, not designed upfront**.

Typical evolution:

1. Start with large context
2. Identify conflicts
3. Extract smaller contexts
4. Introduce explicit integration

Refactoring contexts is **normal and expected**.

---

## 8. Key Takeaways

* Bounded Context protects **meaning**
* Language consistency is more important than reuse
* Integration should be **explicit, not implicit**
* Context boundaries enable independent evolution

> **If Ubiquitous Language defines meaning,
> Bounded Context protects it from corruption.**

---
