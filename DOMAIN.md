# Domain Context: Coffee Shop Platform

## System Overview
This system is the backend infrastructure for a modern, multi-location coffee shop chain. It powers the customer-facing mobile app (Mobile Order & Pay), the in-store Point of Sale (POS), and the barista order-fulfillment screens (Kitchen Display System).

The system must handle high-throughput bursts during morning rush hours, requiring fast reads for menus and highly consistent writes for payments and inventory.

---

## 1. System Actors

| Actor | Description | Key Permissions |
| :--- | :--- | :--- |
| **Customer** | End-user using the mobile app or web portal. | Create orders, manage payment methods, view loyalty points. |
| **Barista** | Store employee fulfilling orders. | View order queue, update order status to `PREPARING` or `READY`. |
| **Store Manager** | Local store administrator. | Update local inventory statuses (e.g., marking "Oat Milk" out of stock), refund orders. |
| **System Admin** | Corporate administrator. | Manage global menu items, pricing tiers, and global store configurations. |

---

## 2. Ubiquitous Language (Dictionary)
To maintain consistency across the codebase and database, always use these exact terms. Do not use synonyms (e.g., use `Order`, not `Cart` or `Purchase`).

| Term | Definition |
| :--- | :--- |
| **Product** | A globally defined menu item (e.g., "Latte", "Croissant"). |
| **Modifier** | A customization applied to a Product (e.g., "Extra Espresso Shot", "Oat Milk"). Modifiers can affect the final price. |
| **LineItem** | A specific instance of a Product + Modifiers added to an Order. |
| **Order** | The entire transaction for a Customer. Contains multiple LineItems, a calculated total, and a tax amount. |
| **Ticket** | The routing representation of an Order sent to the barista's screen. (One Order might split into a "Drink Ticket" and a "Food Ticket"). |
| **LoyaltyAccount** | A record tracking a Customer's earned points and available rewards. |

---

## 3. Core Business Rules & Invariants

### Order Lifecycle
1. **State Machine:** Orders must strictly follow this state flow: `DRAFT` -> `PENDING_PAYMENT` -> `PAID` -> `PREPARING` -> `READY` -> `FULFILLED`.
2. **Immutability:** Once an Order reaches `PAID`, its `LineItems` cannot be modified. If a customer wants to change their drink, the order must be `CANCELED` and refunded, and a new order created.
3. **No Deletion:** Financial records (`Order`, `Payment`, `Refund`) are **never** hard-deleted from the database. Use soft deletes or status changes for auditing purposes.

### Pricing and Modifiers
1. **Dynamic Pricing:** The final price of a `LineItem` is `Product.BasePrice + sum(Modifier.Price)`. 
2. **Taxation:** Taxes are calculated at the `Order` level based on the Store's local tax rate, not globally.

### Inventory & Fulfillment
1. **Out of Stock:** If a Store marks a `Product` or `Modifier` as `OUT_OF_STOCK`, the system must prevent any new `DRAFT` orders containing that item from transitioning to `PENDING_PAYMENT` for that specific store.
2. **Idempotent Payments:** Payment processing must be strictly idempotent to prevent charging a customer twice if the mobile app retries a network-timeout request.

---

## 4. Key Subdomains (Bounded Contexts)

When organizing Go packages (`internal/`), map them to these subdomains:

* **`internal/catalog`**: Manages Products, Modifiers, and Store Menus. (High read, low write).
* **`internal/ordering`**: Manages the cart, order calculation, and the state machine. (High read, high write).
* **`internal/fulfillment`**: Manages the Barista queue and Ticket routing. (Real-time/Event-driven).
* **`internal/billing`**: Manages payment gateway integrations, refunds, and receipts. (Strict consistency required).