### Domain Model là gì? (Theo Eric Evans và DDD chuẩn)

Theo sách **Domain-Driven Design** của Eric Evans (2003), **Domain Model** là:

- Một **mô hình khái niệm** (conceptual model) về domain (lĩnh vực kinh doanh) mà phần mềm đang giải quyết.
- Nó bao gồm các abstraction (khái niệm trừu tượng) quan trọng nhất của domain, được biểu diễn dưới dạng **code** (thường là object model trong OOP).
- Mục tiêu: Làm cho code **phản ánh chính xác** cách business hoạt động, các quy tắc kinh doanh (business rules), invariants, và hành vi (behavior), chứ không chỉ là data storage.
- Domain Model **không phải** là database schema, không phải UI model, không phải DTO – nó là phần **sạch nhất**, tập trung vào **business logic**.

**Key quote từ Eric Evans**:

> "The domain model is not a diagram or a set of diagrams. It is a system of abstractions that describes selected aspects of the domain and can be used to solve problems related to that domain."

Nói cách khác: Domain Model = **Code thể hiện Ubiquitous Language** + **Behavior** + **Rules** của domain.

### Domain Model trong Strategic DDD vs Tactical DDD

| Khía cạnh              | Strategic DDD                                                                               | Tactical DDD                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Mức độ**             | Bức tranh lớn (Big Picture)                                                                 | Chi tiết bên trong một Bounded Context                                       |
| **Domain Model ở đây** | Là mô hình khái niệm chung, được chia sẻ qua Ubiquitous Language giữa team & domain experts | Là implementation cụ thể: các class, method, behavior thực sự trong code     |
| **Focus**              | Bounded Contexts, Subdomains, Context Mapping                                               | Entity, Value Object, Aggregate, Repository, Domain Service, Domain Event... |
| **Mục đích**           | Định nghĩa "domain là gì", giới hạn scope                                                   | Xây dựng code giàu ý nghĩa (rich domain model)                               |

→ **Tactical DDD chính là nơi Domain Model được "hiện thực hóa" thành code**.

### Đặc điểm của một Domain Model tốt (Rich Domain Model)

1. **Rich in behavior** (không phải Anemic Domain Model):
   - Không chỉ getter/setter + data.
   - Có method thể hiện hành vi kinh doanh: `order.Place()`, `order.Cancel()`, `order.ApplyDiscount(discount)`...

2. **Encapsulates business rules & invariants**:
   - Ví dụ: Trong Order, không cho phép thêm OrderLine nếu Order đã Shipped → invariant được bảo vệ trong Aggregate Root.

3. **Sử dụng Ubiquitous Language**:
   - Tên class, method, variable khớp với ngôn ngữ mà domain expert dùng hàng ngày.

4. **Immutable khi có thể** (đặc biệt Value Objects).

5. **Tập trung vào consistency trong Aggregate**:
   - Transaction boundary là Aggregate, không phải toàn bộ database.

### Ví dụ thực tế: Domain Model trong hệ thống Order (Ordering Bounded Context)

Giả sử chúng ta đang làm module Ordering trong một hệ thống bán hàng online.

**Anemic Domain Model** (xấu – hay gặp ở dự án CRUD truyền thống):

```typescript
// Trạng thái đơn hàng (Value Object / Enum)
export enum OrderStatus {
  Pending = "Pending",
  Paid = "Paid",
  Shipped = "Shipped",
  Cancelled = "Cancelled",
}

// Địa chỉ giao hàng (Value Object)
export class Address {
  constructor(
    public readonly street: string,
    public readonly city: string,
    public readonly zipCode: string,
  ) {
    if (!street || !city) throw new Error("Địa chỉ không hợp lệ");
    Object.freeze(this);
  }
}

class OrderItem {
  constructor(
    public readonly id: string,
    public readonly productId: string,
    public readonly unitPrice: Money,
    public readonly quantity: number,
  ) {}

  get total(): Money {
    return Money.create(
      this.unitPrice.amount * this.quantity,
      this.unitPrice.currency,
    );
  }
}
```

→ Chỉ là data container, logic nằm ở service/controller → khó bảo trì khi business phức tạp.

**Rich Domain Model** (tốt – Tactical DDD):

```typescript
export class Order {
  private _items: OrderItem[] = [];
  private _status: OrderStatus;
  private _shippingAddress: Address;

  private constructor(
    public readonly id: string,
    private customerId: string,
    shippingAddress: Address,
  ) {
    this._status = OrderStatus.Pending;
    this._shippingAddress = shippingAddress;
  }

  public static create(
    id: string,
    customerId: string,
    address: Address,
  ): Order {
    return new Order(id, customerId, address);
  }

  // Nghiệp vụ: Thêm sản phẩm
  public addProduct(productId: string, price: Money, quantity: number): void {
    if (this._status !== OrderStatus.Pending) {
      throw new Error("Không thể thêm sản phẩm vào đơn hàng đã thanh toán/hủy");
    }
    if (quantity <= 0) throw new Error("Quantity must be positive");

    this._items.push(new OrderItem(productId, price, quantity));
  }

  // Nghiệp vụ: Thanh toán đơn hàng
  public pay(): void {
    if (this._items.length === 0) throw new Error("Đơn hàng trống");
    this._status = OrderStatus.Paid;

    // Tại đây có thể bắn ra một Domain Event: OrderPaidEvent
  }

  get totalAmount(): Money {
    return this._items.reduce(
      (acc, item) => acc.add(item.total),
      Money.create(0, "VND"),
    );
  }
}
```

→ Logic kinh doanh nằm **trong** Entity/Aggregate → dễ test, dễ hiểu, dễ mở rộng.

### Lời khuyên áp dụng thực tế

1. Bắt đầu bằng **Event Storming** hoặc **Domain Storytelling** → khám phá Ubiquitous Language → vẽ ra các Entity, Aggregate tiềm năng.
2. Xác định **Aggregate Root** đầu tiên (thường là khái niệm chính: Order, Booking, Account...).
3. Đẩy **hành vi** vào Entity/Value Object càng nhiều càng tốt.
4. Tránh **anemic model** bằng cách hỏi: "Logic này thuộc về ai?" → Nếu thuộc về Order → đẩy vào Order.
5. Dùng **Specification**, **Factory**, **Domain Service** khi logic không thuộc về một Entity duy nhất.
6. Test invariants: Viết unit test cho mỗi rule kinh doanh.

### Quy trình thiết kế Domain Model

#### Bước 1: Hiểu sâu Domain (Domain Discovery) – Trước khi code

- **Mục tiêu**: Xây dựng **Ubiquitous Language** chung giữa dev team và domain experts (chuyên gia nghiệp vụ, PO, business analyst).
- **Công cụ phổ biến 2025-2026**:
  - **Event Storming** (Big Picture & Process Level): Workshop 4-8 giờ.
  - **Domain Storytelling** hoặc **Example Mapping**.
- **Làm gì**:
  1. Thu thập **Domain Events** (sự kiện xảy ra): "OrderPlaced", "PaymentReceived", "OrderShipped", "OrderCancelled", "ItemOutOfStock"...
  2. Xác định **Commands** (hành động): "PlaceOrder", "PayOrder", "ShipOrder"...
  3. Vẽ timeline → nhóm thành **Bounded Contexts** (ví dụ: Ordering, Inventory, Payment, Shipping).
  4. Ghi chú các **khái niệm** (nouns) mà mọi người dùng: Order, Customer, Product, Line Item, Address...

**Mẹo**: Ghi âm/ghi chú ngôn ngữ thực tế của business (ví dụ: họ nói "đơn hàng bị hủy vì hết hàng" chứ không phải "update status = cancelled").

#### Bước 2: Xác định Core Domain & Subdomains

- Hỏi: Phần nào tạo giá trị cạnh tranh cao nhất? (Core Domain)
- Ví dụ:
  - Core: Ordering logic (tính giá, áp dụng khuyến mãi, xử lý hủy đơn phức tạp).
  - Supporting: Inventory (kiểm tra tồn kho).
  - Generic: Payment gateway integration.

→ Tập trung thiết kế Domain Model chi tiết cho **Core Domain** trước.

#### Bước 3: Xác định Aggregates (đơn vị nhất quán lớn nhất)

- Quy tắc vàng: **Một Aggregate = Một transaction boundary**.
- Cách tìm:
  1. Bắt đầu từ **Domain Event** → hỏi "Sự kiện này thay đổi trạng thái của những gì cùng lúc?"
  2. Nhóm các entity cần nhất quán với nhau.
  3. Chọn **Aggregate Root** (thực thể chính, chỉ truy cập qua nó).

Ví dụ Ordering Context:

- Aggregate: **Order**
  - Root: Order
  - Bên trong: OrderLines (Entity), ShippingAddress (Value Object), TotalAmount (tính toán)
- Không nên: Customer là Aggregate riêng (không nhất quán cùng Order).
- Aggregate khác: InventoryItem (nếu cần reserve stock).

**Mẹo tránh sai lầm**: Nếu Aggregate > 50-100 dòng code hoặc load > 10-20 entities → chia nhỏ.

#### Bước 4: Thiết kế Entities & Value Objects

- **Entity**: Có identity (ID), lifecycle, mutable → dùng khi cần theo dõi sự thay đổi theo thời gian.
  - Ví dụ: Order, OrderLine.
- **Value Object**: Không identity, immutable, so sánh bằng giá trị.
  - Ví dụ: Money (Amount + Currency), Address, Quantity.

Quy trình:

1. Liệt kê tất cả nouns từ Event Storming.
2. Hỏi: "Hai cái này giống nhau hoàn toàn có phải là một không?" → Nếu có → Value Object.
3. Hỏi: "Cần theo dõi sự thay đổi của nó riêng biệt không?" → Nếu có → Entity.

#### Bước 5: Đẩy Behavior vào Model (Rich Domain Model)

- Tránh **Anemic Domain Model** (chỉ data + getter/setter).
- Mỗi Entity/Aggregate phải có **hành vi** (methods) thể hiện business rules.

Ví dụ Order (C#):

```typescript
// Định nghĩa các kiểu bổ trợ (Giả định đã có sẵn)
type OrderId = string;
type CustomerId = string;
type ProductId = string;

export enum OrderStatus {
  Draft = "Draft",
  PendingPayment = "PendingPayment",
  Paid = "Paid",
}

// Giả định class Money đã tạo ở ví dụ trước
export class Order {
  private readonly _id: OrderId;
  private readonly _customerId: CustomerId;
  private readonly _lines: OrderLine[] = [];
  private _status: OrderStatus = OrderStatus.Draft;
  private _domainEvents: any[] = []; // Chứa các Domain Events

  // Private constructor để bắt buộc dùng static factory method
  private constructor(id: OrderId, customerId: CustomerId) {
    this._id = id;
    this._customerId = customerId;
  }

  // Getter cho ID và Status
  get id(): OrderId {
    return this._id;
  }
  get status(): OrderStatus {
    return this._status;
  }

  // IReadOnlyList tương đương với ReadonlyArray trong TS
  get lines(): ReadonlyArray<OrderLine> {
    return [...this._lines];
  }

  // Tính tổng tiền (Total) dựa trên các OrderLine
  get total(): Money {
    return this._lines.reduce(
      (acc, line) => acc.add(line.subTotal),
      Money.create(0, "VND"), // Giả định tiền tệ mặc định
    );
  }

  // Static Factory Method (Thay thế cho PlaceOrder)
  public static placeOrder(customerId: CustomerId, items: CartItem[]): Order {
    const order = new Order(crypto.randomUUID(), customerId);

    for (const item of items) {
      order.addLine(item);
    }

    order._status = OrderStatus.PendingPayment;
    order.raise(new OrderPlacedEvent(order.id));
    return order;
  }

  public addLine(item: CartItem): void {
    // Invariants: Kiểm tra trạng thái
    if (
      this._status !== OrderStatus.Draft &&
      this._status !== OrderStatus.PendingPayment
    ) {
      throw new Error("Cannot modify confirmed order");
    }

    // Invariant: Không trùng sản phẩm
    if (this._lines.some((l) => l.productId === item.productId)) {
      throw new Error("Duplicate product");
    }

    this._lines.push(
      new OrderLine(item.productId, item.quantity, item.unitPrice),
    );
  }

  public confirmPayment(): void {
    if (this._status !== OrderStatus.PendingPayment) {
      throw new Error("Invalid transition");
    }

    this._status = OrderStatus.Paid;
    this.raise(new OrderPaidEvent(this._id));
  }

  // Cơ chế quản lý Domain Events
  private raise(event: any): void {
    this._domainEvents.push(event);
  }

  public clearEvents(): void {
    this._domainEvents = [];
  }

  get domainEvents(): ReadonlyArray<any> {
    return [...this._domainEvents];
  }
}

// Lớp phụ trợ OrderLine (Entity bên trong Aggregate)
class OrderLine {
  constructor(
    public readonly productId: ProductId,
    public readonly quantity: number,
    public readonly unitPrice: Money,
  ) {}

  get subTotal(): Money {
    return Money.create(
      this.unitPrice.amount * this.quantity,
      this.unitPrice.currency,
    );
  }
}
```

→ Behavior nằm trong Aggregate → invariants được bảo vệ.

#### Bước 6: Xử lý Cross-Aggregate Logic

- Dùng **Domain Service** hoặc **Domain Event** + eventual consistency.
- Ví dụ: Khi OrderPlaced → cần reserve stock → raise OrderPlacedEvent → Inventory handler reserve.

#### Bước 7: Iterate & Refine liên tục

- Viết **unit tests** cho invariants và behavior (test-first nếu có thể).
- Refactor khi Ubiquitous Language thay đổi.
- Dùng **code review** + **domain expert review** định kỳ.

## Aggregate

### 1. Aggregate là gì? (Nhắc lại ngắn gọn để thống nhất)

- **Aggregate** = một nhóm các Entity + Value Object **liên kết chặt chẽ**, được coi là **một đơn vị duy nhất** cho:
  - Consistency (nhất quán transactional)
  - Transaction boundary
  - Persistence (thường load/save cùng lúc)
  - Concurrency control
- **Aggregate Root** = Entity duy nhất mà code bên ngoài được phép giữ reference trực tiếp và gọi method lên. Tất cả thay đổi bên trong Aggregate phải đi qua Root.

### 2. Bộ quy tắc cốt lõi (Rules of Thumb) – Vaughn Vernon & Eric Evans

Đây là 4 quy tắc vàng từ sách **Implementing Domain-Driven Design** (Red Book) của Vaughn Vernon, được cộng đồng DDD công nhận rộng rãi nhất:

| #   | Quy tắc (Rule)                                      | Ý nghĩa chính                                                                            | Tại sao quan trọng?                                         | Khi nào vi phạm? (exception)                                |
| --- | --------------------------------------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------- |
| 1   | **Model True Invariants in Consistency Boundaries** | Chỉ nhóm những gì **phải nhất quán transactional** (true business invariant)             | Tránh over-clustering → Aggregate to → chậm, conflict nhiều | Rất hiếm – nếu không phải invariant thật thì tách ra        |
| 2   | **Design Small Aggregates**                         | Aggregate càng nhỏ càng tốt (thường chỉ Root + vài Entity/Value Object)                  | Giảm lock contention, tăng throughput, dễ scale, dễ test    | Khi business thực sự yêu cầu consistency mạnh               |
| 3   | **Reference Other Aggregates by Identity Only**     | Không giữ reference trực tiếp đến Aggregate khác, chỉ giữ ID (hoặc Value Object chứa ID) | Tránh cascade load, giảm memory, tránh cyclic dependency    | Hiếm – chỉ khi cần UI hiển thị immediate                    |
| 4   | **Use Eventual Consistency Outside the Boundary**   | Giữa các Aggregate khác nhau → dùng eventual consistency (qua Domain Event)              | Giảm coupling, tăng scalability, phù hợp distributed system | Khi user mong đợi immediate consistency (rất ít trường hợp) |

### 3. Cách nhận biết Aggregate trong dự án thực tế (Discovery Process)

Sử dụng các câu hỏi + kỹ thuật này theo thứ tự:

#### Bước 1: Event Storming hoặc Domain Storytelling (bắt buộc)

- Liệt kê **Domain Events** và **Commands** trước.
- Hỏi: “Sự kiện này thay đổi những gì cùng lúc? Những thay đổi nào phải thành công hoặc thất bại cùng nhau?”
- Nhóm các object bị ảnh hưởng bởi cùng một command/event → candidate cho Aggregate.

#### Bước 2: Áp dụng các câu hỏi kiểm tra Invariant (Rule #1)

Hỏi domain expert + tự hỏi:

- “Business rule nào **phải luôn đúng** ngay sau khi command hoàn thành?”
- “Nếu hai thay đổi xảy ra đồng thời, cái nào không được phép xảy ra?”
- “Nếu không enforce rule này trong cùng transaction → có thể gây lỗi kinh doanh nghiêm trọng không?”

Ví dụ Ordering Context:

- Invariant thật: “Tổng tiền Order phải bằng tổng OrderLines” → phải trong cùng Aggregate.
- Invariant thật: “Không được ship Order nếu chưa Paid” → trong Order Aggregate.
- **Không phải** invariant thật: “Order phải có ít nhất 1 line khi tạo” → có thể eventual (hoặc chỉ validation lúc create).
- “Số lượng tồn kho phải giảm khi Order confirmed” → **không** trong Order Aggregate → tách Inventory Aggregate, dùng eventual consistency.

#### Bước 3: Kiểm tra kích thước & performance (Rule #2)

- Aggregate có > 50–100 dòng code hoặc load > 10–20 entities → nghi ngờ quá to.
- Hỏi: “Nếu 100 user cùng modify Aggregate này → conflict rate bao nhiêu?”
- Nếu cao → chia nhỏ.

Ví dụ sai phổ biến:

- Order + Customer + Payment + Shipment + Invoice → quá to → chia thành Order, Payment, Shipment...

#### Bước 4: Kiểm tra reference (Rule #3)

- Nếu class A giữ reference trực tiếp đến B (không phải ID) → hỏi: “Có cần load toàn bộ B mỗi khi load A không?”
- Nếu không cần → chuyển thành chỉ ID.

#### Bước 5: Xác định Aggregate Root

- Là Entity mà **ngoài Aggregate không được phép truy cập trực tiếp** các phần tử bên trong.
- Thường là khái niệm chính trong use-case: Order, Booking, Account, Cart...
- Root phải expose method để thay đổi toàn bộ Aggregate (ví dụ: `order.addLine(...)`, `order.ship()`).

### 4. Checklist nhanh khi review Aggregate (dùng trong code review)

- [ ] Aggregate chỉ chứa **true invariants** (Rule 1)?
- [ ] Aggregate nhỏ, load nhanh, ít entity (Rule 2)?
- [ ] Chỉ reference Aggregate khác bằng **ID** (Rule 3)?
- [ ] Consistency với Aggregate khác dùng **eventual** (Domain Event + handler) (Rule 4)?
- [ ] Tất cả thay đổi đi qua **Root** (không public setter cho child entities)?
- [ ] Có **concurrency token** (RowVersion/ETag) để optimistic lock?
- [ ] Repository chỉ làm việc với Aggregate Root (GetById, Save toàn bộ Aggregate)?

### 5. Ví dụ thực tế Ordering Context (TypeScript như trước)

```typescript
// Aggregate Root: Order
export class Order {
  private readonly _lines: OrderLine[] = [];

  // ... constructor & factory như ví dụ trước

  public addLine(
    productId: ProductId,
    quantity: number,
    unitPrice: Money,
  ): void {
    // Invariant: chỉ thêm khi còn ở trạng thái cho phép
    if (
      ![OrderStatus.Draft, OrderStatus.PendingPayment].includes(this._status)
    ) {
      throw new DomainError("Cannot modify confirmed order");
    }
    // Invariant: không trùng product (nếu business yêu cầu)
    if (this._lines.some((l) => l.productId.equals(productId))) {
      throw new DomainError("Duplicate product");
    }
    this._lines.push(OrderLine.create(productId, quantity, unitPrice));
  }

  public confirmPayment(): void {
    // Invariant: transition hợp lệ
    if (this._status !== OrderStatus.PendingPayment)
      throw new DomainError("Invalid transition");
    this._status = OrderStatus.Paid;
    this.addDomainEvent(new OrderPaidEvent(this.id));
  }
}
```

- **Inventory Aggregate** riêng → khi OrderPaidEvent → handler giảm stock (eventual consistency).

### Lời khuyên cuối

- **Bắt đầu nhỏ**: Đừng cố thiết kế hoàn hảo ngay từ đầu. Bắt đầu với Aggregate lớn → refactor khi thấy conflict/performance issue.
- **Iterate**: Sau 1–2 sprint, xem log concurrency, size của Aggregate trong DB → điều chỉnh.
- **Hỏi domain expert liên tục**: “Rule này có thật sự phải immediate consistency không?”

### VÍ DỤ THỰC TẾ

### Giả định Bounded Context

Chúng ta đang ở **Booking Context** (hoặc Reservation Context) – nơi xử lý việc tạo, xác nhận, hủy, thay đổi đặt chỗ.

**Không** phải toàn bộ hệ thống (không bao gồm Inventory/Availability, Payment, Customer Profile... – những cái đó thường là Aggregate riêng).

### Aggregate chính: Booking (hoặc Reservation)

**Aggregate Root**: `Booking`

**Lý do chọn Booking làm Aggregate Root**:

- Đây là khái niệm chính mà user/business quan tâm: "Tôi muốn đặt chỗ X cho ngày Y".
- Invariants (quy tắc bất biến) cần enforce transactional nằm quanh Booking: ngày check-in/out, trạng thái, tổng giá, điều kiện hủy, số lượng người...
- Booking có lifecycle rõ ràng: Draft → Pending → Confirmed → CheckedIn → Completed → Cancelled.
- Thay đổi Booking (thêm dịch vụ, thay đổi ngày, hủy) phải nhất quán ngay lập tức.

**Các phần tử bên trong Aggregate (Entities + Value Objects)**:

- **Booking** (Root) – Entity
- **GuestInfo** hoặc **PrimaryGuest** – Value Object (hoặc Entity nếu cần theo dõi riêng)
- **StayPeriod** / **DateRange** – Value Object (check-in, check-out)
- **BookingItems** / **ReservedUnits** – Collection of Entity (ví dụ: phòng, dịch vụ thêm, suất ăn...)
- **PricingDetails** – Value Object (base price, taxes, discounts, total)
- **CancellationPolicy** – Value Object
- **PaymentStatus** – Value Object hoặc enum + related info

**Không** nên nhồi nhét quá nhiều:

- **Room** / **Resource** (phòng, ghế, slot lịch) → thường là Aggregate riêng (Resource/Inventory Aggregate) → chỉ reference bằng ID.
- **Customer** → Aggregate riêng → chỉ CustomerId.
- **Payment** → Aggregate riêng (Payment Aggregate) → eventual consistency qua event.

### Ví dụ code TypeScript (Booking Aggregate)

```typescript
// domain/aggregates/booking.ts
import { AggregateRoot } from "../base/aggregate-root";
import { BookingId } from "../value-objects/booking-id";
import { CustomerId } from "../value-objects/customer-id";
import { RoomId } from "../value-objects/room-id"; // Chỉ ID, không reference Room object
import { DateRange } from "../value-objects/date-range";
import { Money } from "../value-objects/money";
import { GuestInfo } from "../value-objects/guest-info";
import { BookingStatus } from "../enums/booking-status";
import { DomainEvent } from "../base/domain-event";

// Events ví dụ
export class BookingCreatedEvent extends DomainEvent {
  constructor(public readonly bookingId: BookingId) {
    super();
  }
}
export class BookingConfirmedEvent extends DomainEvent {
  constructor(public readonly bookingId: BookingId) {
    super();
  }
}
export class BookingCancelledEvent extends DomainEvent {
  constructor(
    public readonly bookingId: BookingId,
    public readonly reason: string,
  ) {
    super();
  }
}

export class Booking extends AggregateRoot {
  private _status: BookingStatus = BookingStatus.Draft;
  private _items: BookingItem[] = []; // Collection of Entity

  private constructor(
    public readonly id: BookingId,
    public readonly customerId: CustomerId,
    public readonly dateRange: DateRange,
    public readonly totalPrice: Money,
    private readonly domainEvents: DomainEvent[] = [],
  ) {
    super();
  }

  // Factory method - tạo Booking mới
  public static create(
    customerId: CustomerId,
    roomId: RoomId, // Chỉ ID
    checkIn: Date,
    checkOut: Date,
    guestInfo: GuestInfo,
    basePrice: Money,
  ): Booking {
    const dateRange = DateRange.create(checkIn, checkOut);
    if (dateRange.isInvalid()) {
      throw new DomainError("Invalid stay period");
    }

    const booking = new Booking(
      BookingId.create(),
      customerId,
      dateRange,
      basePrice, // Có thể tính thêm tax/discount sau
    );

    // Thêm item đầu tiên (phòng chính)
    booking.addItem(roomId, basePrice);

    booking._status = BookingStatus.PendingConfirmation;
    booking.addDomainEvent(new BookingCreatedEvent(booking.id));

    return booking;
  }

  // Behavior: Thêm dịch vụ/phòng phụ (nếu business cho phép multiple items)
  public addItem(resourceId: RoomId | ServiceId, price: Money): void {
    if (
      this._status !== BookingStatus.Draft &&
      this._status !== BookingStatus.PendingConfirmation
    ) {
      throw new DomainError("Cannot modify confirmed booking");
    }

    this._items.push(BookingItem.create(resourceId, price));
    // Có thể recalculate totalPrice ở đây nếu cần
  }

  // Confirm booking (ví dụ sau khi check availability và payment intent)
  public confirm(): void {
    if (this._status !== BookingStatus.PendingConfirmation) {
      throw new DomainError("Invalid transition");
    }
    this._status = BookingStatus.Confirmed;
    this.addDomainEvent(new BookingConfirmedEvent(this.id));
  }

  // Cancel với policy check
  public cancel(reason: string): void {
    if (
      ![BookingStatus.PendingConfirmation, BookingStatus.Confirmed].includes(
        this._status,
      )
    ) {
      throw new DomainError("Cannot cancel this booking");
    }
    // Kiểm tra cancellation policy (nếu có Value Object riêng)
    // Ví dụ: if (now > freeCancellationUntil) throw error...

    this._status = BookingStatus.Cancelled;
    this.addDomainEvent(new BookingCancelledEvent(this.id, reason));
  }

  public get status(): BookingStatus {
    return this._status;
  }

  public get items(): ReadonlyArray<BookingItem> {
    return [...this._items];
  }

  // Domain Events
  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event);
  }

  public getDomainEvents(): DomainEvent[] {
    return [...this.domainEvents];
  }

  public clearDomainEvents(): void {
    this.domainEvents.length = 0;
  }
}

// Entity con bên trong Aggregate
export class BookingItem {
  readonly id: string;
  readonly resourceId: RoomId | ServiceId;
  readonly price: Money;

  private constructor(
    id: string,
    resourceId: RoomId | ServiceId,
    price: Money,
  ) {
    this.id = id;
    this.resourceId = resourceId;
    this.price = price;
  }

  public static create(
    resourceId: RoomId | ServiceId,
    price: Money,
  ): BookingItem {
    return new BookingItem(crypto.randomUUID(), resourceId, price);
  }
}
```

### Một số Aggregate liên quan khác (thường tách riêng)

1. **Availability / Calendar Aggregate** (Root: ResourceCalendar hoặc Room)
   - Invariant: Không cho double-book cùng slot.
   - Khi Booking → Confirmed → raise `BookingConfirmedEvent` → handler reserve slot (eventual consistency).

2. **Customer Aggregate** → chỉ reference bằng CustomerId.

3. **Payment Aggregate** → xử lý thanh toán, capture, refund.

### Lý do thiết kế như vậy (áp dụng rules)

- **Rule 1 (True Invariants)**: Trạng thái Booking, tổng giá, items phải nhất quán ngay khi thay đổi.
- **Rule 2 (Small Aggregate)**: Booking không load toàn bộ Room/Customer → chỉ ID → nhỏ gọn.
- **Rule 3 (Reference by ID)**: RoomId, CustomerId.
- **Rule 4 (Eventual Consistency)**: Kiểm tra phòng trống, giảm tồn kho → qua Domain Event + handler ở Availability Context.
