# ER Diagram – Indian E-Commerce Platform

ER model for an Indian e-commerce platform (in the style of Flipkart or Amazon.in) with the entities **Customer, Product, Order, OrderItem, Seller, Category, Payment, Delivery and Address**.

---

## 1. Notation Used

| Symbol in diagram | Meaning |
|---|---|
| `PK` | Primary key |
| `FK` | Foreign key |
| `UK` | Unique (candidate) key |
| `PK, FK` | Attribute is both part of the primary key and a foreign key (weak-entity key) |
| **Solid line** `──` | **Identifying** relationship (owner → weak entity) |
| **Dashed line** `- -` | **Non-identifying** (normal) relationship |
| `\|\|` | Exactly one → **total (mandatory)** participation |
| `\|o` | Zero or one → **partial (optional)** participation |
| `\|{` | One or more → **total (mandatory)** participation |
| `o{` | Zero or more → **partial (optional)** participation |
| `"composite: X"` | Attribute is a component of composite attribute **X** |
| `"multi-valued"` | Attribute is multi-valued (stored in its own table) |
| `"partial key"` | Discriminator of a weak entity |
| `"derived"` | Derived attribute (computed, not stored) |

---

## 2. Main ER Diagram

```mermaid
erDiagram
    CUSTOMER {
        string customer_id PK
        string first_name "composite: name"
        string middle_name "composite: name"
        string last_name "composite: name"
        string email UK
        date date_of_birth
        int age "derived from date_of_birth"
        date registered_on
    }

    CUSTOMER_PHONE {
        string customer_id PK, FK
        string phone_number PK "multi-valued, +91 mobile"
    }

    ADDRESS {
        string customer_id PK, FK "owner key"
        int address_seq PK "partial key (weak entity)"
        string house_no "composite: street_address"
        string street "composite: street_address"
        string landmark "composite: street_address"
        string city
        string state
        string pincode "6-digit PIN code"
        string address_type "Home, Work, Other"
    }

    SELLER {
        string seller_id PK
        string business_name
        string gstin UK "15-char GST number"
        string pan UK "10-char PAN"
        string contact_first_name "composite: contact_name"
        string contact_last_name "composite: contact_name"
        string email UK
        string pickup_city "composite: pickup_address"
        string pickup_state "composite: pickup_address"
        string pickup_pincode "composite: pickup_address"
        float rating "derived from reviews"
    }

    SELLER_PHONE {
        string seller_id PK, FK
        string phone_number PK "multi-valued"
    }

    CATEGORY {
        string category_id PK
        string category_name UK
        string description
        string parent_category_id FK "recursive: sub-category"
    }

    PRODUCT {
        string product_id PK
        string seller_id FK
        string category_id FK
        string product_name
        string brand
        decimal mrp "INR"
        decimal selling_price "INR"
        string hsn_code "GST HSN code"
        decimal gst_rate "0, 5, 12, 18 or 28 percent"
        int stock_qty
        string product_type "discriminator for specialization"
    }

    PRODUCT_IMAGE {
        string product_id PK, FK
        string image_url PK "multi-valued"
    }

    ELECTRONICS {
        string product_id PK, FK
        string model_number
        int warranty_months
        string bis_certification "BIS registration no."
    }

    CLOTHING {
        string product_id PK, FK
        string size "XS to XXL"
        string fabric
        string gender "Men, Women, Kids, Unisex"
    }

    BOOK {
        string product_id PK, FK
        string isbn UK
        string author
        string publisher
        string language "Hindi, English, etc."
    }

    GROCERY {
        string product_id PK, FK
        string fssai_license "FSSAI licence no."
        date expiry_date
        boolean is_vegetarian "green or brown dot"
    }

    ORDER {
        string order_id PK
        string customer_id FK
        int address_seq FK "ship-to address of the customer"
        datetime order_date
        string order_status "Placed, Shipped, Delivered, Cancelled, Returned"
        decimal total_amount "derived: sum of order items"
    }

    ORDER_ITEM {
        string order_id PK, FK "owner key"
        int line_no PK "partial key (weak entity)"
        string product_id FK
        int quantity
        decimal unit_price "price at time of order"
        decimal gst_amount
    }

    PAYMENT {
        string payment_id PK
        string order_id FK
        decimal amount "INR"
        string payment_mode "UPI, Card, NetBanking, Wallet, COD, EMI"
        string transaction_ref UK "UPI UTR or gateway ref"
        string payment_status "Pending, Success, Failed, Refunded"
        datetime paid_at
    }

    DELIVERY {
        string order_id PK, FK "owner key"
        int shipment_no PK "partial key (weak entity)"
        string courier_partner "e.g. Delhivery, Ekart, India Post"
        string awb_number UK "airway bill / tracking no."
        date dispatch_date
        date expected_date
        date delivered_date
        string delivery_status
    }

    %% ---------- Relationships ----------
    CUSTOMER ||--|{ CUSTOMER_PHONE : "has"
    CUSTOMER ||--o{ ADDRESS : "owns"
    CUSTOMER ||--o{ ORDER : "places"
    ADDRESS  ||..o{ ORDER : "ships to"

    SELLER   ||--|{ SELLER_PHONE : "has"
    SELLER   ||..o{ PRODUCT : "sells"

    CATEGORY ||..o{ PRODUCT : "classifies"
    CATEGORY |o..o{ CATEGORY : "parent of"

    PRODUCT  ||--|{ PRODUCT_IMAGE : "has"
    PRODUCT  ||..o{ ORDER_ITEM : "appears in"

    ORDER    ||--|{ ORDER_ITEM : "contains"
    ORDER    ||..|{ PAYMENT : "paid by"
    ORDER    ||--o{ DELIVERY : "shipped as"

    PRODUCT  ||--o| ELECTRONICS : "is a"
    PRODUCT  ||--o| CLOTHING : "is a"
    PRODUCT  ||--o| BOOK : "is a"
    PRODUCT  ||--o| GROCERY : "is a"
```

---

## 3. Specialization of Product (ISA Hierarchy)

```mermaid
flowchart TD
    P["PRODUCT<br/>(superclass)<br/>product_id, name, brand, mrp,<br/>selling_price, hsn_code, gst_rate"]
    D(("d"))
    E["ELECTRONICS<br/>model_number, warranty_months,<br/>bis_certification"]
    C["CLOTHING<br/>size, fabric, gender"]
    B["BOOK<br/>isbn, author, publisher, language"]
    G["GROCERY<br/>fssai_license, expiry_date,<br/>is_vegetarian"]

    P ---|"product_type"| D
    D --- E
    D --- C
    D --- B
    D --- G
```

| Property | Value | Reason |
|---|---|---|
| **Disjointness** | **Disjoint (d)** | A product belongs to at most one subclass (a book cannot also be clothing). |
| **Completeness** | **Partial** | Some products (e.g. furniture, toys) belong to none of the four subclasses and stay plain `PRODUCT`. |
| **Defining attribute** | `product_type` | Attribute-defined specialization: its value decides the subclass. |
| **Inheritance** | Each subclass inherits all `PRODUCT` attributes and uses `product_id` as its PK and FK. |

---

## 4. Primary Keys

| Entity | Primary Key | Type |
|---|---|---|
| Customer | `customer_id` | Simple |
| Address | (`customer_id`, `address_seq`) | Composite: owner key + partial key |
| Seller | `seller_id` | Simple (`gstin`, `pan` are candidate keys) |
| Category | `category_id` | Simple |
| Product | `product_id` | Simple |
| Electronics / Clothing / Book / Grocery | `product_id` | Inherited from Product |
| Order | `order_id` | Simple |
| OrderItem | (`order_id`, `line_no`) | Composite: owner key + partial key |
| Payment | `payment_id` | Simple (`transaction_ref` is a candidate key) |
| Delivery | (`order_id`, `shipment_no`) | Composite: owner key + partial key |

---

## 5. Composite Attributes

| Entity | Composite Attribute | Components |
|---|---|---|
| Customer | `name` | first_name, middle_name, last_name |
| Address | `street_address` | house_no, street, landmark |
| Address | full address | street_address, city, state, pincode |
| Seller | `contact_name` | contact_first_name, contact_last_name |
| Seller | `pickup_address` | pickup_city, pickup_state, pickup_pincode |

## 6. Multi-valued Attributes

| Entity | Multi-valued Attribute | Mapped To |
|---|---|---|
| Customer | phone_numbers | `CUSTOMER_PHONE` (customer_id, phone_number) |
| Seller | phone_numbers | `SELLER_PHONE` (seller_id, phone_number) |
| Product | images | `PRODUCT_IMAGE` (product_id, image_url) |

## 7. Derived Attributes

| Entity | Attribute | Derived From |
|---|---|---|
| Customer | `age` | date_of_birth |
| Order | `total_amount` | Σ (quantity × unit_price + gst_amount) of its order items |
| Seller | `rating` | Customer reviews |

---

## 8. Weak Entities

| Weak Entity | Owner (Identifying) Entity | Identifying Relationship | Partial Key | Full Key |
|---|---|---|---|---|
| **ADDRESS** | CUSTOMER | owns | `address_seq` | (customer_id, address_seq) |
| **ORDER_ITEM** | ORDER | contains | `line_no` | (order_id, line_no) |
| **DELIVERY** | ORDER | shipped as | `shipment_no` | (order_id, shipment_no) |

A weak entity cannot exist without its owner, so its participation in the identifying relationship is always **total**. Deleting a customer deletes their addresses, and deleting an order deletes its items and shipments.

---

## 9. Relationships, Cardinality and Participation Constraints

| Relationship | Entities | Cardinality | Participation | Meaning |
|---|---|---|---|---|
| places | Customer – Order | 1 : N | Customer **partial**, Order **total** | A customer may have no orders; every order belongs to exactly one customer. |
| owns *(identifying)* | Customer – Address | 1 : N | Customer **partial**, Address **total** | A customer may save 0+ addresses; an address cannot exist without its customer. |
| ships to | Address – Order | 1 : N | Address **partial**, Order **total** | Every order is shipped to exactly one address. |
| sells | Seller – Product | 1 : N | Seller **partial**, Product **total** | A new seller may have no listings; every product has one seller. |
| classifies | Category – Product | 1 : N | Category **partial**, Product **total** | Every product is in one category. |
| parent of *(recursive)* | Category – Category | 1 : N | **Partial** both sides | Electronics → Mobiles → Smartphones; top-level categories have no parent. |
| contains *(identifying)* | Order – OrderItem | 1 : N | **Total** both sides | An order has at least one item; an item belongs to exactly one order. |
| appears in | Product – OrderItem | 1 : N | Product **partial**, OrderItem **total** | A product may never be ordered; every item refers to one product. |
| paid by | Order – Payment | 1 : N | **Total** both sides | Every order has at least one payment record (retries and COD included). |
| shipped as *(identifying)* | Order – Delivery | 1 : N | Order **partial**, Delivery **total** | An order may be split into several shipments; a cancelled order has none. |
| is a *(specialization)* | Product – Subclasses | 1 : 0..1 | Product **partial**, Subclass **total** | Disjoint, partial specialization. |

> **Many-to-many resolved:** Order ↔ Product is M : N, resolved through the weak entity **ORDER_ITEM**, which stores `quantity`, `unit_price` and `gst_amount`.

---

## 10. Indian-Specific Design Choices

- **GSTIN, PAN** on Seller and **HSN code, GST rate** on Product for GST invoicing.
- **6-digit PIN code** and **state** on Address for delivery serviceability and IGST/CGST/SGST calculation.
- **Payment modes:** UPI, Cards, NetBanking, Wallets, **Cash on Delivery** and EMI; `transaction_ref` stores the UPI UTR.
- **Courier partners** and **AWB number** on Delivery for shipment tracking.
- **FSSAI licence** and veg/non-veg marking on Grocery; **BIS certification** on Electronics.
- All amounts are in **INR (₹)**.
