---
name: withdraw
description: Withdraw crypto from Eterna to an external wallet
metadata.openclaw.requires.bins:
  - name: eterna
    install: npm install -g @eterna-hybrid-exchange/cli
    postInstall: eterna login
---

# Withdraw Skill

Guide the user through withdrawing funds from Eterna.

## Step 1 — Check withdrawable amount

```typescript
const amount = await eterna.getWithdrawableAmount("USDT");
return amount;
```

If the user has open positions, available balance will be less than total equity. Explain this if it surprises them.

## Step 2 — Get chain options

```typescript
const info = await eterna.getCoinInfo("USDT");
return info.rows[0].chains.filter((c) => c.chainWithdraw === "1").map((c) => ({
  chain: c.chain, chainType: c.chainType,
}));
```

Recommend Arbitrum for low fees unless the user specifies a chain.

## Step 3 — Submit withdrawal

**Always confirm address and amount with the user before submitting.** Double-check the chain matches their destination wallet.

```typescript
const result = await eterna.submitWithdrawal("USDT", "50", "0xabc123...", "ARBI");
return result;
```

## Step 4 — Check status

```typescript
const status = await eterna.getWithdrawalStatus();
return status;
```

## SDK methods used

| Method | Returns |
|--------|---------|
| `eterna.getWithdrawableAmount(coin)` | `{ coin, withdrawableAmount, availableBalance }` |
| `eterna.getCoinInfo(coin)` | `{ rows: [{ coin, chains: [{ chain, chainType, chainDeposit, chainWithdraw }] }] }` |
| `eterna.submitWithdrawal(coin, amount, address, chain)` | `{ withdrawalRequestId, status, coin, amount, address, chain }` |
| `eterna.getWithdrawalStatus(id?)` | `{ requests: [{ id, status, coin, amount, address, chain, createdAt }] }` |
