# Ashtech Pay — Direct API SDK

> Initiate Mobile Money payments directly from your server, without redirects. Supports USSD Push, OTP SMS, OTP USSD, and Wave flows across 16 African countries.

[![API Version](https://img.shields.io/badge/API-v1-blue)](https://ashtechpay.top)
[![Protocol](https://img.shields.io/badge/protocol-HTTPS-green)](https://ashtechpay.top)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Authentication](#authentication)
3. [Countries & Operators](#countries--operators---get-v1countries)
4. [Initiate a Payment](#initiate-a-payment---post-v1collect)
5. [Payment Flows](#payment-flows)
6. [Transaction Status](#transaction-status---get-v1transactionid)
7. [Fee Schedule](#fee-schedule---get-v1fees)
8. [Webhooks](#webhooks)
9. [Error Codes](#error-codes)

---

## Introduction

The Ashtech Pay API unifies multiple African payment gateways into a single REST interface. Initiate Mobile Money payments across 16+ African countries without redirects. Routing between operators is automatic — no need to choose a provider.

| Property | Value |
|---|---|
| **Base URL** | `https://ashtechpay.top` |
| **Format** | JSON only — `Content-Type: application/json` |
| **Authentication** | Bearer token in `Authorization` header |
| **Protocol** | HTTPS required |
| **Versioning** | `/v1/` prefix on all endpoints |

---

## Authentication

All requests must include your API key in the `Authorization` HTTP header.

```http
Authorization: Bearer YOUR_API_KEY
```

> ⚠️ **Security:** Only use your API key from your server (Node.js, Python, PHP…). Never include it in browser-side or mobile app code.

**Authenticated request example (Node.js):**

```javascript
const response = await fetch("https://ashtechpay.top/v1/collect", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ /* ... */ })
});
```

---

## Countries & Operators — `GET /v1/countries`

Returns the full list of active countries and their available Mobile Money operators. This list is managed by the administrator — any addition or removal is immediately reflected via this endpoint.

```javascript
fetch("https://ashtechpay.top/v1/countries", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
})
```

**Response:**

```json
[
  {
    "code": "CM",
    "name": "Cameroun",
    "currency": "XAF",
    "operators": ["MTN Mobile Money", "Orange Money"]
  },
  {
    "code": "SN",
    "name": "Senegal",
    "currency": "XOF",
    "operators": ["Free Money", "Orange Money", "Wave"]
  }
]
```

### Supported Countries (16)

| Country | Code | Currency | Operators |
|---|---|---|---|
| Benin | `BJ` | XOF | Moov Money, MTN Mobile Money |
| Burkina Faso | `BF` | XOF | Moov Money, Orange Money (OTP) |
| Cameroon | `CM` | XAF | MTN Mobile Money, Orange Money |
| Central African Rep. | `CF` | XAF | Orange Money (OTP) |
| Congo | `CG` | XAF | Airtel Money, MTN Mobile Money |
| Côte d'Ivoire | `CI` | XOF | Moov Money, MTN, Orange (OTP), Wave |
| Gabon | `GA` | XAF | Airtel Money, Moov Money |
| Guinea Conakry | `GN` | GNF | MTN Mobile Money, Orange Money (OTP) |
| Equatorial Guinea | `GQ` | XAF | Orange Money (OTP) |
| Guinea-Bissau | `GW` | XOF | Orange Money (OTP) |
| Mali | `ML` | XOF | Moov Money, Orange Money (OTP) |
| Niger | `NE` | XOF | Airtel Money |
| DR Congo | `CD` | CDF | Afrimoney, Airtel, Orange (OTP), Vodacom M-Pesa |
| Senegal | `SN` | XOF | Free Money, Orange Money (OTP), Wave |
| Chad | `TD` | XAF | Airtel Money, Moov Money |
| Togo | `TG` | XOF | Flooz (Moov), T-Money |

> `(OTP)` = OTP required (received by SMS or USSD) • `Wave` = Wave payment link

---

## Initiate a Payment — `POST /v1/collect`

Initiates a Mobile Money payment. The customer receives a validation request on their phone. Routing between providers is automatic based on country and operator. Fees are deducted automatically — `credited_amount` is the net amount credited to your account.

### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `amount` | number | ✅ | Gross amount to collect |
| `currency` | string | ✅ | Country currency (XAF, XOF, GNF, CDF…) |
| `phone` | string | ✅ | Payer's phone number |
| `operator` | string | ✅ | Exact operator name (from `/v1/countries`) |
| `country_code` | string | ✅ | ISO country code (CM, SN, CI…) |
| `reference` | string | ➖ | Your unique order reference |
| `otp` | string | ➖ | OTP code if required (see `400 otp_required` response) |
| `notify_url` | string | ➖ | Webhook URL to receive payment result |

### Example Request

```javascript
fetch("https://ashtechpay.top/v1/collect", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    amount: 5000,
    currency: "XAF",
    phone: "670000000",
    operator: "MTN Mobile Money",
    country_code: "CM",
    reference: "ORDER-001",
    notify_url: "https://yoursite.com/webhook"
  })
})
```

### Response `202` — Success (USSD Push)

```json
{
  "transaction_id": "8f3e1c2d-...",
  "reference": "ORDER-001",
  "status": "pending",
  "amount": 5000,
  "credited_amount": 4750,
  "fee_amount": 250,
  "currency": "XAF",
  "operator": "MTN Mobile Money",
  "phone": "670000000",
  "country_code": "CM",
  "created_at": "2026-03-15T14:00:00Z"
}
```

### OTP Required (Orange Money CI, SN, ML, BF…)

Some operators require an OTP code. If so, the API returns a `400 otp_required` error. Re-submit the request with the `otp` field included.

```json
// Initial 400 response:
{
  "error": "otp_required",
  "message": "OTP required. Dial #144*82# to get your OTP code.",
  "ussd_code": "#144*82#"  // null for SMS OTP
}
```

```javascript
// Re-submit with OTP:
{
  "amount": 5000,
  "currency": "XOF",
  "phone": "07XXXXXXXX",
  "operator": "Orange Money",
  "country_code": "CI",
  "otp": "123456",
  "notify_url": "https://yoursite.com/webhook"
}
```

---

## Payment Flows

Depending on the country and operator, the API automatically uses one of 4 flows below. Your code must handle each differently since the response and steps vary.

| Flow | Operators | Initial Response | Required Action |
|---|---|---|---|
| **USSD Push** | MTN, Moov, Airtel, Orange CM, Free SN, T-Money, Flooz, M-Pesa | `202 pending` | Wait for webhook. Customer validates on their phone. |
| **OTP SMS** | Orange Money (CI, SN, ML, GN, CF, CG, GA, GW, GQ, CD, TD) | `400 otp_required`, `ussd_code: null` | Customer receives SMS OTP. Re-submit request with `otp`. |
| **OTP USSD** | Orange Money BF only | `400 otp_required`, `ussd_code: *144*4*6*5000#` | Customer dials USSD code, receives OTP. Re-submit with `otp`. |
| **Wave** | Wave CI, Wave SN | `202 pending`, `flow: "wave"`, `wave_url: ...` | Display `wave_url` as a button or QR code. Customer completes in Wave app. |

### Flow Detection

```javascript
async function collectPayment(params) {
  const res = await fetch("https://ashtechpay.top/v1/collect", {
    method: "POST",
    headers: {
      "Authorization": "Bearer YOUR_API_KEY",
      "Content-Type": "application/json"
    },
    body: JSON.stringify(params)
  });

  const data = await res.json();

  if (res.status === 202 && data.flow === "wave") {
    // Wave flow: display data.wave_url
    return { type: "wave", waveUrl: data.wave_url, transactionId: data.transaction_id };
  }

  if (res.status === 202) {
    // USSD Push flow: wait for webhook
    return { type: "ussd_push", transactionId: data.transaction_id };
  }

  if (res.status === 400 && data.error === "otp_required") {
    if (data.ussd_code) {
      // OTP USSD (BF Orange): code to dial
      return { type: "otp_ussd", ussdCode: data.ussd_code };
    } else {
      // OTP SMS (Orange CI, SN, ML…): automatic SMS
      return { type: "otp_sms" };
    }
  }

  throw new Error(data.message);
}
```

---

## Transaction Status — `GET /v1/transaction/:id`

Check the status of a transaction at any time using the `transaction_id` returned at initiation. Can be used alongside webhooks.

```javascript
fetch("https://ashtechpay.top/v1/transaction/8f3e1c2d-...", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
})
```

**Response:**

```json
{
  "transaction_id": "8f3e1c2d-...",
  "reference": "ORDER-001",
  "status": "success",
  "amount": 5000,
  "credited_amount": 4750,
  "fee_amount": 250,
  "currency": "XAF",
  "phone": "670000000",
  "created_at": "2026-03-15T14:00:00Z",
  "confirmed_at": "2026-03-15T14:02:17Z"
}
```

### Transaction Statuses

| Status | Description | Final? |
|---|---|---|
| `pending` | Awaiting operator confirmation | ❌ |
| `success` | Payment confirmed — merchant account credited | ✅ |
| `failed` | Payment declined, expired, or cancelled | ✅ |

---

## Fee Schedule — `GET /v1/fees`

Returns the current fee schedule for each active country. Fees are configured by the administrator and may change at any time. Query this endpoint to calculate the net amount before calling `/v1/collect`.

```javascript
fetch("https://ashtechpay.top/v1/fees", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
})
```

**Response:**

```json
[
  {
    "country_code": "CM",
    "country_name": "Cameroun",
    "currency": "XAF",
    "deposit_fee_pct": 3.5,
    "withdrawal_fee_pct": 1.5,
    "transfer_fee_pct": 1.0,
    "total_fee_pct": 5.5
  }
]
```

### Calculating Net Amount Before Collecting

```javascript
// Cache fees at startup or refresh hourly
const fees = await fetch("https://ashtechpay.top/v1/fees", {
  headers: { "Authorization": "Bearer YOUR_API_KEY" }
}).then(r => r.json());

// Calculate net amount credited to your account
function computeNet(grossAmount, countryCode) {
  const fee = fees.find(f => f.country_code === countryCode);
  if (!fee) return grossAmount;

  const feeAmount = Math.round(grossAmount * fee.total_fee_pct / 100);
  return {
    gross: grossAmount,
    fee: feeAmount,
    net: grossAmount - feeAmount,
    fee_pct: fee.total_fee_pct,
  };
}

// Example:
console.log(computeNet(10000, "CM"));
// → { gross: 10000, fee: 550, net: 9450, fee_pct: 5.5 }
```

---

## Webhooks

When a transaction reaches a final state, Ashtech Pay automatically sends a `POST` request to the `notify_url` passed in your `/v1/collect` call. The `amount` field is the net amount after fees; `total_amount` is the gross collected amount.

### Payload — Successful Payment

```json
{
  "event": "payment.completed",
  "transaction_id": "8f3e1c2d-...",
  "reference": "ORDER-001",
  "status": "completed",
  "amount": 4750,         // net after fees
  "total_amount": 5000,   // gross collected
  "currency": "XAF",
  "type": "deposit",
  "phone": "670000000",
  "timestamp": "2026-03-15T14:02:17.000Z"
}
```

### Events

| Event | Trigger |
|---|---|
| `payment.completed` | Deposit payment confirmed successfully |
| `payment.failed` | Payment declined, expired, or cancelled |
| `payout.completed` | Withdrawal or outgoing transfer confirmed |
| `payout.failed` | Withdrawal or transfer failed |

### Handler — Node.js / Express

```javascript
app.post("/webhook", express.json(), async (req, res) => {
  // Always respond 200 first
  res.status(200).json({ received: true });

  const { event, transaction_id, reference, amount, currency } = req.body;

  if (event === "payment.completed") {
    // amount = net amount (after fees)
    await markOrderAsPaid(reference, { transactionId: transaction_id, amount, currency });
  }
  if (event === "payment.failed") {
    await cancelOrder(reference);
  }
  if (event === "payout.completed") {
    await markPayoutDone(reference, { transactionId: transaction_id });
  }
  if (event === "payout.failed") {
    await markPayoutFailed(reference);
  }
});
```

> **Best practices:**
> - Always respond HTTP 200 immediately.
> - Process business logic **after** responding 200 (asynchronously).
> - Check `transaction_id` against your database to prevent duplicate processing.
> - Your `notify_url` must be a public HTTPS URL (not localhost).

---

## Error Codes

On error, the API returns a JSON object with `error` and `message` fields.

```json
{
  "error": "bad_request",
  "message": "Required fields: amount, currency, phone, operator, country_code"
}
```

| HTTP | Error | Meaning |
|---|---|---|
| `400` | `bad_request` | Missing parameter or invalid format |
| `400` | `otp_required` | OTP required — check `ussd_code` in the response |
| `401` | `unauthorized` | API key missing, invalid, or revoked |
| `403` | `forbidden` | This transaction does not belong to your account |
| `404` | `not_found` | Transaction not found |
| `422` | `unprocessable` | Country or operator not supported / incorrect currency |
| `429` | `rate_limited` | Too many requests — slow down |
| `502` | `gateway_error` | Operator network rejected the payment |
| `500` | `server_error` | Internal error — retry |

---

## Support

For any technical questions not covered by this documentation, contact the team via the in-app support.

📧 [support@ashtechpay.top](mailto:support@ashtechpay.top) • 🌐 [ashtechpay.top](https://ashtechpay.top)
