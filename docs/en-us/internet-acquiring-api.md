# Internet Acquiring API (E-Commerce) — BRICS Pay QR

Source: https://outline.ilink.dev/s/689d8f67-d0fb-41ca-9895-677ecdc7171d

## 1. Purpose

This document describes the Internet Acquiring API protocol for integration with e-commerce platforms to accept payments through BRICS Pay QR.

The BRICS Pay QR processing solution is built on Joys processing.

## 2. Glossary (translated)

- Processing: BRICS Pay QR payment processing.
- E-commerce platform: website/online store/marketplace used to sell goods and services.
- Merchant (TSP): trade/service enterprise using the payment service.
- POS terminal (`Pos`): merchant terminal identifier linked to merchant account.

## 3. Protocol parameters

- Data format: JSON ([RFC 7159](https://datatracker.ietf.org/doc/html/rfc7159.html) / [RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259) style in examples).
- Transport: HTTPS POST (HTTP can be used for test server).
- Request authentication: electronic signature generated with terminal private key.
- Signature verification: performed by processing using corresponding public key.

Supported value kinds: object, array, numbers, string, literals (`true`, `false`, `null`).

## 4. Error model

Two categories:

1. **Protocol errors** (global): response includes `Error` object only (no `Reply`).
2. **Logic errors** (per method): nested inside `Reply`.

### 4.1 Protocol errors (common)

| Number | Description | Recommended action |
|---|---|---|
| 400 | Request syntax error | Validate parameters, types, and request structure |
| 401 | Signature verification failed | Re-check public key, terminal number, signature; re-activate terminal if needed |
| 404 | Object not found | Verify input data |
| 408 | Request timeout | Retry with same parameters |
| 409 | Object already exists | Verify input data |
| 423 | Terminal is blocked | Contact support |
| 500 | Internal server error | Contact support |

## 5. Merchant onboarding and terminal setup

Before using payments in e-commerce:

1. Sign an agreement to join BRICS Pay QR service.
2. Register in the service.
3. Obtain terminal ID (`Pos`) linked to merchant ID.
4. Configure terminal settings:
   - Shop URL (return URL after payment page)
   - CSS URL (payment page styling)
   - Callback URL (operation status notifications)
   - TTL (payment form lifetime, default 5 minutes)
   - Public key and selected cryptographic algorithm

## 6. Cryptographic algorithms

Asymmetric key pair is required.

- Private key remains on client side.
- Public key is provided for terminal generation/activation.

Supported algorithms from source:

| Algorithm | Parameters | Digest function |
|---|---|---|
| ECDSA | Curve `prime256v1` or `prime224v1` | SHA-256 |
| RSA | 1024 or 2048-bit key | SHA-256 |
| GOST 34.10 | 256-bit, curve `2001-CryptoProA` | GOST 34.11-2012 256 |
| GOST 34.10 | 256-bit, curve `2012-TC26A` | GOST 34.11-2012 256 |
| GOST 34.10 | 512-bit, curve `2012-TC26A` | GOST 34.11-2012 512 |

The exact request bytes must match the bytes used for signing.

## 7. Creating a payment page

Use POST request:

- Request body: JSON structure.
- URL includes `signature` value derived from JSON digest signed by private key.

If key does not match configured terminal public key, authorization fails (`401`).

Response returns payment page URL containing QR flow.

Callbacks are sent to terminal/operation callback URL after payment attempt, with final operation status.

## 8. Main API paths

| Path | Method | Description |
|---|---|---|
| `URL/api` | POST | Create transaction and display payment form |
| `URL/get` | GET | Retrieve check/operation status |
| `URL/refund` | POST | Full/partial refund by transaction |

Header recommendation from source: send `Accept-Language` (IETF BCP 47).

## 9. Create transaction (`URL/api`, POST)

Purpose: create operation and render payment page.

### 9.1 Request body (key fields)

| Parameter | Required | Type | Description |
|---|---|---|---|
| `Pos` | Yes | string | Merchant terminal number |
| `Sequence` | Yes | string | Unique terminal operation/check number |
| `Amount` | Yes | string | Floating-point transaction amount |
| `User` | Yes | string | SHA-256 from IP + User-Agent |
| `Callback` | No | string | Custom callback URL |
| `CSS` | No | string | CSS file URL |
| `Return` | No | string | Return URL to merchant shop |
| `TTL` | No | uint8 | Custom operation TTL |
| `Receipt` | No | object | Receipt structure |

Receipt highlights:

| Field | Required | Type | Description |
|---|---|---|---|
| `Receipt.Number` | Yes | string | Merchant receipt number |
| `Receipt.Discount` | No | string | Discount amount (informational) |
| `Receipt.Items` | Yes | array/object | Items collection |
| `Item.ID` | No | string | Product ID |
| `Item.Group` | No | string | Product group identifier |
| `Item.Title` | Yes | string | Product title |
| `Item.Quantity` | No | string | Quantity |
| `Item.Measure` | No | string | Unit |
| `Item.Discount` | No | string | Per-item discount |
| `Item.Price` | Yes | string | Per-item price |

### 9.2 Response (payment page)

| Parameter | Required | Type | Description |
|---|---|---|---|
| `URL` | Yes | string | Payment URL |

## 10. Get operation/check state (`URL/get`, GET)

Pass terminal and sequence plus URL signature.

Example pattern:

`ia/get/?pos=<POS>&sequence=<SEQUENCE>&signature=<BASE64_SIGNATURE>`

Response includes transaction/check status fields such as:

- `Transaction`
- `Paid`
- `Processed`
- `Amount`
- `Currency` (`Code`, `Precision`, `Name`, `Symbol`)
- `Time` (`Created`, `Processed`, `Timeout`)
- optional `Reference`
- optional `Error` (`Code`, `Message`)

## 11. Full or partial refund (`URL/refund`, POST)

Used from merchant back office to return funds for successful payment transaction.

Important points:

- Refund `Sequence` must be a new unique terminal operation identifier.
- `Reference`/`Transaction` must point to parent payment transaction.
- Refund amount cannot exceed original payment amount.

Typical refund body fields:

| Parameter | Required | Type | Description |
|---|---|---|---|
| `POS` | Yes | string | Terminal number |
| `Sequence` | Yes | string | Unique refund operation number |
| `Reference` | Yes | string | Parent transaction reference |
| `Amount` | Yes | string | Refund amount |

The response structure follows check-state response semantics.

## 12. Callback behavior

When transaction reaches final state:

- `Processed=true`, `Paid=true` (success), or
- `Processed=true`, `Paid=false` (failed),

processing sends POST callback with operation/check payload.

## 13. Integration checklist

1. Activate and configure terminal.
2. Generate and securely store private key.
3. Register matching public key in processing.
4. Build canonical JSON before signing.
5. Sign request data and send signature in URL parameter as specified.
6. Handle protocol and logic errors separately.
7. Implement callback receiver and idempotent status updates.
8. Implement GET status fallback polling.
9. Implement refund flow with amount validation.
