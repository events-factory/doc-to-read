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
