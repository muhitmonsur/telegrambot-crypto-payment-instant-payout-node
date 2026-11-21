# PayerURL Telegram Bot Crypto Payment SDK

[![npm version](https://img.shields.io/npm/v/@muhitmonsur/telegrambot-crypto-payment-instant-payout-node.svg)](https://www.npmjs.com/package/@muhitmonsur/telegrambot-crypto-payment-instant-payout-node)
[![license](https://img.shields.io/npm/l/@muhitmonsur/telegrambot-crypto-payment-instant-payout-node.svg)](LICENSE)

Server-side Node.js SDK for creating PayerURL crypto payments and verifying payment webhooks in Telegram bot and Node.js applications.

**Powered by [PayerURL](https://payerurl.com)** — instant crypto payments to your wallet.

- [Get API keys](https://dash.payerurl.com/profile/get-api-credentials)
- [PayerURL dashboard](https://dash.payerurl.com)
- [Telegram support](https://t.me/Payerurl)

## Requirements

- Node.js 18 or newer
- A PayerURL public key and secret key

## Installation

```bash
npm install @muhitmonsur/telegrambot-crypto-payment-instant-payout-node
```

```bash
yarn add @muhitmonsur/telegrambot-crypto-payment-instant-payout-node
```

```bash
pnpm add @muhitmonsur/telegrambot-crypto-payment-instant-payout-node
```

## Import

### ES modules / TypeScript

```ts
import { Payerurl } from '@muhitmonsur/telegrambot-crypto-payment-instant-payout-node';
```

### CommonJS

```js
const { Payerurl } = require('@muhitmonsur/telegrambot-crypto-payment-instant-payout-node');
```

## Server-side credentials

This SDK signs API requests with a secret key. Keep it in server environment variables and never expose it in browser code or Telegram bot source distributed to users.

```env
PAYERURL_PUBLIC_KEY=your_public_key
PAYERURL_SECRET_KEY=your_secret_key
```

```js
const { PAYERURL_PUBLIC_KEY, PAYERURL_SECRET_KEY } = process.env;

if (!PAYERURL_PUBLIC_KEY || !PAYERURL_SECRET_KEY) {
  throw new Error('Missing PayerURL API credentials');
}
```

## Quick start

```js
import { Payerurl } from '@muhitmonsur/telegrambot-crypto-payment-instant-payout-node';

const payerurl = new Payerurl({
  publicKey: process.env.PAYERURL_PUBLIC_KEY,
  secretKey: process.env.PAYERURL_SECRET_KEY,
});

const result = await payerurl.payment({
  invoiceId: `ORDER-${Date.now()}`,
  amount: 1000,
  currency: 'usd',
  data: {
    first_name: 'Alice',
    last_name: 'Smith',
    email: 'alice@example.com',
    redirect_url: 'https://yourdomain.com/payment/success',
    cancel_url: 'https://yourdomain.com/payment/cancel',
    notify_url: 'https://yourdomain.com/api/payerurl/notify',
  },
  orderItems: [
    {
      name: 'Telegram bot subscription',
      qty: 1,
      price: '10.00',
    },
  ],
});

if (result.status) {
  console.log(result.redirectUrl);
} else {
  console.error(result.message);
}
```

The SDK uses the PayerURL Telegram payment API endpoint by default:

```text
https://api-v2.payerurl.com/api/telegramPayment
```

You can override it through the optional `apiUrl` constructor option.

## Express webhook example

```js
import express from 'express';
import { Payerurl } from '@muhitmonsur/telegrambot-crypto-payment-instant-payout-node';

const app = express();
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

const payerurl = new Payerurl({
  publicKey: process.env.PAYERURL_PUBLIC_KEY,
  secretKey: process.env.PAYERURL_SECRET_KEY,
});

app.post('/api/payment', async (req, res) => {
  const result = await payerurl.payment({
    invoiceId: req.body.invoiceId,
    amount: Number(req.body.amount),
    currency: req.body.currency || 'usd',
    data: {
      first_name: req.body.first_name,
      last_name: req.body.last_name,
      email: req.body.email,
      redirect_url: 'https://yourdomain.com/payment/success',
      cancel_url: 'https://yourdomain.com/payment/cancel',
      notify_url: 'https://yourdomain.com/api/payerurl/notify',
    },
  });

  if (!result.status) {
    return res.status(400).json({ message: result.message });
  }

  return res.json({ redirectUrl: result.redirectUrl });
});

app.post('/api/payerurl/notify', (req, res) => {
  const result = payerurl.verifyWebhook({
    authorization: req.headers.authorization,
    authStr: req.body?.authStr,
    body: req.body,
  });

  if (!result.ok) {
    return res.status(400).json({
      status: result.status,
      message: result.message,
    });
  }

  // Fulfill the invoice only after successful verification.
  console.log(result.payload);
  return res.json({ status: result.status, message: result.payload });
});

app.listen(3000);
```

## Telegram bot usage

Send the SDK's `redirectUrl` to the user as a normal link or inline keyboard button. A Telegram bot is not a browser for PayerURL checkout, so the hosted checkout link must be opened separately and the payment callback must be verified by the bot server.

```js
// Telegram bot framework pseudocode
const result = await payerurl.payment({
  invoiceId: `TG-${chatId}-${Date.now()}`,
  amount: 1000,
  currency: 'usd',
  data: {
    redirect_url: 'https://yourdomain.com/payment/success',
    cancel_url: 'https://yourdomain.com/payment/cancel',
    notify_url: 'https://yourdomain.com/api/payerurl/notify',
  },
});

if (result.status) {
  await bot.sendMessage(chatId, 'Complete your payment:', {
    reply_markup: {
      inline_keyboard: [[
        { text: 'Pay with crypto', url: result.redirectUrl },
      ]],
    },
  });
}
```

## Next.js App Router

```ts
// app/api/payment/route.ts
import { Payerurl } from '@muhitmonsur/telegrambot-crypto-payment-instant-payout-node';
import { NextResponse } from 'next/server';

const payerurl = new Payerurl({
  publicKey: process.env.PAYERURL_PUBLIC_KEY!,
  secretKey: process.env.PAYERURL_SECRET_KEY!,
});

export async function POST(req: Request) {
  const body = await req.json();
  const result = await payerurl.payment({
    invoiceId: body.invoiceId,
    amount: body.amount,
    currency: body.currency || 'usd',
    data: {
      first_name: body.first_name,
      last_name: body.last_name,
      email: body.email,
      redirect_url: `${process.env.APP_URL}/payment/success`,
      cancel_url: `${process.env.APP_URL}/payment/cancel`,
      notify_url: `${process.env.APP_URL}/api/payerurl/notify`,
    },
  });

  return NextResponse.json(result);
}
```

```ts
// app/api/payerurl/notify/route.ts
import { Payerurl } from '@muhitmonsur/telegrambot-crypto-payment-instant-payout-node';
import { NextResponse } from 'next/server';

const payerurl = new Payerurl({
  publicKey: process.env.PAYERURL_PUBLIC_KEY!,
  secretKey: process.env.PAYERURL_SECRET_KEY!,
});

export async function POST(req: Request) {
  const contentType = req.headers.get('content-type') || '';
  const body = contentType.includes('application/json')
    ? await req.json()
    : Object.fromEntries(new URLSearchParams(await req.text()));

  const result = payerurl.verifyWebhook({
    authorization: req.headers.get('authorization'),
    authStr: body.authStr,
    body,
  });

  if (!result.ok) {
    return NextResponse.json(
      { status: result.status, message: result.message },
      { status: 400 }
    );
  }

  return NextResponse.json({
    status: result.status,
    message: result.payload,
  });
}
```

## API

### `new Payerurl(config)`

| Option | Type | Required | Description |
|---|---|---|---|
| `publicKey` | `string` | Yes | PayerURL API public key |
| `secretKey` | `string` | Yes | PayerURL API secret key; keep it server-side |
| `apiUrl` | `string` | No | Defaults to `https://api-v2.payerurl.com/api/telegramPayment` |
| `fetch` | `typeof fetch` | No | Custom Fetch-compatible transport |

The constructor throws if `publicKey` or `secretKey` is missing.

### `payment(request)`

| Field | Type | Required | Description |
|---|---|---|---|
| `invoiceId` | `string` | Yes | Unique order or invoice ID |
| `amount` | `number` | Yes | Positive integer in the smallest currency unit |
| `currency` | `string` | No | Fiat currency code; defaults to `usd` |
| `data` | `object` | Yes | Customer and callback information |
| `data.first_name` | `string` | No | Customer first name |
| `data.last_name` | `string` | No | Customer last name |
| `data.email` | `string` | No | Customer email |
| `data.redirect_url` | `string` | Yes | Success redirect URL |
| `data.cancel_url` | `string` | Yes | Cancel redirect URL |
| `data.notify_url` | `string` | Yes | Webhook notification URL |
| `orderItems` | `array` | No | Optional checkout line items |
| `type` | `string` | No | Integration type; defaults to `nodejs` |

Each `orderItems` entry accepts:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | Yes | Product name; spaces are converted to underscores |
| `qty` | `number \| string` | Yes | Quantity |
| `price` | `number \| string` | Yes | Item price |

**Success**

```js
{ status: true, redirectUrl: 'https://...' }
```

**Error**

```js
{ status: false, message: 'Human-readable error' }
```

### `verifyWebhook(input)`

```js
const result = payerurl.verifyWebhook({
  authorization: req.headers.authorization,
  authStr: req.body?.authStr,
  body: req.body,
});
```

| Input | Required | Description |
|---|---|---|
| `authorization` | Preferred | Raw `Authorization` header |
| `authStr` | Fallback | Base64 webhook token when the Authorization header is unavailable |
| `body` | Yes | Parsed JSON or URL-encoded webhook fields |

**Webhook result**

```js
{ ok: true, status: 2040, message: payload, payload }
```

```js
{ ok: false, status: 2030, message: 'Signature not matched' }
```

| Status | Meaning |
|---|---|
| `2040` | Signature is valid and the order is complete |
| `2030` | Authorization, public key, or signature validation failed |
| `2050` | Required order data is missing or the order is not complete |
| `20000` | Order was cancelled |

Only fulfill an order when `result.ok === true`. Store the `transaction_id` and make fulfillment idempotent.

## Utility exports

The package also exports:

- `buildQueryString`
- `signPayload`
- `createAuthHeader`
- `parseAuthToken`
- `safeEqual`

These helpers are intended for advanced integrations. Most applications should use `payment()` and `verifyWebhook()`.

## Production checklist

- Store API credentials only on the server.
- Use HTTPS for `redirect_url`, `cancel_url`, and `notify_url`.
- Verify every webhook before fulfilling an order.
- Match `order_id` with an existing invoice.
- Verify the paid amount and currency in your own database.
- Record processed `transaction_id` values to prevent duplicate fulfillment.
- Do not expose the secret key to Telegram users or browser bundles.

## License

MIT
