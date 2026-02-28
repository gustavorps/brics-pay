# POS API Protocol — BRICS Pay QR

Source: https://outline.ilink.dev/s/05717c4a-9396-4d53-b13d-d6898f7f5b5c

## 1. Purpose

This document describes the POS API protocol for integration with offline point-of-sale software to accept BRICS Pay QR payments.

The technology stack references Joys processing.

## 2. Protocol and message format

- Transport: HTTPS POST to processing endpoint (HTTP for test server).
- Data format: JSON (RFC 7159/8259 style).
- Signature: request and response signatures in header `X-Signature` (base64).
- Language negotiation: optional `Accept-Language` (BCP 47, for example `ru`, `en`).

### 2.1 Request header

| Parameter | Type | Description |
|---|---|---|
| `X-Signature` | base64 | Signature of ASN.1 structure/digest for request body |
| `Accept-Language` | string | Response language code |

### 2.2 Request body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `Query` | string | Yes | Request identifier (`<type>/<section>/<process>`) |
| `ID` | number | Yes | Terminal identifier |
| `Message` | number | No | Correlation/message identifier |
| `Request` | object | Yes | Query-specific payload |

### 2.3 Response body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `Query` | string | Yes | Same query identifier |
| `ID` | number | Yes | Terminal identifier |
| `Message` | number | No | Echoed message identifier |
| `Error` | object | No | Protocol-level error |
| `Reply` | object | Yes* | Query-specific response payload |

`Reply` is omitted on protocol errors.

## 3. Error model

### 3.1 Protocol errors (global)

| Number | Description | Recommended action |
|---|---|---|
| 400 | Request syntax error | Verify request structure and types |
| 401 | Signature verification failed | Verify key/signature/terminal; re-activate terminal if needed |
| 403 | Metadata or processing key changed | Re-activate with same successful activation parameters |
| 404 | Object not found | Verify request data |
| 405 | Operation forbidden for terminal | Verify permissions/terminal profile |
| 406 | Request unavailable with current parameters | Verify request values |
| 408 | Request timeout | Retry with same parameters |
| 409 | Object already exists | Verify uniqueness fields |
| 423 | Terminal blocked | Contact support |
| 500 | Internal server error | Contact support |

### 3.2 Logic errors (query-specific)

Returned in `Reply` context; each query may define additional error codes (for example `409`, `412`, `404`, `405`).

## 4. Trade session (`Session`)

Payment requests may include optional `Session` value (cashier shift/session number). Session lifecycle integrity is controlled by terminal, not processing.

## 5. Signature requirements

Supported asymmetric algorithms include ECDSA, RSA, and GOST variants (same family as IA doc).

Flow:

1. Terminal signs request body digest with private key.
2. Signature is base64-encoded to `X-Signature`.
3. Processing verifies with terminal public key.
4. Processing signs response; terminal verifies with processing public key.

Body bytes must exactly match signed bytes.

## 6. Terminal activation (`pos/activate`)

### 6.1 Request

| Parameter | Required | Type | Description |
|---|---|---|---|
| `Query` | Yes | string | `pos/activate` |
| `ID` | Yes | number | Terminal ID |
| `Message` | No | number | Message ID |
| `Request.Activation` | Yes | string | Signature of activation code |
| `Request.Key` | Yes | string | Terminal public key (base64) |
| `Request.Serial` | Yes | string | Terminal serial number |
| `Request.License` | No | string | Terminal software license number |

### 6.2 Response (highlights)

Returns processing key and terminal profile metadata, including:

- `Activation` signature from processing side
- processing `Key` (public key)
- `Sequence`, `Transaction` last values
- `Timezone`
- `Services`
- `Blocked`
- `Currency` (`Code`, `Name`, `Symbol`, `Precision`)
- merchant metadata (`Address`, `Brand`, `Legal`, `Phone`)

Activation notes from source:

- Wrong activation code path may yield `406`.
- Limited number of activation attempts.
- If any request returns `403`, re-activation is required.

## 7. Payments and checks

All payment methods produce a check/transaction structure with status fields:

- `Processed`: processing status
- `Paid`: payment status
- `Amount`, `Currency`
- `Sequence` (cash register check number)
- `Transaction` (payment transaction ID)
- `Receipt` with printable result (`Code`, `Message`, `Time`)

### 7.1 Receipt input structure (`Request.Receipt`)

| Field | Required | Type | Description |
|---|---|---|---|
| `Receipt.Number` | Yes | string | Check number |
| `Receipt.Discount` | No | number | Informational discount |
| `Receipt.Items` | Yes | array/object | Purchased items |
| `Item.ID` | No | string | Product code |
| `Item.Group` | No | string | Accounting group |
| `Item.Title` | Yes | string | Display title |
| `Item.Quantity` | No | number | Quantity |
| `Item.Measure` | No | string | Unit |
| `Item.Discount` | No | number | Per-item discount |
| `Item.Price` | Yes | number | Per-item price |

### 7.2 Payment by customer QR (`pos/payment/token`)

Request fields:

- `Token` (customer QR)
- `Amount`
- `Sequence`
- optional `Session`, `Receipt`

Typical logic errors:

- `409`: duplicate check (`Sequence` already exists)
- `412`: invalid amount

### 7.3 Payment by merchant QR (`pos/payment/invoice`)

Request fields:

- `Amount`, `Sequence`
- optional `TTL`, `Service`, `Session`, `Receipt`, `Callback`

Response includes check structure containing QR/token data for QR generation.

Typical logic errors:

- `409`: duplicate check
- `412`: invalid amount

### 7.4 Cancel unpaid check (`pos/sequence/cancel`)

Cancels current unpaid check (`Processed=false`).
Cannot reverse an already paid transaction.

Typical logic error:

- `405`: check cannot be canceled in current state

### 7.5 Full reversal of paid check (`pos/payment/reversal`)

Requires:

- new `Sequence`
- parent `Transaction`
- optional `Session`

Important constraints:

- Reversal is for paid transaction (`Processed=true`, `Paid=true`).
- Usually same-terminal and same operating-day constraints apply.
- If partial refunds already exist, full reversal may fail with `412`.

Typical errors:

- `409` duplicate sequence
- `404` check not found
- `405` operation forbidden
- `412` reversal forbidden due to prior refunds/reversal

### 7.6 Full/partial refund (`pos/payment/refund`)

Requires:

- parent `Transaction`
- `Amount` (full or partial)
- `Sequence`
- optional `Session`

Typical errors:

- `404` check not found
- `405` operation forbidden
- `409` duplicate sequence
- `412` invalid amount (too large / precision mismatch)

## 8. Search and reporting

### 8.1 Get check by sequence (`pos/sequence/get`)

Gets check details by `Sequence` within the same terminal.

### 8.2 Get check by transaction (`pos/receipt/get`)

Finds check by transaction number, including checks created on other terminals under same account/currency context.

### 8.3 Search checks list (`pos/receipt/search`)

Supports filters:

- optional `Session`
- optional `Time.Start`, `Time.End`
- pagination (`Elements`, `Offset`)

Sorted by creation time descending.

### 8.4 Session totals / Z-report (`pos/receipt/session`)

Request by `Session`; response includes aggregated totals (for example credit, debit, invoice count), enabling shift-level reconciliation.

## 9. Operational recommendations

1. Always verify processing response signatures.
2. Keep `Sequence` globally unique per terminal lifecycle.
3. Handle protocol vs logic errors separately.
4. Before paid transaction reversal/refund, fetch transaction/check data via search APIs.
5. Enforce amount precision rules by terminal currency settings.
