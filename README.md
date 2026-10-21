# ScopeDiff
Clear scope. Fair payment.

ScopeDiff helps independent designers, video editors and developers agree on paid scope changes with international clients. Paste the original agreement and the new request, review Gemini's evidence-backed classification, set your own prices, then let the client approve and pay for selected extras through PayPal Sandbox. Cancelled extras can be refunded individually.

## Run locally
Requires Node.js 22+. No npm dependencies.

```sh
cp .env.example .env
# Fill the sandbox/AI/signing environment variables privately.
node --env-file=.env server.js
# Open http://127.0.0.1:8790
npm test
```

Environment: `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET`, `GEMINI_API_KEY`, `SCOPEDIFF_SIGNING_SECRET`, optional `DAILY_ANALYSIS_LIMIT` and `PORT` (8790). PayPal keys must belong to the SAME sandbox REST app. A separate Sandbox Personal account approves purchases. Never use real-money credentials. Gemini model: `gemini-3.8-flash`.

## How it works
1. Paste the agreed brief and client request, or try either built-in example.
2. Gemini returns structured JSON: included / outside scope / clarification needed. Every item contains exact quotations from both inputs.
3. Server verifies every quote is a nonempty substring of its source; invalid evidence is rejected and retried once. Persistent invalid output fails closed.
4. The freelancer confirms prices in USD. AI can suggest a rate calculation but never sets the final amount.
5. Client checks the desired extras, approves via PayPal JS SDK, and the server captures the approved order.
6. The client sees a payment confirmation and order/capture IDs. The freelancer refreshes Payment trail from PayPal using that order ID, then cancels an item to refund its frozen line-item amount.

## PayPal integration
All calls use `https://api-m.sandbox.paypal.com` only.
- `POST /v1/oauth2/token`: server-side OAuth2 client credentials.
- `POST /v2/checkout/orders`: create CAPTURE-intent order, total computed from server-side selected item IDs.
- `POST /v2/checkout/orders/{id}/capture`: capture after PayPal JS SDK `onApprove`; verify capture status, currency and amount.
- `POST /v2/payments/captures/{id}/refund`: exact partial refund for cancelled paid item.
- PayPal JavaScript SDK renders the client approval button. The SDK client ID is public; secrets stay server-side.

Money uses integer cents; prices freeze in the signed priced token. Duplicate items, unpriced/included items and excessive/repeated refunds are rejected. Create/capture/refund requests use stable PayPal idempotency keys. AI never calls payment APIs.

## Structure and deployment
`index.html`, `style.css`, `app.js`: responsive English frontend. `api/index.js`: serverless-compatible handler, also reused by `server.js`. `lib/core.js`: pure safety checks; `lib/token.js`: expiring HMAC tokens. Vercel configuration is supplied but this project has not been deployed.

## Limitations and security
This is a sandbox hackathon demo, not a production payment service or legal advice. No user accounts or verified freelancer identity. Signed role capabilities separate freelancer and client permissions. Tokens are bearer capabilities, not encrypted. Never enter confidential client text. Exact source quotes do not prove AI classification is correct; human review is mandatory. No live payments are supported. See v2 limitations below.

## Attribution
Code and documentation were written with an AI coding agent and require human review. Product concept: WooYoung Choi. References: user-provided Houselights local-server structure; official PayPal Orders v2 and Payments v2 API documentation. No third-party app code/assets were copied.

MIT license, copyright WooYoung Choi.

## Stateless Vercel deployment (v2)
Use Node.js 22 in Vercel project settings. Import this folder as the project root, with no build command. The `api/index.js` Node serverless function serves API requests and root assets are static. Configure `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET` (sandbox credentials only), `GEMINI_API_KEY`, and `SCOPEDIFF_SIGNING_SECRET` (at least 32 random characters, identical on every instance). Optional `DAILY_ANALYSIS_LIMIT` defaults to 200. For local execution export the same variables and run `npm start`.

No server files or database store change orders. The server HMAC-SHA256 signs project details, quotes, item categories, human-confirmed integer-cent prices, currency, and a 30-day expiry. Analysis returns a signed `freelancer` role token stored only in the originating browser localStorage, never in its URL. Price confirmation issues a separate signed `client` role token for the same change-order ID. Use “Open client approval link” after confirming prices; only that client token enters the URL fragment (`#t=...`). Both roles can create and capture sandbox payments; only freelancer tokens can confirm prices or refund. Client refund requests return 403 before any PayPal call. Clients see a payment confirmation and order/capture IDs, not refund controls. In the originating browser, open Payment trail, paste the client’s order ID and refresh from PayPal to cancel a paid item and refund it. Treat both tokens as bearer capabilities: the freelancer token grants refund authority, the client link grants payment authority, and both contain original source text (signed, not encrypted). LocalStorage is vulnerable to XSS/device access; do not share the freelancer token. Roles prove capability possession, not real-world identity. Legacy tokens without a role are rejected. Losing the token loses access; secret rotation invalidates existing links. Keep a stable signing secret across deployments.

Orders include item SKU/quantity/unit prices, item-total breakdown and change-order custom ID. Amounts are recalculated from verified signed data, never caller totals. Capture and refunds re-query PayPal; audit events are reconstructed from order/capture/refund IDs, statuses and provider timestamps. Order IDs are remembered in localStorage and share-link fragments; they are hints only and ownership is verified against signed data. No `.data` directory is used. Create/capture/refund use deterministic PayPal-Request-Id keys. Refund notes record the SKU and duplicate/refund-budget checks use PayPal data.

AI requests allow 10 per IP per rolling hour and a default 200 per UTC day per instance. Inputs are limited to 18,000 characters combined. These are instance-memory counters, so the global limit may not be exact (인스턴스별 메모리 카운터라 전체 상한이 정확하지 않을 수 있음). Cold starts reset counters. Production needs shared rate limiting, authentication/authorization, webhook reconciliation and durable order discovery. There is no distributed lock; PayPal idempotency protects repeated identical refunds, while PayPal's remaining capture limit is the final guard for concurrent refunds. PayPal idempotency retention is provider-limited, so this is a sandbox demo, not production accounting software.

References: PayPal Orders v2 https://developer.paypal.com/docs/api/orders/v2/ and Payments v2 https://developer.paypal.com/docs/api/payments/v2/ (items, custom_id, GET order/capture/refund, refund notes, idempotency).
