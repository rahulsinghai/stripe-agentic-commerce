# Module 4: Building the Merchant Service

The Merchant Service receives checkout requests from the Agent and processes payments. In this module, you'll implement the ACP checkout endpoints that make this possible.

## What's Already Set Up

The starter kit includes a working Merchant Service with:

- ✅ Express server (`server.js`)
- ✅ Product catalog API (`/api/products`)
- ✅ Product store with inventory management
- ✅ Checkout router with stubbed endpoints (you'll implement these)

## What You'll Build

You'll implement these ACP endpoints in `merchant-service/routes/checkouts.js`:

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/checkouts` | POST | Create checkout session |
| `/checkouts/:id` | GET | Retrieve checkout |
| `/checkouts/:id` | PUT | Update checkout |
| `/checkouts/:id/complete` | POST | Complete with payment |
| `/checkouts/:id/cancel` | POST | Cancel checkout |

## Checkout Flow

```
Agent                         Merchant
  │                              │
  │ POST /checkouts              │
  │─────────────────────────────►│ Create session
  │       checkout_xxx           │
  │◄─────────────────────────────│
  │                              │
  │ PUT /checkouts/:id           │
  │─────────────────────────────►│ Add address
  │       updated                │
  │◄─────────────────────────────│
  │                              │
  │ POST /checkouts/:id/complete │
  │─────────────────────────────►│ Process SPT
  │       order confirmed        │
  │◄─────────────────────────────│
```

## Module Objectives

By the end of this module, you'll have:

- ✅ ACP checkout endpoints implemented
- ✅ Checkout state management working
- ✅ SPT payment processing integrated with Stripe
- ✅ Error handling for edge cases

> **Note**: This module will take approximately 45 minutes to complete.

## ACP Checkout Endpoints

### The Checkout Conversation

When a customer wants to buy something, the Agent orchestrates a multi-step checkout flow by calling endpoints on the Merchant's server. The Merchant owns the products, pricing, and checkout logic — the Agent just facilitates the conversation and handles payment credentials.

Here's what a typical purchase looks like:

```
Customer: "I'd like to buy the Blizzard Rustler 10"
    │
    │  Agent calls POST /checkouts with the item
    ▼
Agent: "Great choice! That's $749. Where should we ship it?"
    │
    │  Customer provides address
    │  Agent calls PUT /checkouts/:id with address
    ▼
Agent: "Got it! Choose your shipping speed:"
       • Standard (5-7 days) - $4.99
       • Express (2-3 days) - $9.99
    │
    │  Customer picks shipping
    │  Agent calls PUT /checkouts/:id with shipping option
    ▼
Agent: "Your total is $823.90. Ready to complete your purchase?"
    │
    │  Customer confirms
    │  Agent generates SPT from customer's saved payment method
    │  Agent calls POST /checkouts/:id/complete with SPT
    ▼
Agent: "Order confirmed! Your skis will arrive in 5-7 days."
```

### The Merchant Endpoints

These are the endpoints you'll build on the Merchant Service. The Agent calls them to create and manage checkouts:

| Step | Endpoint | What Happens |
| --- | --- | --- |
| Customer picks a product | `POST /checkouts` | Creates checkout with items, returns shipping options |
| Customer provides address | `PUT /checkouts/:id` | Adds shipping address to checkout |
| Customer picks shipping | `PUT /checkouts/:id` | Sets shipping option, checkout becomes "ready" |
| Customer confirms | Agent creates SPT | Agent generates a Shared Payment Token from customer's saved card |
| Agent completes purchase | `POST /checkouts/:id/complete` | Merchant processes SPT payment, marks order complete |
| Customer changes mind | `POST /checkouts/:id/cancel` | Cancels the checkout |

> **Note**: The Agent also uses `GET /checkouts/:id` to check the current state before each step.

### Checkout States

```
                ┌───────────────────────┐
                │ not_ready_for_payment │
                └───────────┬───────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │ (add address &    │                   │
        │  shipping option) │                   │
        ▼                   │                   │
┌───────────────────┐       │           ┌───────────────┐
│ ready_for_payment │       │           │   canceled    │
└─────────┬─────────┘       │           └───────────────┘
          │                 │
          │ (complete       │
          │  with SPT)      │
          ▼                 │
    ┌───────────┐           │
    │ completed │◄──────────┘
    └───────────┘
```

### State Transitions

| Current State | Action | New State |
| --- | --- | --- |
| `not_ready_for_payment` | Add address + shipping | `ready_for_payment` |
| `not_ready_for_payment` | Cancel | `canceled` |
| `ready_for_payment` | Complete with SPT | `completed` |
| `ready_for_payment` | Cancel | `canceled` |

### The Starter File

Open `merchant-service/routes/checkouts.js` — you'll find a starter file with:

**✅ Working infrastructure:**

- Express router setup
- In-memory checkout store (`Map`)
- Helper functions (`calculateLineItems`, `calculateTotals`, `formatCheckoutResponse`)
- Fulfillment options defined

**⏳ Stubbed endpoints (you'll implement these):**

```js
POST /          // Returns "TODO: Implement"
GET /:id        // Returns "TODO: Implement"
PUT /:id        // Returns "TODO: Implement"
POST /:id/complete  // Returns "TODO: Implement"
POST /:id/cancel    // Returns "TODO: Implement"
```

### Key Components Already Provided

**In-Memory Checkout Store**

```js
const checkouts = new Map();
```

Stores active checkout sessions. In production, this would be a database — but a `Map()` keeps things simple for the workshop.

**Unique ID Generator**

```js
const generateId = () => `checkout_${crypto.randomBytes(12).toString('hex')}`;
```

Creates IDs like `checkout_a1b2c3d4e5f6...` for each session.

**Helper Functions**

```js
calculateLineItems(items)    // Builds line items from cart
calculateTotals(lineItems)   // Calculates subtotal, tax, shipping, total
determineStatus(checkout)    // Returns current checkout status
formatCheckoutResponse()     // Formats checkout for API response
```

You'll use these helpers in your endpoint implementations!

## Create Checkout Session

### Implementing POST /checkouts

Open `merchant-service/routes/checkouts.js` and find the stubbed `POST /` route. Replace the entire route handler with:

```js
router.post('/', (req, res) => {
  try {
    const { items, buyer, fulfillment_address, catalog } = req.body;

    // Validate items
    if (!items || !Array.isArray(items) || items.length === 0) {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'missing_items',
        message: 'Items array is required and must not be empty'
      });
    }

    // Validate each item exists and has stock - use the specified catalog
    const products = getProducts(catalog);
    for (const item of items) {
      if (!item.id || typeof item.quantity !== 'number' || item.quantity < 1) {
        return res.status(400).json({
          type: 'invalid_request',
          code: 'invalid_item',
          message: 'Each item must have an id and positive quantity'
        });
      }

      const product = products.find(p => p.id === item.id);
      if (!product) {
        return res.status(400).json({
          type: 'invalid_request',
          code: 'product_not_found',
          message: `Product not found: ${item.id}`
        });
      }

      if (!product.inStock || product.stock < item.quantity) {
        return res.status(400).json({
          type: 'invalid_request',
          code: 'insufficient_stock',
          message: `Insufficient stock for: ${product.title}`
        });
      }
    }

    // Create the checkout object
    const lineItems = calculateLineItems(items, catalog);
    const checkout = {
      id: generateId(),
      currency: 'usd',
      line_items: lineItems,
      catalog: catalog,
      payment_provider: {
        provider: 'stripe',
        supported_payment_methods: ['card']
      },
      messages: [],
      links: [
        { type: 'terms_of_use', url: 'https://example.com/terms' },
        { type: 'privacy_policy', url: 'https://example.com/privacy' }
      ],
      created_at: new Date().toISOString()
    };

    if (buyer) checkout.buyer = buyer;
    if (fulfillment_address) checkout.fulfillment_address = fulfillment_address;

    // Store in our in-memory Map
    checkouts.set(checkout.id, checkout);

    console.log('🛒 Checkout created:', checkout.id);
    res.status(201).json(formatCheckoutResponse(checkout));

  } catch (error) {
    console.error('Create checkout error:', error);
    res.status(500).json({
      type: 'processing_error',
      code: 'internal_error',
      message: 'An error occurred while creating the checkout'
    });
  }
});
```

> **Note**: The `catalog` parameter tells the Merchant which product catalog to use (e.g., coffee, vinyl, skis). This allows the same checkout system to work with any product catalog!

### What This Code Does

1. **Validates the request** — ensures items array exists and each item has valid data
2. **Checks stock availability** — prevents overselling by verifying products exist and are in stock
3. **Creates the checkout object** — uses `calculateLineItems()` helper to build line items with prices
4. **Stores in memory** — saves to the `checkouts` Map (a database in production)
5. **Returns formatted response** — uses `formatCheckoutResponse()` helper for consistent output

### Test Your Implementation

Run this command in your terminal:

```bash
curl -X POST http://localhost:4000/checkouts \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "id": "SKI-001", "quantity": 1 }
    ],
    "buyer": {
      "email": "customer@example.com",
      "name": "Jane Doe"
    }
  }'
```

The response includes a checkout ID used for all subsequent operations. Notice the status is `not_ready_for_payment` because we haven't added a shipping address yet!

## Update Checkout Session

### Implementing GET /checkouts/:id

Find the stubbed GET route and replace it with:

```js
router.get('/:id', (req, res) => {
  try {
    const { id } = req.params;
    const checkout = checkouts.get(id);

    if (!checkout) {
      return res.status(404).json({
        type: 'invalid_request',
        code: 'checkout_not_found',
        message: `Checkout with id '${id}' not found`
      });
    }

    res.json(formatCheckoutResponse(checkout));

  } catch (error) {
    console.error('Get checkout error:', error);
    res.status(500).json({
      type: 'processing_error',
      code: 'internal_error',
      message: 'An error occurred while retrieving the checkout'
    });
  }
});
```

### Implementing PUT /checkouts/:id

Find the stubbed PUT route and replace it with:

```js
router.put('/:id', (req, res) => {
  try {
    const { id } = req.params;
    const { items, buyer, fulfillment_address, fulfillment_option_id } = req.body;

    const checkout = checkouts.get(id);

    if (!checkout) {
      return res.status(404).json({
        type: 'invalid_request',
        code: 'checkout_not_found',
        message: `Checkout with id '${id}' not found`
      });
    }

    // Can't modify completed/canceled checkouts
    if (checkout.status === 'completed') {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'checkout_completed',
        message: 'Cannot modify a completed checkout'
      });
    }

    if (checkout.status === 'canceled') {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'checkout_canceled',
        message: 'Cannot modify a canceled checkout'
      });
    }

    // Update items if provided
    if (items && Array.isArray(items)) {
      const products = getProducts();

      for (const item of items) {
        if (!item.id || typeof item.quantity !== 'number' || item.quantity < 1) {
          return res.status(400).json({
            type: 'invalid_request',
            code: 'invalid_item',
            message: 'Each item must have an id and positive quantity'
          });
        }

        if (!products.find(p => p.id === item.id)) {
          return res.status(400).json({
            type: 'invalid_request',
            code: 'product_not_found',
            message: `Product not found: ${item.id}`
          });
        }
      }

      checkout.line_items = calculateLineItems(items);
    }

    // Update buyer, address, and shipping option
    if (buyer) checkout.buyer = { ...checkout.buyer, ...buyer };
    if (fulfillment_address) checkout.fulfillment_address = { ...checkout.fulfillment_address, ...fulfillment_address };

    if (fulfillment_option_id) {
      const validOption = defaultFulfillmentOptions.find(fo => fo.id === fulfillment_option_id);
      if (!validOption) {
        return res.status(400).json({
          type: 'invalid_request',
          code: 'invalid_fulfillment_option',
          message: `Invalid fulfillment option: ${fulfillment_option_id}`
        });
      }
      checkout.fulfillment_option_id = fulfillment_option_id;
    }

    checkout.updated_at = new Date().toISOString();
    checkouts.set(id, checkout);

    console.log('✏️ Checkout updated:', id, '- Status:', determineStatus(checkout));
    res.json(formatCheckoutResponse(checkout));

  } catch (error) {
    console.error('Update checkout error:', error);
    res.status(500).json({
      type: 'processing_error',
      code: 'internal_error',
      message: 'An error occurred while updating the checkout'
    });
  }
});
```

### Status Transitions

| Condition | Status |
| --- | --- |
| Just created, missing info | `not_ready_for_payment` |
| Has items + address + shipping | `ready_for_payment` |
| Payment processed | `completed` |
| User cancelled | `canceled` |

### Test the Update Endpoint

Using the checkout ID from your previous test, run this command to add a shipping address:

```bash
curl -X PUT http://localhost:4000/checkouts/checkout_f5c42d1ddb7ea1b1240bb4ef \
  -H "Content-Type: application/json" \
  -d '{
    "fulfillment_address": {
      "line_one": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94105",
      "country_code": "US"
    },
    "fulfillment_option_id": "shipping_standard"
  }'
```

After this update, the status changes to `ready_for_payment` — the checkout is now ready for the Agent to complete with an SPT!

## Complete Checkout with SPT

### Implementing POST /checkouts/:id/complete

This is the most critical endpoint — it processes the payment using the SPT token from the Agent.

Find the stubbed `POST /:id/complete` route and replace it with:

```js
router.post('/:id/complete', async (req, res) => {
  try {
    const { id } = req.params;
    const { payment_data, buyer } = req.body;

    const checkout = checkouts.get(id);

    if (!checkout) {
      return res.status(404).json({
        type: 'invalid_request',
        code: 'checkout_not_found',
        message: `Checkout with id '${id}' not found`
      });
    }

    if (checkout.status === 'completed') {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'checkout_already_completed',
        message: 'Checkout has already been completed'
      });
    }

    if (checkout.status === 'canceled') {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'checkout_canceled',
        message: 'Cannot complete a canceled checkout'
      });
    }

    // Validate payment data
    if (!payment_data || !payment_data.token) {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'missing_payment_token',
        message: 'Payment token is required'
      });
    }

    // Validate SPT format
    if (!payment_data.token.startsWith('spt_')) {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'invalid_token',
        message: 'Invalid SPT token format. Token must start with spt_'
      });
    }

    if (buyer) checkout.buyer = { ...checkout.buyer, ...buyer };

    // FINAL STOCK CHECK - Right before payment!
    const products = getProducts(checkout.catalog);
    for (const lineItem of checkout.line_items) {
      const product = products.find(p => p.id === lineItem.id);

      if (!product) {
        return res.status(400).json({
          type: 'checkout_error',
          code: 'product_not_found',
          message: `Product no longer available: ${lineItem.title}`
        });
      }

      if (!product.inStock || product.stock < lineItem.item.quantity) {
        console.log(`❌ Stock check failed: ${product.title} has ${product.stock} but need ${lineItem.item.quantity}`);
        return res.status(400).json({
          type: 'checkout_error',
          code: 'insufficient_stock',
          message: `Insufficient stock for: ${lineItem.title}`
        });
      }
    }
    console.log('✅ Final stock check passed');

    console.log('💳 Processing payment for checkout:', id);
    console.log('   Token:', payment_data.token.substring(0, 30) + '...');

    // Calculate total amount
    const fulfillmentOption = checkout.fulfillment_option_id
      ? defaultFulfillmentOptions.find(fo => fo.id === checkout.fulfillment_option_id)
      : null;
    const totals = calculateTotals(checkout.line_items, fulfillmentOption);
    const totalAmount = totals.find(t => t.type === 'total')?.amount || 0;

    // Process payment with Stripe using the SPT
    const stripeSecretKey = process.env.STRIPE_SECRET_KEY;

    if (stripeSecretKey && payment_data.provider === 'stripe') {
      try {
        const params = new URLSearchParams({
          amount: totalAmount.toString(),
          currency: checkout.currency,
          shared_payment_granted_token: payment_data.token,
          'payment_method_types[0]': 'card',
          confirm: 'true'
        });

        const response = await fetch('https://api.stripe.com/v1/payment_intents', {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${stripeSecretKey}`,
            'Content-Type': 'application/x-www-form-urlencoded'
          },
          body: params.toString()
        });

        const paymentIntent = await response.json();

        if (paymentIntent.error) {
          console.error('Payment error:', paymentIntent.error.message);
          checkout.messages.push({
            type: 'error',
            code: 'payment_declined',
            content: paymentIntent.error.message
          });
          checkouts.set(id, checkout);
          return res.status(400).json(formatCheckoutResponse(checkout));
        }

        if (paymentIntent.status !== 'succeeded') {
          checkout.messages.push({
            type: 'error',
            code: 'payment_failed',
            content: 'Payment could not be processed'
          });
          checkouts.set(id, checkout);
          return res.status(400).json(formatCheckoutResponse(checkout));
        }

        console.log('   ✅ Payment succeeded:', paymentIntent.id);
        checkout.payment_intent_id = paymentIntent.id;

      } catch (stripeError) {
        console.error('Stripe API error:', stripeError.message);
        return res.status(500).json({
          type: 'processing_error',
          code: 'payment_failed',
          message: 'Payment processing failed'
        });
      }
    } else {
      // Demo mode without Stripe key
      console.log('   ⚠️  Demo mode - simulating successful payment');
    }

    // NOTE: Stock is decremented via webhook, not here
    console.log('   ⏳ Stock will be reserved when webhook confirms payment');

    // Mark as completed
    checkout.status = 'completed';
    checkout.completed_at = new Date().toISOString();
    checkout.order = {
      id: `order_${crypto.randomBytes(12).toString('hex')}`,
      checkout_session_id: checkout.id,
      permalink_url: `https://example.com/orders/${checkout.id}`
    };

    checkout.messages.push({
      type: 'info',
      content: 'Order placed successfully! Thank you for your purchase.'
    });

    checkouts.set(id, checkout);

    console.log('🎉 Checkout completed:', id);
    res.json(formatCheckoutResponse(checkout));

  } catch (error) {
    console.error('Complete checkout error:', error);
    res.status(500).json({
      type: 'processing_error',
      code: 'internal_error',
      message: 'An error occurred while completing the checkout'
    });
  }
});
```

### What This Code Does

1. **Final stock check** — validates products are still available right before payment
2. **Validates the SPT** — ensures the token starts with `spt_` (Stripe's SPT format)
3. **Creates a PaymentIntent** — uses `shared_payment_granted_token` to charge the customer
4. **Marks checkout complete** — creates an order and stores confirmation

> **Warning**: **The Final Stock Check is Critical!** This catches edge cases where stock changed after the customer created their cart. Without it, you might charge customers for items you can't ship!

> **Note**: Stock is decremented via webhooks (covered in Module 5), not in this endpoint. This ensures inventory only changes when Stripe definitively confirms payment.

### The SPT Payment

The key line is:

```js
shared_payment_granted_token: payment_data.token,
```

This tells Stripe: "Use this SPT to charge the customer's card." The Merchant never sees the actual card details — they only receive a secure token.

The SPT is generated by the Agent from the customer's saved payment method. The Merchant's Stripe account receives the funds directly.

## Cancel Checkout Session

### Implementing POST /checkouts/:id/cancel

Find the stubbed `POST /:id/cancel` route and replace it with:

```js
router.post('/:id/cancel', (req, res) => {
  try {
    const { id } = req.params;
    const { reason } = req.body;

    const checkout = checkouts.get(id);

    if (!checkout) {
      return res.status(404).json({
        type: 'invalid_request',
        code: 'checkout_not_found',
        message: `Checkout with id '${id}' not found`
      });
    }

    if (checkout.status === 'completed') {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'checkout_completed',
        message: 'Cannot cancel a completed checkout'
      });
    }

    if (checkout.status === 'canceled') {
      return res.status(400).json({
        type: 'invalid_request',
        code: 'already_canceled',
        message: 'Checkout has already been canceled'
      });
    }

    checkout.status = 'canceled';
    checkout.canceled_at = new Date().toISOString();
    checkout.messages.push({
      type: 'info',
      content: reason ? `Checkout cancelled: ${reason}` : 'Checkout has been cancelled'
    });

    checkouts.set(id, checkout);

    console.log('❌ Checkout cancelled:', id);
    res.json(formatCheckoutResponse(checkout));

  } catch (error) {
    console.error('Cancel checkout error:', error);
    res.status(500).json({
      type: 'processing_error',
      code: 'internal_error',
      message: 'An error occurred while canceling the checkout'
    });
  }
});
```

### What This Code Does

1. **Validates the checkout** — ensures it exists and returns a 404 if not found
2. **Checks cancellation eligibility** — prevents canceling completed or already-canceled checkouts
3. **Updates the status** — sets status to `canceled` and records the timestamp
4. **Adds a message** — stores the cancellation reason (if provided) in the checkout messages
5. **Returns the updated checkout** — sends back the formatted response

## Testing the Merchant Service

### Verify Your Implementation

Confirm all five endpoints are implemented in `merchant-service/routes/checkouts.js`:

| Endpoint | Status |
| --- | --- |
| `POST /` (create) | ✅ Implemented |
| `GET /:id` (retrieve) | ✅ Implemented |
| `PUT /:id` (update) | ✅ Implemented |
| `POST /:id/complete` | ✅ Implemented |
| `POST /:id/cancel` | ✅ Implemented |

### Testing via the Frontend

The checkout endpoints work together as part of a flow orchestrated by the Agent. When you use the chat interface to make a purchase:

1. Agent calls `POST /checkouts` — when you say "I want to buy this"
2. Agent calls `PUT /checkouts/:id` — when you provide your address
3. Agent calls `PUT /checkouts/:id` — again when you select shipping
4. Agent generates an SPT — from your saved payment method
5. Agent calls `POST /checkouts/:id/complete` — with the SPT to finish

Watch all these calls happen in real-time using the **ACP Inspector**!

### What You've Built

Your Merchant Service now has a complete ACP checkout implementation:

```
┌─────────────────────────────────────────────────────────┐
│                    Merchant Service                     │
├─────────────────────────────────────────────────────────┤
│  POST /checkouts              → Create checkout session │
│  GET  /checkouts/:id          → Retrieve checkout state │
│  PUT  /checkouts/:id          → Update address & ship   │
│  POST /checkouts/:id/complete → Process SPT payment     │
│  POST /checkouts/:id/cancel   → Cancel checkout         │
└─────────────────────────────────────────────────────────┘
```

## Module 4 Review

Congratulations! You've implemented the complete Merchant Service with ACP checkout endpoints.

### ACP Endpoints

- `POST /checkouts` — Create checkout session
- `GET /checkouts/:id` — Retrieve checkout
- `PUT /checkouts/:id` — Update address/shipping
- `POST /checkouts/:id/complete` — Complete with SPT
- `POST /checkouts/:id/cancel` — Cancel checkout

### Key Features

- Checkout state management (`not_ready` → `ready` → `completed`)
- Line item calculations with tax
- Fulfillment options (standard, express shipping)
- SPT token validation
- Stock reservation on completion
- Error handling for all edge cases

### Code Summary

**Checkout Creation**

```js
const checkout = {
  id: generateId(),
  currency: 'usd',
  line_items: calculateLineItems(items),
  // ...
};
checkouts.set(checkout.id, checkout);
```

**Status Determination**

```js
const determineStatus = (checkout) => {
  const hasAddress = checkout.fulfillment_address?.line_one;
  const hasShipping = checkout.fulfillment_option_id;

  if (hasAddress && hasShipping) return 'ready_for_payment';
  return 'not_ready_for_payment';
};
```

**SPT Payment Processing**

```js
const params = new URLSearchParams({
  amount: totalAmount.toString(),
  currency: checkout.currency,
  shared_payment_granted_token: payment_data.token,
  'payment_method_types[0]': 'card',
  confirm: 'true'
});

const response = await fetch('https://api.stripe.com/v1/payment_intents', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${stripeSecretKey}`,
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: params.toString()
});
```

### Knowledge Check

**Q1: What makes a checkout "ready_for_payment"?**
The checkout must have items, a fulfillment address (`line_one` and `city`), and a selected fulfillment option.

**Q2: What happens when the complete endpoint receives an SPT?**
It validates the SPT format, creates a PaymentIntent with Stripe using `shared_payment_granted_token`, marks the checkout as completed, and creates an order. Stock is decremented later via webhooks to ensure inventory only changes when payment is confirmed.

**Q3: Why use an in-memory Map for checkouts?**
For simplicity in this workshop. In production, you'd use a database to persist checkouts across server restarts.

**Q4: What validations happen on checkout creation?**
Items array must exist and not be empty, each item needs an ID and positive quantity, products must exist, and stock must be available.

### Congratulations! 🎉

You've completed both the Agent Service and Merchant Service! The full ACP flow is now working:

1. User chats with the AI Agent
2. Agent creates checkout on Merchant (your endpoints!)
3. Agent collects payment via PaymentSetup
4. Agent creates SPT from saved card
5. Agent completes checkout with SPT
6. Merchant charges card using the SPT

### Test the Complete Flow

Go to the chat interface and try buying something:

1. "I want to buy the Blizzard Rustler 10"
2. Provide your shipping address
3. Select a shipping option
4. Enter test card: `4242 4242 4242 4242`
5. Confirm the purchase

Watch the **ACP Inspector** to see Agent ↔ Merchant communication in real-time!
