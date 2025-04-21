# Integrating Supabase and Stripe

1. User management with supabase auth

- Handle sign-up/sign-in via Supabase
- Use supabase `auth.user()` to access user ID -- used as the Stripe `customer.metadata.user_id`

2. Create stripe customer on signup

- On `auth.signUp`, trigger a backend function (e.g., supabase edge function or webhook) to:
- Create stripe customer
- Store the `stripe_customer_id` in `public.users` table

3. Product and pricing setup in Stripe

- Define products and prices in the stripe dashboard
- user `lookup_key` or `price_id` in your app for reference

4. Checkout or portal session

On frontend:

- Call a server-side function to create a `checkout.session` or `billing portal session` using the users `stripe_customer_id`
- Redirect the user to stripe

5. Handle webhooks

- Setup Stripe webhooks to listen for:
  - `checkout.session.completed`
  - `invoice.paid`
  - `customer.subscription.updated`
  - `customer.subscription.deleted`
- On webhook, update Suapabase `users` or `subscriptions` table accordingly

6. Restrict access based on subscription
   On supabase: - Use RLS policies or conditional logic in your app to restrict content/features - Example: `subscription_status = 'active'`

## Steps

- Create account Stripe
- Grab the id: developers/API Keys/Secret key
- Setup the webhook: webhooks/add endpoint/test local environment
  - stripe login
  - test endpoint:
    - create script for it: `stripe listen --forward-to localhost:<port>/<webhook-api-route-path>`

### Setup flow

- Setup stripe instance

```js
// stripe setup: lib/stripe.ts
import Stripe from "stripe";

export const stripe = new Stripe(process.env.STRIPE_KEY, {
  apiVersion: "2024-06-20",
  typescript: true,
});
```

- Setup action

```js
// actions/stripe.ts
("use server");

export async function subscribeAction({ userId }) {
  // our stripe instance from lib/stripe.ts
  const { url } = await stripe.checkout.sessions.create({
    payment_method_types: ["card"],
    line_items: [
      {
        price: process.env.STRIPE_PRICE_ID,
        quantity: 1,
      },
    ],
    metadata: {
      userId,
    },
    mode: "subscription",
    sucess_url: `${process.env.NEXT_PUBLIC_URL}`,
    cancel_url: `${process.env.NEXT_PUBLIC_URL}`,
  });

  return url;
}
```

- Setup client interaction

```js
// subscribe button
("use client");
function SubscribeButton({ userId }) {
  const router = useRouter();

  const [isPending, startTransition] = useTransition();

  async function handleClickSubscribeButton() {
    startTransition(async () => {
      // our server action
      const url = await subscribeAction({ userId });

      if (url) {
        router.push(url);
      } else {
        console.error("Failed to create subscription session");
      }
    });
  }

  return (
    <button onClick={handleClickSubscribeButton} disable={isPending}>
      subscribe
    </button>
  );
}
```

- Setup webhook

```js
// api/stripe/route.ts

export async function POST(request: Request) {
  const body = request.text();
  const signature = headers().get("Stripe-Signature");
  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SINGING_SECRET
    );
  } catch (error) {
    console.log(error);
    return new NextResponse("webhook error", { status: 400 });
  }

  if (
    event.type === "checkout.session.completed" &&
    event.data.object.payment.status === "paid"
  ) {
  }
}
```

- Create product on stripe
  - products/create
  - price/recurring/monthly
  - take the product id, and add to the `env.`
