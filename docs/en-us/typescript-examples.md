# TypeScript Examples (Option C)

This file provides inferred TypeScript examples for integrating BRICS Internet Acquiring and POS APIs.

> These examples are illustrative integration patterns, not normative protocol text.

## 1. Shared signing helpers

```ts
import crypto from "node:crypto";

export function sha256(input: string): Buffer {
  return crypto.createHash("sha256").update(input, "utf8").digest();
}

export function signRsaSha256Base64(payload: string, pemPrivateKey: string): string {
  const signer = crypto.createSign("RSA-SHA256");
  signer.update(payload, "utf8");
  signer.end();
  return signer.sign(pemPrivateKey).toString("base64");
}

export function verifyRsaSha256Base64(
  payload: string,
  signatureBase64: string,
  pemPublicKey: string,
): boolean {
  const verifier = crypto.createVerify("RSA-SHA256");
  verifier.update(payload, "utf8");
  verifier.end();
  return verifier.verify(pemPublicKey, Buffer.from(signatureBase64, "base64"));
}
```

## 2. Internet Acquiring: create transaction (`URL/api`)

```ts
type IaCreateTransactionRequest = {
  Pos: string;
  Sequence: string;
  Amount: string;
  User: string;
  Callback?: string;
  CSS?: string;
  Return?: string;
  TTL?: number;
  Receipt?: {
    Number: string;
    Discount?: string;
    Items: Array<{
      ID?: string;
      Group?: string;
      Title: string;
      Quantity?: string;
      Measure?: string;
      Discount?: string;
      Price: string;
    }>;
  };
};

type IaCreateTransactionReply = {
  URL: string;
};

async function createIaTransaction(
  baseUrl: string,
  body: IaCreateTransactionRequest,
  privateKeyPem: string,
): Promise<IaCreateTransactionReply> {
  const payload = JSON.stringify(body);
  const signature = encodeURIComponent(signRsaSha256Base64(payload, privateKeyPem));

  const res = await fetch(`${baseUrl}/api?signature=${signature}`, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "accept-language": "en",
    },
    body: payload,
  });

  if (!res.ok) {
    throw new Error(`IA create transaction failed: HTTP ${res.status}`);
  }

  return (await res.json()) as IaCreateTransactionReply;
}
```

## 3. Internet Acquiring: get check state (`URL/get`)

```ts
type IaCheckState = {
  Transaction?: string;
  Paid: boolean;
  Processed: boolean;
  Amount: string;
  Currency: { Code: number; Precision: number; Name: string; Symbol: string };
  Time: { Created: string; Processed?: string; Timeout: string };
  Reference?: string;
  Error?: { Code: string; Message: string };
};

async function getIaCheckState(
  baseUrl: string,
  pos: string,
  sequence: string,
  privateKeyPem: string,
): Promise<IaCheckState> {
  const canonical = `pos=${pos}&sequence=${sequence}`;
  const signature = encodeURIComponent(signRsaSha256Base64(canonical, privateKeyPem));

  const res = await fetch(
    `${baseUrl}/get?pos=${encodeURIComponent(pos)}&sequence=${encodeURIComponent(sequence)}&signature=${signature}`,
    {
      headers: { "accept-language": "en" },
    },
  );

  if (!res.ok) {
    throw new Error(`IA get state failed: HTTP ${res.status}`);
  }

  return (await res.json()) as IaCheckState;
}
```

## 4. POS API: signed request envelope

```ts
type PosEnvelope<TRequest> = {
  Query: string;
  ID: number;
  Message?: number;
  Request: TRequest;
};

type PosResponse<TReply> = {
  Query: string;
  ID: number;
  Message?: number;
  Error?: { Number: number; Description?: string };
  Reply?: TReply;
};

async function postPosSigned<TReq extends object, TReply>(
  endpointUrl: string,
  envelope: PosEnvelope<TReq>,
  terminalPrivateKeyPem: string,
  processingPublicKeyPem?: string,
): Promise<PosResponse<TReply>> {
  const payload = JSON.stringify(envelope);
  const xSignature = signRsaSha256Base64(payload, terminalPrivateKeyPem);

  const res = await fetch(endpointUrl, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "accept-language": "en",
      "x-signature": xSignature,
    },
    body: payload,
  });

  const text = await res.text();
  const responseSignature = res.headers.get("x-signature");

  if (processingPublicKeyPem && responseSignature) {
    const ok = verifyRsaSha256Base64(text, responseSignature, processingPublicKeyPem);
    if (!ok) {
      throw new Error("POS response signature verification failed");
    }
  }

  return JSON.parse(text) as PosResponse<TReply>;
}
```

## 5. POS activation example (`pos/activate`)

```ts
type ActivateRequest = {
  Activation: string;
  Key: string;
  Serial: string;
  License?: string;
};

async function activatePos(
  endpointUrl: string,
  terminalId: number,
  req: ActivateRequest,
  terminalPrivateKeyPem: string,
) {
  return postPosSigned<ActivateRequest, {
    Activation: string;
    Key: string;
    Sequence: string;
    Timezone: string;
    Services: Array<unknown>;
    Transaction: string;
    Blocked: boolean;
    Currency: { Code: number; Name: string; Symbol: string; Precision: number };
  }>(
    endpointUrl,
    {
      Query: "pos/activate",
      ID: terminalId,
      Request: req,
    },
    terminalPrivateKeyPem,
  );
}
```

## 6. POS invoice payment example (`pos/payment/invoice`)

```ts
type ReceiptItem = {
  ID?: string;
  Group?: string;
  Title: string;
  Quantity?: number;
  Measure?: string;
  Discount?: number;
  Price: number;
};

type CashReceipt = {
  Number: string;
  Discount?: number;
  Items: ReceiptItem[];
};

type InvoiceRequest = {
  Amount: number;
  Sequence: string;
  TTL?: number;
  Service?: number;
  Session?: number;
  Receipt?: CashReceipt;
  Callback?: string;
};

async function createInvoice(
  endpointUrl: string,
  terminalId: number,
  req: InvoiceRequest,
  terminalPrivateKeyPem: string,
  processingPublicKeyPem: string,
) {
  return postPosSigned<InvoiceRequest, {
    Paid: boolean;
    Processed: boolean;
    Amount: number;
    Currency: number;
    Token?: string;
    Sequence: string;
    Receipt?: {
      Number?: string;
      Transaction?: string;
      Time?: string;
      Result?: { Code: number; Message: string };
    };
  }>(
    endpointUrl,
    {
      Query: "pos/payment/invoice",
      ID: terminalId,
      Request: req,
    },
    terminalPrivateKeyPem,
    processingPublicKeyPem,
  );
}
```

## 7. POS refund example (`pos/payment/refund`)

```ts
type RefundRequest = {
  Transaction: string;
  Amount: number;
  Sequence: string;
  Session?: number;
};

async function refundPayment(
  endpointUrl: string,
  terminalId: number,
  req: RefundRequest,
  terminalPrivateKeyPem: string,
  processingPublicKeyPem: string,
) {
  return postPosSigned<RefundRequest, { Paid: boolean; Processed: boolean; Sequence: string }>(
    endpointUrl,
    {
      Query: "pos/payment/refund",
      ID: terminalId,
      Request: req,
    },
    terminalPrivateKeyPem,
    processingPublicKeyPem,
  );
}
```

## 8. Error handling pattern

```ts
function assertNoProtocolError<T>(resp: PosResponse<T>): asserts resp is PosResponse<T> & { Reply: T } {
  if (resp.Error) {
    throw new Error(`Protocol error ${resp.Error.Number}: ${resp.Error.Description ?? "No description"}`);
  }
  if (!resp.Reply) {
    throw new Error("Missing Reply payload");
  }
}
```

## 9. Idempotency and state transitions

Recommended rules for client implementation:

1. Store each `Sequence` once and never reuse it.
2. Persist all response payloads for reconciliation.
3. Treat final states as:
   - success: `Processed=true && Paid=true`
   - failure: `Processed=true && Paid=false`
4. For pending checks (`Processed=false`), poll/get status and/or use callback updates.
5. Before reversal/refund, retrieve latest transaction state via search endpoints.
