You’re using the **Mastercard Payment Gateway Services (MPGS)** `RETRIEVE_ORDER` API to verify a payment.
To build a **scheduler (cron job / background worker)** that periodically checks if an order was paid, follow this pattern.

---

# 1️⃣ Official API Concept (MPGS)

The API you are calling:

```
apiOperation=RETRIEVE_ORDER
```

Purpose:

* Retrieve **latest status of an order**
* Check **authorization**
* Check **captured amount**
* Inspect **transactions**

This is the **correct API for payment verification**.

---

# 2️⃣ Scheduler Logic (Recommended Flow)

### Step 1 — Store order when payment starts

When the user starts checkout, save:

```
order_id
amount
status = PENDING
created_at
```

Example

```
REG-6-1773337455524
```

---

### Step 2 — Scheduler runs every X minutes

Example cron:

```
*/2 * * * *
```

Runs **every 2 minutes**.

The scheduler:

1. Get orders with `status = PENDING`
2. Call `RETRIEVE_ORDER`
3. Update database.

---

# 3️⃣ Example Request (your current one)

```bash
curl -X POST https://test-gateway.mastercard.com/api/nvp/version/70 \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "apiOperation=RETRIEVE_ORDER" \
-d "apiPassword=API_PASSWORD" \
-d "apiUsername=merchant.xxxxxxxxxxxxxxxxx" \
-d "merchant=xxxxxxxxxxx" \
-d "order.id=REG-6-1773337455524"
```

---

# 4️⃣ Fields Scheduler Must Check

From the response check:

### Payment success

```
status=CAPTURED
```

AND

```
totalCapturedAmount > 0
```

Example success:

```
status=CAPTURED
totalCapturedAmount=1000.00
```

---

### Payment still pending

```
status=AUTHENTICATION_PENDING
```

or

```
status=CREATED
```

---

### Payment failed

```
status=AUTHENTICATION_UNSUCCESSFUL
```

or

```
gatewayCode=DECLINED
```

---

# 5️⃣ Scheduler Decision Table

| status                      | Meaning                     | Action          |
| --------------------------- | --------------------------- | --------------- |
| CAPTURED                    | Payment successful          | mark **PAID**   |
| AUTHENTICATION_PENDING      | waiting user authentication | retry           |
| CREATED                     | payment started             | retry           |
| AUTHENTICATION_UNSUCCESSFUL | failed                      | mark **FAILED** |

---

# 6️⃣ Example Scheduler (Node.js)

```javascript
import axios from "axios";
import qs from "qs";

async function checkPayment(orderId) {

  const data = qs.stringify({
    apiOperation: "RETRIEVE_ORDER",
    apiPassword: process.env.MPGS_PASSWORD,
    apiUsername: "merchant.TESTBOK000012",
    merchant: "TESTBOK000012",
    "order.id": orderId
  });

  const res = await axios.post(
    "https://test-gateway.mastercard.com/api/nvp/version/70",
    data,
    { headers: { "Content-Type": "application/x-www-form-urlencoded" } }
  );

  const result = res.data;

  if (result.status === "CAPTURED") {
    console.log("Payment successful");
  } else {
    console.log("Payment not completed yet");
  }
}
```

---

# 7️⃣ Example Cron Scheduler

```javascript
import cron from "node-cron";

cron.schedule("*/2 * * * *", async () => {

  const pendingOrders = await db.orders.find({
    status: "PENDING"
  });

  for (const order of pendingOrders) {
    await checkPayment(order.id);
  }

});
```

---

# 8️⃣ Important Production Rule

Never trust **redirect responses from frontend**.

Always verify payment with:

```
RETRIEVE_ORDER
```

from your **server**.

---

# 9️⃣ Best Practice (Very Important)

Stop checking after:

```
status = CAPTURED
status = FAILED
```

Retry only for:

```
CREATED
AUTHENTICATION_PENDING
```

---

✅ If you want, I can also show you **the best architecture used by large payment systems**:

Webhook + Scheduler fallback (used with **Mastercard MPGS integrations).
It prevents **lost payments and double processing**.


Below is a **focused documentation of the REFUND flow** in **Mastercard Payment Gateway Services (MPGS)** using Hosted Checkout.

The key idea: **refund happens only after a successful payment transaction exists**.

---

# MPGS Refund Flow (Step-by-Step)

## 1️⃣ Successful Payment Must Exist

Before a refund can happen, the order must already have a **successful payment transaction**.

Typical earlier flow:

```
INITIATE_CHECKOUT
      ↓
Customer payment
      ↓
PURCHASE transaction
      ↓
Payment approved
```

You must know:

* `order.id`
* `transaction.id` (original payment)

---

# 2️⃣ Retrieve the Original Transaction

First get the **transaction ID** using `RETRIEVE_ORDER`.

### Request

```bash
curl -X POST "https://test-gateway.mastercard.com/api/nvp/version/70" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "apiOperation=RETRIEVE_ORDER" \
  -d "apiUsername=merchant.TESTBOK000012" \
  -d "apiPassword=YOUR_PASSWORD" \
  -d "merchant=TESTBOK000012" \
  -d "order.id=ORDER123"
```

### Example Response

```
result=SUCCESS
order.id=ORDER123

transaction[0].id=TXN_ABC123
transaction[0].type=PAYMENT
transaction[0].result=SUCCESS
transaction[0].amount=100.00
```

Save:

```
TXN_ABC123
```

This is the **target transaction** for the refund.

---

# 3️⃣ Send Refund Request

Use **`apiOperation=REFUND`**.

### Request

```bash
curl -X POST "https://test-gateway.mastercard.com/api/nvp/version/70" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "apiOperation=REFUND" \
  -d "apiUsername=merchant.TESTBOK000012" \
  -d "apiPassword=YOUR_PASSWORD" \
  -d "merchant=TESTBOK000012" \
  -d "order.id=ORDER123" \
  -d "transaction.targetTransactionId=TXN_ABC123" \
  -d "transaction.amount=100.00" \
  -d "transaction.currency=USD"
```

### Important Parameters

| Parameter                         | Purpose                      |
| --------------------------------- | ---------------------------- |
| `transaction.targetTransactionId` | original payment transaction |
| `transaction.amount`              | amount to refund             |
| `transaction.currency`            | currency                     |

---

# 4️⃣ Refund Response

Example response:

```
result=SUCCESS
transaction.type=REFUND
transaction.result=SUCCESS
transaction.amount=100.00
```

This means:

✅ Refund successfully processed.

---

# 5️⃣ Partial Refund Example

Refund part of the amount.

Example original payment:

```
100.00 USD
```

Refund:

```bash
transaction.amount=30.00
```

Remaining refundable amount:

```
70.00 USD
```

Multiple partial refunds are allowed until the total equals the original payment.

---

# 6️⃣ Refund Flow Diagram

```
Customer Payment
      │
      ▼
PURCHASE transaction created
      │
      ▼
Merchant retrieves order
      │
      ▼
Merchant sends REFUND request
      │
      ▼
Gateway creates REFUND transaction
      │
      ▼
Funds returned to card
```

---

# 7️⃣ Refund Transaction Structure

After refund, the order will have **multiple transactions**:

```
transaction[0] = PAYMENT
transaction[1] = REFUND
```

Example:

```
transaction[0].type=PAYMENT
transaction[0].amount=100

transaction[1].type=REFUND
transaction[1].amount=100
```

---

# 8️⃣ Important Rules

Refund rules in MPGS:

| Rule                                 | Explanation              |
| ------------------------------------ | ------------------------ |
| Refund requires original transaction | Must reference payment   |
| Cannot exceed original amount        | Max = payment amount     |
| Currency must match                  | Same as original payment |
| Refund creates a new transaction     | Does not modify original |

---

# 9️⃣ Common Errors

### Invalid operation

```
interaction.operation=REFUND
```

❌ Invalid in checkout.

Refund must use:

```
apiOperation=REFUND
```

---

### Missing target transaction

```
transaction.targetTransactionId missing
```

Refund will fail.

---

# 🔟 Complete Refund Flow

```
1. Customer pays
2. Payment approved
3. Merchant retrieves order
4. Merchant sends REFUND request
5. Gateway processes refund
6. Refund transaction created
7. Funds returned to customer
```

---

💡 **Integration tip**

Production systems normally store:

```
order.id
transaction.id
amount
currency
status
```

This makes **refund operations very easy to trigger later**.

---

If you want, I can also show you the **three refund architectures used in real payment systems**:

1️⃣ **Direct refund (MPGS API)**
2️⃣ **Back-office refund (merchant portal)**
3️⃣ **Automated refund service (microservice)**

The **third one is what large fintech systems use.**
