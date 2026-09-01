# @mansafi/sdk

TypeScript client for [MansaFi](https://mansafi.xyz), where people and the software working for them hold accounts side by side, and the figures moving between them stay encrypted.

The package is a typed layer over the REST API and nothing more: read accounts, move confidential money, hand an agent a wallet the chain itself keeps within bounds, and take webhook deliveries you can prove came from us.

> Beta. The surface is settled but still moving. Pin a version, and read the changelog before you bump it.

## Install

```bash
npm install @mansafi/sdk
```

Node 18 or newer, which is where global `fetch` and Web Crypto arrive. Browsers and edge runtimes work too, though a live key has no business being shipped to a browser.
