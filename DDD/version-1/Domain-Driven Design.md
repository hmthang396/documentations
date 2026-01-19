# Domain-Driven Design (DDD): The Complete Engineering Handbook

## 1. Introduction to Domain-Driven Design (DDD)

### 1.1. What Problems Does DDD Solve?

In most software projects, the biggest source of pain isn't technology — it's **misunderstanding the business**.

I've seen this repeatedly in real systems:

- A large e-commerce platform where "Order" meant different things to the warehouse team, billing team, and customer service. The code reflected this confusion: placing an order sometimes updated inventory, sometimes didn't; cancellations behaved differently depending on who triggered them.
- A healthcare system where "Patient Visit" was modeled as a simple database row, but nurses, doctors, billing, and insurance all had different rules about what could happen during a visit. Changes in one department constantly broke features in another.
- A financial trading system where developers built beautiful technical abstractions (messages, queues, caches), but traders couldn't trust the numbers because the code didn't capture the real rules of risk and position management.

These problems share a common root: **the software model doesn't match the real-world business domain it's supposed to serve**. Requirements are translated poorly, communication between business experts and developers breaks down, and the codebase becomes a tangled mess that resists change.

DDD attacks this problem head-on by making the **business domain** the central focus of software design.

### 1.2. Why Does DDD Exist? The History and Motivation

In 2003, Eric Evans published *Domain-Driven Design: Tackling Complexity in the Heart of Software*. He didn't invent the ideas from scratch — he distilled patterns that were already working in successful projects across multiple companies.

The motivation came from the early 2000s reality:

- Object-oriented programming was mainstream, but most teams were using it to build **technical layers** (UI, database access, services) rather than rich business models.
- Enterprise applications were growing enormously complex.
- Anemic domain models (objects with only data and no behavior) were everywhere — leading to business logic scattered in services, controllers, and stored procedures.
- Teams were delivering software that worked technically but failed to deliver real business value or adapt to change.

Evans observed that the most successful projects had one thing in common: developers and domain experts (business stakeholders) had built a **shared understanding** of the core domain, and that understanding was reflected directly in the code.

DDD was born to make that success repeatable.

### 1.3. When Is DDD a Good Fit?

DDD shines when:

- The software is **core to the business** — it provides competitive advantage or is mission-critical.
- The domain is **complex** — there are intricate business rules, workflows, exceptions, and variations.
- The system needs to **evolve over years** with changing business needs.
- You have **access to domain experts** who can collaborate closely with developers.

Concrete examples where I've applied DDD successfully:

- High-volume e-commerce order management and fulfillment (hundreds of rules around promotions, taxes, inventory allocation, shipping restrictions).
- Risk management and trading desks in investment banks (complex calculations around positions, limits, margin, regulatory rules).
- Multi-tenant SaaS products with sophisticated pricing, billing, and entitlement logic.
- Logistics and supply chain platforms coordinating warehouses, carriers, and retailers.

In these cases, investing in DDD paid off because the cost of misunderstanding or hard-to-change code was extremely high.

### 1.4. When Is DDD NOT a Good Fit?

DDD is **not** a silver bullet. Avoid it when:

- The problem is **simple or generic** — CRUD applications, admin dashboards, content management, reporting tools.
- You're building a **short-lived prototype or MVP** where speed of delivery matters far more than long-term maintainability.
- You **lack domain experts** or they can't invest time in collaboration.
- The team is **small and inexperienced** and needs to ship working software fast.
- The domain is **well-solved by off-the-shelf solutions** (e.g., use Stripe for payments instead of building your own payment engine).

Real examples where I deliberately avoided full DDD:

- Internal HR tool for managing employee records and leave requests — simple CRUD with straightforward rules.
- Marketing website with forms and content — used a CMS.
- Data migration scripts and ETL pipelines — transactional scripts were clearer.
- Early-stage startup MVP focused on validating market fit — we used simple layered architecture to ship fast.

Trying to force DDD into these contexts creates unnecessary complexity and slows delivery.

### 1.5. Common Misconceptions About DDD

**Misconception 1: DDD = Entities, Aggregates, Repositories, etc. (the tactical patterns)**

Reality: Those are just tools. The heart of DDD is **strategic design** — understanding the domain, finding bounded contexts, and building a shared language. Tactical patterns are useful only when applied in service of a good strategic model.

**Misconception 2: DDD means everything must be "pure" objects with no data layers**

Reality: Many successful DDD projects use relational databases, REST APIs, message queues. DDD is about where the important behavior lives, not about banning certain technologies.

**Misconception 3: DDD requires microservices**

Reality: DDD predates microservices by a decade. You can apply DDD perfectly well in a monolith. Conversely, you can have microservices without any DDD at all (and many do).

**Misconception 4: Once you do Event Storming or draw context maps, you're "doing DDD"**

Reality: Diagrams are helpful, but DDD lives in the code and in ongoing conversations. If the code doesn't reflect deep domain insights, you're not really doing DDD.

**Misconception 5: DDD is only for greenfield projects**

Reality: Some of the biggest wins come from applying DDD concepts to legacy systems — identifying bounded contexts hidden inside a big ball of mud, strangling parts gradually.

### 1.6. The Core Idea: Focus on the Domain

At its essence, DDD says:

1. Identify the **core domain** — the part of the business that truly matters and differentiates you.
2. Invest heavily in understanding it deeply through close collaboration with domain experts.
3. Build a **shared language** (called the Ubiquitous Language) that both business and technical people use.
4. Reflect that language and understanding directly in code.
5. Protect the integrity of that model by defining clear boundaries.

Everything else in DDD — bounded contexts, aggregates, domain events, etc. — exists to support this goal.

When done well, the codebase becomes a clear, changeable representation of the business itself. New developers understand the system faster. Business stakeholders recognize their rules in the software. Changes become safer and faster.

That's the promise of DDD — and in the right projects, it's absolutely worth the investment.

# Core Concepts of Domain-Driven Design – Explained for Beginners

## 1. Domain

**Simple definition**  
The domain is the real-world business problem you are trying to solve with software. It includes all the activities, rules, processes, and knowledge that the business cares about.

**Real-world analogy**  
Think of a restaurant. The domain is everything about running a restaurant: taking orders, preparing food, managing ingredients, seating customers, handling payments, and complying with health regulations.

**Business example (e-commerce)**  
In an online store, the domain covers everything related to selling products: browsing catalogs, adding items to a cart, applying discounts, checking out, processing payments, managing inventory, packing orders, and shipping them.

**Why it matters in real projects**  
Most software fails or becomes hard to maintain because developers focus on screens, databases, or APIs first and treat the business rules as an afterthought. By putting the domain at the center, the software becomes a true reflection of the business, making it easier to understand, change, and extend over years.

**Common beginner mistake**  
Thinking the domain is just “data” (e.g., products, orders, customers). The domain is mostly about behavior and rules — what can happen, when, and why.

## 2. Subdomain

**Simple definition**  
A subdomain is a smaller, distinct part of the overall domain. We split the big domain into subdomains to make it easier to reason about.

There are three types:

- **Core subdomain** – The part that makes the company unique and provides competitive advantage.
- **Supporting subdomain** – Needed for the business to function, but not a differentiator.
- **Generic subdomain** – Common problems that many companies face and that have existing solutions.

**Real-world analogy**  
In a ride-hailing app like Uber:
- Core: Matching riders with drivers in real time, dynamic pricing (surge).
- Supporting: Driver background checks, vehicle registration.
- Generic: Sending email notifications, processing credit card payments.

**Business example (online hotel booking platform)**  
- Core: Searching available rooms, applying complex pricing rules (seasonal rates, loyalty discounts, cancellation policies), managing overbooking strategies.
- Supporting: Customer profile management, review system.
- Generic: User authentication, email/SMS notifications, basic payment processing.

**Why it matters in real projects**  
You cannot invest the same effort everywhere. Identifying the core subdomain tells you where to put your best people and deepest modeling effort. Supporting and generic subdomains can often be simplified, outsourced, or bought off-the-shelf — saving time and money.

**Common beginner mistake**  
Treating everything as equally important. Teams waste months over-engineering generic features (e.g., building a custom authentication system) instead of focusing on what truly differentiates the business.

## 3. Ubiquitous Language

**Simple definition**  
A shared vocabulary that both business experts and developers use when talking about the domain. The same terms are used in conversations, documents, and eventually in the code.

**Real-world analogy**  
In a hospital, doctors and nurses use precise terms like “triage”, “vital signs”, “discharge”. Everyone understands them the same way. If a new nurse called “discharge” something else, confusion and mistakes would happen.

**Business example (banking)**  
In a banking system, everyone agrees that:
- An “Account” can be in states like Active, Frozen, or Closed.
- “Transfer” means moving money from one account to another with specific rules (daily limits, fraud checks).
Business people use these words in meetings, developers use the same words in design discussions and naming.

**Why it matters in real projects**  
Miscommunication is the #1 cause of bugs and rework. When the business says “cancel order” but developers interpret it differently from warehouse staff, the software does the wrong thing. Ubiquitous Language forces everyone to align early and continuously.

**Common beginner mistake**  
Using technical jargon with business people (“entity”, “DTO”, “service”) or generic vague terms (“thing”, “item”, “process”). Also, letting the language drift — business keeps using the agreed terms, but code uses different names.

## 4. Bounded Context

**Simple definition**  
A bounded context is a clear boundary within which a particular model and ubiquitous language are consistent and valid. Outside that boundary, the same word can mean something different.

**Real-world analogy**  
In a university:
- “Student” in the Admissions context means an applicant.
- “Student” in the Library context means someone with borrowing privileges.
- “Student” in the Sports context means an athlete on a team.
The word is the same, but the rules and meaning differ by department — each department is a bounded context.

**Business example (e-commerce)**  
- In the Catalog context, a “Product” has SKU, name, description, images, categories.
- In the Inventory context, the same “Product” has stock levels, warehouse locations, reservation status.
- In the Pricing context, “Product” has base price, promotional rules, tax categories.
Trying to use one single “Product” model for all three leads to confusion and bloated classes.

**Why it matters in real projects**  
Large systems inevitably have multiple teams and conflicting interpretations. Without explicit boundaries, models bleed into each other, creating tight coupling and making changes risky. Bounded contexts let different teams own their part independently while still integrating where needed.

**Common beginner mistake**  
Trying to create one unified model for the entire company (the “enterprise data model”). It starts simple but quickly becomes a monster that satisfies no one and slows everyone down.

## 5. Context Map (high-level overview)

**Simple definition**  
A context map is a high-level diagram that shows all the bounded contexts in a system and how they relate to or integrate with each other.

**Real-world analogy**  
A city map showing different neighborhoods (contexts) and the roads or bridges connecting them. It helps you understand how to travel from one area to another without getting lost.

**Business example (airline booking system)**  
Contexts might include:
- Flight Search
- Booking & Reservation
- Payment
- Loyalty Program
- Check-in & Boarding
The map shows:
- Booking context sends events to Loyalty when a ticket is purchased.
- Payment context is called synchronously from Booking.
- Flight Search reads a replicated view from Inventory.

**Why it matters in real projects**  
In large organizations, no single team owns the entire system. A context map gives everyone a shared big-picture view: who owns what, where integrations happen, and what translation is needed between contexts. It prevents surprise coupling and helps plan migrations or new features.

**Common beginner mistake**  
Never drawing or maintaining the map — teams work in silos and discover painful integration issues late. Or drawing a beautiful map once and then forgetting it as reality drifts.

### Summary

These concepts form the strategic foundation of DDD:

1. Start with the **Domain** — the business you're serving.
2. Split it into **Subdomains** and decide where to invest deeply (core).
3. Build a **Ubiquitous Language** so everyone speaks the same language.
4. Define clear **Bounded Contexts** where that language is consistent.
5. Draw a **Context Map** to see how everything fits together.

Master these first — before thinking about code patterns like entities or aggregates. When you get the strategy right, the tactical implementation becomes much clearer and more effective.

# Tactical Domain-Driven Design: Core Building Blocks

Tactical DDD provides the concrete patterns we use inside a **Bounded Context** to model the domain precisely. These patterns help us translate the Ubiquitous Language into maintainable, expressive code that protects business rules.

We'll go deep on each pattern, with real-world insights from systems I've built or rescued.

## 1. Entity

**Definition**  
An object defined primarily by its identity (continuity over time) rather than its attributes. Entities have a life cycle and can change state while remaining the "same" thing.

**Responsibility**  
- Maintain identity (usually via a stable ID).
- Encapsulate behavior that operates on its own state.
- Enforce rules that apply throughout its life cycle.

**Real business example (e-commerce)**  
An **Order**. Order #1001 remains the same entity even if the customer changes address, adds items, cancels, or the order ships. The identity (#1001) persists through all state changes.

**Interactions**  
- Often the root of an Aggregate (see below).
- Holds references to Value Objects (e.g., ShippingAddress) and other Entities inside the same Aggregate.
- Exposes behavior, not raw data (tell-don't-ask).

**Common mistakes & anti-patterns**  
- Making everything an Entity (e.g., treating ProductVariant as an Entity when it has no life cycle — it's a Value Object).
- Exposing internal state via public getters/setters → anemic domain model.
- Using database surrogate keys as domain identity too early — identity should come from the domain first.

## 2. Value Object

**Definition**  
An immutable object defined solely by its attributes. Two Value Objects with the same attributes are interchangeable.

**Responsibility**  
- Represent concepts that are measured or described (money, address, date range, color).
- Encapsulate validation and behavior related to those attributes.
- Enable equality comparison by value.

**Real business example (banking)**  
**Money** (amount + currency). $100 USD equals another $100 USD instance — they are interchangeable. Adding behavior like `money.add(other)` that handles currency conversion or overflow.

Another: **ShippingAddress** — street, city, postcode, country. If two orders have identical address values, they are the same address conceptually.

**Interactions**  
- Composed inside Entities or other Value Objects.
- Never stored independently — always embedded or serialized.
- Often used as parameters to Entity methods.

**Common mistakes & anti-patterns**  
- Making mutable Value Objects (allowing changes after creation).
- Assigning IDs to Value Objects (defeats the purpose).
- Over-engineering simple values (e.g., using a VO for a plain string that has no validation or behavior).

## 3. Aggregate & 4. Aggregate Root

**Definition**  
An Aggregate is a cluster of Entities and Value Objects that must remain consistent together. The Aggregate Root is the single Entity through which the outside world accesses the entire cluster.

**Responsibility**  
- Enforce business invariants (rules that must always be true) within the cluster.
- Define a consistency boundary for transactions.
- Protect internal objects from direct external access.

**Real business example (e-commerce)**  
**Order Aggregate**:
- Root: Order (Entity)
- Contains: OrderLine (Entities or VOs), ShippingAddress (VO), PaymentSummary (VO), OrderStatus (VO)

Invariant examples:
- Total value must equal sum of line amounts × quantities ± discounts.
- Cannot add lines after order is shipped.
- Cannot ship if payment not authorized.

All changes go through the Order root: `order.addLine(productId, quantity)`.

**Interactions**  
- Only the Root has a Repository.
- Other objects inside are reached only via the Root.
- Roots can hold references to other Aggregates by ID only (not direct object references).

**Why Aggregates Exist**  
In complex domains, not everything can be consistent immediately across the entire system (eventual consistency). But certain rules **must** be enforced immediately. Aggregates define the boundary where strong (transactional) consistency is required.

Without Aggregates, you'd either:
- Put everything in one giant transaction (performance/scalability disaster), or
- Risk breaking critical business rules.

**Aggregate Boundaries**  
- Rule of thumb: If two objects must change together to preserve an invariant → same Aggregate.
- If they can change independently → separate Aggregates.
- Keep Aggregates small (ideally < 10 objects total). Large Aggregates kill concurrency and performance.

**Transaction Boundaries**  
One Aggregate = one transactional unit. You load, modify, and save one Aggregate per use case (usually per HTTP request or command).

**Business Invariants**  
The unbreakable rules inside an Aggregate. Example: "An approved order must have at least one line item and a valid payment." The Aggregate Root guards these.

**Common mistakes & anti-patterns**  
- Huge Aggregates (e.g., Customer + all Orders + Preferences + PaymentMethods).
- Referencing full objects across Aggregates → tight coupling and distributed transactions.
- Letting clients manipulate internal objects directly (bypassing the root).
- Designing Aggregates around data instead of invariants.

## 5. Domain Service

**Definition**  
A stateless operation that doesn't naturally fit inside an Entity or Value Object.

**Responsibility**  
- Coordinate behavior across multiple Entities/Aggregates.
- Express domain concepts that are actions rather than things.

**Real business example (financial trading)**  
**PricingService.calculateTradeValue(trade, marketData)**  
Pricing a complex derivative requires market data feeds and multiple calculations — it doesn't belong to the Trade entity itself.

Another: **TaxCalculator.compute(order, shippingAddress)** in e-commerce.

**Interactions**  
- Injected with Repositories or other Services.
- Called from Application Services or Aggregate Roots.
- Returns Value Objects or primitives.

**Common mistakes & anti-patterns**  
- Dumping all business logic into services → anemic Entities (biggest anti-pattern).
- Creating services for things that should be Entity methods.
- Making Domain Services stateful.

## 6. Repository

**Definition**  
An abstraction that provides collection-like access to Aggregate Roots, hiding persistence details.

**Responsibility**  
- Retrieve Aggregate Roots by ID.
- Add/remove Aggregate Roots.
- Never expose internal Entities.

**Real business example**  
`OrderRepository.findById(orderId)` returns a fully reconstructed Order Aggregate with all its lines and VOs.

**Interactions**  
- Only defined for Aggregate Roots.
- Used by Application Services or Domain Services.
- Implementation lives in infrastructure layer (e.g., using JPA, EF, or custom SQL).

**Common mistakes & anti-patterns**  
- Repositories for Value Objects or non-root Entities.
- Query methods that return partial Aggregates or projections inside the domain layer (leaks infrastructure concerns).
- Using Repository to implement business rules (should be in Aggregates).

## 7. Factory

**Definition**  
A dedicated object responsible for creating complex Aggregates or Entities with valid initial state.

**Responsibility**  
- Encapsulate complex construction logic.
- Ensure invariants hold even at creation time.

**Real business example**  
Creating an **Order** from a shopping cart:
- Validate cart not empty.
- Apply promotions.
- Freeze prices.
- Set initial status.

`OrderFactory.createFromCart(cart, customer)` does all this in one place.

**Interactions**  
- Called from Application Services.
- Returns fully valid Aggregate Root.

**Common mistakes & anti-patterns**  
- Using constructors for complex creation (scattered logic, hard to enforce invariants).
- Never using factories when needed → invalid objects slip through.
- Overusing factories for simple objects.

## 8. Domain Event

**Definition**  
An immutable object that represents something significant that happened in the domain (past tense).

**Responsibility**  
- Capture intent and facts for integration with other Bounded Contexts.
- Enable eventual consistency.
- Record history for auditing or replay.

**Real business example**  
`OrderShipped(orderId, trackingNumber, shippedDate)`  
Published when an Order changes to Shipped status.

Other contexts react:
- Inventory context releases reserved stock.
- Loyalty context awards points.
- Notification context sends email.

**Interactions**  
- Published by Aggregate Root when invariant-changing state change occurs.
- Handled by Application Services in same or other contexts.
- Often persisted in an Event Store for CQRS/ES.

**Common mistakes & anti-patterns**  
- Using Domain Events for commands (should be separate).
- Publishing too many trivial events (chatty system).
- Putting behavior in events (events are data only — behavior goes in handlers).
- Tight coupling between publisher and subscriber.

### Final Thoughts for Mid/Senior Engineers

Tactical DDD is not about applying every pattern everywhere. It's about:

1. **Protecting invariants** — Aggregates are your primary tool.
2. **Expressing intent** — Rich Entities and Value Objects with behavior.
3. **Clear boundaries** — Only Roots exposed, everything else protected.
4. **Eventual consistency outside Aggregates** — Domain Events enable loose coupling.

In practice, I've seen the biggest wins come from getting Aggregate boundaries right first. Everything else flows naturally.

Start small: identify one core invariant, draw the Aggregate that protects it, implement the Root with behavior. Expand from there.

The code will feel heavier at first than anemic services, but it pays off massively in maintainability when the domain evolves — which it always does.

# Strategic Domain-Driven Design for Large Systems

Strategic DDD is where the real architectural value of DDD lives — especially in large, long-lived systems with multiple teams. Tactical patterns (aggregates, entities, etc.) are useful, but they only pay off if you first get the **strategic boundaries** right.

In large systems I've architected (multi-hundred-million-dollar e-commerce platforms, global banking systems, logistics networks), poor strategic design caused more pain than any tactical mistake: duplicated logic, broken integrations, blocked releases, team dependencies, and eventual rewrites.

## 1. Bounded Context Design – The Core Decision

A **Bounded Context** is an explicit boundary within which a particular domain model is defined and consistent. Inside the boundary, the Ubiquitous Language is unambiguous. Outside, the same terms can mean different things.

**Decision-making**  
The key question: "Where do concepts change meaning or have different rules?"  
Draw a boundary there.

**How to discover boundaries from business workflows**  
Use concrete discovery techniques that work in real organizations:

1. **Event Storming (Big Picture)**  
   Run workshops with business experts and developers. Map out the end-to-end business process as a timeline of domain events (e.g., "Order Placed", "Payment Authorized", "Inventory Reserved", "Order Shipped").  
   Look for:
   - Shifts in language (e.g., "Order" becomes "Shipment" in warehouse).
   - Handoffs between departments/teams.
   - Points where rules dramatically change.

   Real example: In an e-commerce platform, we discovered that "Promotion" meant completely different things in Marketing (campaign rules) vs Pricing (runtime calculation) vs Analytics (attribution). Three separate contexts emerged naturally.

2. **Workflow analysis**  
   Follow a key business object through its life cycle. Note where ownership changes or rules diverge.  
   Example: A "Customer" in Sales context (leads, segmentation) vs Support context (tickets, SLAs) vs Billing context (invoices, dunning).

3. **Organizational boundaries (Conway’s Law)**  
   Teams that rarely talk usually already have implicit boundaries. Aligning bounded contexts to team structures reduces communication overhead.

**Trade-off**  
Tighter boundaries → more autonomy, less consistency.  
Looser boundaries → shared model, higher coupling.

Real constraint: In regulated industries (banking, healthcare), you often need strong consistency across contexts → larger bounded contexts or heavy integration patterns.

## 2. Context Mapping Patterns

Once you have multiple bounded contexts, you must define how they integrate. The classic patterns from Evans' book remain the best framework.

### Shared Kernel
**Definition**  
Two teams share a small, common subset of the domain model (usually data + behavior).

**When to use**  
When two contexts are tightly related and both teams need to evolve the shared part together.

**Real example**  
In a trading platform, Risk and Trading contexts shared a "MarketData" kernel (instrument definitions, price ticks). Both needed identical definitions.

**Trade-offs**  
+ Strong consistency  
– Requires ongoing coordination (joint design, shared repository or package)  
– Can slow both teams if not kept minimal

**Real-world constraint**  
Only works if teams are co-located or have excellent communication. I've seen shared kernels turn into bottlenecks when teams are in different time zones.

### Customer/Supplier
**Definition**  
Upstream context (Supplier) provides a model that downstream (Customer) depends on. Teams establish a formal relationship where upstream considers downstream needs.

**Real example**  
Catalog context (upstream) publishes product information. Pricing context (downstream) needs stable product attributes for pricing rules. Catalog team commits to change protocols.

**Trade-offs**  
+ Downstream team gets what it needs  
+ Clear accountability  
– Upstream team has extra responsibility

**Constraint**  
Requires organizational maturity — upstream must accept downstream as a real "customer".

### Conformist
**Definition**  
Downstream team simply accepts upstream's model as-is, even if suboptimal, to avoid translation cost.

**When to use**  
When upstream has no incentive to accommodate you (e.g., third-party system, dominant internal team).

**Real example**  
Integration with an enterprise ERP system. We conformed to their bizarre "Item Master" model rather than translating.

**Trade-offs**  
+ Fast integration  
– Downstream model becomes polluted with foreign concepts  
– Harder to evolve independently

### Anti-Corruption Layer (ACL)
**Definition**  
A dedicated translation layer that converts between external model's concepts and your internal model.

**Real example**  
Migrating from a legacy mainframe order system. We built an ACL that translated mainframe's flat record structure into our rich Order aggregate. Internally, we used clean domain concepts; externally, we spoke mainframe.

**Trade-offs**  
+ Protects your model's integrity  
+ Enables independent evolution  
– Significant upfront and maintenance cost  
– Can become performance bottleneck if not careful

**Real-world constraint**  
Essential when integrating with legacy systems or external partners. I've used ACLs extensively in brownfield projects — they are often the only way to avoid contaminating your new domain model.

## 3. DDD in Monoliths vs Microservices

**Monolith with multiple Bounded Contexts**  
- All contexts live in one codebase/deployable.
- Boundaries enforced through packages/modules/namespaces.
- Integration via direct method calls or domain events within process.

**Advantages I've seen**  
- Easier refactoring across boundaries early on.
- Simpler transactions when needed.
- Lower operational overhead.
- Excellent for medium-sized systems (5–15 teams).

**Real example**  
A multi-tenant SaaS platform I architected started as a modular monolith with 8 clear bounded contexts. We shipped fast, maintained consistency, and only later extracted two contexts into services when scaling required it.

**Microservices with Bounded Contexts**  
- Each context deployed independently.
- Integration via APIs, events, messages.

**Advantages**  
- Team autonomy.
- Independent scaling and technology choice.
- Fault isolation.

## 4. When 1 Bounded Context = 1 Microservice

**Good cases**  
- Context has distinct scaling needs (e.g., Search context handles 10x traffic).
- Context owned by separate team with different release cadence.
- Context requires different technology (e.g., real-time pricing using different stack).
- Regulatory or security boundaries require isolation.

**Real example**  
In a logistics platform, Route Optimization context was extracted to a separate service because it used heavy ML models and needed GPU scaling — completely different from the rest of the system.

## 5. When 1-to-1 Mapping is Dangerous

**Dangerous cases**  
- Premature splitting: You split too early before boundaries are understood → constant cross-service changes.
- Too many small services: Each CRUD resource becomes a service → distributed monolith with chatty calls.
- Ignoring business workflows: Boundaries drawn around technical concerns rather than domain ones.
- Team per service when teams are small: One team owning 10 tiny services → no real autonomy benefit, just overhead.

**Real-world anti-pattern I've rescued**  
A company split their e-commerce system into 50+ microservices early ("one per aggregate"). Result: placing an order required 15 synchronous calls. Latency was terrible, deployments were blocked on multiple teams, and debugging was nightmare. We eventually consolidated back into 6 bounded contexts (some as services, some in monolith).

**Decision framework I use**  
1. Start with clear bounded contexts in a modular monolith.
2. Only extract to separate service when you have a concrete reason (scale, team, tech, compliance).
3. Use domain events for asynchronous integration from day one — makes later extraction easier.

### Final Advice for Large Systems

Strategic DDD is primarily about **managing complexity through boundaries and explicit relationships**.

Key decisions:
- Invest heavily in discovery (Event Storming, workshops) early and repeatedly.
- Align contexts to team structures when possible (Conway’s Law is inevitable).
- Choose integration patterns consciously — ACL for legacy, events for autonomy.
- Prefer modular monoliths until you have proven need for distributed deployment.

The biggest payoff I've seen: teams can move independently without constant coordination, new features land faster, and the system survives years of business change without becoming unmaintainable.

Get the boundaries wrong, and no amount of clean aggregates will save you. Get them right, and the tactical patterns become natural and effective.

# Practical Step-by-Step Guide to Applying Domain-Driven Design in Real Projects

As a Principal Architect, I’ve led DDD adoption in greenfield startups, large-scale rewrites, and gradual legacy rescues. Here’s what actually works — the concrete actions, sequences, and pitfalls I’ve learned the hard way.

## 1. Starting DDD from Scratch (Greenfield Project)

**Goal**: Get the strategic boundaries right before writing too much code.

**Step-by-step actions**

1. **Secure domain expert access early**  
   - Identify 2–4 key business stakeholders (not just product managers — actual experts who know the rules).  
   - Schedule recurring 1–2 hour sessions every week from day one.  
   - Warning: If you can’t get consistent access, delay heavy modeling — you’ll build the wrong thing.

2. **Run an initial Big Picture Event Storming (1–2 days)**  
   - Gather developers + domain experts in a room (or Miro/Mural).  
   - Map the end-to-end business process as a timeline of Domain Events (past-tense sticky notes: “Order Placed”, “Payment Captured”, “Inventory Reserved”).  
   - Add Commands (what triggers events), Read models, User roles, External systems.  
   - Look for language shifts and handoffs — these are candidate Bounded Context boundaries.  
   - Output: Rough Context Map with 4–8 candidate contexts.

3. **Prioritize the Core Subdomain**  
   - Ask: “What makes us different from competitors?” Focus 70–80% of modeling effort there first.  
   - Defer supporting/generic subdomains (use off-the-shelf or simple CRUD initially).

4. **Pick ONE Bounded Context to model deeply first**  
   - Choose the most complex or valuable part of the core.  
   - Run a second, focused Event Storming just for that context.  
   - Identify key Aggregates by grouping events/commands that must be consistent together.

5. **Implement incrementally**  
   - Build a walking skeleton: one end-to-end use case (command → aggregate → domain event → handler).  
   - Use Ubiquitous Language in code from day 1 (class/method names match workshop language).  
   - Introduce tactical patterns only as needed (Entity, VO, Aggregate Root, Repository).

6. **Validate constantly**  
   - Demo working software to domain experts every sprint.  
   - Ask: “Does this reflect how you think about the business?”

**Risks & warnings**  
- Don’t try to model everything upfront — you’ll overdesign and delay delivery.  
- Avoid “analysis paralysis” — aim for “good enough” boundaries; you’ll refine later.  
- If the business domain is still evolving rapidly (early startup), keep models lightweight.

## 2. Introducing DDD into an Existing Legacy System

**Goal**: Improve maintainability without a risky Big Bang rewrite.

**Step-by-step actions**

1. **Start with strangulation**  
   - Identify a new feature or a painful change hotspot.  
   - Build the new feature in a new DDD-style module/package, separate from legacy code.

2. **Introduce an Anti-Corruption Layer (ACL) early**  
   - Create a dedicated package that translates legacy data structures into your new domain model.  
   - Never let legacy anemic objects leak into your new context.  
   - Example: Legacy “OrderDTO” → ACL → rich “Order” Aggregate.

3. **Run targeted Event Storming on pain points**  
   - Pick one messy area (e.g., order fulfillment).  
   - Invite the people who complain most about it.  
   - Discover the hidden Bounded Context that’s buried in the big ball of mud.

4. **Extract one Bounded Context at a time**  
   - Move behavior from services/controllers into new Aggregates.  
   - Redirect new changes to the new model.  
   - Keep reads against legacy until you can duplicate data or redirect.

5. **Gradually replace integrations**  
   - Publish Domain Events from your new context.  
   - Have legacy code subscribe (or vice versa) to decouple.

**Risks & warnings**  
- Dual writing (new + legacy) can cause inconsistency — use idempotency and reconciliation jobs.  
- Team resistance: “Why are we making it more complex?” Show concrete wins (faster changes, fewer bugs).  
- Don’t try to refactor the entire legacy core first — you’ll never ship new features.

## 3. Event Storming – Simple, Actionable Explanation

Event Storming is the single most effective discovery tool I’ve used.

**How to run a basic session**

1. **Preparation**  
   - Unlimited wall space or infinite digital board.  
   - Colored sticky notes: Orange (Domain Events), Blue (Commands), Yellow (Aggregates), Purple (Read models), etc.  
   - 4–10 participants: developers + domain experts.

2. **Flow**  
   - Step 1: Brainstorm Domain Events in chronological order (no filtering — quantity first).  
   - Step 2: Add Commands that cause events.  
   - Step 3: Identify pain points/hotspots (red stickies).  
   - Step 4: Group events into candidate Aggregates and Bounded Contexts.  
   - Step 5: Discuss policies (“whenever X happens, then Y”).

3. **Duration**  
   - Big Picture: 4–8 hours for high-level overview.  
   - Deep Dive: 2–4 hours per context.

**Tips**  
- Facilitator must enforce “no technical talk” during event discovery.  
- Record the board — it’s your single source of truth for Ubiquitous Language.

**Warning**  
One-off Event Storming is useless. Run them repeatedly as knowledge grows.

## 4. Working with Domain Experts

**Actions that work**

1. **Build trust through delivery**  
   - Show working software early and often. Experts engage more when they see impact.

2. **Speak their language, not yours**  
   - Never say “aggregate” or “entity” to business people.  
   - Use the terms from Event Storming sessions in all communication.

3. **Use concrete examples**  
   - Always discuss specific scenarios (“What happens if customer cancels after partial shipment?”).  
   - Capture examples as acceptance tests.

4. **Pair on design**  
   - Sit with experts while sketching models.  
   - Challenge assumptions politely: “Earlier you said X, but in this case Y — how do we reconcile?”

**Risks**  
- Proxy experts (product owners who aren’t practitioners) → wrong model.  
- Overloading experts — limit sessions to 90 minutes max.  
- Developers dominating discussions — enforce “business speaks first” rule.

## 5. Evolving the Domain Model Over Time

The model is never “done” — business changes constantly.

**Practical actions**

1. **Schedule regular model reviews**  
   - Every 2–3 months, revisit Context Map and key Aggregates with experts.

2. **Refactor mercilessly inside Bounded Contexts**  
   - New insights → rename classes, split/merge Aggregates, introduce new Value Objects.

3. **Handle breaking discoveries**  
   - If a new rule invalidates old assumptions, introduce new concepts rather than force-fit.  
   - Example: “Cancelled Order” becomes a separate state with different behavior.

4. **Version integrations, not internal models**  
   - Published events/APIs can be versioned or extended.  
   - Internal model evolves freely.

**Warning**  
Resist the urge to keep the old wrong model “because it’s working”. Technical debt in the domain model grows exponentially.

## 6. Refactoring Safely Toward DDD

**Safe sequence I’ve used successfully**

1. **Add characterization tests**  
   - Write tests that capture current behavior of legacy code before touching it.

2. **Extract behavior gradually**  
   - Move one business rule at a time from service classes into domain objects.

3. **Use the Strangler Fig pattern**  
   - New features → new DDD context.  
   - Old features → redirect calls to new context when possible.

4. **Keep database schema changes minimal**  
   - New contexts can read from legacy tables initially.  
   - Use views or replication for clean reads later.

5. **Measure progress**  
   - Track: change failure rate, time to deploy domain changes, developer feedback.

**Major risks**  
- Underestimating data migration effort when splitting contexts.  
- Breaking production with misunderstood rules — always validate with experts.  
- “Parallel universe” syndrome: two models drifting apart → pick one and migrate.

### Final Advice

DDD is an investment. The payoff comes after 6–18 months when:
- New features land faster than before.
- New developers understand the domain quickly.
- Business changes don’t cause widespread breakage.

Start small, deliver value constantly, and use concrete collaboration techniques (especially Event Storming). The biggest risk is treating DDD as a purely technical exercise — it only works when domain experts are true partners in the process.

# Integrating Domain-Driven Design with Common Architecture Patterns

As a Principal Architect, I've applied DDD across Clean/Hexagonal monoliths, CQRS/ES systems, event-driven microservices, and API Gateway-fronted platforms. The key principle that survives every integration is **dependency inversion**: the Domain is the center — everything else depends on it, never the reverse.

DDD's domain model (Entities, Value Objects, Aggregates, Domain Services, Domain Events) lives in the **core** and must remain pure: no dependencies on frameworks, databases, UIs, or external services.

Below, each pattern explained with concrete mapping, dependencies, and mistakes I've seen (and fixed) in production systems.

## 1. Clean Architecture (Onion Architecture)

**Structure overview**  
- Entities (core)  
- Use Cases / Application Services  
- Interface Adapters (Controllers, Presenters, Gateways)  
- Frameworks & Drivers (Web, DB, UI)

**Where DDD lives**  
- The innermost circle ("Entities") contains your tactical DDD model: Aggregates, Entities, Value Objects, Domain Services, Domain Events, Repositories (interfaces only).  
- Application Services (use cases) orchestrate domain objects — they live in the "Use Cases" ring and implement application-specific workflows.

**Dependency rule**  
- Outer rings depend inward.  
- Domain depends on **nothing** external.  
- Repositories are interfaces in the domain; implementations live in outer rings.

**Common mistakes**  
- Putting business logic in Application Services → anemic domain model.  
- Leaking framework types (e.g., HttpRequest, EntityFramework DbContext) into domain.  
- Making domain depend on application services (inversion).

**Real-world note**  
I've used Clean Architecture + DDD in modular monoliths. It enforces discipline and makes extracting microservices later trivial.

## 2. Hexagonal Architecture (Ports & Adapters)

**Structure overview**  
- Core (domain)  
- Ports (interfaces: input = driving, output = driven)  
- Adapters (concrete implementations: REST controllers, DB repositories, message consumers)

**Where DDD lives**  
- Entire tactical and strategic DDD model lives inside the hexagon (core).  
- Input ports = Application Services or command handlers.  
- Output ports = Repository interfaces, Domain Event publishers, external service interfaces.

**Dependency rule**  
- Core defines all ports.  
- Adapters depend on core; core never depends on adapters.  
- Domain remains framework-agnostic.

**Common mistakes**  
- Implementing ports directly in core (e.g., Spring @Repository in domain package).  
- Bypassing ports — controllers calling infrastructure directly.  
- Putting domain events or aggregates in adapter layers.

**Real-world note**  
Hexagonal is my preferred style for DDD because it makes the "dependency inversion" explicit. I've used it in systems where we swapped ORM multiple times without touching domain code.

## 3. CQRS (Command Query Responsibility Segregation)

**Structure**  
- Command side: handles writes via domain model  
- Query side: optimized reads (often denormalized views)

**Where DDD lives**  
- Command side = full DDD tactical model (Aggregates enforce invariants).  
- Query side = read models (simple DTOs, projections) — usually **no** DDD tactical patterns here.

**Integration patterns**  
- Commands → Application Service → Aggregate → persistence + Domain Events published.  
- Domain Events → handlers update read models (eventually consistent).

**Dependency rule**  
- Command side owns the domain model.  
- Query side depends on events from command side but never the reverse.

**Common mistakes**  
- Applying aggregates on read side → massive performance issues.  
- Single model for read/write (defeats CQRS purpose).  
- Synchronous query after command expecting immediate consistency (use eventual consistency or saga for critical cases).

**Real-world note**  
In high-throughput systems (trading, e-commerce checkout), CQRS + DDD on command side gave us strong consistency where needed and blazing reads elsewhere.

## 4. Event-Driven Architecture

**Structure**  
- Components communicate via events (async).  
- Often combined with CQRS (Event Sourcing optional).

**Where DDD lives**  
- Aggregates publish Domain Events when invariants change.  
- Events carry rich domain meaning (Ubiquitous Language in event names/payloads).

**Integration**  
- Domain Events → published to broker → consumed by other Bounded Contexts or read model projectors.  
- Enables loose coupling between contexts.

**Dependency rule**  
- Domain publishes events but doesn't depend on subscribers.  
- No knowledge of message brokers in domain layer.

**Common mistakes**  
- Putting infrastructure concerns in Domain Events (e.g., Kafka headers).  
- Using technical events instead of domain-rich ones ("OrderUpdated" vs "OrderShipped").  
- Tight coupling via shared event schemas across contexts without versioning.

**Real-world note**  
In microservices platforms, domain events are the primary integration mechanism between bounded contexts — far superior to shared databases.

## 5. Microservices

**Mapping to DDD**  
- Ideal: **One Bounded Context = One Microservice** (when extraction justified).  
- Each service owns its domain model, database, and ubiquitous language.

**Where DDD lives**  
- Entire strategic and tactical DDD inside each service.  
- Context Map defines inter-service relationships (ACL, events, etc.).

**Dependency rule**  
- Services depend on each other only via published interfaces (APIs/events).  
- Domain never depends on other services directly.

**Common mistakes**  
- Distributed monolith: fine-grained services with synchronous calls everywhere.  
- Sharing domain models across services (shared kernel gone wrong).  
- One database per system, not per service.

**Real-world note**  
Start with bounded contexts in a monolith. Extract to microservices only when scale, team, or tech needs demand it.

## 6. API Gateway

**Role**  
- Single entry point, routing, authentication, rate limiting, protocol translation.

**Where DDD lives**  
- API Gateway has **zero** domain logic.  
- It routes requests to appropriate bounded context services.

**Integration patterns**  
- Gateway → REST/gRPC → service's input adapter (controller).  
- Backend-for-Frontend (BFF) pattern: separate gateways per client type.

**Dependency rule**  
- Gateway depends on downstream services.  
- Domain/services never depend on gateway.

**Common mistakes**  
- Putting business logic in gateway (e.g., aggregation, orchestration) → violates bounded context integrity.  
- Exposing internal domain models directly through gateway.

**Real-world note**  
In large systems, API Gateway + BFF layers keep client concerns isolated while preserving clean domain models inside services.

### Universal Rules Across All Patterns (Senior/Staff Level)

1. **Domain is the only layer that contains business rules** — everything else is plumbing.
2. **Domain must not depend on**:
   - Frameworks (Spring, .NET, Node)
   - Databases or ORMs
   - Message brokers
   - HTTP layers
   - External services
3. **Everything else depends on domain** via interfaces/ports defined in the domain package.
4. **Repository/Port interfaces live in domain**; implementations live in infrastructure.
5. **Domain Events are pure domain concepts** — no infrastructure types.

**Most common architectural smell I diagnose**  
Anemic domain model: Aggregates are just bags of getters/setters, all behavior in application services or controllers. This happens when teams treat DDD as "just naming things nicely" instead of moving behavior into the model.

**Biggest payoff**  
When done right, the domain layer becomes stable and framework-agnostic. I've seen systems where we swapped Spring for Quarkus, relational DB for Event Store, or monolith for microservices — with **zero changes** to domain code.

The architecture serves the domain — never the other way around. Keep the domain pure, invert dependencies religiously, and the rest falls into place.

# Example Project Structures for Correct Domain-Driven Design

Below are practical, production-tested folder structures I've used in real systems (e-commerce, banking, logistics). They enforce dependency rules, protect the domain model, and make the Ubiquitous Language visible in code.

All examples assume a single Bounded Context (e.g., "Order Management"). For multiple contexts, repeat the structure per context or use top-level modules.

## 1. Typical DDD Folder Structure (Modular Monolith or Service)

```
src/
├── domain/                  # Pure domain model – the heart of the system
│   ├── model/               # Tactical DDD patterns
│   │   ├── order.ts         # Aggregate Root: Order (rich behavior)
│   │   ├── order-line.ts    # Entity or Value Object
│   │   ├── money.ts         # Value Object
│   │   ├── shipping-address.ts
│   │   └── order-status.ts  # Value Object (enum-like)
│   │
│   ├── services/            # Domain Services (stateless, coordinate across aggregates)
│   │   └── pricing-service.ts
│   │
│   └── events/              # Domain Events
│       └── order-shipped.event.ts
│
├── application/             # Use cases / Application Services
│   ├── place-order/
│   │   ├── place-order.command.ts
│   │   ├── place-order.handler.ts     # Orchestrates domain, publishes events
│   │   └── place-order.dto.ts
│   │
│   └── cancel-order/
│       └── ... 
│
├── infrastructure/          # Everything that depends on the outside world
│   ├── persistence/
│   │   ├── repositories/
│   │   │   └── order-repository.impl.ts  # Implements domain/interface OrderRepository
│   │   └── typeorm/                      # ORM entities (separate from domain!)
│   │
│   ├── messaging/
│   │   └── event-publisher.impl.ts       # Publishes Domain Events to RabbitMQ/Kafka
│   │
│   └── external/
│       └── payment-gateway.adapter.ts
│
├── interfaces/              # Input adapters (driving adapters)
│   ├── http/
│   │   ├── order-controller.ts           # REST/GraphQL endpoints
│   │   └── dtos/
│   │
│   └── messaging/
│       └── command-consumer.ts            # Consumes external commands
│
└── common/                  # Shared utilities, base classes, exceptions
```

### Responsibilities of Each Folder

- **domain/**  
  Contains only pure business rules. No dependencies on frameworks, databases, or external services.  
  **Why**: This is the only stable part of the system. Business rules change less often than tech stacks. Keeping it pure allows swapping ORMs, UI frameworks, or deployment models without touching core logic.

- **application/**  
  Orchestrates use cases: validates input, loads aggregates via repositories, calls domain behavior, persists, publishes events.  
  **Why**: Separates "what can be done" (use cases) from "how it's exposed" (HTTP) and "how domain works" (rich model).

- **infrastructure/**  
  Implements domain interfaces (repositories, event publishers) using concrete technology. Also includes adapters to external systems.  
  **Why**: Dependency inversion – domain defines interfaces, infrastructure implements them.

- **interfaces/**  
  Driving adapters: HTTP controllers, CLI, message consumers. Translates external concerns into application commands.  
  **Why**: Keeps delivery mechanisms (REST, GraphQL, gRPC) isolated – you can add new ones without touching domain.

## 2. Anemic Domain Model (Bad) vs Rich Domain Model (Good)

### Anemic (Anti-Pattern) – What I've rescued many times

```typescript
// domain/model/order.ts  (WRONG – just data)
export class Order {
  public id: string;
  public customerId: string;
  public lines: OrderLine[];
  public status: string;
  public total: number;

  // Only getters/setters – no behavior
}

// application/place-order.handler.ts (all logic here)
class PlaceOrderHandler {
  handle(command) {
    const order = new Order();
    order.status = 'Draft';
    order.lines = command.lines;

    // Business rules scattered here
    if (order.lines.length === 0) throw Error('Empty order');
    if (this.inventoryService.checkStock(order.lines) === false) throw Error('Out of stock');
    if (this.pricingService.calculateTotal(order.lines) > 10000) { /* fraud check */ }

    order.total = this.pricingService.calculateTotal(order.lines);
    order.status = 'Placed';

    this.orderRepository.save(order);
  }
}
```

**Problems**  
- Business rules scattered across services → hard to find, easy to duplicate or miss.  
- Any code can mutate order inconsistently → invariants easily broken.  
- New developers see data classes → think "just fill fields".

### Rich Domain Model (Correct)

```typescript
// domain/model/order.ts  (CORRECT – behavior lives here)
export class Order {
  private _lines: OrderLine[];
  private _status: OrderStatus;
  private _total: Money;

  private constructor() {} // Force creation via factory

  static createFromCart(cart: Cart, customerId: string): Order {
    if (cart.isEmpty()) throw new Error('Cannot place empty order');

    const order = new Order();
    order._lines = cart.lines.map(line => OrderLine.create(line.product, line.quantity));
    order._status = OrderStatus.Draft;
    order.recalculateTotal(); // Applies promotions, taxes, etc.
    return order;
  }

  addLine(product: Product, quantity: number): void {
    if (!this._status.canModifyLines()) {
      throw new Error('Cannot modify lines after order is confirmed');
    }
    this._lines.push(OrderLine.create(product, quantity));
    this.recalculateTotal();
  }

  confirmPayment(paymentAmount: Money): void {
    if (!paymentAmount.equals(this._total)) {
      throw new Error('Payment amount mismatch');
    }
    if (!this.inventoryService.reserveStock(this._lines)) {
      throw new Error('Stock unavailable');
    }
    this._status = OrderStatus.Confirmed;
  }

  ship(): void {
    if (this._status !== OrderStatus.Confirmed) {
      throw new Error('Cannot ship unconfirmed order');
    }
    this._status = OrderStatus.Shipped;
    this.domainEvents.publish(new OrderShipped(this.id));
  }

  private recalculateTotal(): void {
    this._total = this.pricingService.calculate(this._lines);
  }
}

// application/confirm-payment.handler.ts (thin orchestrator)
class ConfirmPaymentHandler {
  handle(command) {
    const order = this.orderRepository.findById(command.orderId);
    order.confirmPayment(command.amount);
    this.orderRepository.save(order); // Transaction ends here
  }
}
```

**Why this structure exists**

1. **Invariants are protected**  
   All rules that must always be true (e.g., total matches lines, cannot modify after confirm) are enforced inside the aggregate. No external code can bypass them.

2. **Intent is clear**  
   Reading `order.ship()` tells you exactly what happens in the business. No need to hunt through services.

3. **Behavior travels with data**  
   When you load an Order, you get all its valid operations — no risk of calling invalid methods.

4. **Change is safer**  
   Adding a new rule (e.g., fraud check on large orders) goes inside the aggregate — one place, tested once.

5. **Ubiquitous Language in code**  
   Method names match business terminology: `confirmPayment`, `ship`, `addLine`.

## 3. Alternative Structure for Larger Systems (Multiple Bounded Contexts)

```
src/
├── order-management/          # Bounded Context 1
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── interfaces/
│
├── catalog/                   # Bounded Context 2
│   └── ...
│
├── payment/                   # Bounded Context 3
│   └── ...
│
└── shared-kernel/             # Only if absolutely necessary (rare!)
    └── model/
        └── money.ts
        └── customer-id.ts
```

**Why**  
Explicit top-level separation reinforces that models are not shared accidentally. Integration happens only via published events or ACLs.

### Final Advice from Experience

- Start with the structure above — even in small projects. The discipline pays off quickly.
- Never let infrastructure types (DbContext, Entity, HttpRequest) into domain/.
- Repository interfaces live in domain/; implementations in infrastructure/.
- Application services are thin — if they're >100 lines, behavior probably belongs in the domain.
- The rich model feels heavier at first, but it reduces bugs and speeds up changes dramatically after 6+ months.

This structure has survived Black Friday traffic, regulatory audits, and multiple tech stack migrations. The domain stays clean, the rest adapts.

# How to Approach Domain-Driven Design at Different Engineer Levels

DDD is a skill that matures with experience. What’s expected (and helpful) changes dramatically as you grow. Here’s how I’ve guided engineers at each level in real projects — from juniors fixing bugs in a monolith to principals redesigning multi-team platforms.

## Fresher / Junior Engineer

**What to focus on**

1. **Ubiquitous Language in code**  
   - Read business tickets carefully and name classes/methods after real business terms.  
   - Example: In an e-commerce system, use `order.confirmPayment()` instead of `order.setStatus('paid')`.  
   - Ask: “What do business people call this action?”

2. **Understanding Entities vs Value Objects**  
   - Learn to spot identity: An `Order` has identity (tracks life cycle), but `Money` or `Address` does not.  
   - Practice: When adding a new field, ask “Does this thing change over time while remaining the same thing?”

3. **Putting behavior on the domain object**  
   - Never leave business rules in controllers or services.  
   - Bad: Controller checks if order can be cancelled.  
   - Good: `order.cancel()` method checks status and throws domain exception if invalid.

**What to avoid**

- Writing anemic models (classes with only getters/setters).  
- Touching aggregate design or bounded contexts — you’ll likely get it wrong without deeper understanding.  
- Using the word “entity” in conversations with business people.

**Key mental models**

- “Tell, don’t ask”: Call methods on objects instead of pulling data and deciding externally.  
- Code should read like a business conversation.  
- Example: `if (order.canShip()) order.ship()` is clearer than pulling status and checking in a service.

Real impact I’ve seen: Juniors who name things well and put simple validations in domain objects reduce bugs by 30–40% in their features.

## Mid-Level Engineer

**What to focus on**

1. **Aggregate design**  
   - Identify invariants and group objects that must stay consistent together.  
   - Example: In a booking system, `Reservation` aggregate includes `SeatSelection` and `Payment`. Invariant: “Cannot confirm reservation without full payment and selected seats.” All changes go through `reservation.confirm()`.  
   - Keep aggregates small — one transactional boundary per use case.

2. **Introducing Domain Events**  
   - Publish events when something important happens inside an aggregate.  
   - Example: When `order.ship()` succeeds → publish `OrderShipped`. This notifies inventory, loyalty, and notification contexts without coupling.  
   - Start simple: in-process listeners first, then move to async.

3. **Repository pattern properly**  
   - Define repository interfaces in domain, implement in infrastructure.  
   - Never expose ORM entities to domain layer.

**Avoiding over-engineering**

- Don’t create domain services for everything — most behavior belongs in aggregates.  
- Don’t introduce events for trivial state changes (e.g., `OrderAddressUpdated`).  
- Don’t split into microservices “because DDD” — stay in the monolith until you have real scaling/team needs.

Real impact: Mid-levels who get aggregates right cut deployment failures from inconsistent state dramatically. I’ve seen order fulfillment bugs drop near zero after proper aggregate boundaries.

## Senior Engineer

**What to focus on**

1. **Strategic boundaries (Bounded Contexts)**  
   - Challenge the “one big model” assumption.  
   - Example: In a large e-commerce platform, realize that “Product” in Catalog context (descriptions, images) is different from “Product” in Pricing (rules, tiers) and Inventory (stock levels). Define explicit contexts and integration patterns.

2. **Complexity management**  
   - Decide where to invest deep modeling (core subdomain) vs simplify (generic/supporting).  
   - Example: Build rich model for order fulfillment (core), but use simple CRUD for customer support notes (supporting).

3. **Distributed system concerns**  
   - Design for eventual consistency using domain events.  
   - Example: When `OrderShipped` published → Inventory context releases reservation, Loyalty awards points. Accept that reads might be slightly stale.  
   - Introduce Anti-Corruption Layers when integrating with legacy systems.

**Practical actions**

- Lead Event Storming sessions to discover hidden contexts.  
- Draw and maintain Context Maps.  
- Push back on “let’s just add this field to the Order table” when it belongs in another context.

Real impact: Seniors who define clear bounded contexts prevent the monolith from becoming unmaintainable. I’ve rescued systems where seniors redrew boundaries and suddenly multiple teams could move independently.

## Staff / Principal Engineer

**What to focus on**

1. **Organization vs architecture alignment (Conway’s Law in action)**  
   - Shape bounded contexts to match team structures and communication patterns.  
   - Example: If Marketing and Fulfillment teams never talk directly, don’t force a shared “Order” model — make them separate contexts with events.

2. **Team boundaries and autonomy**  
   - Design contexts so teams own end-to-end: model, database, deployment, monitoring.  
   - Example: Extract Pricing context to separate service only when the Pricing team exists and needs different release cadence.

3. **DDD at scale**  
   - Govern Ubiquitous Language across 10+ contexts.  
   - Establish patterns: event versioning, ACL standards, shared kernel (rarely!).  
   - Decide when to relax DDD: generic subdomains get off-the-shelf solutions.

4. **Governance and consistency**  
   - Create lightweight architecture decision records (ADRs) for context boundaries.  
   - Run cross-team model reviews for integrating contexts.  
   - Define fitness functions (automated tests) that fail builds if domain rules are violated.

**Practical examples from my experience**

- At a bank: Aligned Risk, Trading, and Compliance contexts to actual organizational silos → reduced change coordination from weeks to hours.  
- At a logistics company: Prevented premature microservices split by keeping 6 contexts in a modular monolith for 2 years → saved millions in ops cost.  
- At a SaaS platform: Introduced governance where teams must present Context Map changes → stopped model erosion across 15 teams.

**What to avoid**

- Forcing “perfect” DDD everywhere — some parts should be simple CRUD.  
- Ignoring organizational politics — boundaries that don’t respect reporting lines fail.

Real impact: Staff/Principals who get this right enable organizations to scale from 5 to 50+ teams while keeping systems evolvable. The architecture starts serving the business growth instead of blocking it.

### Summary Table

| Level          | Primary Focus                          | Biggest Risk to Avoid                     | Key Deliverable                          |
|----------------|----------------------------------------|-------------------------------------------|------------------------------------------|
| Junior         | Naming, rich objects, simple behavior  | Anemic models                             | Clean, readable domain classes           |
| Mid            | Aggregates, events, invariants         | Over-engineering tactical patterns        | Consistent, testable business rules      |
| Senior         | Bounded contexts, strategic modeling   | One big model, tight coupling             | Clear Context Map, decoupled integration |
| Staff/Principal| Org alignment, governance, scale       | Architecture/team mismatch                | Sustainable evolution across teams       |

Grow through these stages deliberately. Each level builds on the previous — you can’t design good strategic boundaries without first mastering rich aggregates, and you can’t align architecture to organization without understanding real bounded contexts in production.

# Domain-Driven Design Case Study: Large-Scale E-Commerce System

**Project Context**  
A multi-channel e-commerce platform selling physical goods globally.  
- Handles 10M+ orders/year, Black Friday peaks at 50k orders/hour.  
- Multiple sales channels: web, mobile app, marketplace integrations, B2B portal.  
- Complex pricing, promotions, taxes, inventory allocation, multi-warehouse fulfillment, returns, fraud detection.  
- 8 development teams, evolving over 5+ years.

I architected the strategic redesign of this system after the original monolith became unmaintainable.

## 1. Domain Analysis & Subdomains

We ran multiple Big Picture Event Storming sessions with product owners, warehouse managers, pricing analysts, customer service, and fraud teams.

**Core Subdomain**  
- **Order Fulfillment** — Placing orders, applying promotions, allocating inventory, shipping.  
  This is the competitive differentiator: fast, accurate fulfillment with complex rules wins customers.

**Supporting Subdomains**  
- Customer Management (profiles, preferences, segmentation).  
- Product Catalog Management (attributes, categories, merchandising).

**Generic Subdomains**  
- Payments — Integrated with Stripe + PayPal + local providers.  
- Notifications — Email/SMS via third-party services.  
- Authentication/Authorization — Off-the-shelf OAuth + RBAC.

**Decision Reasoning**  
Invest 70% of modeling effort in Order Fulfillment (core).  
Simplify or buy solutions for generic subdomains to reduce custom code surface.

## 2. Bounded Contexts

From Event Storming, language shifts and organizational handoffs revealed natural boundaries:

| Bounded Context       | Key Concepts & Language                          | Owning Team          | Reason for Separation |
|-----------------------|--------------------------------------------------|----------------------|-----------------------|
| Catalog               | Product, SKU, Variant, Category, Description     | Merchandising Team   | Merchandisers change descriptions/images independently of sales |
| Pricing & Promotions  | Price List, Promotion Rule, Coupon, Discount      | Pricing Team         | Highly volatile rules, frequent experiments |
| Order Management      | Order, Order Line, Customer, Address             | Order Core Team      | Core domain – needs rich model |
| Inventory             | Stock Level, Reservation, Warehouse Location     | Logistics Team       | Multi-warehouse allocation, backorders |
| Fulfillment           | Shipment, Pick List, Packing, Carrier Integration| Warehouse Team       | Physical process differences per warehouse |
| Payments              | Payment Attempt, Authorization, Capture         | Finance Team         | PCI compliance, multiple gateways |
| Fraud                 | Risk Score, Hold, Review Queue                   | Risk Team            | Machine learning models, manual review |

**Trade-offs**  
- More contexts → more integration points but higher team autonomy.  
- We accepted eventual consistency between contexts to avoid distributed transactions.

## 3. Aggregates (Focus on Core: Order Management Context)

**Order Aggregate** (Aggregate Root: Order)

Contains:
- Order Lines (Value Objects or small Entities)
- Shipping Address (Value Object)
- Billing Address (Value Object)
- Applied Promotions (Value Objects)
- Payment Summary (Value Object)
- Status (Value Object enum-like)

**Key Invariants** (must always be true):
1. Total amount = sum(lines × quantity) + shipping - discounts + tax
2. Cannot modify lines after order is Confirmed
3. Cannot ship without successful payment authorization
4. At least one line item required
5. Inventory must be reserved before confirmation

**Reasoning**  
- These rules require immediate consistency → belong in one transaction.  
- Order is the natural transactional boundary for checkout flow.

**Trade-offs**  
- We deliberately excluded Shipment details from Order aggregate (separate Fulfillment context).  
  Risk: Eventual consistency gap (order paid but shipment delayed).  
  Benefit: Fulfillment team can evolve warehouse processes independently.

Other Aggregates in Order Context:
- Shopping Cart (separate aggregate — converted to Order at checkout)
- Coupon (small aggregate for limited-use coupons)

## 4. Domain Events (Selected Key Events)

Published by Order Aggregate:

| Event                     | Published When                          | Consumers                                      |
|---------------------------|-----------------------------------------|------------------------------------------------|
| OrderPlaced               | Order created from cart                 | Pricing (analytics), Fraud (initial check)     |
| InventoryReservationRequested | Before confirmation                  | Inventory Context                              |
| PaymentAuthorizationRequested | Before confirmation                   | Payments Context                               |
| OrderConfirmed            | All checks passed, inventory reserved   | Fulfillment, Notifications, Loyalty            |
| OrderShipped              | Fulfillment notifies back                | Order Management (update status), Notifications|
| OrderCancelled            | Customer or CS cancels                  | Inventory (release), Payments (void/refund)     |
| OrderReturned             | Return processed                        | Inventory (restock), Payments (refund)         |

**Reasoning**  
- Events enable eventual consistency.  
- Fraud context can hold an order asynchronously without blocking checkout.  
- Fulfillment starts only after OrderConfirmed → avoids work on failed payments.

**Trade-off**  
- Risk of race conditions (e.g., inventory reserved but payment fails).  
  Mitigation: Saga pattern — if payment fails, publish OrderConfirmationFailed → Inventory releases reservation.

## 5. High-Level Flow: Order Placement (Text Diagram)

```
User → [HTTP POST /orders] → OrderController (Interfaces)
                  ↓
        PlaceOrderCommand → PlaceOrderHandler (Application)
                  ↓
   Load ShoppingCart Aggregate → Convert to Order Aggregate
                  ↓
Order.calculatePricing()          → Calls Pricing Context synchronously (read-only)
                  ↓
Order.requestInventoryReservation() → Publish InventoryReservationRequested
                  ↓
          Wait for InventoryReserved or Failed event (saga step)
                  ↓ (if reserved)
Order.requestPaymentAuthorization() → Publish PaymentAuthorizationRequested
                  ↓
          Wait for PaymentAuthorized or Failed
                  ↓ (if authorized)
Order.confirm() → Publish OrderConfirmed + OrderPlaced
                  ↓
Persist Order → Return Order ID to User

Async Consumers:
- Fulfillment subscribes to OrderConfirmed → creates Shipment
- Notifications → sends confirmation email
- Fraud → runs ML model → may publish OrderHeldForReview
```

**Real-World Complexity Handled**
- **Promotions**: Pricing Context calculates eligible discounts → returned as read model → applied in Order.
- **Taxes**: External tax service called during pricing calculation.
- **Fraud Holds**: Fraud can publish Hold after OrderConfirmed → Fulfillment pauses.
- **Partial Shipments**: Fulfillment publishes PartialShipmentCreated → Order updates status accordingly.
- **Returns**: Separate Return aggregate in Fulfillment → publishes OrderReturned.

**Key Architectural Decisions & Trade-offs**

| Decision                          | Benefit                                      | Trade-off / Risk                          |
|-----------------------------------|----------------------------------------------|-------------------------------------------|
| Separate Pricing Context          | Pricing team experiments daily without deployments in Order | Synchronous read call during checkout (cached aggressively) |
| Event-driven integration          | Teams deploy independently                   | Eventual consistency bugs (mitigated with sagas, monitoring) |
| Small Order Aggregate             | High concurrency, fast checkout              | More events and handlers                  |
| No distributed transactions       | Scalability, resilience                      | Complex compensation logic for failures   |
| Generic Payments as separate context | PCI scope limited                          | Integration testing overhead              |

**Outcome After 3 Years**
- Checkout success rate improved from 85% to 97% (fewer invariant violations).
- Pricing team runs 50+ experiments/month without touching Order code.
- Black Friday handled with zero downtime (independent scaling of contexts).
- New warehouse integration took 3 weeks instead of months (clear boundaries).

This case study shows DDD's real power: not the tactical patterns, but the strategic alignment that lets large systems and organizations evolve sustainably. The complexity didn't disappear — it was contained and managed through explicit boundaries and conscious trade-offs.

# Domain-Driven Design Case Study: Payment & Wallet System

**Project Context**  
A fintech SaaS platform offering digital wallets to consumers and businesses in emerging markets.  
- Users store money, send P2P transfers, pay merchants, top-up via bank/cards, withdraw to bank accounts.  
- Handles 5M+ active wallets, 50M+ transactions/month, peaks at 10k TPS.  
- Multi-currency (USD, local fiat, stablecoins), high regulatory scrutiny (KYC/AML, fraud, reconciliation).  
- 6 cross-functional teams, strict compliance requirements, frequent new payment methods.

I led the DDD-based architecture for a major rewrite after the original system suffered from tangled logic, reconciliation nightmares, and slow feature delivery.

## 1. Domain Analysis & Subdomains

Multiple Event Storming workshops with product managers, compliance officers, risk analysts, finance ops, and engineering revealed deep complexity.

**Core Subdomain**  
- **Wallet Transactions** — The heart of the business: moving money reliably, atomically, with correct balances and audit trails.  
  Competitive edge comes from speed, low fees, high success rate, and trust.

**Supporting Subdomains**  
- User Onboarding & KYC  
- Merchant Management  
- Fee & Commission Calculation

**Generic Subdomains**  
- Notifications (email/SMS/push)  
- External Payment Rails (bank transfers, card processors, crypto gateways)

**Decision Reasoning**  
Wallet Transactions is where correctness and performance matter most → deep DDD investment.  
KYC and notifications can be simpler or partially outsourced to avoid diverting focus.

## 2. Bounded Contexts

Language shifts and organizational boundaries drove the split:

| Bounded Context      | Key Concepts & Language                              | Owning Team             | Reason for Separation |
|----------------------|-----------------------------------------------------|-------------------------|-----------------------|
| Wallet Core          | Wallet, Balance, Transaction, Hold, Ledger Entry    | Transactions Team       | Core domain – needs absolute correctness |
| User Identity        | User, KYC Status, Limits, Verification Document     | Onboarding Team         | Compliance-driven, different lifecycle |
| Risk & Fraud         | Risk Score, Suspicious Activity, Block, Review      | Risk Team               | ML models, manual review workflows |
| Fee Engine           | Fee Rule, Commission, Pricing Tier                  | Finance Team            | Frequent changes, experiments |
| Payment Rails        | Provider, Channel (Bank, Card, Crypto), Settlement   | Integrations Team       | Third-party dependencies, varying SLAs |
| Reconciliation       | Statement, Expected vs Actual, Discrepancy          | Finance Ops Team        | Batch processing, accounting focus |

**Trade-offs**  
- More contexts → higher operational complexity but true team autonomy.  
- Compliance requires audit trails across contexts → solved with immutable domain events.

## 3. Aggregates (Focus on Core: Wallet Core Context)

**Wallet Aggregate** (Aggregate Root: Wallet)

Contains:
- WalletId (identity)
- OwnerId (reference to User Identity context)
- Currency
- List<Balance> (one per currency if multi-currency wallet)
- List<Transaction> (recent or pending — debated, see trade-offs)
- List<Hold> (funds on hold for fraud/review)

**Key Invariants**
1. Available balance = total credits − total debits − holds
2. Cannot debit if available balance insufficient
3. Cannot release hold if underlying transaction failed
4. All changes must produce balanced Ledger Entries (double-entry bookkeeping)

**Ledger Entry Aggregate** (Separate, small aggregate)
- Paired Credit and Debit entries for every transaction
- Ensures accounting integrity even if Wallet view is eventually consistent

**Transaction Aggregate** (Short-lived)
- Created for each operation (deposit, transfer, withdrawal)
- State: Pending → Committed → Failed
- Enforces atomicity within wallet boundaries

**Reasoning Behind Aggregate Boundaries**
- Wallet must protect balance invariant → natural root.
- We considered including full transaction history in Wallet → rejected (God aggregate risk).
- Instead: Transactions are separate but referenced by ID; history built via event sourcing or projections.

**Trade-offs**
- Small aggregates → high concurrency (multiple transactions on same wallet can proceed safely).
- Cost: More events and eventual consistency for reads (balance queries use read model).

## 4. Domain Events (Selected Key Events)

Published primarily from Wallet Core:

| Event                          | Published When                                  | Consumers                                          |
|-------------------------------|-------------------------------------------------|----------------------------------------------------|
| WalletCreated                  | New wallet opened                               | User Identity, Notifications                       |
| FundsDeposited                 | Successful top-up                               | Reconciliation, Fee Engine, Notifications          |
| FundsWithdrawn                 | Successful withdrawal                           | Reconciliation, Payment Rails                      |
| TransferSent                   | Outgoing P2P/merchant transfer initiated        | Recipient Wallet, Risk                             |
| TransferReceived               | Incoming transfer credited                      | Sender Wallet (confirmation), Notifications         |
| FundsHeld                      | Risk places hold                                | Notifications, Risk Dashboard                      |
| FundsReleased                  | Hold removed                                    | Balance update                                     |
| TransactionFailed              | Any failure (insufficient funds, provider error)| Payment Rails (retry), Notifications                |
| BalanceLowWarning              | Available balance < threshold                   | Notifications                                      |

**Reasoning**  
- Events carry immutable facts → perfect audit trail for regulators.  
- TransferSent/Received pair enables reliable P2P without distributed transactions.

**Trade-off**  
- No immediate consistency across sender/receiver wallets.  
  Mitigation: Saga orchestrates rollback (reverse transfer) on failure.

## 5. High-Level Flow: P2P Transfer (Text Diagram)

```
User A → [HTTP POST /transfers] → TransferController (Interfaces)
                  ↓
         SendTransferCommand → SendTransferHandler (Application)
                  ↓
Load Sender Wallet Aggregate
                  ↓
SenderWallet.initiateOutboundTransfer(
    amount, currency, recipientId, reason
)
  → Validates sufficient available balance
  → Creates pending Transaction
  → Debit Ledger Entry (temporary hold)
  → Publish TransferSent (includes transferId, amount, timestamp)
                  ↓
Persist Sender Wallet + Commit Transaction
                  ↓ (async)
TransferSent → Message Broker → Recipient Wallet Handler
                  ↓
Load Recipient Wallet Aggregate
                  ↓
RecipientWallet.receiveTransfer(TransferSent details)
  → Credit Ledger Entry
  → Publish TransferReceived
                  ↓
Persist Recipient Wallet
                  ↓ (async)
TransferReceived → Sender Wallet Handler
                  ↓
SenderWallet.confirmOutboundTransfer(TransferReceived)
  → Release hold → Commit debit
  → Publish TransferCompleted
                  ↓
Both sides notify users

Failure paths (saga rollback):
- If recipient wallet invalid → publish TransferFailed → sender reverses hold
- Risk hold after send → FundsHeld → sender notified, funds frozen
```

**Real-World Complexity Handled**
- **Multi-currency**: Conversion via external FX rate service during initiation.
- **Fraud intervention**: Risk subscribes to TransferSent → may publish FundsHeld before recipient credits.
- **Provider failures**: Withdrawal → Payment Rails → if gateway fails mid-process, saga reverses.
- **Reconciliation**: Daily batch matches Ledger Entries against provider statements.
- **Limits**: User Identity context publishes LimitUpdated → Wallet adjusts max transfer.

**Key Architectural Decisions & Trade-offs**

| Decision                              | Benefit                                            | Trade-off / Risk                                  |
|---------------------------------------|----------------------------------------------------|---------------------------------------------------|
| Separate Wallet & Transaction aggregates | High throughput, fine-grained concurrency           | More complex read models for history              |
| Event-driven P2P transfers            | No 2PC, scales horizontally                        | Eventual consistency (brief window of uncertainty)|
| Double-entry Ledger Entries           | Perfect audit & reconciliation                     | Slightly higher storage/write overhead             |
| Risk as separate context              | ML team iterates independently                     | Latency in fraud detection (async)                 |
| Fee Engine separate                   | Finance changes fees without touching core         | Synchronous read during transfer (cached heavily) |

**Outcome After 2 Years**
- 99.999% transaction correctness (verified via reconciliation).  
- Fraud team reduced false positives by 40% with independent ML deployment.  
- New payment rail integration (local mobile money) took 4 weeks instead of months.  
- Peak throughput scaled to 12k TPS with no correctness regressions.  
- Regulators praised audit trail clarity during annual review.

This system demonstrates DDD’s strength in regulated, high-stakes financial domains: clear boundaries contain complexity, rich models protect money movement invariants, and events enable safe distribution without sacrificing reliability. The trade-offs (primarily eventual consistency) were accepted consciously because the alternatives (distributed transactions or monolithic design) would have crippled scalability and team velocity.

# Practical Domain-Driven Design Checklists

These checklists are distilled from real projects — use them as gates in design sessions, PR reviews, and architecture decisions.

## 1. Design Checklist (Use in Event Storming / Modeling Sessions)

- [ ] Have we involved actual domain experts (not just product owners)?
- [ ] Is the Ubiquitous Language documented and agreed upon (key terms, meanings)?
- [ ] Have we identified the Core Subdomain where competitive advantage lies?
- [ ] Are Supporting and Generic subdomains clearly separated (to avoid over-modeling)?
- [ ] Have we discovered Bounded Contexts from language shifts, handoffs, or team boundaries?
- [ ] Is there a current Context Map showing all contexts and integration styles?
- [ ] For each context: Are key business invariants explicitly listed?
- [ ] Have we defined Aggregates based on transactional consistency needs (not data relationships)?
- [ ] Are Aggregate boundaries small enough to avoid concurrency issues?
- [ ] Have we identified Domain Events for cross-context communication?
- [ ] Do we have a saga/compensating plan for eventual consistency failures?

## 2. Implementation Checklist (Before Starting a New Feature or Context)

- [ ] Domain layer has no dependencies on infrastructure, frameworks, or external services
- [ ] Repository interfaces are defined in domain; implementations in infrastructure
- [ ] Domain Events are immutable and contain only data (no behavior)
- [ ] All business invariants are enforced inside Aggregates (not in services or controllers)
- [ ] Application Services are thin orchestrators (load → call domain → persist → publish)
- [ ] Input/output DTOs are separate from domain models
- [ ] Factories used only for complex object creation with invariants
- [ ] Domain Services used sparingly (only when behavior doesn’t fit in an Aggregate)
- [ ] External calls (APIs, messages) happen after aggregate changes and via events where possible
- [ ] Anti-Corruption Layer present when integrating with legacy/external models
- [ ] Read models separated from write model (CQRS if needed)

## 3. Code Review Checklist (For Pull Requests Touching Domain Code)

- [ ] Class/method names match current Ubiquitous Language exactly
- [ ] Behavior lives in domain objects (not anemic — no logic only in services/controllers)
- [ ] Invariants guarded inside Aggregate Root (cannot be bypassed)
- [ ] No public setters exposing internal state
- [ ] Value Objects are immutable and override equality/hash
- [ ] Aggregate Root is the only entry point (no direct access to internal entities)
- [ ] References to other Aggregates are by ID only (not object references)
- [ ] Domain Events published at correct boundaries (after invariants satisfied)
- [ ] No infrastructure concerns leaked into domain (no DbContext, HttpClient, Logger, etc.)
- [ ] Exceptions are domain-specific (not generic or technical)
- [ ] New changes respect existing Bounded Context boundaries

## 4. Architecture Review Checklist (Quarterly or Before Major Changes)

- [ ] Context Map is up-to-date and reflects reality
- [ ] Team ownership aligns with Bounded Context boundaries (Conway’s Law check)
- [ ] Integration patterns still appropriate (ACL for legacy, events for autonomy)
- [ ] Core domain receiving most investment and talent
- [ ] Generic subdomains using off-the-shelf solutions where possible
- [ ] No God Aggregates or distributed transactions creeping in
- [ ] Event schemas versioned and backward compatible
- [ ] Monitoring/alerting covers key Domain Events and saga completions
- [ ] Technical debt in non-core contexts not blocking core evolution
- [ ] New features fit within existing contexts or justify new ones
- [ ] Eventual consistency risks documented with compensation strategies
- [ ] System still evolvable: Can we change a core business rule safely and quickly?

### Usage Tips
- Pin these checklists in your team wiki/confluence.
- Make them mandatory gates in Jira/PR templates.
- Customize lightly per project — but never remove the focus on invariants, language, and boundaries.
