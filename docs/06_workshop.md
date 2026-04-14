# Workshop Wrap-Up

Congratulations! 🎉 You've completed the Agentic Commerce Workshop and built a fully functional AI shopping assistant!

## What You Built

### Merchant Service

- ✅ Product catalog with AI-friendly selectors
- ✅ ACP checkout endpoints (create, update, complete, cancel)
- ✅ SPT payment processing
- ✅ Webhook-based stock management

### Agent Service

- ✅ Payment method storage and management
- ✅ Shared Payment Token (SPT) creation
- ✅ Checkout orchestration via ACP
- ✅ AI service integration

### Integration

- ✅ End-to-end purchase flow
- ✅ Stripe webhooks for payment confirmation
- ✅ Frontend chat interface
- ✅ Stripe payment processing

## Key Concepts Learned

### Agentic Commerce Protocol (ACP)

```
POST /checkouts               → Create session
PUT  /checkouts/:id           → Update details
POST /checkouts/:id/complete  → Process payment
```

### Shared Payment Tokens (SPT)

```
Agent creates SPT → Agent sends to Merchant → Merchant charges card
```

### AI Function Calling

```json
{ "action": "create_checkout", "items": [...] }
```

→ Agent executes action → Returns result to user

## Architecture Summary

```
┌──────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│   Frontend   │────►│   Agent Service      │────►│ Merchant Service│
│   (UI)       │◄────│   (Orchestration)    │◄────│ (ACP/Payments)  │
└──────────────┘     └──────────┬───────────┘     └────────┬────────┘
                                │                          │
                   ┌────────────┴────────────┐             │
                   ▼                         ▼             ▼
            ┌─────────────┐    ┌─────────────────┐  ┌──────────────────┐
            │ AI Service  │    │  Agent Stripe   │  │ Merchant Stripe  │
            │ (Lambda)    │    │  (Cards, SPT)   │  │   (Payments)     │
            └─────────────┘    └─────────────────┘  └──────────────────┘
```

The Agent and Merchant each have separate Stripe accounts. Payment methods are stored on the Agent's account, and the Merchant processes payments via their own account using SPT.

## Next Steps

- **Explore the acp-demo codebase** — See the complete implementation
- **Deploy to production** — Use the deployment guides in the repo
- **Extend your agent** — Add order tracking, returns, or multi-merchant support
- **Join the community** — Share what you've built!

## Resources

- [Stripe Documentation](https://stripe.com/docs)
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
- [AWS Lambda](https://docs.aws.amazon.com/lambda/)
- [Express.js](https://expressjs.com/)

## Feedback

We'd love to hear about your experience! Please share:

- What worked well
- What could be improved
- What you'd like to learn next

---

Thank you for participating in this workshop. You now have the knowledge to build AI-powered commerce experiences with Stripe.

Happy building! 🚀

## Quick Reference

| Module | File | What You Implemented |
| --- | --- | --- |
| 3 | `agent-service/routes/payment.js` | SPT creation, payment method retrieval |
| 3 | `agent-service/routes/checkout.js` | ACP checkout orchestration |
| 3 | `frontend/components/PaymentSetup.tsx` | Stripe Elements integration |
| 4 | `merchant-service/routes/checkouts.js` | All 5 ACP endpoints |
| 5 | `merchant-service/routes/webhooks.js` | Webhook handler for stock updates |
