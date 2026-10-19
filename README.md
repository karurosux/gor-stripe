# gor-stripe

A [gor](https://github.com/karurosux/go-on-rails) generator plugin that
drops Stripe subscriptions into any gor app: Checkout, the customer billing
portal, a signed webhook that keeps a `subscriptions` table in sync, and
middleware that gates paid features. Once installed, it runs as
`gor g stripe`.

Requires `gor g auth` to have run first (the middleware gates by signed-in
user, and the table references `users`), and a gor version with
`{{timestamp}}` plugin-path support.

## What it generates

Backend:

- `migrations/<timestamp>_create_subscriptions.sql` — one row per user:
  Stripe customer/subscription IDs, status, price, period end.
- `queries/subscriptions.sql` + sqlc codegen.
- `internal/stripe/stripe.go` — the endpoints:
  - `POST /api/stripe/checkout` — starts a subscription Checkout session
    (body `{"priceId": "..."}`, defaults to `STRIPE_PRICE_ID`); returns
    `{"url"}` to redirect to.
  - `POST /api/stripe/portal` — opens the Stripe billing portal (manage
    plan, payment method, cancel); returns `{"url"}`.
  - `GET /api/stripe/subscription` — the signed-in user's status
    (`none` until a first checkout completes).
  - `POST /api/stripe/webhook` — Stripe's callback. Verifies
    `Stripe-Signature`, then mirrors `checkout.session.completed` and
    `customer.subscription.{created,updated,deleted}` into the table.
- `internal/stripe/middleware.go` — the paid-feature guards:

  ```go
  // any active or trialing subscription
  paid := api.Group("", auth.Require, stripe.RequireSubscription(q))

  // only specific tiers
  pro := api.Group("", auth.Require, stripe.RequirePrice(q, os.Getenv("STRIPE_PRICE_PRO")))

  // inside a gated handler
  sub := stripe.CurrentSubscription(c)
  ```

  Both reject with `402 Payment Required`. Tune `stripe.ActiveStatuses`
  (e.g. add `"past_due"` for a dunning grace period).

Frontend:

- `web/src/features/billing/` — a `/billing` page (subscribe / manage
  billing buttons, renewal date) plus `useSubscription`, `useCheckout`,
  `usePortal` hooks; wired into routes (behind auth) and the nav.

Env (appended to `.env` / `.env.example`): `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID`.

## Use it

```bash
gor plugins install https://github.com/karurosux/gor-stripe
cd yourapp && gor g stripe
gor db migrate
# fill the STRIPE_* keys in .env, then in another terminal:
stripe listen --forward-to localhost:8080/api/stripe/webhook
```

Create a product with a recurring price in the Stripe test dashboard and
put its price ID in `STRIPE_PRICE_ID`. Sign in, open `/billing`, subscribe
with card `4242 4242 4242 4242` — the webhook flips the status to
`active` and gated routes open up.

## Develop

```bash
gor new testapp && cd testapp
gor g auth
gor plugins install ../gor-stripe --project --link   # symlinked; edits are live
gor g stripe                                         # --force to rerun
go build ./... && go test ./... && (cd web && npx tsc -b)
```

The plugin is declarative (`plugin.json` + templates) — gor renders the
templates, injects at `gor:` markers, appends env defaults, and runs sqlc.
The migration path's `{{timestamp}}` token becomes a fresh goose version at
run time; re-runs reuse the same file instead of duplicating history.
