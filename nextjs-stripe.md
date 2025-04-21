# Next.js & Stripe

1. Setup Stripe product

   - Go to stripe dashboard
   - Search `Product catalog`
     add: name, description, billing period, recurring, etc
   - Once it's created, click on `create payment link`

2. Setup Stripe payment link
3. Setup Stripe webhook API on nextjs

   - Search `web hooks`
   - We will have an api on our next.js app and everytime a user makes a payment, it fires this api
   - user makes payment -> fires api -> change db

4. Review different event types

### Main events:

- `checkout.session.completed`:

  - when: user completes checkout
  - use:
    - create or update subscriptions row
    - set `status = "active"`, fill in `current_period_start`, `current_period_end`
    - link to correct `user_id`, `price_id` and store `stripe_subscription_id`

- `invoice.paid`:

  - when: payment is successfully made (initial or recurring)
  - use:
    - update `subscriptions.status = "active"`
    - update billing period (`current_period_start_` and `end`)

- `invoice.payment_failed`:

  - when: payment failed
  - use:
    - set `status = "past_due"`
    - optionally notify user

- `customer.subscription.updated`:

  - when: plan is changed (upgrade/downgrade), or billing period is updated
  - use:
    - sync `price_id`, `status` and period dates

- `customer.subscription.deleted`:

  - when: subscription is canceled or ends
  - use:
    - update `status = "canceled"`
    - optionally offer reactivation or downgrade logic
