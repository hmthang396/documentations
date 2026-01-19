Here is a clear and professional English translation of the provided Vietnamese text on **Subdomains** in Domain-Driven Design (Strategic DDD):

# Domain-Driven Design - Subdomain

A **Subdomain** is a way to divide a large domain (the entire business field/problem space) into smaller, meaningful parts that each represent distinct business capabilities.

The domain (problem domain) of a company is usually very broad and complex. For example, an e-commerce company does not just involve “selling products”; it also includes product management, payments, shipping & logistics, customer support, marketing, inventory management, and much more.

When analyzing the domain, we break it down into **subdomains** in order to:

- Better understand the overall structure  
- Identify which parts are **core** (differentiating), which are **supporting**, and which are **generic**  
- Make better decisions about team organization, **Bounded Contexts**, and integration strategies  

## Subdomain Classification (according to Eric Evans)

Eric Evans classifies subdomains into three main types:

Here are some illustrative diagrams showing the three types of subdomains and their strategic importance:◊

| Subdomain Type     | Description                                                                 | Business Characteristics                          | Example in E-commerce                              | Investment Strategy                          |
|--------------------|-----------------------------------------------------------------------------|---------------------------------------------------|----------------------------------------------------|----------------------------------------------|
| **Core Domain**    | The part that provides the primary competitive advantage and unique value | Most important, where the company must excel      | Order placement process, personalized promotions, product recommendations | Heavy investment, deep DDD application       |
| **Supporting Domain** | Supports the core domain but is not a source of differentiation          | Important but not differentiating                 | Inventory management, shipping cost calculation, product reviews | Moderate investment, can use off-the-shelf solutions |
| **Generic Domain** | Common functionality that most companies implement in a similar way       | No competitive advantage                          | User management (registration/login), payments (Stripe/PayPal), email sending | Use third-party solutions or buy off-the-shelf |

## Relationship between Subdomain and Bounded Context

- A subdomain **often** (but not always) corresponds to one **Bounded Context** (not necessarily a strict 1:1 mapping).  

Possible relationships include:

- 1 subdomain → 1 Bounded Context (most common case)  
- 1 subdomain → multiple Bounded Contexts (when the subdomain is very large/complex)  
- Multiple subdomains → 1 Bounded Context (rare and usually not recommended)  

Here are some common visual representations of how subdomains relate to bounded contexts:

### Example in E-commerce

| Subdomain              | Type          | Corresponding Bounded Context       | Notes                                              |
|------------------------|---------------|-------------------------------------|----------------------------------------------------|
| Order Management       | Core          | Order Context                       | Order placement, cancellation, status tracking     |
| Product Catalog        | Core / Supporting | Catalog Context                  | Product listings, attributes, categories           |
| Inventory Management   | Supporting    | Inventory Context                   | Stock checking, inbound/outbound operations        |
| Payment Processing     | Generic       | Payment Context (or use Stripe)     | Usually integrated with third-party providers      |
| Customer Account       | Supporting    | Identity / Account Context          | Registration, login, personal information          |
| Marketing & Promotion  | Core          | Promotion Context                   | Promotions, vouchers, campaigns                    |
| Shipping & Logistics   | Supporting    | Shipping Context                    | Shipping fee calculation, carrier selection        |

## Why Distinguish Subdomains?

1. **Prioritization & Resource Allocation**  
   Focus people, time, and technology on the **core domain** to create real competitive advantage. Use off-the-shelf or third-party solutions for generic domains to save time and cost.

2. **Guides Bounded Context Design**  
   Each subdomain is usually a strong candidate for its own Bounded Context.

3. **Helps Define Ubiquitous Language**  
   Different subdomains may use the same term with different meanings.  
   Example: “Order” in Order Management means a customer purchase → while in Shipping it may mean a transportation shipment.

4. **Supports Team Organization**  
   Core domain teams usually need higher expertise.  
   Generic domain teams can often use vendors, outsource, or buy ready solutions.

## Practical Ways to Discover Subdomains

- **Large-scale Domain Discovery Workshops** (e.g., Big Picture Event Storming):  
  - Draw the complete end-to-end business timeline  
  - Group domain events and activities into larger clusters → each cluster is a potential subdomain  

- Ask domain experts key questions:  
  - “Which part makes our company different from competitors?”  
  - “If we didn’t do this part, could the company still survive?”  
  - “Which parts do most companies in our industry do in almost the same way?”  

- Look at **Business Capabilities**:  
  Each major business capability is usually a good candidate for a subdomain.

These concepts form the foundation of **Strategic Domain-Driven Design**. Correctly identifying subdomains helps teams make better architectural, organizational, and investment decisions.
