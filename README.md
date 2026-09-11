# @mansafi/sdk

TypeScript client for [MansaFi](https://mansafi.xyz), where people and the software working for them hold accounts side by side, and the figures moving between them stay encrypted.

The package is a typed layer over the REST API and nothing more: read accounts, move confidential money, hand an agent a wallet the chain itself keeps within bounds, and take webhook deliveries you can prove came from us.

> Beta. The surface is settled but still moving. Pin a version, and read the changelog before you bump it.

## Install

```bash
npm install @mansafi/sdk
```

Node 18 or newer, which is where global `fetch` and Web Crypto arrive. Browsers and edge runtimes work too, though a live key has no business being shipped to a browser.

## First transfer

```ts
import { MansaFi } from "@mansafi/sdk";

const mansafi = new MansaFi({ apiKey: process.env.MANSAFI_API_KEY! });

// The amount is encrypted on-chain. What comes back confirms it settled and
// pointedly does not repeat the figure you just sent.
const transfer = await mansafi.transfers.create({
  to: "@vendor",
  amount: "125.00",
  asset: "USDG",
  memo: "Invoice #4471",
});

console.log(transfer.status, transfer.txHash);
```

Each `create` goes out with an `Idempotency-Key`, invented for you when you do not supply one. Supplying your own makes a retry safe even across a restart, since two calls sharing a key can only ever produce one transfer:

```ts
await mansafi.transfers.create(
  { to: "@vendor", amount: "125.00" },
  { idempotencyKey: `invoice-4471` },
);
```

## Keys

One bearer key per request, minted at **Dashboard → Developer → API Keys**. The prefix says which network it acts on and the client reads that for itself, so there is no mode switch to set wrongly:

| Prefix | Network | What it touches |
|---|---|---|
| `hc_live_` | Mainnet, chain 4663 | Real USDG |
| `hc_test_` | Testnet, chain 46630 | Nothing real |

```ts
const mansafi = new MansaFi({ apiKey: "hc_test_..." });
mansafi.environment; // "test"
```

A key can move money. Keep it in the environment or a secrets manager, never in a commit, and replace it the moment you suspect it has been seen.

## What confidentiality means here

The servers hold ciphertext and no key to open it. Two consequences fall directly out of that, and they shape how this SDK behaves:

- No response carries a plaintext amount, including the response to the call that specified one. Reading a figure means decrypting it yourself.
- `accounts.balances()` returns numbers only when handed a `decryptionProof` built on your side. Without one you learn which assets are non-empty, and nothing more.

```ts
const { balances } = await mansafi.accounts.balances({ decryptionProof });
```

## Agents

An agent gets a wallet and a mandate. The mandate is a smart-account constraint rather than a server-side rule, so overspending is not something the agent is discouraged from, it is something it cannot do, and a compromised backend cannot wave it through either:

```ts
const agent = await mansafi.agents.create({
  name: "research-bot",
  spendPolicy: {
    dailyLimitUsdg: 500,
    perTransactionLimitUsdg: 50,
    allowedRecipients: ["api.market", "*.anthropic.com"],
    assets: ["USDG"],
    activeHours: "00:00-23:59",
    hitlThresholdUsdg: 25, // at or above this, a person decides
  },
});

// Work through whatever is waiting on you.
const { pending } = await mansafi.agents.listPendingTransactions();
for (const tx of pending) {
  await mansafi.agents.approveTransaction(tx.transactionId);
}
```

## Webhooks

Subscribe, then prove each delivery is ours before you act on it. Hand the verifier the body exactly as it arrived; a parsed and rebuilt object hashes to something else and will be rejected:

```ts
const webhook = await mansafi.webhooks.create({
  url: "https://yourapp.com/hooks/mansafi",
  events: ["transfer.confirmed", "agent.transaction.pending_approval"],
});

// Shown once. Store it before you move on.
const secret = webhook.secret;

// In the handler: verify and parse together. A bad signature throws
// MansaFiWebhookVerificationError, so nothing unproven reaches the logic below.
const event = await mansafi.webhooks.constructEvent({
  payload: rawBody, // string or Uint8Array, untouched
  signature: request.headers["x-mansafi-signature"],
  secret,
});

switch (event.event) {
  case "transfer.confirmed":
    console.log(`settled: ${event.txHash}`);
    break;
  case "agent.transaction.pending_approval":
    // get a person involved
    break;
}
```

If a plain boolean suits you better, `mansafi.webhooks.verifySignature({ payload, signature, secret })` answers true or false and throws nothing.
