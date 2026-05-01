# Stripe Integration Guide (Documentation Only)

This repository contains **only documentation** for integrating Stripe Checkout with the AI Prompt Template Library subscription service. No real payment processing occurs in this MVP; the steps below can be followed to enable live payments in a production environment.

---

## 1. Prerequisites

- A Stripe account (sign‑up at https://dashboard.stripe.com/register).
- Access to the Stripe Dashboard for API keys.
- A server or static‑site host that can serve the checkout page and handle the success/webhook endpoints.
- (Optional) Domain with HTTPS – Stripe requires a secure origin for Checkout.

---

## 2. Obtain API Keys

1. Log in to the Stripe Dashboard.
2. Navigate to **Developers → API keys**.
3. Copy the **Publishable key** (`pk_live_…` or `pk_test_…`).
4. Copy the **Secret key** (`sk_live_…` or `sk_test_…`).
5. Store these keys securely (environment variables, secret manager, etc.).

---

## 3. Create a Product & Price in Stripe

1. In the Dashboard, go to **Products → Add product**.
2. Name: **AI Prompt Template Library – Monthly Subscription**.
3. Description: *Unlimited access to 30+ curated AI prompts across blogs, socials, ads, emails, and image generation.*
4. Click **Add price**.
   - **Pricing model:** Recurring
   - **Currency:** USD
   - **Amount:** 999 (represents $9.99)
   - **Billing interval:** Monthly
5. Save the product. Note the **Price ID** – it looks like `price_1Hh…`.

---

## 4. Checkout Link Template (Static Site Example)

Below is a minimal HTML snippet you can embed on `pricing.html` (or a dedicated checkout page). Replace `YOUR_PUBLISHABLE_KEY` and `YOUR_PRICE_ID` with the values from steps 2‑3.

```html
<script src="https://js.stripe.com/v3/"></script>
<button id="checkout-button">Subscribe for $9.99/mo</button>
<script>
  const stripe = Stripe('YOUR_PUBLISHABLE_KEY');
  document.getElementById('checkout-button').addEventListener('click', async () => {
    const {error} = await stripe.redirectToCheckout({
      lineItems: [{price: 'YOUR_PRICE_ID', quantity: 1}],
      mode: 'subscription',
      successUrl: window.location.origin + '/success.html?session_id={CHECKOUT_SESSION_ID}',
      cancelUrl: window.location.origin + '/cancel.html',
    });
    if (error) console.error(error);
  });
</script>
```

The button redirects the user to Stripe’s hosted Checkout page. After a successful payment Stripe redirects to `success.html` with a `session_id` query parameter.

---

## 5. Member Delivery Flow (Post‑Payment)

1. **Checkout Session Completed** – Stripe sends a `checkout.session.completed` webhook.
2. **Webhook Handler** (server‑side) receives the event, verifies the signature, and extracts:
   - `customer_email`
   - `subscription.id`
   - `session_id`
3. **Create / Update Member Record** in your database:
   ```json
   {
     "email": "customer@example.com",
     "stripe_customer_id": "cus_…",
     "stripe_subscription_id": "sub_…",
     "status": "active",
     "created_at": "2026-04-30T20:00:00Z"
   }
   ```
4. **Send Welcome Email** – include:
   - Login link to the member‑only area.
   - Quick‑start guide.
   - Link to the full prompt library.
5. **Grant Access** – when the user logs in, check the subscription status via Stripe’s API (`/v1/subscriptions/{id}`). If `status` is `active` or `trialing`, allow full library download.
6. **Handle Cancellations / Expirations** – listen for `customer.subscription.deleted` and `customer.subscription.updated` webhooks. Update the member record and revoke access accordingly.

---

## 6. Testing Locally

- Use Stripe test keys (`pk_test_…` / `sk_test_…`).
- Set up the webhook endpoint locally with **ngrok** (e.g., `ngrok http 3000`).
- Add the **Signing secret** from the Dashboard to your server to verify events.
- Use Stripe’s test card numbers (e.g., `4242 4242 4242 4242`).

---

## 7. Deploying to Production

1. Replace test keys with live keys.
2. Ensure your domain uses HTTPS.
3. Update webhook URLs in the Dashboard to point to your live server.
4. Run a final end‑to‑end test with a real card (or use Stripe’s test mode on live keys).

---

### Quick Reference Summary
| Step | Action |
|------|--------|
| 1 | Create Stripe account & get API keys |
| 2 | Add product & price ($9.99/mo) |
| 3 | Embed Checkout button (replace keys) |
| 4 | Implement webhook handler to create member record |
| 5 | Send welcome email & grant library access |
| 6 | Handle cancellations & renewals |
| 7 | Test with ngrok, then go live |

---

**Note:** This documentation is intentionally static – no code is executed in this repository. Implement the server‑side parts in your preferred language/framework (Node, Python, Ruby, etc.).
