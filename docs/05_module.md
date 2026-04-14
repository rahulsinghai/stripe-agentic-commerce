# Module 5: Integration & Testing

Now it's time to bring everything together! In this module, you'll implement webhooks to close the payment loop, run all services, and test the complete flow.

You'll learn:

- Implementing webhooks for stock management
- Running all services together
- Testing end-to-end purchase flows
- Using the frontend interface
- Debugging common issues

```
┌──────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│   Frontend   │────►│   Agent Service      │────►│ Merchant Service│
│   :3000      │◄────│   :3001              │◄────│ :4000           │
└──────────────┘     └──────────┬───────────┘     └────────▲────────┘
                                │                          │
                      ┌─────────┴─────────┐                │ Webhooks
                      │                   │                │
               ┌──────▼──────┐    ┌───────▼───────┐ ┌──────┴────────┐
               │ AI Service  │    │ Agent Stripe  │ │Merchant Stripe│
               │ (Lambda)    │    │ (Cards, SPT)  │ │  (Payments)   │
               └─────────────┘    └───────────────┘ └───────────────┘
```

The key addition in this module: Merchant's Stripe account sends webhooks to the Merchant Service to confirm payment succeeded, which then decrements stock.

## Module Objectives

By the end of this module, you'll have:

- ✅ Webhooks handling payment confirmations and stock updates
- ✅ All services running together
- ✅ Completed an end-to-end purchase
- ✅ Tested with the provided frontend
- ✅ Debugged and verified the integration

## Handling Edge Cases: Stock & Price Changes

One of ACP's key strengths is handling real-world edge cases that occur between when a customer adds items to their cart and when they complete payment. Let's explore this hands-on!

### The Real-World Problem

Consider this scenario:

1. Customer adds "Salomon QST 98" skis to cart (5 in stock)
2. Customer fills in shipping details
3. Customer adds payment method
4. Meanwhile, another customer buys all 5 pairs
5. Original customer clicks "Pay Now"

**What should happen?** The payment should fail gracefully with a clear message — not charge the customer for skis that can't be shipped!

### Try It Yourself: The "Sold Out" Scenario

Let's simulate this exact scenario:

#### Step 1: Start a Purchase

1. Open the chat interface
2. Ask to buy a product: "I want to buy the Salomon QST 98 skis"
3. Complete all the profile steps (address, shipping, payment method)
4. Get to the point where you can click "Pay Now" — but DON'T click it yet!

#### Step 2: Sell Out the Stock

With your checkout ready but not completed:

1. Click the 🏪 Merchant button in the header to open the Merchant Admin
2. Find the "Salomon QST 98" in the inventory list
3. Click the − button repeatedly until stock reaches 0
4. Watch the stock count update in real-time

#### Step 3: Try to Complete the Purchase

Now go back to the chat and click "Pay Now" (or say "complete my order").

**What happens?**

```
❌ Unable to complete purchase: Insufficient stock for Salomon QST 98
```

The payment was never attempted because ACP validates stock at the final moment!

### How ACP Handles This

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Agent     │     │    Merchant     │     │ Merchant Stripe │
│  (AI Chat)  │     │   (Your Store)  │     │    (Payment)    │
└──────┬──────┘     └────────┬────────┘     └────────┬────────┘
       │                     │                       │
       │ POST /complete      │                       │
       │ (with SPT token)    │                       │
       │────────────────────►│                       │
       │                     │                       │
       │                     │ ✓ Validate checkout   │
       │                     │ ✓ Check stock AGAIN   │
       │                     │ ✗ Stock = 0!          │
       │                     │                       │
       │    400 Error        │                       │
       │◄────────────────────│                       │
       │  "Insufficient      │     (Never called!)   │
       │   stock"            │                       │
```

**Key insight**: The Merchant's `/complete` endpoint performs a final validation before processing payment:

```js
// In POST /checkouts/:id/complete
for (const lineItem of checkout.line_items) {
  const product = products.find(p => p.id === lineItem.id);

  // Final stock check - right before charging!
  if (!product || product.stock < lineItem.item.quantity) {
    return res.status(400).json({
      type: 'checkout_error',
      code: 'insufficient_stock',
      message: `Insufficient stock for: ${lineItem.title}`
    });
  }
}

// Only AFTER validation passes do we charge
const paymentIntent = await stripe.paymentIntents.create({...});
```

### Why This Matters

| Without ACP | With ACP |
| --- | --- |
| Customer charged, then refund needed | Payment never attempted |
| Customer frustration | Clear error message |
| Inventory mismatch | Always accurate |
| Manual reconciliation | Automatic protection |

### Other Edge Cases ACP Handles

**Price Changes**

If a product's price changes between cart creation and checkout:

- The Merchant recalculates totals at `/complete`
- The SPT amount is validated against the new total
- Mismatch? Transaction rejected before payment

**Product Removed**

If a product is discontinued:

- `/complete` checks product exists
- Returns clear error: "Product no longer available"

**Fulfillment Option Changed**

If shipping costs change:

- Totals recalculated with new shipping
- Customer sees updated total before confirming

> **Note**: **Production Best Practice**: In a real store, you might also implement "soft holds" — temporarily reserving stock when a checkout is created, releasing it if the checkout expires without completion.

### What You've Learned

- ✅ ACP validates at every step, not just at cart creation
- ✅ The Merchant has final authority on whether a checkout can complete
- ✅ Payments are never attempted for invalid checkouts
- ✅ Customers get clear error messages instead of surprise refunds

## Implementing Webhooks

### Why Webhooks for Stock Management?

When a customer completes a purchase, when should we decrement stock?

| Option | Problem |
| --- | --- |
| When checkout is created | Customer might abandon cart — stock incorrectly reserved |
| When `/complete` is called | Network could fail after payment succeeds — stock not decremented |
| **When webhook confirms payment** ✅ | Stock only decremented when Stripe confirms money was received |

Webhooks are the safest place to decrement stock because they're the source of truth — Stripe directly confirming the payment succeeded.

### The Problem We're Solving

Imagine this scenario:

1. Customer clicks "Pay Now"
2. Payment goes through on Stripe's side
3. But your server crashes before recording the sale
4. Customer paid, but your stock count is wrong!

With webhooks, Stripe will keep retrying until your server confirms receipt. Stock is only decremented when you definitively know payment succeeded.

### The Plan

We'll implement a webhook endpoint on the Merchant Service that:

1. Receives `payment_intent.succeeded` events from Stripe
2. Verifies the signature (security!)
3. Finds the matching checkout
4. Decrements stock for each item purchased
5. Records the sale in the catalog

```
Merchant Stripe             Merchant Service
  │                              │
  │  POST /webhooks/stripe       │
  │  { payment_intent.succeeded} │
  │─────────────────────────────►│
  │                              │ 1. Verify signature
  │                              │ 2. Find checkout by payment_intent_id
  │                              │ 3. Decrement stock for each item
  │                              │ 4. Record sale in catalog
  │       { received: true }     │
  │◄─────────────────────────────│
```

> **Note**: The payment was processed on the Merchant's Stripe account (using the SPT), so webhooks are configured on and sent from the Merchant's Stripe dashboard.

> **Warning**: Stock is **ONLY** decremented by webhooks. The `/complete` endpoint does NOT decrement stock — it only charges the customer. The webhook is responsible for updating inventory. This is the production-safe pattern.

### Setting Up the Webhook Endpoint

Open `merchant-service/routes/webhooks.js` — you'll see a starter file with TODO placeholders.

#### Step 1: Verify the Webhook Signature

This is critical for security. Without verification, anyone could send fake events to your endpoint.

Find this section and replace the entire block:

```js
// TODO: Verify webhook signature
let event;
return res.status(501).json({
  error: 'TODO: Implement webhook signature verification',
  hint: 'Replace this block with stripe.webhooks.constructEvent()'
});
```

Replace it with:

```js
// Verify webhook signature
let event;
try {
  const sig = req.headers['stripe-signature'];
  event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
} catch (err) {
  console.error('⚠️ Webhook signature verification failed:', err.message);
  return res.status(400).send(`Webhook Error: ${err.message}`);
}
```

Stripe signs each webhook with a secret key unique to your endpoint. `constructEvent()` uses this secret to verify the signature matches the payload. If someone sends a fake event, the signature won't match and the request is rejected.

#### Step 2: Handle payment_intent.succeeded

Find this TODO:

```js
// TODO: Handle payment_intent.succeeded event
// - Find the checkout by payment_intent_id
// - Mark it as webhook-confirmed
// - Call the catalog stock endpoint to decrement stock
// - Log the confirmation
```

Replace it with:

```js
if (event.type === 'payment_intent.succeeded') {
  const paymentIntent = event.data.object;
  console.log('✅ Webhook: Payment confirmed:', paymentIntent.id);

  // Find checkout with this payment_intent_id
  for (const [checkoutId, checkout] of checkouts) {
    if (checkout.payment_intent_id === paymentIntent.id) {
      // Mark as webhook-confirmed
      checkout.webhook_confirmed = true;
      checkout.webhook_confirmed_at = new Date().toISOString();

      // Record sales via the catalog API
      // Stock is ONLY decremented when webhook confirms payment!
      if (!checkout.stock_reserved) {
        console.log('📦 Webhook: Recording sales via Catalog API');
        const merchantBaseUrl = `http://localhost:${process.env.PORT || 4000}`;
        const catalogName = checkout.catalog;

        for (const lineItem of checkout.line_items) {
          try {
            const response = await fetch(`${merchantBaseUrl}/api/${catalogName}/sale`, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({
                productId: lineItem.id,
                quantity: lineItem.item.quantity,
                orderId: checkout.order?.id,
              }),
            });
            const result = await response.json();
            if (result.success) {
              console.log(`   ✅ Sale recorded: ${lineItem.item.quantity}x ${lineItem.title}`);
            }
          } catch (err) {
            console.error(`   ❌ Sale recording failed:`, err.message);
          }
        }
        checkout.stock_reserved = true;
      }

      console.log('🎉 Webhook: Order confirmed for fulfillment:', checkoutId);
      break;
    }
  }
}
```

Stock is only decremented after Stripe definitively confirms payment succeeded. This handles edge cases like 3D Secure authentication delays or network failures during the sync `/complete` call.

#### Step 3: Handle payment_intent.payment_failed (Optional)

Find the optional TODO and replace it with:

```js
if (event.type === 'payment_intent.payment_failed') {
  const paymentIntent = event.data.object;
  console.log('❌ Webhook: Payment failed:', paymentIntent.id);
  console.log('   Reason:', paymentIntent.last_payment_error?.message || 'Unknown');

  // You could notify the Agent or update checkout status here
}
```

### Setting Up Your Webhook Secret

To test webhooks locally, you'll use the Stripe CLI. The CLI gives you a webhook signing secret.

#### Step 1: Install Stripe CLI (if not already)

```bash
brew install stripe/stripe-cli/stripe
```

#### Step 2: Login to Stripe

```bash
stripe login
```

Follow the prompts to authenticate with your Stripe account.

#### Step 3: Forward Webhooks to Your Local Server

In a new terminal, run:

```bash
stripe listen --forward-to localhost:4000/webhooks/stripe
```

You'll see output like:

```
> Ready! Your webhook signing secret is whsec_abc123...
```

#### Step 4: Add the Secret to Your Environment

Copy the `whsec_...` value and add it to `merchant-service/.env`:

```bash
STRIPE_WEBHOOK_SECRET=whsec_abc123...
```

#### Step 5: Restart Your Services

```bash
./dev.sh
```

> **Note**: Keep the `stripe listen` terminal running in a separate window alongside your services terminal.

### Testing Webhooks

With `stripe listen` running, complete a purchase through the chat. Watch both terminals:

**Stripe CLI terminal:**

```
2024-01-15 10:30:45   --> payment_intent.created [evt_...]
2024-01-15 10:30:46   --> payment_intent.succeeded [evt_...]
2024-01-15 10:30:46  <--  [200] POST http://localhost:4000/webhooks/stripe
```

**Merchant Service terminal:**

```
✅ Webhook: Payment confirmed: pi_3abc123...
📦 Webhook: Order confirmed for fulfillment: checkout_xyz789
Amount: 823.90 USD
```

You can also trigger test events directly with the Stripe CLI:

```bash
stripe trigger payment_intent.succeeded
```

### Seeing Webhook Confirmation in the Admin Panel

Open the Merchant Admin Panel and go to the **Sales** tab. After a webhook is received, you'll see orders marked as confirmed!

> **Note**: In a production system, this webhook confirmation would trigger: shipping label generation, inventory update in ERP, confirmation email to customer, and commission calculation for affiliates.

### What You've Learned

- ✅ Why webhooks are essential for production
- ✅ How to verify webhook signatures
- ✅ Handling `payment_intent.succeeded` events
- ✅ Decrementing stock only when payment is confirmed
- ✅ Using Stripe CLI for local webhook testing

Webhooks close the loop — instead of hoping your sync call worked, Stripe confirms it directly.

## Testing with Different Card Types

Now that you have webhooks working, let's explore what happens when payments fail — and how Stripe protects you from fraud.

### Using the Stripe Test Helper

The frontend includes a powerful testing tool: the Stripe Test Helper. This button appears when you're adding a payment method and lets you simulate different card scenarios.

**Finding the Test Helper**

1. Click the 👤 Profile button in the chat header
2. Go to the 💳 Payment tab
3. Click **Add Payment Method**
4. Look for the **STRIPE** button in the Payment Element

The Stripe button opens a test card selector with various scenarios.

### Test Card Scenarios

**Successful Payments**

| Card Number | Description |
| --- | --- |
| `4242 4242 4242 4242` | Standard successful payment |
| `4000 0566 5566 5556` | Visa (debit) |
| `5555 5555 5555 4444` | Mastercard |

**Declined Cards**

| Card Number | Description |
| --- | --- |
| `4000 0000 0000 0002` | Generic decline |
| `4000 0000 0000 9995` | Insufficient funds |
| `4000 0000 0000 0069` | Expired card |
| `4000 0000 0000 0127` | Incorrect CVC |

**Fraud Testing**

| Card Number | Description |
| --- | --- |
| `4100 0000 0000 0019` | 🚨 Blocked as fraudulent — Radar blocks this |
| `4000 0000 0000 0259` | Dispute (chargeback) after success |

> **Note**: The card `4100 0000 0000 0019` is special — Stripe Radar automatically flags it as high-risk and blocks the payment. This simulates what happens when a real fraudster tries to use a stolen card.

### Exercise: Test a Fraudulent Card

**Step 1: Add the Fraudulent Card**

1. Open the Profile → Payment tab
2. Click **Add Payment Method**
3. Click the **STRIPE** test helper button
4. Select "Fraudulent card" or manually enter: `4100 0000 0000 0019`
5. Use any future expiry date and any CVC
6. Click **Save Payment Method**

The card will be saved (Stripe doesn't block at save time — only at charge time).

**Step 2: Attempt a Purchase**

1. Go back to the chat
2. Ask to buy something: "I want to buy the Salomon QST 98"
3. Complete the checkout flow
4. Watch what happens when payment is attempted...

**Step 3: Observe the Failure**

You should see the AI report the payment was declined due to suspected fraud. The checkout doesn't complete and no webhook is received (payment never succeeded).

Check your terminal logs for the Merchant Service:

```
💳 Processing payment for checkout: checkout_xxx
❌ Payment failed: Your card was declined
Decline code: fraudulent
```

### Viewing in Stripe Dashboard

**Step 1: Open Stripe Dashboard**

Go to [dashboard.stripe.com/test/payments](https://dashboard.stripe.com/test/payments).

**Step 2: Find the Failed Payment**

Look for a payment with status **Failed** (red), matching amount, and recent timestamp. Click on it to see details.

**Step 3: Review the Decline Reason**

In the payment details, you'll see:

- **Outcome**: Blocked by Radar
- **Risk evaluation**: High risk
- **Decline reason**: fraudulent

This is Stripe Radar in action — it analyzed the card and blocked it before any money moved.

> **Note**: This test card triggers Stripe's fraud detection system. In production, Radar uses machine learning trained on billions of transactions to identify real fraud patterns like unusual purchase velocity, mismatched billing/shipping addresses, known fraudulent card numbers, suspicious device fingerprints, and hundreds of other signals.

### Other Decline Scenarios

**Insufficient Funds**

```
Card: 4000 0000 0000 9995
Result: Payment declined - insufficient funds
```

The AI should tell the user their card was declined and suggest trying another card.

**Expired Card**

```
Card: 4000 0000 0000 0069
Result: Payment declined - expired card
```

**3D Secure Required**

```
Card: 4000 0000 0000 3220
Result: Requires authentication (3DS popup)
```

This simulates Strong Customer Authentication (SCA) required in Europe.

### What You've Learned

- ✅ How to use the Stripe test helper for different card scenarios
- ✅ What happens when a fraudulent card is used
- ✅ How to view payment failures in Stripe Dashboard
- ✅ How Stripe Radar blocks suspicious transactions

## Understanding Stripe Radar

Stripe Radar is built into every Stripe account — it's your first line of defense against fraud. Let's explore how it works and what you can see in the Dashboard.

### What is Stripe Radar?

Radar is Stripe's machine learning-powered fraud prevention system. It analyzes every payment attempt using:

- **Card signals** — Is this card known to be fraudulent?
- **Behavioral patterns** — Is this purchase velocity normal?
- **Device fingerprinting** — Has this device been used for fraud before?
- **Network intelligence** — Data from millions of businesses using Stripe
- **Custom rules** — Your own business-specific fraud rules

```
Payment Attempt
      │
      ▼
┌─────────────────────────────────────────────┐
│              STRIPE RADAR                   │
│                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │  Card   │  │Behavior │  │ Device  │      │
│  │ Signals │  │Patterns │  │  Data   │      │
│  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │            │            │           │
│       └────────────┼────────────┘           │
│                    ▼                        │
│            ┌───────────────┐                │
│            │  Risk Score   │                │
│            │   0 - 100     │                │
│            └───────┬───────┘                │
│                    │                        │
│     ┌──────────────┼──────────────┐         │
│     ▼              ▼              ▼         │
│  ┌─────┐      ┌─────────┐    ┌────────┐     │
│  │BLOCK│      │ REVIEW  │    │ ALLOW  │     │
│  └─────┘      └─────────┘    └────────┘     │
└─────────────────────────────────────────────┘
```

### Viewing Radar in the Dashboard

1. Go to [dashboard.stripe.com](https://dashboard.stripe.com/)
2. Make sure you're in **Test mode** (toggle in the top right)
3. Click **Radar** in the left sidebar (under "More")

The Radar dashboard shows blocked payments, payments flagged for manual review, and risk distribution across your payments.

> **Note**: In test mode, you won't see real risk scores or ML predictions — the dashboard shows test data. In live mode, you'd see actual fraud patterns and scores.

### How Risk Scores Work

Every payment gets a risk score from 0–100:

| Score Range | Risk Level | Default Action |
| --- | --- | --- |
| 0–20 | Low | Allow |
| 21–65 | Elevated | Allow (monitor) |
| 66–75 | High | Review |
| 76–100 | Highest | Block |

These thresholds are customizable with Radar for Fraud Teams (paid add-on).

### Viewing a Blocked Payment

1. Go to **Payments** in the Dashboard
2. Find the failed payment from the fraudulent card
3. Click to open details

You'll see:

```
Payment Details
───────────────────────────────────
Status:         Failed
Amount:         $XXX.XX
Decline reason: fraudulent

Risk Insights
───────────────────────────────────
Outcome:        Blocked by Radar rule
Risk level:     Highest (99)
Risk factors:
  • Card reported as fraudulent
  • Card testing pattern detected
```

### Radar Rules

Radar comes with default rules, and you can add custom ones:

**Default Rules (Always Active)**

- Block if CVC check fails
- Block if card is on fraud lists
- Block payments from sanctioned countries

**Custom Rules (Examples)**

```
# Block large orders from new customers
BLOCK IF :amount_in_usd: > 500 AND :is_new_customer:

# Review international orders over $200
REVIEW IF :amount_in_usd: > 200 AND :card_country: != 'US'

# Allow known good customers
ALLOW IF :customer_email: ENDS_WITH '@trustedcompany.com'
```

> **Note**: Custom rules require the paid "Radar for Fraud Teams" add-on. The basic Radar is free and included with every Stripe account.

### Fraud Signals in Webhooks

When a payment fails due to fraud, the webhook includes detailed information:

```json
{
  "type": "payment_intent.payment_failed",
  "data": {
    "object": {
      "id": "pi_xxx",
      "status": "requires_payment_method",
      "last_payment_error": {
        "code": "card_declined",
        "decline_code": "fraudulent",
        "message": "Your card was declined.",
        "outcome": {
          "network_status": "declined_by_network",
          "reason": "highest_risk_level",
          "risk_level": "highest",
          "risk_score": 99,
          "seller_message": "Stripe blocked this payment as high risk.",
          "type": "blocked"
        }
      }
    }
  }
}
```

Your application can use this information to show appropriate error messages, log fraud attempts for analysis, and trigger additional verification steps.

### Best Practices for Fraud Prevention

1. **Collect Full Billing Information** — More data = better fraud detection. Always collect full billing address, CVC, and cardholder name.
2. **Use 3D Secure** — Enable 3D Secure for high-risk transactions. It shifts liability to the card issuer.
3. **Monitor Your Radar Dashboard** — Check regularly for unusual spikes in blocked payments, patterns in fraud attempts, and false positives (legitimate customers blocked).
4. **Review Before Fulfilling** — For high-value orders, manually verify before shipping.

### What You've Learned

- ✅ How Stripe Radar evaluates payment risk
- ✅ Where to find Radar data in the Dashboard
- ✅ How risk scores determine payment outcomes
- ✅ What information is available when fraud is detected
- ✅ Best practices for fraud prevention

Radar runs automatically on every payment — no code required. It's one of the key benefits of using Stripe for payments.

## Module 5 Review

### What You've Accomplished

You've successfully integrated all the services and completed end-to-end testing!

**Services Running**

- ✅ Merchant Service (port 4000)
- ✅ Agent Service (port 3001)
- ✅ Frontend (port 3000)

**Flows Tested**

- ✅ Product discovery via chat
- ✅ Checkout creation
- ✅ Address and shipping updates
- ✅ Payment with SPT
- ✅ Order completion

**Tools Used**

- ✅ Frontend chat interface
- ✅ ACP Inspector
- ✅ curl for API testing
- ✅ Stripe Dashboard

### Complete Flow Summary

```
1. User: "I want skis"
   └─► AI suggests products

2. User: "I'll take the Rustler 10"
   └─► Agent creates checkout via Merchant ACP

3. User: "Ship to 123 Main St, SF"
   └─► Agent updates checkout with address

4. User: "Complete my order"
   └─► Agent creates SPT
   └─► Agent sends SPT to Merchant
   └─► Merchant charges card via Stripe
   └─► Order confirmed!
```

### Key Integration Points

| From | To | Protocol |
| --- | --- | --- |
| Frontend | Agent | REST API |
| Agent | AI Service | REST (Lambda) |
| Agent | Merchant | ACP REST |
| Agent | Stripe | Stripe API (SPT) |
| Merchant | Stripe | Stripe API (PaymentIntent) |

### Knowledge Check

**Q1: What order should services be started?**
Merchant → Agent → Frontend. The Agent depends on Merchant being available.

**Q2: What happens if the Merchant service restarts mid-checkout?**
The in-memory checkout store is cleared, so existing checkouts are lost. The user would need to start over. In production, use a database.

**Q3: How can you verify a payment succeeded?**
Check the Stripe Dashboard for a `succeeded` PaymentIntent, or look for the `payment_intent_id` in the completed checkout response.

**Q4: What does the ACP Inspector show?**
All HTTP calls between Agent and Merchant (create, update, complete checkout), plus Stripe API calls and SPT token creation.
