# Documentation: Tactical Domain-Driven Design (DDD)

## Introduction

Domain-Driven Design (DDD) is a software design approach that focuses on accurately and effectively modeling the business domain. DDD is divided into two main parts:

- **Strategic DDD** — focuses on analyzing and dividing a large domain into bounded contexts, defining a ubiquitous language, and context mapping
- **Tactical DDD** — provides concrete design patterns and building blocks for implementing the domain model in code

Tactical DDD offers specific tools (building blocks) to create a rich, expressive, and maintainable domain model. It helps developers avoid common pitfalls such as the **anemic domain model** (where domain objects are just data containers without behavior) by strongly emphasizing the combination of data and behavior within the same classes.

This document is written based on many years of practical experience applying DDD in real-world projects — from financial systems and insurance platforms to large-scale e-commerce and logistics systems.

## Core Building Blocks of Tactical DDD

### 1. Entities

- **Definition**: Objects that have a unique **identity** and can change state over time. Identity is independent of attribute values.
- **Characteristics**:
  - Mutable (state can change)
  - Compared by identity (usually an ID — UUID, auto-increment, etc.)
  - Usually represent core business concepts: Customer, Order, Invoice, Booking…
- **When to use**: When you need to track changes of an object throughout its lifecycle
- **Example** (TypeScript):
  > Ví dụ thực tế: Trong hệ thống Order Management, một Order (Đơn hàng) là Entity vì mỗi đơn hàng có ID riêng (ví dụ: OrderID=123). Bạn có thể cập nhật trạng thái từ "Pending" sang "Shipped", nhưng identity vẫn giữ nguyên.

```typescript
export abstract class Entity {
  // Common entity logic (if any)
}

type OrderStatus = "Pending" | "Shipped";

export class Order extends Entity {
  private readonly id: string; // Guid → string (UUID)
  private status: OrderStatus;
  private readonly items: ReadonlyArray<OrderItem>;

  constructor(id: string, items: OrderItem[]) {
    super();
    this.id = id;
    this.items = Object.freeze([...items]); // enforce immutability
    this.status = "Pending";
  }

  get Id(): string {
    return this.id;
  }

  get Status(): OrderStatus {
    return this.status;
  }

  get Items(): ReadonlyArray<OrderItem> {
    return this.items;
  }

  ship(): void {
    if (this.status !== "Pending") {
      throw new Error("Only pending orders can be shipped.");
    }

    this.status = "Shipped";
  }
}
```

### 2. Value Objects

- **Definition**: Objects without conceptual identity — defined completely by their attributes. They are **immutable**.
- **Characteristics**:
  - So sánh bằng giá trị (equals based on attributes).
  - Không có lifecycle riêng; chỉ tồn tại như phần của Entity.
  - Giúp tránh primitive obsession (sử dụng quá nhiều string/int thay vì object ý nghĩa).
  - No identity field
  - Usually small, descriptive concepts: Money, Address, DateRange, Email, Percentage…
- **When to use**: To eliminate primitive obsession and make domain concepts more expressive
- **Example** (TypeScript):
  > Ví dụ thực tế: Trong Order, Address (Địa chỉ giao hàng) là Value Object vì hai địa chỉ giống nhau hoàn toàn là tương đương, không cần ID riêng.

```typescript
export abstract class ValueObject {
  protected abstract getEqualityComponents(): unknown[];

  equals(other: ValueObject): boolean {
    if (!other || other.constructor !== this.constructor) {
      return false;
    }

    const thisComponents = this.getEqualityComponents();
    const otherComponents = other.getEqualityComponents();

    if (thisComponents.length !== otherComponents.length) {
      return false;
    }

    return thisComponents.every(
      (value, index) => value === otherComponents[index],
    );
  }
}

export class Money extends ValueObject {
  public readonly amount: number; // decimal → number
  public readonly currency: string;

  constructor(amount: number, currency: string) {
    super();

    if (amount < 0) {
      throw new Error("Amount cannot be negative.");
    }

    if (!currency) {
      throw new Error("Currency is required.");
    }

    this.amount = amount;
    this.currency = currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currencies must match.");
    }

    return new Money(this.amount + other.amount, this.currency);
  }

  protected getEqualityComponents(): unknown[] {
    return [this.amount, this.currency];
  }
}
```

### 3. Aggregates

- **Definition**: Aggregate là nhóm các Entity và Value Object liên kết chặt chẽ, được quản lý qua một Aggregate Root (Entity chính). Aggregate đảm bảo tính nhất quán (consistency) và là đơn vị nhỏ nhất để persist (lưu trữ) hoặc truy vấn.
- **Key Rules**:
  - All external access must go through the Aggregate Root
  - Transaction boundary = Aggregate boundary
  - Sử dụng để tránh các vấn đề concurrency hoặc data inconsistency.
  - Maintain invariants within the Aggregate | Đảm bảo invariants (quy tắc bất biến) trong toàn Aggregate
  - Reference other Aggregates only by ID (never direct object reference)

- **Apply**:
  - Áp dụng vào dự án: Xác định Aggregate bằng cách vẽ diagram: Nhóm các object có quan hệ "part-of" (phần của). Trong database, một Aggregate thường map với một transaction. Tránh Aggregate quá lớn để không làm chậm performance.
- **Classic example**: Order (Aggregate Root) + OrderItems + ShippingAddress
  > Ví dụ thực tế: Order là Aggregate Root, chứa các OrderLine (Entity con, như sản phẩm trong đơn hàng) và ShippingAddress (Value Object). Bạn không thể thêm OrderLine trực tiếp; phải qua Order.

```typescript
class OrderId {
  constructor(public readonly value: string) {}
}

// 2. Entity phụ bên trong Aggregate
class OrderItem {
  constructor(
    public readonly productId: string,
    public readonly price: Money, // Sử dụng Value Object Money từ ví dụ trước
    public readonly quantity: number,
  ) {}

  get total(): Money {
    return Money.create(this.price.amount * this.quantity, this.price.currency);
  }
}

// 3. AGGREGATE ROOT
export class Order {
  private _items: OrderItem[] = [];
  private _totalPrice: Money;

  private constructor(
    public readonly id: OrderId,
    private customerId: string,
    currency: string,
  ) {
    this._totalPrice = Money.create(0, currency);
  }

  // Factory method để khởi tạo Order
  public static create(
    id: string,
    customerId: string,
    currency: string = "VND",
  ): Order {
    return new Order(new OrderId(id), customerId, currency);
  }

  // Phương thức duy nhất để chỉnh sửa dữ liệu bên trong (Invariants)
  public addItem(productId: string, price: Money, quantity: number): void {
    if (quantity <= 0) throw new Error("Số lượng phải lớn hơn 0");
    if (price.currency !== this._totalPrice.currency) {
      throw new Error("Tiền tệ không đồng nhất với đơn hàng");
    }

    const newItem = new OrderItem(productId, price, quantity);
    this._items.push(newItem);

    // Cập nhật lại tổng tiền (Logic nghiệp vụ được đóng gói tại Root)
    this._totalPrice = this._totalPrice.add(newItem.total);
  }

  // Chỉ trả về bản sao hoặc Read-only để bảo vệ tính đóng gói
  get items(): readonly OrderItem[] {
    return [...this._items];
  }

  get totalPrice(): Money {
    return this._totalPrice;
  }
}
```

```C#
public class Order : AggregateRoot
{
    private List<OrderLine> _lines = new List<OrderLine>();
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    public void AddLine(ProductId productId, int quantity)
    {
        // Kiểm tra invariants, ví dụ: quantity > 0
        _lines.Add(new OrderLine(productId, quantity));
    }
}
```

### 4. Repositories

- Repository là lớp trừu tượng hóa việc lưu trữ và truy xuất Aggregates, như một collection in-memory. Nó che giấu persistence layer (DB, file) và chỉ làm việc với domain objects.

- **Purpose**: Provide collection-like interface for accessing and persisting **Aggregate Roots** (never individual Entities inside an Aggregate)
- **Important principles**:
  - Work only with Aggregate Roots
  - Hide all persistence details
  - Usually one Repository per Aggregate type
  - Do **not** expose IQueryable or complex query methods — keep domain logic in the domain

- **Apply**:
  - Áp dụng vào dự án: Đặt Repository ở domain layer, implement ở infrastructure. Điều này giúp tách biệt domain khỏi DB, dễ test và thay đổi storage.

- **Example** (TypeScript):

```typescript
export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

### 5. Domain Services

- Khi logic không thuộc về Entity hay Value Object (ví dụ: phối hợp nhiều Aggregates), dùng Domain Service. Chúng stateless và chứa business rules.

- **Definition**: Stateless operations that do not naturally belong to any single Entity or Value Object
- **When to use**:
  - Business logic spans multiple Aggregates
  - Complex calculations or transformations
  - Coordinating domain rules across entities
- **Examples**: PricingEngine, PaymentAuthorizationService, LoyaltyPointsCalculator

- **Apply**:
  - Chỉ dùng khi cần; ưu tiên đặt logic vào Entity trước. Tránh lạm dụng thành "anemic domain model" (model chỉ có data, không có behavior).

### 6. Domain Events

- Domain Event ghi nhận các sự kiện quan trọng trong domain (ví dụ: OrderShipped), dùng để decoupling modules hoặc trigger side effects.

- **Definition**: Something important that happened in the domain that other parts of the system may care about
- **Common pattern**:
  - OrderPlaced
  - OrderShipped
  - PaymentReceived
  - CustomerReachedGoldTier
- **Purpose**: Enable loose coupling between different parts of the system (especially across bounded contexts)

- **Apply**:
  - Áp dụng: Sử dụng event bus như MediatR (C#) hoặc RabbitMQ để tích hợp.

### 7. Factories

- Factory tạo ra các object phức tạp, đảm bảo invariants từ đầu.
- **Purpose**: Encapsulate complex object creation logic, especially for Aggregates
- **Benefits**:
  - Đảm bảo các bất biến được thỏa mãn ngay từ thời điểm tạo ra sản phẩm.
  - Hides construction complexity
  - Improves readability

- **Apply**:
  - Dùng cho constructors phức tạp, tránh code lộn xộn trong Entity.

### 8. Modules / Packages

- Organize code by bounded context and by tactical pattern
- Common structure example:

```
src/
  Ordering/
    Domain/
      Aggregates/
      Entities/
      ValueObjects/
      DomainEvents/
      DomainServices/
      Repositories/
    Application/
    Infrastructure/
```

## Practical Application Summary

1. Start with **Ubiquitous Language** (from strategic DDD)
2. Identify **Aggregates** → define consistency boundaries
3. Model important behavior inside **Entities** and **Aggregates**
4. Use **Value Objects** aggressively to enrich the model
5. Create **Domain Services** only when logic doesn’t belong naturally to an Aggregate
6. Protect invariants through **Aggregate Roots**
7. Use **Repositories** to persist/load whole Aggregates
8. Publish **Domain Events** for important state changes
9. Keep infrastructure (database, external services) completely separated

## Quick Reference Table

| Concept        | Has Identity? | Mutable? | Lifecycle Tracking? | Typical Size    | Access Through    |
| -------------- | ------------- | -------- | ------------------- | --------------- | ----------------- |
| Entity         | Yes           | Yes      | Yes                 | Medium–Large    | Directly / via AR |
| Value Object   | No            | No       | No                  | Small           | By value          |
| Aggregate Root | Yes           | Yes      | Yes                 | Large (cluster) | Only entry point  |
| Domain Service | No            | No       | No                  | Stateless       | Directly          |

If you would like a more detailed explanation of any particular pattern, real-world code examples in a specific language (C#, Java, TypeScript, Python, etc.), or how to combine Tactical DDD with modern architectures (Clean Architecture, Vertical Slice, Modular Monolith, Microservices…), feel free to ask!
