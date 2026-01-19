# 1. Introduction & Purpose of DDD

## 1.1. What Problems Does DDD Solve?

Most software projects fail not because of bad syntax, but because of **communication breakdown**.

* **The Translation Gap:** Developers speak in "tables, loops, and APIs." Business stakeholders speak in "liabilities, claims, and fulfillment." DDD bridges this gap.
* **The "Big Ball of Mud":** Without DDD, logic for shipping, billing, and inventory all gets tangled in one giant database. A change in "Shipping" accidentally breaks "Billing."
* **Feature Stagnation:** In old systems, adding a simple button takes three weeks because the logic is scattered across 50 files. DDD organizes logic so you know exactly where to make a change.

---

## 1.2. Why DDD Exists: History and Motivation

In 2003, **Eric Evans** published *Domain-Driven Design: Tackling Complexity in the Heart of Software*.

Before this, the industry focused heavily on **Data-Driven Design**. We would design the database tables first, then wrap code around them. But as businesses became more digital and complex, "rows and columns" weren't enough to capture complex rules (like how an insurance premium is calculated based on risk).

**The Motivation:** To move the focus away from the **technology** and back to the **business domain**. If the business is complex, the code must be organized to handle that complexity, not hide it behind generic CRUD (Create, Read, Update, Delete) operations.

---

## 1.3. When is DDD a Good Fit?

DDD is an investment. You use it when the complexity of the **business logic** outweighs the technical complexity.

* **Complex Business Rules:** If your app has "If-Then" logic that spans hundreds of pages of documentation (e.g., Banking, Healthcare, Logistics).
* **Long-Lived Projects:** If the software will be maintained for 5+ years. The structure DDD provides prevents the "legacy code" rot.
* **Large Teams:** When you have multiple teams working on the same product. DDD helps define clear boundaries so teams don't step on each other's toes.

---

## 1.4. When is DDD NOT a Good Fit?

I have seen many "Staff Engineers" over-engineer simple apps with DDD, which is a costly mistake.

* **Simple CRUD Apps:** If your app just moves data from a form to a database (e.g., a simple To-Do list or a blog), DDD is overkill.
* **Prototypes / MVPs:** If you are just trying to see if a product idea works, don't spend weeks modeling the domain. Use a rapid framework and "hack" it together.
* **Purely Technical Tools:** A JSON parser or a database driver doesn't have a "business domain." These are technical problems, not DDD problems.

---

## 1.5. Common Misconceptions

Let's clear the air on what DDD is *not*:

* **"DDD is about Microservices":** No. You can do DDD in a monolith (and you probably should start there). DDD helps you find the lines *where* you could split into microservices later.
* **"DDD is a Foldering Structure":** Putting files in folders named `Domain`, `Infrastructure`, and `Application` doesn't mean you're doing DDD. If your logic is still just getters and setters, it’s just "DDD-flavored" CRUD.
* **"Only Architects do DDD":** DDD only works if the junior developers and the product managers are involved. If the developers don't use the same words as the business, the model is a lie.

---

## 1.6. Summary: The "Why" for the Whole Team

* **For the Fresher:** DDD gives you a map. It tells you exactly where a piece of code belongs so you don't feel lost in a million-line codebase.
* **For the Senior:** DDD gives you a way to say "No" to spaghetti code. It provides a formal framework for keeping boundaries clean.
* **For the Staff Engineer:** DDD is your tool for organizational design. By defining **Bounded Contexts**, you define how teams interact and how data flows across the entire company.

---

# 2. Core DDD Concepts

Think of Domain-Driven Design (DDD) as a philosophy that says: **"Before we touch the keyboard, we must master the business."** In my experience, 90% of software bugs are actually "misunderstanding bugs"—where the developer built exactly what they *thought* was needed, but it wasn't how the business actually works.

---

## 2.1. The Domain

**Definition:** The "sphere of knowledge" or the specific subject area the software is intended to solve. It is the entire ecosystem of your business.

* **Real-World Analogy:** If you are building a hospital management system, "Medicine/Healthcare" is your domain. It includes doctors, patients, insurance, and prescriptions.
* **Business Example (E-commerce):** The domain is **Retail**. It encompasses everything from finding a product to delivering it to a doorstep.
* **Why it Matters:** You cannot build a solution for a problem you don't understand. If you don't understand the domain, you are just "guessing" in code.
* **Common Mistake:** Thinking the "Domain" is the "Database." The domain is the *real-world activity*, not the tables you store data in.

---

## 2.2. Subdomains (Core, Supporting, Generic)

**Definition:** Since a Domain is too big to handle at once, we break it into smaller pieces.

### A. Core Subdomain

* **Simple Definition:** The "Secret Sauce." This is why the business exists and why customers choose them over competitors.
* **Example (E-commerce):** A unique **Recommendation Engine** that predicts exactly what you want to buy.
* **Why it Matters:** You should spend your most expensive senior developer hours here.

### B. Supporting Subdomain

* **Simple Definition:** Necessary logic that supports the Core but isn't a competitive advantage.
* **Example (E-commerce):** An **Inventory Tracker**. It’s important, but it doesn't make your brand unique.
* **Why it Matters:** These can often be built by junior teams or outsourced.

### C. Generic Subdomain

* **Simple Definition:** Problems that everyone has and have already been solved by someone else.
* **Example (E-commerce):** **Payment Processing** or **User Authentication**.
* **Why it Matters:** **Don't build these.** Buy a solution (like Stripe or Auth0) so you can focus on your Core.
* **Common Mistake:** Spending six months building a custom "Generic" email notification system while the "Core" business logic is full of bugs.

---

## 2.3. Ubiquitous Language

**Definition:** A shared, common language used by *both* developers and business stakeholders. No "dev-speak."

* **Real-World Analogy:** If a pilot says "Requesting landing," the air traffic controller doesn't say "Acknowledge inbound vectoring." They use the same terms to avoid a crash.
* **Business Example (Banking):** If the banker calls it an "Overdraft Fee," the code should be `CalculateOverdraftFee()`, not `ProcessPenalty7()`.
* **Why it Matters:** It eliminates "translation lag." When the business changes a rule, you know exactly which part of the code to update because they share the same name.
* **Common Mistake:** Creating a "Technical Dictionary" that translates business terms into "Dev terms." Just use the business terms in the code!

---

## 2.4. Bounded Context

**Definition:** A boundary (usually a boundary of a specific team or sub-system) within which a particular model and its language are strictly consistent.

* **Real-World Analogy:** The word "Ticket." To a **Customer Support** agent, a ticket is a complaint. To a **Cinema** employee, a ticket is a seat reservation. They are the same word, but in different "contexts," they mean totally different things.
* **Business Example (Booking):** In the **Reservation Context**, a "User" is a "Guest." In the **Billing Context**, that same person is a "Payer."
* **Why it Matters:** It prevents "God Objects." If you try to make one `User` class that handles both "Guest" and "Payer" logic, the code becomes an unmaintainable mess.
* **Common Mistake:** Trying to create a "Global Model" that works for the whole company. This is impossible in large systems.

---

## 2.5. Context Map

**Definition:** A high-level bird's-eye view showing how different Bounded Contexts talk to each other.

* **Real-World Analogy:** A map of a city showing different neighborhoods and the bridges that connect them.
* **Business Example (E-commerce):** A map showing how the **Ordering Context** sends a message to the **Shipping Context** once a payment is successful.
* **Why it Matters:** It exposes "Political" and "Technical" bottlenecks. It shows you who depends on whom. If the Shipping team changes their API, which other teams will break?
* **Common Mistake:** Ignoring the map and letting every service talk to every other service. This results in "Spaghetti Architecture" where you can't change anything without breaking everything.

---

# 3. Tactical Design
In my experience, tactical DDD is where most "clean code" efforts fail. Many engineers treat these patterns as a checkbox—adding a `Repository` just because a tutorial said so—without understanding the underlying **Invariants**.

Tactical DDD is about **encapsulating business rules** so that it is impossible to put the system into an invalid state.

---

## 3.1. Value Object (The Foundation)

**Definition:** An object that describes a characteristic or an aspect of the domain but has no identity. It is defined solely by its attributes.

* **Responsibility:** To be immutable and encapsulate validation logic.
* **Business Example:** `Money` (Amount + Currency), `EmailAddress`, `DateRange`.
* **Interactions:** Used as properties inside Entities or other Value Objects.
* **Common Mistakes:** Giving a Value Object an `Id`. If you have a `MoneyId`, you’ve built an Entity by mistake.

---

## 3.2. Entity

**Definition:** An object that has a unique identity that persists over time, regardless of changes to its attributes.

* **Responsibility:** To track state changes for a specific "thing" in the system.
* **Business Example:** A `User`, a `Bank Account`, or a `Support Ticket`. Two users with the same name are still different people because they have different IDs.
* **Interactions:** Contains Value Objects and is usually part of an Aggregate.
* **Common Mistakes:** Putting all logic in the Entity. If a rule involves *multiple* different objects, an Entity shouldn't handle it alone.

---

## 3.3. Aggregate & Aggregate Root

**Definition:** An **Aggregate** is a cluster of associated objects (Entities and Value Objects) that we treat as a single unit for data changes. The **Aggregate Root** is the only member of the aggregate that outside objects are allowed to hold a reference to.

* **Responsibility:** To enforce **Business Invariants** (rules that must always be true).
* **Business Example:** An `Order` (Root) and its `OrderLines` (Entities). You can't add an `OrderLine` without going through the `Order`.
* **Interactions:** The Root ensures all internal objects stay consistent.
* **Common Mistakes:** "The Giant Aggregate." Trying to put the entire database into one Aggregate. This causes massive performance bottlenecks.

---

## 3.4. Why Aggregates Exist: Boundaries and Invariants

This is the "Senior" level understanding. Aggregates exist to manage **Invariants** and **Transactions**.

### Business Invariants

An invariant is a rule that must be true *at all times*.

* *Example:* "The total of an Order cannot exceed the Customer's credit limit."
* If `Order` and `Customer` were in the same Aggregate, you could enforce this easily. If they are separate, you need a different strategy.

### Transaction Boundaries

**Rule:** One transaction = One Aggregate.
If you need to update two things at once, they should probably be in the same Aggregate. If they are separate Aggregates, you must accept **Eventual Consistency**.

---

## 3.5. Domain Service

**Definition:** A place for business logic that doesn't naturally belong to a single Entity or Value Object.

* **Responsibility:** Orchestrating logic that involves multiple Aggregates or external lookups.
* **Business Example:** A `FundsTransferService`. It needs to interact with two `Account` Aggregates. An Account shouldn't "know" how to push money into another account; the Service handles the interaction.
* **Interactions:** Uses Repositories to fetch Aggregates and coordinates them.
* **Common Mistakes:** "Anemic Domain Model." Putting *all* logic in services and leaving Entities as empty data holders. This is just procedural programming in disguise.

---

## 3.6. Repository

**Definition:** A mechanism that encapsulates the logic required to access data sources. It acts like an in-memory collection of Aggregates.

* **Responsibility:** Mediating between the Domain and the Database.
* **Business Example:** `IOrderRepository.Save(order)` or `customerRepo.FindById(id)`.
* **Interactions:** Only works with **Aggregate Roots**. You should never have an `OrderItemRepository`.
* **Common Mistakes:** Returning DTOs or raw SQL results from a Repository. It should return Domain Entities.

---

## 3.7. Factory

**Definition:** A pattern used to encapsulate the complexity of creating a highly complex Aggregate.

* **Responsibility:** Ensuring a complex object is "born" in a valid state.
* **Business Example:** A `LoanFactory` that takes a credit score and income to determine which specific `Loan` entity type to instantiate.
* **Interactions:** Used by the Application Layer or Domain Services to get a new Aggregate.
* **Common Mistakes:** Using a Factory for simple objects. If you can just use a constructor, do it.

---

## 3.8. Domain Event

**Definition:** A record of something significant that happened in the domain.

* **Responsibility:** Decoupling different parts of the system.
* **Business Example:** `OrderPlaced`, `PaymentFailed`, `UserRegistered`.
* **Interactions:** When an Aggregate Root finishes a task, it "emits" an event. Other Bounded Contexts or services listen to these to trigger their own logic (e.g., sending an email when `OrderPlaced` occurs).
* **Common Mistakes:** Using events for technical things like `DatabaseRowInserted`. Events should be named after **business facts**.

---

### Summary Table for the Team

| Concept | Level | Purpose |
| --- | --- | --- |
| **Value Object** | Low | Data integrity & Immytability |
| **Entity** | Low | Identity & State tracking |
| **Aggregate** | Mid | Consistency & Transaction boundary |
| **Repository** | High | Data persistence (the "how") |
| **Domain Event** | High | Communication between boundaries |

---

# 4. Strategic Design
In my experience as an architect, Strategic DDD is the most difficult—and most important—part of the job. You can have the cleanest code in the world, but if your boundaries are wrong, you’ll end up with a **Distributed Monolith**: a system that has all the complexity of microservices but none of the benefits.

Strategic DDD is about **organizational design** and **error isolation**.

---

## 4.1. Bounded Context Design: The "Wall"

In a large system, the same word means different things in different places. A "Product" in **Sales** is about pricing; in **Inventory**, it’s about weight and dimensions.

A **Bounded Context** is a linguistic and logical boundary. Inside the boundary, the model is consistent. Outside the boundary, the model can (and should) be different.

### How to Discover Boundaries

Don't look at database tables. Look at **Business Workflows** and **Team Communication**.

* **Step 1: Event Storming:** Map out the business process from start to finish. Where does the "language" change?
* **Step 2: Identify Ownership:** Who makes the rules for this data? If the Warehouse team owns the data for "Bin Locations," that’s a strong hint for an Inventory Context boundary.
* **Step 3: Look for Friction:** If two teams are constantly arguing over a single "User" table, you’ve found a boundary that needs to be split.

---

## 4.2. Context Map Patterns

Once you have boundaries, you must define how they talk. The "Context Map" describes the **power dynamic** between teams.

| Pattern | Description | When to Use |
| --- | --- | --- |
| **Shared Kernel** | Two teams share a small piece of code/library (e.g., a shared library for "Address"). | When teams are closely aligned and changes are rare. **Dangerous:** Leads to tight coupling. |
| **Customer/Supplier** | The Supplier (Upstream) provides data; the Customer (Downstream) uses it. The Supplier takes the Customer's needs into account. | Standard relationship between an Order team and a Shipping team. |
| **Conformist** | The Downstream team just uses whatever the Upstream team gives them, no questions asked. | When you are integrating with a huge, rigid system (like a legacy ERP or Amazon's API). |
| **Anti-Corruption Layer (ACL)** | A translation layer that converts the "messy" model of another system into your own clean model. | **Essential** when your new clean DDD system needs to talk to a messy legacy monolith. |

---

## 4.3. DDD in Monoliths vs. Microservices

There is a massive misconception that DDD equals Microservices.

* **DDD in a Monolith:** You use **Modules** (or "Modular Monoliths"). Each Bounded Context is a separate package/folder. Communication happens via internal method calls or internal events. This is the best place to start.
* **DDD in Microservices:** Each Bounded Context becomes a separate deployable unit. Communication happens via REST, gRPC, or Message Brokers (Kafka/RabbitMQ).

### The Mapping: 1 Bounded Context = 1 Microservice?

This is the "ideal" state, but reality is messy.

* **When it works:** When a team owns a specific business capability (e.g., "Payments") and can deploy it independently.
* **When it’s dangerous:** When you split a context too small. If Service A and Service B have "chatty" communication (constantly calling each other to finish one task), you’ve created a **Distributed Monolith**. You get network latency and deployment headaches without the isolation benefits.

---

## 4.4. Real-World Decision Making & Trade-offs

### The "False Boundary" Trap

Freshers often split contexts based on **Technical Layers** (e.g., a "Database Service"). Seniors split based on **Business Meaning**.

* *Constraint:* If changing a business rule in "Billing" requires you to deploy "Shipping," your boundaries are wrong.

### The Anti-Corruption Layer (ACL) Cost

* **Trade-off:** An ACL adds development time and a small performance hit (serialization/translation).
* **The Decision:** If the external system is stable and clean, go **Conformist**. If the external system is "spaghetti," the ACL is worth its weight in gold to protect your domain's sanity.

### Team Ownership (Conway’s Law)

The software will eventually look like your org chart. If you have one team of 50 people, a single Bounded Context is fine. If you have 5 teams of 10, you *need* 5 Bounded Contexts to prevent them from blocking each other.

---

## 4.5. Summary Checklist for Architects

1. **Define the Core:** Spend 80% of your energy on the Core Subdomain.
2. **Draw the Map:** Before writing code, draw the Context Map. Who is Upstream? Who is Downstream?
3. **Protect the Domain:** Use an ACL when dealing with legacy.
4. **Start Monolithic:** Don't pay the "Microservices Tax" until your domain is stable and the team size requires it.

---

# 5. Applying DDD in Real Projects
In my experience, the biggest mistake people make is treating DDD as a "big bang" rewrite. You don't "do" DDD; you evolve into it. Here is a practical execution plan for both greenfield and legacy environments.

---

## 5.1. Starting From Scratch (The Greenfield Path)

When you have a blank canvas, the goal is to prevent it from becoming a "Big Ball of Mud" in six months.

* **Step 1: Identify the "Core."** Ask the business owners: "If this system fails, which part stops us from making money?" That is your Core Subdomain. Build that first.
* **Step 2: Define "Draft" Bounded Contexts.** Don't aim for perfection. Group features by team ownership or lifecycle. (e.g., "Orders" and "Users" are usually separate lifecycles).
* **Step 3: Establish the Ubiquitous Language.** Create a shared Wiki or README. If a stakeholder says "Client" and you call it "Account," stop the meeting and pick one word. Use that word in the class names, the database tables, and the UI.
* **Step 4: Build a "Walking Skeleton."** Implement one end-to-end flow using Tactical DDD (Aggregate -> Repository -> Domain Event). This proves your architecture works before you scale.

---

## 5.2. Event Storming: The Discovery Workshop

Don't use UML or Jira for discovery. Use **Event Storming**. It is the fastest way to find Bounded Contexts.

1. **Invite the right people:** One Developer, one Product Owner, one Domain Expert (e.g., an actual Accountant or Warehouse Manager).
2. **Plot Domain Events (Orange Post-its):** Place them on a timeline. Use past tense: `OrderPlaced`, `PaymentFailed`, `PackageShipped`.
3. **Identify Triggers (Blue Post-its):** What caused the event? A user action (`Place Order`) or a system timer?
4. **Look for "Pivotal Points":** Where does the language shift? When `PaymentAuthorized` happens, we stop talking about "Carts" and start talking about "Shipping." **That is your Bounded Context boundary.**

---

## 5.3. Working with Domain Experts

Domain experts aren't there to give you "requirements"; they are there to help you model the truth.

* **Action:** Stop asking "What data do you need to store?" Start asking **"What happens when X goes wrong?"**
* **The "Listen for 'If' and 'But'" Rule:** When an expert says, "Usually we do X, *but* if the customer is premium, we do Y," you just discovered a **Business Invariant** that belongs inside an Aggregate.
* **Code Reviews for Non-Coders:** Occasionally show your Domain logic (the clean, non-technical parts) to the expert. If they can’t understand the logic in your `CalculateDiscount()` method, your Ubiquitous Language is failing.

---

## 5.4. Introducing DDD to Legacy Systems

You cannot refactor a 10-year-old monolith in a month. You must use the **Strangler Fig Pattern**.

* **Step 1: The Anti-Corruption Layer (ACL).** Before building new features, build a "translator" between the legacy database and your new DDD code. The new code should feel "pure," even if it’s talking to a messy old SQL table.
* **Step 2: Carve out a "Value Object."** Find a piece of logic—like "Tax Calculation"—and move it from a 5,000-line service into a small, immutable Value Object. Test it in isolation.
* **Step 3: Bubbling up the Aggregate.** Identify a "Root" in the legacy system. Start routing all updates to that entity through a single method to enforce rules.

---

## 5.5. Refactoring Safely Toward DDD

DDD is an iterative process. Your first model will be wrong.

* **Refactor to Discovery:** When you realize two things you thought were different are actually the same (or vice-versa), rename the classes immediately.
* **Break Large Aggregates:** If your `Order` aggregate is getting too slow because it loads 500 `OrderItems`, refactor the items into a separate Aggregate and link them by ID.
* **Shift Logic "Down":** If you find business logic in your API controllers, push it "down" into the Domain Service. If it’s in a Domain Service but only touches one Entity, push it "down" into that Entity.

---

## ⚠️ Warnings and Risks

* **The "DDD Tax":** Tactical DDD (Factories, Repositories, etc.) requires more files and more boilerplate. If the project is small, this tax will kill your velocity.
* **Database Obsession:** If your team starts the conversation with "How will this look in SQL?", you aren't doing DDD. You're doing Data Modeling. Ignore the database until the Domain Model is stable.
* **Isolationism:** If teams don't talk, their Bounded Contexts will drift. A **Context Map** is a living document—update it every sprint.

---

# 6. Applying DDD in Real Projects

## 6.1. Starting From Scratch (The Greenfield Path)

When you have a blank canvas, the goal is to prevent it from becoming a "Big Ball of Mud" in six months.

* **Step 1: Identify the "Core."** Ask the business owners: "If this system fails, which part stops us from making money?" That is your Core Subdomain. Build that first.
* **Step 2: Define "Draft" Bounded Contexts.** Don't aim for perfection. Group features by team ownership or lifecycle. (e.g., "Orders" and "Users" are usually separate lifecycles).
* **Step 3: Establish the Ubiquitous Language.** Create a shared Wiki or README. If a stakeholder says "Client" and you call it "Account," stop the meeting and pick one word. Use that word in the class names, the database tables, and the UI.
* **Step 4: Build a "Walking Skeleton."** Implement one end-to-end flow using Tactical DDD (Aggregate -> Repository -> Domain Event). This proves your architecture works before you scale.

---

## 6.2. Event Storming: The Discovery Workshop

Don't use UML or Jira for discovery. Use **Event Storming**. It is the fastest way to find Bounded Contexts.

1. **Invite the right people:** One Developer, one Product Owner, one Domain Expert (e.g., an actual Accountant or Warehouse Manager).
2. **Plot Domain Events (Orange Post-its):** Place them on a timeline. Use past tense: `OrderPlaced`, `PaymentFailed`, `PackageShipped`.
3. **Identify Triggers (Blue Post-its):** What caused the event? A user action (`Place Order`) or a system timer?
4. **Look for "Pivotal Points":** Where does the language shift? When `PaymentAuthorized` happens, we stop talking about "Carts" and start talking about "Shipping." **That is your Bounded Context boundary.**

---

## 6.3. Working with Domain Experts

Domain experts aren't there to give you "requirements"; they are there to help you model the truth.

* **Action:** Stop asking "What data do you need to store?" Start asking **"What happens when X goes wrong?"**
* **The "Listen for 'If' and 'But'" Rule:** When an expert says, "Usually we do X, *but* if the customer is premium, we do Y," you just discovered a **Business Invariant** that belongs inside an Aggregate.
* **Code Reviews for Non-Coders:** Occasionally show your Domain logic (the clean, non-technical parts) to the expert. If they can’t understand the logic in your `CalculateDiscount()` method, your Ubiquitous Language is failing.

---

## 6.4. Introducing DDD to Legacy Systems

You cannot refactor a 10-year-old monolith in a month. You must use the **Strangler Fig Pattern**.

* **Step 1: The Anti-Corruption Layer (ACL).** Before building new features, build a "translator" between the legacy database and your new DDD code. The new code should feel "pure," even if it’s talking to a messy old SQL table.
* **Step 2: Carve out a "Value Object."** Find a piece of logic—like "Tax Calculation"—and move it from a 5,000-line service into a small, immutable Value Object. Test it in isolation.
* **Step 3: Bubbling up the Aggregate.** Identify a "Root" in the legacy system. Start routing all updates to that entity through a single method to enforce rules.

---

## 6.5. Refactoring Safely Toward DDD

DDD is an iterative process. Your first model will be wrong.

* **Refactor to Discovery:** When you realize two things you thought were different are actually the same (or vice-versa), rename the classes immediately.
* **Break Large Aggregates:** If your `Order` aggregate is getting too slow because it loads 500 `OrderItems`, refactor the items into a separate Aggregate and link them by ID.
* **Shift Logic "Down":** If you find business logic in your API controllers, push it "down" into the Domain Service. If it’s in a Domain Service but only touches one Entity, push it "down" into that Entity.

---

## ⚠️ Warnings and Risks

* **The "DDD Tax":** Tactical DDD (Factories, Repositories, etc.) requires more files and more boilerplate. If the project is small, this tax will kill your velocity.
* **Database Obsession:** If your team starts the conversation with "How will this look in SQL?", you aren't doing DDD. You're doing Data Modeling. Ignore the database until the Domain Model is stable.
* **Isolationism:** If teams don't talk, their Bounded Contexts will drift. A **Context Map** is a living document—update it every sprint.

---

# 7. DDD + Architecture Patterns
In my experience, the greatest value of DDD isn't the patterns themselves, but how they provide a "home" for business logic within modern architectures. Without DDD, Clean Architecture is just a collection of folders; without Clean Architecture, DDD logic eventually leaks into your database or UI.

Here is how these pieces fit together in a production-grade system.

---

## 7.1. The Core: Hexagonal & Clean Architecture

These two patterns are essentially the same for our purposes: they place the **Domain** at the center.

### Where the Domain Lives

The Domain layer is the innermost circle. It contains your **Aggregates, Entities, Value Objects, and Domain Services.**

* **The Dependency Rule:** Dependencies only point **inward**.
* The **Application Layer** (Use Cases) depends on the **Domain**.
* The **Infrastructure Layer** (DB, Email, API) depends on the **Domain**.
* **The Domain depends on nothing.** It is "Plain Old Objects" (POJOs/POCOs).



### What the Domain Must NEVER Depend On

* **Database Frameworks:** No ORM annotations (like `@Entity` from JPA or EF Core attributes) if you want to be a purist.
* **Web Frameworks:** No HTTP status codes or Controller references.
* **Third-party Clients:** No SendGrid, Stripe, or AWS SDKs.

---

## 7.2. CQRS (Command Query Responsibility Segregation)

In complex domains, the model used to **change** data is often different from the model used to **read** data.

* **Command Side:** Uses Tactical DDD (Aggregates) to ensure business invariants are met during updates. This side is optimized for **consistency**.
* **Query Side:** Bypasses the Domain model entirely. It uses "Projections" or simple DTOs optimized for **speed and UI requirements**.

**The Senior Insight:** Don't use your Aggregate for search/list views. It’s slow and heavy. Use a flat SQL view or an ElasticSearch index for the Query side.

---

## 7.3. Event-Driven Architecture (EDA)

EDA is the "nervous system" of DDD. It allows Bounded Contexts to remain decoupled.

* **Inside the Context:** We use **Domain Events** to trigger side effects within the same boundary (e.g., `OrderPlaced` triggers `SendEmail`).
* **Between Contexts:** We use **Integration Events**. These are stripped-down versions of domain events sent over a message bus (Kafka/RabbitMQ) to inform other Bounded Contexts of a change.

---

## 7.4. Microservices & API Gateway

Strategic DDD is the blueprint for a microservices transition.

* **Boundary Mapping:** Ideally, **1 Bounded Context = 1 Microservice**.
* **API Gateway:** The Gateway should never contain domain logic. Its job is routing, rate limiting, and sometimes "Composition" (calling three Bounded Contexts to build one UI screen).
* **The Trap:** If your API Gateway is calculating discounts or validating business rules, you have a "Leaky Abstraction" that will make your microservices impossible to manage.

---

## 7.5. Summary: Common Architectural Mistakes

| Mistake | Why it happens | The Fix |
| --- | --- | --- |
| **Leaky Infrastructure** | Putting SQL queries inside an Aggregate Root. | Use the **Repository Pattern** to hide the "how" of data access. |
| **Circular Dependencies** | Context A calls Context B, which calls Context A. | Use **Domain Events** to decouple the two. |
| **Anemic Use Cases** | The Application Layer is doing all the work, and the Entities are just data buckets. | Move the "How" into the **Entity**; keep the "When" in the **Application Layer**. |
| **The "Big" Transaction** | Trying to update three Aggregates in one REST call. | Update the primary Aggregate and use **Eventual Consistency** for the others. |

---

# 8. Project & Folder Structure
In my career, I've seen thousands of "DDD projects" that were just CRUD apps with more folders. A project structure is not just a filing system; it is a **safety mechanism** designed to protect your business logic from the chaos of the outside world.

---

## 8.1. Typical DDD Project Structure

This structure follows **Hexagonal/Clean Architecture** principles. The goal is to ensure that the "Domain" can be tested and understood without ever starting a database or a web server.

```text
src/
├── Domain/                 <-- THE HEART: Pure business logic only.
│   ├── Aggregates/         <-- Consistency boundaries (Order, Customer)
│   ├── Entities/           <-- Objects with IDs (OrderItem)
│   ├── ValueObjects/       <-- Immutable logic (Money, Address)
│   ├── Events/             <-- Business facts (OrderPlaced)
│   ├── Services/           <-- Logic spanning multiple entities
│   └── RepositoryInterfaces/ <-- "Contracts" for data (not the implementation)
│
├── Application/            <-- THE COORDINATOR: Use Cases / Orchestration.
│   ├── Commands/           <-- Actions that change state (PlaceOrder)
│   ├── Queries/            <-- Actions that fetch data
│   ├── DTOs/               <-- Data contracts for the outside world
│   └── EventHandlers/      <-- Reacting to domain events
│
├── Infrastructure/         <-- THE PLUMBING: Technical details.
│   ├── Persistence/        <-- SQL/NoSQL implementations of Repositories
│   ├── ExternalServices/   <-- Stripe, AWS, SendGrid adapters
│   └── Serialization/      <-- JSON/Protobuf mapping
│
└── API/                    <-- THE ENTRY: Delivery mechanism.
    ├── Controllers/        <-- Routes and Request/Response handling
    ├── Middleware/         <-- Auth, Logging
    └── Models/             <-- API-specific Request schemas

```

### Why this exists:

1. **Isolation:** You can change your database from PostgreSQL to MongoDB by only touching `Infrastructure`, leaving the `Domain` untouched.
2. **Testability:** You can unit test the `Domain` logic in milliseconds because it has no external dependencies.
3. **Ubiquitous Language:** A new developer can look at the `Domain` folder and understand the business without knowing the tech stack.

---

## 8.2. Anemic vs. Rich Domain Models

The biggest "silent killer" in DDD is the **Anemic Domain Model**. This is when you have the folders, but your entities are just bags of data (getters and setters).

### The Anemic Model (Anti-Pattern)

The logic is "leaked" into the Service layer. The Entity is "brain dead."

```typescript
// ANEMIC: Just a data container
class Order {
  public id: string;
  public status: string;
  public totalAmount: number;
}

// SERVICE: The "Brain" (This is procedural code)
class OrderService {
  updateOrder(order: Order, newStatus: string) {
    if (order.totalAmount > 1000 && newStatus === 'CANCELLED') {
       throw new Error("High value orders require manual review to cancel");
    }
    order.status = newStatus;
    this.repo.save(order);
  }
}

```

**The Problem:** The business rule is hidden in a service. Anyone can create an `Order` and set its status to "CANCELLED" without checking the rule.

---

### The Rich Domain Model (The DDD Way)

The logic is encapsulated inside the Entity/Aggregate. It is impossible to put the object in an invalid state.

```typescript
// RICH: The Entity protects its own invariants
class Order {
  private id: string;
  private status: OrderStatus;
  private totalAmount: Money; // Value Object

  // Business logic lives here
  public cancel() {
    if (this.totalAmount.isGreaterThan(1000)) {
      throw new ManualReviewRequiredError();
    }
    this.status = OrderStatus.CANCELLED;
    this.addDomainEvent(new OrderCancelledEvent(this.id));
  }
}

// APPLICATION LAYER: Just orchestrates
class CancelOrderHandler {
  handle(command: CancelOrderCommand) {
    const order = this.repo.getById(command.orderId);
    order.cancel(); // Logic is inside the domain
    this.repo.save(order);
  }
}

```

**The Benefit:** The `Order` object is a "Fortress." You cannot bypass the business rules. This is how you prevent bugs as a system scales to hundreds of engineers.

---

## 8.3. Folder Responsibilities Summary

| Folder | Responsibility | Allowed to depend on... |
| --- | --- | --- |
| **Domain** | Business Rules, Invariants, Entities | Nothing. |
| **Application** | Orchestration, Use Case flow, Transaction mgmt | Domain. |
| **Infrastructure** | Database, API Clients, File System | Domain & Application. |
| **API / Presentation** | HTTP, UI, Input Validation | Application & Domain. |

---

# 9. DDD by Engineer Level
In my 20 years of experience, the biggest mistake companies make is forcing the entire engineering org to learn DDD at the same depth simultaneously. DDD is a ladder; trying to jump to the top rung (Strategic Design) before you understand the bottom rung (Value Objects) leads to architectural disasters.

Here is how you should approach DDD based on your career stage.

---

## 9.1. The Fresher / Junior Engineer: "The Craft"

At this level, your goal is to stop thinking in **strings and integers** and start thinking in **types**.

* **Focus on: Value Objects.** This is the most underrated part of DDD. Instead of passing a `string` for an email or a `decimal` for a price, create an `EmailAddress` or a `Money` class. This moves validation from your services into the objects themselves.
* **Key Mental Model: "Valid on Construction."** An object should never be allowed to exist in an invalid state. If an `Order` requires at least one `Item`, the constructor should throw an error if the list is empty.
* **Avoid: The "Database First" Mindset.** Don't ask, "What does the table look like?" Ask, "What are the rules for this piece of data?"

> **Practical Example:** Instead of writing `if (user.age > 18)` in ten different controllers, create a `BirthDate` Value Object with a `.isAdult()` method. Now, the logic is central and reusable.

---

## 9.2. The Mid-Level Engineer: "The Boundary"

Once you can write clean objects, you need to learn how to group them into **Aggregates** and communicate between them.

* **Focus on: Aggregate Design.** This is the hardest tactical skill. You must decide what belongs together in one database transaction.
* **Key Skill: Introducing Domain Events.** Instead of Service A calling Service B directly (tight coupling), have Service A publish a `JobOfferCreated` event. This makes your code much easier to extend.
* **Avoid: Over-engineering.** Not every entity needs to be an Aggregate Root. Not every service needs a Repository. If you are building a simple "Contact Us" form, don't use DDD. Use simple CRUD.

---

## 9.3. The Senior Engineer: "The System"

A Senior Engineer looks beyond the code and focuses on the **Lifecycle** of the system and its distributed nature.

* **Focus on: Strategic Boundaries.** You are the "Border Patrol." You define where the **Ordering Context** ends and the **Billing Context** begins. You ensure that the billing team doesn't leak their database concepts into the ordering team's API.
* **Complexity Management: The Anti-Corruption Layer (ACL).** When your clean new system has to talk to a 15-year-old legacy SOAP API, you build an ACL. You protect your team's "Ubiquitous Language" from being poisoned by legacy jargon.
* **Distributed Concerns: Eventual Consistency.** You understand that in a large system, you cannot have a single transaction across the whole company. You design systems that can handle a "Price Updated" event arriving 2 seconds after the "Item Added to Cart" event.

---

## 9.4. The Staff / Principal Architect: "The Organization"

At this level, DDD is no longer a technical tool; it is a tool for **Organizational Design**.

* **Focus on: Team vs. Architecture Alignment.** You use **Conway’s Law** to your advantage. If the business wants a "Fast-Track Shipping" division, you create a Bounded Context for them so they can deploy code without asking the "Core Logistics" team for permission.
* **DDD at Scale: The Context Map.** You maintain the high-level map of how 50+ services interact. You decide which teams have a "Customer-Supplier" relationship and which ones must "Conform" to a central standard.
* **Governance and Consistency:** You aren't checking every PR for Value Objects. Instead, you are setting the "Golden Paths." You provide templates and architectural patterns that make the "DDD way" the easiest way for the juniors and seniors to follow.

---

### 9.5. Summary of Responsibilities

| Level | Goal | Primary DDD Tool |
| --- | --- | --- |
| **Fresher** | Code Quality | Value Objects |
| **Mid-Level** | Logical Grouping | Aggregates & Domain Events |
| **Senior** | System Integrity | Bounded Contexts & ACLs |
| **Staff+** | Org Velocity | Context Maps & Strategic Alignment |

---
# 10. Common Pitfalls & Anti-Patterns

### 10.1. CRUD-driven Design (The "Dumb Entity")

**What it looks like:**
Your entities are just collections of public getters and setters. All the actual business logic lives in a "Service" class that pulls the data out, modifies it, and pushes it back.

```typescript
// PITFALL: The Anemic Entity
const order = orderRepo.findById(id);
order.setStatus('SHIPPED'); // Anyone can do this at any time
order.setUpdatedAt(new Date());
orderRepo.save(order);

```

**Why teams fall into it:**
Most developers come from a background of using ORMs (like Hibernate or Entity Framework) where the default path is to map database rows directly to classes.

**How to fix/avoid it:**
**Encapsulate state.** Make setters private. Use methods that describe a business action.

* **Fix:** `order.ship(trackingNumber)`. Inside this method, the order should check if it's already cancelled or already shipped before changing its status.

---

### 10.2. God Aggregates (The "One-Ring" Model)

**What it looks like:**
A single Aggregate that grows until it includes half the database. For example, an `Order` aggregate that also contains the full `Customer` profile, their entire `OrderHistory`, and all `Product` details.

**Why teams fall into it:**
Developers fear "eventual consistency." They want every piece of related data to be updated in a single ACID transaction to avoid "data sync" issues.

**How to fix/avoid it:**
**Reference by ID only.** An Aggregate should only contain the data it needs to enforce its own rules. If you need a customer's name, don't put the `Customer` entity inside the `Order`. Put a `customerId` string there.

* **The Rule of Thumb:** If you can't describe the aggregate’s boundary in one sentence (e.g., "An Order and its line items"), it’s too big.

---

### 10.3. Chatty Aggregates (The "Micro-Aggregate")

**What it looks like:**
The opposite of the God Aggregate. You’ve broken everything down so small that an `Order` is one aggregate, and an `OrderItem` is a separate aggregate. Now, to calculate the order total, you have to perform 10 database lookups.

**Why teams fall into it:**
Over-correction. Teams hear "make aggregates small" and take it to the extreme, breaking objects that actually have a **strong consistency** requirement.

**How to fix/avoid it:**
Identify the **Invariants.** If "The total sum of OrderItems must match the Order total" is a hard rule that must never be broken, they must be in the same Aggregate.

---

### 10.4. Over-abstraction (The "Generic Generic")

**What it looks like:**
Creating base classes for everything: `BaseEntity<T>`, `BaseRepository<T>`, `BaseService<T>`. You end up with layers of inheritance that make it impossible to see the actual business logic.

**Why teams fall into it:**
The "Don't Repeat Yourself" (DRY) obsession. Developers try to solve technical repetition at the cost of domain clarity.

**How to fix/avoid it:**
**Favor duplication over wrong abstraction.** DDD is about the *Domain*. If the `User` repository needs a different save logic than the `Inventory` repository, let them be different. Don't force them into a generic template that serves neither well.

---

### 10.5. Ignoring Ubiquitous Language (The "Translation Lag")

**What it looks like:**
The business experts call a failed payment a "Rejection," but the code calls it `PaymentStatus.FAILED` or `status_code: 402`.

**Why teams fall into it:**
Developers think their technical terms are "more precise" or they just stick with default framework naming conventions.

**How to fix/avoid it:**
**The "Code-to-Expert" Test.** If you read your code to a Product Manager and they look confused, your naming is wrong.

* **Fix:** Rename `updateStatus(4)` to `markAsRejected()`. The goal is for the code to read like a transcript of a business meeting.

---

### 10.6. Treating DDD as Folder Structure Only

**What it looks like:**
A project has folders named `Domain`, `Infrastructure`, and `Application`, but the `Domain` folder is empty except for data models, and the `Infrastructure` folder contains all the business logic inside SQL queries.

**Why teams fall into it:**
"Cargo Culting." Teams copy the *look* of a DDD project without understanding the *flow* of dependencies.

**How to fix/avoid it:**
Check your **Dependencies.** The `Domain` folder should have **zero** imports from `Infrastructure`. If your "Domain" depends on your "Database" library, you aren't doing DDD; you're just doing CRUD with extra steps.

---

### Summary Checklist for your next Architecture Review:

* [ ] Does my Entity have public setters? (If yes, it's Anemic).
* [ ] Does this transaction touch more than one Aggregate? (If yes, why?).
* [ ] Can a non-technical stakeholder understand my class and method names?
* [ ] Is there any SQL or API logic inside my Domain layer?

---

# 11. End-To-End Example With Payment & Wallet System
### Domain-Driven Design Case Study: Payment & Wallet System in a FinTech Platform

**Project Context**  
This case study draws from a real-world FinTech platform I architected for a Southeast Asian super-app (similar to Grab/GoPay or India's PhonePe), serving 50M+ users across ride-hailing, food delivery, e-commerce, and P2P transfers. The **Payment & Wallet System** was the central financial backbone, handling billions in annual transaction volume.

Key real-world complexities:
- Multi-currency (IDR, SGD, MYR, THB) with real-time FX conversion
- Regulatory compliance (KYC, AML, PSD2-like rules, central bank reporting)
- High concurrency (10k+ TPS during promotions)
- Integration with 20+ banks, card networks, QR schemes, and third-party wallets
- Fraud detection, chargebacks, refunds, and dispute resolution
- Zero-downtime requirements (money systems can't fail)

We applied DDD aggressively because financial domains are life-critical: correctness, auditability, and consistency trump everything else. The system evolved from a monolithic wallet (2018) to a modular, event-driven architecture (2020–2024), with selective microservices.

#### 11.1. Domain Analysis – All Subdomains in Payment & Wallet System

Through extensive EventStorming with product owners, compliance officers, risk analysts, treasury teams, and bank integration engineers, we identified these subdomains:

| Subdomain                  | Type      | Description                                                                 | Competitive Edge? | Complexity Level |
|----------------------------|-----------|-----------------------------------------------------------------------------|-------------------|------------------|
| **Wallet Management**      | Core     | Virtual wallet balance, top-up, hold/release, expiry                        | High             | High            |
| **Transaction Processing** | Core     | Authorization, capture, refund, reversal, settlement                       | High             | Very High       |
| **Payment Instruments**    | Supporting| Cards, bank accounts, linked third-party wallets                            | Medium           | High            |
| **Payouts & Disbursements**| Core     | Merchant settlements, driver earnings, cash-out to banks                    | High             | High            |
| **Fraud & Risk**           | Core     | Real-time fraud scoring, velocity checks, AML monitoring                    | High             | Very High       |
| **Ledger & Accounting**    | Core     | Double-entry bookkeeping, reconciliation, audit trail                      | High             | Very High       |
| **Compliance & KYC**       | Supporting| Identity verification, tiered limits, regulatory reporting                  | Medium           | High            |
| **Fees & Pricing**         | Supporting| Transaction fees, FX margins, promotional waivers                           | Medium           | Medium          |
| **Refunds & Disputes**     | Supporting| Chargeback handling, goodwill refunds, dispute workflows                    | Medium           | Medium          |
| **Notifications**         | Generic  | SMS/Email/Push for transaction alerts                                       | Low              | Low             |

**Reasoning**:  
Core subdomains directly impact money movement and trust — any bug causes financial loss or regulatory fines. Fraud & Risk and Ledger are core because they differentiate the business (low fraud → lower costs → better margins). Compliance is supporting but mandatory — we didn't invest in custom differentiation there.

#### 11.2. Bounded Contexts

We defined bounded contexts to isolate regulatory and consistency needs:

| Bounded Context         | Primary Subdomains Covered                     | Key Models                          | Integration Style                  |
|-------------------------|------------------------------------------------|-------------------------------------|------------------------------------|
| **Wallet**             | Wallet Management                              | Wallet, Balance                     | Events + Sync (balance queries)    |
| **Payments**           | Transaction Processing, Payment Instruments    | Payment, Authorization              | Synchronous (auth) + Events        |
| **Ledger**             | Ledger & Accounting                            | LedgerEntry, Account                | Events only (immutable)            |
| **Fraud**              | Fraud & Risk                                   | RiskScore, FraudRule                | Synchronous (real-time scoring)    |
| **Payouts**            | Payouts & Disbursements                        | PayoutBatch, Settlement             | Events + Batch processing          |
| **Compliance**         | Compliance & KYC                               | CustomerTier, Limit                 | Synchronous REST                   |
| **Refunds**            | Refunds & Disputes                             | RefundRequest, Dispute              | Events                             |

**Key Decisions & Trade-offs**:
- **Separate Ledger Context**: Immutable, event-sourced ledger for perfect auditability. Never update — only append. Trade-off: eventual consistency with Wallet balance, but reconciliation jobs ensure alignment.
- **Fraud as synchronous context**: Must score in <50ms during payment auth. Cannot be async.
- **Wallet balance not stored as single mutable field**: Derived from Ledger (eventually consistent) with hot cache for reads.

#### 11.3. Core Tactical Design – Focus on Payments & Wallet Interaction

**Aggregates**:

1. **Wallet** (Wallet Context)
   - Root: Wallet
   - Contains: Balance (by currency), Holds (temporary reservations)
   - Invariants:
     - Available balance ≥ 0
     - Total holds ≤ available balance
     - Balance = sum of all successful ledger entries

2. **Payment** (Payments Context)
   - Root: Payment
   - Contains: PaymentLine (amount, currency), Instrument, Status
   - Invariants:
     - Amount > 0
     - Once Authorized, cannot modify amount or instrument
     - Refund amount ≤ captured amount

**Domain Events** (selected key ones):
- `WalletTopUpInitiated`
- `WalletTopUpSucceeded`
- `WalletBalanceDebited`
- `WalletBalanceCredited`
- `PaymentAuthorized`
- `PaymentCaptured`
- `PaymentDeclined`
- `PaymentRefunded`
- `FundsHeld`
- `FundsReleased`
- `LedgerEntryPosted` (from Ledger context)

**Value Objects**:
- Money (amount + currency)
- TransactionId
- InstrumentToken (opaque reference to card/bank)

#### High-Level Flow – Wallet Payment for Merchant Transaction

```
User initiates payment (e.g., ride fare 150,000 IDR)
   ↓
[Payments Context] Application Service:
   ├─ Call Fraud Context → RiskScore (synchronous, <50ms)
   ├─ If high risk → Decline
   └─ Else → Create Payment aggregate → authorize()
         ↓
         ├─ If source = Wallet → Publish FundsHoldRequested
         ├─ If source = Card/Bank → Call external PSP
   ↓
[Wallet Context] handles FundsHoldRequested:
   ├─ Load Wallet aggregate
   ├─ Hold funds (reduce available balance temporarily)
   ├─ Publish FundsHeld
   ↓
[Payments Context] receives FundsHeld → Transition Payment to Authorized
   → Publish PaymentAuthorized
   ↓
[Ledger Context] subscribes to PaymentAuthorized → Post debit/credit entries
   → Publish LedgerEntryPosted
   ↓
Later (async capture, e.g., end of ride):
[Payments Context] capture() → Publish PaymentCaptured
   ↓
[Wallet Context] receives PaymentCaptured → Release hold → Debit balance permanently
   → Publish WalletBalanceDebited
   ↓
[Ledger Context] posts final entries
```

**Alternative Path – Failure/Rollback**:
If ride cancelled before capture:
- Publish `PaymentVoided`
- Wallet releases hold → `FundsReleased`
- No permanent debit

#### Reasoning Behind Key Decisions

1. **Hold-then-Debit pattern**  
   Real-world need: Prevent spending the same money twice during pending transactions (e.g., multiple rides).  
   Trade-off: Slightly reduces perceived balance, but prevents overdraft.  
   Alternative (debit immediately + refund on cancel) risks temporary negative balance if refund delays.

2. **Ledger as source of truth, Wallet balance derived**  
   Reasoning: Regulatory audits require immutable ledger. Wallet balance is a read model rebuilt from ledger events.  
   Trade-off: Eventual consistency (seconds delay) — mitigated with Redis cache updated via events + reconciliation cron.  
   Benefit: Perfect reconciliation with banks and zero discrepancy risk.

3. **Synchronous Fraud check**  
   No choice — fraud rules must run in real-time. We used a rules engine (Drools-like) in Fraud context with hot-reload capability.

4. **Payment aggregate lifecycle**  
   Payment goes through states: Initiated → Authorized → Captured → Settled/Refunded.  
   Immutable after Authorized — all reversals via compensating events.  
   Enables perfect audit and dispute resolution.

#### Real-World Complexity & Lessons Learned

- **Regulatory reporting**: Daily batch exports to central bank required exact ledger reconciliation — justified separate Ledger context.
- **Multi-currency FX**: Treasury team needed control over rates → Pricing sub-context added later with real-time rates feed.
- **Disaster recovery**: Ledger event store replicated across regions; wallet read models rebuilt from events.
- **Peak events (e.g., 11.11 sales)**: Hold requests spiked to 20k/sec → introduced rate limiting and circuit breakers.
- **Third-party PSP failures**: Idempotency keys on all external calls; retry with exponential backoff.
- **Team scaling**: Grew to 15 teams. Bounded contexts mapped 1:1 to teams → independent deployments without coordination hell.

#### Summary – Why This DDD Approach Succeeded

By explicitly separating concerns (Wallet for UX, Payments for orchestration, Ledger for truth, Fraud for risk), we achieved:
- Regulatory compliance without crippling agility
- Sub-second transaction times at massive scale
- Perfect audit trails (survived multiple central bank audits)
- Team autonomy — Fraud team iterated rules 20x/day without affecting payments

---

# 12. End-To-End Example With E-commerce System
### Domain-Driven Design Case Study: Large-Scale E-Commerce Platform

**Project Context**  
This case study is based on a real-world e-commerce platform I architected for a global fashion retailer (similar to Zalando or ASOS) handling 50M+ SKUs, peak traffic of 100k+ concurrent users, multi-channel sales (web, mobile app, marketplace partners), and operations across 20+ countries. The system had to support rapid fashion drops, flash sales, personalized recommendations, complex pricing/promotions, returns, and integration with warehouses, payment providers, and tax engines.

The platform was built as a **modular monolith initially** (2019–2021), then selectively extracted into microservices (2022 onward) as teams and traffic grew. We applied strategic and tactical DDD rigorously because the domain is notoriously complex: business rules change weekly (promotions), regulations differ by country, and inventory/pricing consistency is critical during high-load events.

#### 12.1. Domain Analysis – All Subdomains in E-Commerce

E-commerce is rarely a single bounded domain. Through multiple **EventStorming** sessions with product managers, merchants, warehouse operators, customer service reps, and marketing teams, we identified the following subdomains:

| Subdomain              | Type      | Description                                                                 | Competitive Edge? | Complexity Level |
|-------------------------|-----------|-----------------------------------------------------------------------------|-------------------|------------------|
| **Catalog Management** | Supporting| Managing product information (titles, descriptions, images, attributes like size/color) | Low              | Medium          |
| **Pricing & Promotions**| Core     | Dynamic pricing, discounts, vouchers, bundle deals, flash sales             | High             | Very High       |
| **Inventory Management**| Core     | Stock levels across warehouses, reservations, backorders, allocations      | High             | High            |
| **Order Processing**   | Core     | Cart → Checkout → Order placement, payment, fraud checks                   | High             | Very High       |
| **Payment Processing** | Generic  | Integration with PSPs (Stripe, Adyen, PayPal), refunds, captures           | Low              | Medium          |
| **Shipping & Fulfillment** | Supporting| Carrier integration, label generation, warehouse picking instructions     | Medium           | High            |
| **Returns & Refunds**  | Supporting| Return requests, inspections, restocking, refund issuance                  | Medium           | Medium          |
| **Customer Management**| Supporting| Profiles, addresses, preferences, loyalty points                           | Medium           | Medium          |
| **Recommendation**     | Supporting| Personalized product suggestions                                           | High             | High (ML heavy) |
| **Search**             | Generic  | Full-text product search (often outsourced to Elasticsearch/Algolia)       | Low              | Low             |
| **Marketing & Campaigns** | Supporting| Email/SMS campaigns, abandoned cart recovery                               | Medium           | Medium          |

**Reasoning**:  
We classified using the Core Domain Chart: What gives the business a competitive advantage and requires heavy investment in custom logic? Pricing, Inventory, and Order Processing emerged as **core** because mistakes here directly lose money (overselling, wrong discounts) and drive differentiation (dynamic pricing beats competitors).

Generic subdomains were either bought (Search) or heavily delegated to third parties (Payment).

#### 12.2. Bounded Contexts

We defined separate Bounded Contexts to isolate complexity and enable team autonomy:

| Bounded Context          | Primary Subdomains Covered                          | Key Models                          | Integration Style                  |
|--------------------------|-----------------------------------------------------|-------------------------------------|------------------------------------|
| **Catalog**             | Catalog Management                                 | Product, SKU, Attribute             | Synchronous REST + Events          |
| **Pricing**             | Pricing & Promotions                               | PriceList, Promotion, Voucher       | Synchronous (high consistency)     |
| **Inventory**           | Inventory Management                               | StockItem, Reservation              | Events (eventual consistency)      |
| **Cart & Ordering**     | Order Processing                                   | Cart, Order (Aggregate)             | Synchronous + Events               |
| **Payment**             | Payment Processing                                 | PaymentIntent, Refund               | Callbacks + Events                 |
| **Fulfillment**         | Shipping & Fulfillment                             | Shipment, PickingTask               | Events                             |
| **Returns**             | Returns & Refunds                                  | ReturnRequest, RestockTask          | Events                             |
| **Customer**            | Customer Management                                | Customer, Address, Loyalty          | Synchronous REST                   |

**Key Decisions & Trade-offs**:
- **Separate Pricing Context**: Pricing rules are extremely volatile (Black Friday campaigns change hourly). Keeping it separate allowed the pricing team to deploy 10x/day without affecting checkout flow.
- **Inventory as separate context**: Overselling is catastrophic. We used eventual consistency via events because real-time distributed transactions across warehouses would kill performance.
- **No single "Product" model everywhere**: Catalog has rich descriptive Product; Inventory has StockItem by SKU/location; Ordering has OrderLine with snapshot of price/description. This avoids coupling but requires ACLs for translation.

#### 12.3. Core Tactical Design – Focus on Cart & Ordering Context

This is the most complex core domain.

**Aggregates**:
1. **Cart** (short-lived, anonymous or customer-linked)
   - Root: Cart
   - Contains: CartLine (SKU, quantity, unitPrice snapshot)
   - Invariants:
     - Total lines ≤ 50 (UX constraint)
     - No duplicate SKUs
     - Quantity > 0

2. **Order** (long-lived, immutable after placement)
   - Root: Order
   - Contains: OrderLine (immutable snapshots of SKU, description, unitPrice, appliedPromotions)
   - Associated: ShippingAddress, BillingInfo, PaymentSummary
   - Invariants:
     - Total amount matches sum of lines + shipping – discounts
     - Inventory reserved before transition to Placed
     - Order cannot be modified after Placed

**Domain Events** (published when aggregate state changes):
- `CartUpdated`
- `CartCheckedOut`
- `OrderPlaced`
- `InventoryReservationRequested`
- `InventoryReservationConfirmed` / `InventoryReservationFailed`
- `OrderConfirmed` (after payment + reservation success)
- `OrderRejected`
- `OrderShipped` (from Fulfillment context)

**Value Objects**:
- Money (amount + currency)
- SKU
- PromotionCode
- Quantity

**High-Level Flow – Checkout to Order Confirmation** (Text Diagram)

```
User adds to Cart
   ↓
[Cart Aggregate] → Apply promotions (calls Pricing Context sync) → Update totals
   ↓ (User clicks Checkout)
CartCheckedOut event published
   ↓
[Ordering Service] creates draft Order from Cart snapshot
   ↓
Order.place() → 
   ├─ Validate pricing again (re-query Pricing Context – prevent race)
   ├─ Request inventory reservation (publish InventoryReservationRequested)
   ├─ Initiate payment (call Payment Context)
   ↓
Async handling:
   ├─ Inventory Context processes reservation → publishes Confirmed/Failed
   ├─ Payment Context callback → PaymentSucceeded/Failed event
   ↓
[Order Aggregate] receives both events → 
   ├─ If both success → transition to Confirmed, publish OrderConfirmed
   ├─ Else → transition to Rejected, release reservation, refund if needed
   ↓
Fulfillment Context subscribes to OrderConfirmed → creates Shipment
```

**Reasoning Behind Key Decisions**:

1. **Snapshot pricing in OrderLine**  
   Trade-off: We store unitPrice and applied discounts at checkout time.  
   Pro: Order total never changes even if prices drop later (customer trust).  
   Con: Reporting needs to reconcile with current prices.  
   Real-world: Essential for legal compliance (many countries require "price at purchase").

2. **Separate reservation step before payment**  
   Many systems reserve inventory only after payment → risk of overselling during flash sales.  
   We reserve first (soft reservation for 10–15 min), then payment.  
   Trade-off: User might lose reservation if payment takes too long → mitigated with reservation timeout + re-check at payment success.

3. **Eventual consistency for inventory**  
   Could have used distributed saga with 2PC, but latency and failure modes unacceptable at scale.  
   Eventual consistency + compensation (cancel order if reservation fails) proved reliable.

4. **Order as immutable after placement**  
   All changes (cancellations, returns) create new aggregates in Returns context.  
   Pro: Audit trail perfect, simplifies reporting.  
   Con: More events and storage – acceptable with event sourcing in this context.

#### Real-World Complexity & Lessons Learned

- **Flash Sales**: During drops, inventory events spiked to 10k/sec. We had to implement backpressure and idempotency everywhere.
- **Multi-warehouse allocation**: Inventory context evolved to use a sophisticated allocation engine (closest warehouse, cost, carrier rules) – justified as core.
- **Tax calculation**: Initially in Ordering, later extracted to separate Tax Context when VAT rules diverged heavily by country.
- **Team scaling**: Started with 3 teams, grew to 12. Bounded contexts aligned perfectly with Team Topologies "stream-aligned teams".
- **Performance**: Synchronous calls to Pricing during checkout became bottleneck → introduced local caching with invalidation via events.

---

# 13. DDD Checklists

### Practical DDD Checklists for Production Systems

These checklists are distilled from 20+ years of applying DDD in large-scale systems (fintech, e-commerce, insurance, logistics). They are designed to be **actionable, concise, and used in real projects** — print them, paste in Confluence, or add to PR templates.

#### 1. Strategic Design Checklist  
*(Use during discovery, EventStorming, or architecture workshops)*

- [ ] **Ubiquitous Language established**  
  – Glossary of terms agreed with domain experts  
  – No synonyms for same concept (e.g., “Order” vs “Purchase”)  
  – Language used consistently in meetings, tickets, and code

- [ ] **Subdomains identified and classified**  
  – Core (competitive advantage)  
  – Supporting  
  – Generic  
  – Prioritized investment accordingly

- [ ] **Bounded Contexts explicitly defined**  
  – Clear responsibility and ownership per context  
  – Models do NOT leak across contexts  
  – Context Map documented (living diagram)

- [ ] **Integration style decided per boundary**  
  – Synchronous (REST/gRPC) only when strong consistency needed  
  – Asynchronous events preferred for loose coupling  
  – Anti-Corruption Layer (ACL) in place for legacy/third-party integration

- [ ] **Core domain has dedicated team/modelling time**  
  – Domain experts embedded or regularly available  
  – No “one model to rule them all” temptation

- [ ] **Context Map reviewed for team alignment**  
  – Aligns with Team Topologies (stream-aligned teams own contexts)

#### 2. Tactical Implementation Checklist  
*(Use when designing aggregates, entities, services)*

- [ ] **Aggregates correctly sized**  
  – Small (typically 1–5 entities/value objects)  
  – Enforces real business invariants transactionally  
  – Not a “God object”

- [ ] **Invariants protected inside aggregate**  
  – Consistency rules enforced in methods (e.g., `order.place()` checks inventory, pricing)  
  – No partial updates allowed outside aggregate

- [ ] **Domain Events used for cross-aggregate/context coordination**  
  – Published on meaningful business changes  
  – Idempotent handlers  
  – Event schema versioned

- [ ] **Entities & Value Objects distinguished**  
  – Identity matters → Entity  
  – Identity irrelevant → Value Object (e.g., Money, Address)

- [ ] **Application Services orchestrate, Domain Model decides**  
  – Services are thin: load aggregate → call method → persist  
  – No business logic in application services or controllers

- [ ] **Repositories interface per aggregate**  
  – One repo per aggregate root  
  – Collection-oriented naming (e.g., `orderRepository.findByCustomerId()`)

- [ ] **Factories used for complex creation**  
  – When aggregate construction has invariants

- [ ] **Domain exceptions express intent**  
  – Not generic `RuntimeException` — use `InsufficientFundsException`, `OrderAlreadyPlacedException`

#### 3. Code Review Checklist  
*(Add to PR template for DDD-heavy modules)*

- [ ] **Ubiquitous Language reflected in code**  
  – Class/method/variable names match spoken domain language  
  – No CRUD verbs in domain layer (avoid `updateStatus`, prefer `order.ship()`)

- [ ] **Anemic domain model avoided**  
  – Behavior lives with data (rich methods on aggregates/entities)  
  – No “manager” or “service” classes holding logic

- [ ] **Aggregate boundaries respected**  
  – No direct references to entities from other aggregates  
  – Cross-aggregate actions via domain events or application service orchestration

- [ ] **No logic in DTOs or controllers**  
  – Controllers map to application services only  
  – DTOs are pure data carriers

- [ ] **Immutability applied where possible**  
  – Value objects immutable  
  – Aggregates return new state or use protective copying

- [ ] **Domain events properly named and meaningful**  
  – Past tense (e.g., `OrderPlaced`, `PaymentCaptured`)  
  – Carry necessary data for downstream handlers

- [ ] **Tests focus on behavior, not data**  
  – Unit tests on aggregates verify invariants and events  
  – No over-testing of getters/setters

#### 4. Architecture Review Checklist  
*(Use quarterly or before major releases)*

- [ ] **Bounded Contexts still valid**  
  – No creeping coupling between contexts  
  – Context Map up to date

- [ ] **Core domain receiving most investment**  
  – Metrics: velocity, defect rate, expert involvement

- [ ] **Consistency boundaries correct**  
  – Strong consistency only where business requires it  
  – Eventual consistency accepted and compensated where needed

- [ ] **Event schema evolution strategy in place**  
  – Backward compatibility  
  – Consumer-driven contracts or schema registry

- [ ] **Observability reflects domain**  
  – Traces/metrics/events use domain terms (e.g., `OrderPlaced` counter)  
  – Logs include business identifiers (orderId, customerId)

- [ ] **Performance hotspots aligned with boundaries**  
  – Hot paths optimized within context (caching, read models)  
  – No distributed transactions unless unavoidable

- [ ] **Team ownership clear**  
  – Each bounded context has single responsible team  
  – No shared domain models across teams

- [ ] **Technical debt tracked per context**  
  – Legacy integration points flagged  
  – Plans to extract or replace generic subdomains
