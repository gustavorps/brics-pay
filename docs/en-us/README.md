# BRICS API Documentation (English)

This folder contains English Markdown documentation generated from the following source specifications:

- Internet Acquiring API (E-Commerce): https://outline.ilink.dev/s/689d8f67-d0fb-41ca-9895-677ecdc7171d
- POS API Protocol: https://outline.ilink.dev/s/05717c4a-9396-4d53-b13d-d6898f7f5b5c

> Translation style: **literal-first** (close to source wording), with minimal interpretation.

## Documents

- [internet-acquiring-api.md](./internet-acquiring-api.md): BRICS Pay QR Internet Acquiring flow and endpoints.
- [pos-api.md](./pos-api.md): POS protocol, message formats, activation, payment, reversal/refund, search, Z-report.
- [typescript-examples.md](./typescript-examples.md): TypeScript integration examples for `brics-pay` style clients.

## Scope

These docs cover protocol behavior, request/response shape, signatures, errors, and operation lifecycle.
UI sketch images from source documents are intentionally omitted.

## Language and terminology

The original documents are primarily in Russian. Field names and query identifiers are kept as specified by protocol (for example `Query`, `ID`, `Reply`, `pos/payment/invoice`).

## Notes

- The source documents include dense tables and repeated structures. This translation preserves semantic meaning and endpoint behavior.
- Where examples are inferred for TypeScript usage, they are explicitly marked as examples and not normative protocol text.
