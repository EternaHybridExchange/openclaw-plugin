---
name: deposit
description: Guide user through depositing crypto and transferring funds to the trading wallet
metadata.openclaw.requires.bins:
  - name: eterna
    install: npm install -g @eterna-hybrid-exchange/cli
    postInstall: eterna login
---

# Deposit Skill

Guide the user through depositing funds into their Eterna trading account.

**Critical flow:** Deposit address → send crypto → monitor with `getDepositRecords()` → transfer to trading wallet → confirm balance.

## Important context

- Deposits arrive in the **Funding wallet**, NOT the trading wallet.
- After deposit confirms, you MUST call `transferToTrading()` before funds are usable.
- `getBalance()` checks the **trading wallet** — it will show zero until you transfer. **Do NOT use `getBalance()` to check if a deposit has arrived.**
- Use `getDepositRecords()` to monitor incoming deposits.
- Recommend **Arbitrum** for USDT deposits — cheapest and fastest.

## Step 1 — Show deposit options

```typescript
const coins = await eterna.getAllowedDepositCoins("USDT");
return coins.configList.map((c) => ({
  coin: c.coin, chain: c.chain, chainName: c.chainType,
  minDeposit: c.minDepositAmount, confirmations: c.blockConfirmNumber,
}));
```

Present as a simple choice: "I'd recommend **Arbitrum** — fast and cheap. Which chain works for your wallet?"

If the user wants a different coin (BTC, ETH, USDC), run `getAllowedDepositCoins` for that coin.

## Step 2 — Get deposit address

```typescript
// Replace coin/chain based on user's choice
const addr = await eterna.getDepositAddress("USDT", "ARBI");
return addr;
```

Show address clearly. If there's a tag/memo, emphasize it — missing tags can lose funds.

## Step 3 — Monitor deposit

Once the user says they've sent funds, check `getDepositRecords()`:

```typescript
const records = await eterna.getDepositRecords("USDT");
const pending = records.rows.filter((r) => r.status !== 3 && r.status !== 4);
const confirmed = records.rows.filter((r) => r.status === 3);
return { pending, confirmed };
```

**Status codes:**
- 0 = unknown
- 1 = waiting for confirmations — tell user: "I can see it, waiting for blockchain confirmations."
- 2 = processing — "Almost there, Eterna is processing it."
- **3 = success** — proceed to Step 4 immediately.
- 4 = failed — "Something went wrong. Check the tx on a block explorer."

Check when the user asks. If they ask you to poll, check every minute.

## Step 4 — Transfer to trading wallet

**This step is mandatory.** Funds sit in the Funding wallet until transferred.

If they deposited USDT:

```typescript
const transfer = await eterna.transferToTrading("USDT", "ALL_BALANCE");
return transfer;
```

If they deposited a non-USDT coin, swap first:

```typescript
const swap = await eterna.swapToUsdt("ETH"); // omit amount for full balance
const transfer = await eterna.transferToTrading("USDT", "ALL_BALANCE");
return { swap, transfer };
```

## Step 5 — Confirm balance is ready

```typescript
const balance = await eterna.getBalance();
const account = balance.list[0];
return {
  totalEquity: account.totalEquity,
  availableBalance: account.totalAvailableBalance,
  coins: account.coin.filter((c) => parseFloat(c.equity) > 0).map((c) => ({
    coin: c.coin, equity: c.equity, usdValue: c.usdValue,
  })),
};
```

Celebrate: "You're funded! $X ready to trade." Then suggest looking at trade ideas.

## SDK methods used

| Method | Returns |
|--------|---------|
| `eterna.getAllowedDepositCoins(coin?, chain?)` | `{ configList: [{ coin, chain, chainType, minDepositAmount, blockConfirmNumber }] }` |
| `eterna.getDepositAddress(coin, chainType)` | `{ coin, chains: { chainType, addressDeposit, tagDeposit } }` |
| `eterna.getDepositRecords(coin?)` | `{ rows: [{ coin, chain, amount, txID, status, confirmations }] }` — status: 0-4 |
| `eterna.transferToTrading(coin, amount)` | `{ transferId: string }` — use `"ALL_BALANCE"` for full transfer |
| `eterna.swapToUsdt(coin, amount?)` | `{ coin, orderId, qty, success, message }` — omit amount for full balance |
| `eterna.getBalance()` | `{ list: [{ totalEquity, totalAvailableBalance, coin: [...] }] }` — **trading wallet only** |
| `eterna.getAllCoinsBalance(accountType)` | Balance by account type: `"FUND"`, `"UNIFIED"`, `"SPOT"` |
