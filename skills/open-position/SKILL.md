---
name: open_position
description: Place trades on Eterna with proper sizing, leverage, and risk management
metadata.openclaw.requires.bins:
  - name: eterna
    install: npm install -g @eterna-hybrid-exchange/cli
    postInstall: eterna login
---

# Open Position Skill

Place trades with proper instrument checks, leverage, position sizing, and TP/SL.

## Before placing any trade

1. **Check balance** — don't trade with zero equity.
2. **Check existing positions** — avoid unintended doubling on the same symbol.
3. **Get instrument specs** — need `minOrderQty`, `qtyStep`, `tickSize` for proper rounding.
4. **Get user confirmation** — NEVER execute without explicit "yes"/"do it"/"go ahead".

## Pre-trade checks

```typescript
const symbol = "LINKUSDT"; // replace

const [balance, positions, instruments, ticker] = await Promise.all([
  eterna.getBalance(),
  eterna.getPositions(symbol),
  eterna.getInstruments(symbol),
  eterna.getTickers(symbol),
]);

const equity = parseFloat(balance.list[0].totalEquity);
const existingPos = positions.list.filter((p) => parseFloat(p.size) > 0);
const spec = instruments.list[0];
const price = parseFloat(ticker.list[0].lastPrice);

return {
  equity,
  availableBalance: balance.list[0].totalAvailableBalance,
  existingPositions: existingPos.map((p) => ({ side: p.side, size: p.size, entry: p.avgPrice })),
  contract: {
    minOrderQty: spec.lotSizeFilter.minOrderQty,
    qtyStep: spec.lotSizeFilter.qtyStep,
    tickSize: spec.priceFilter.tickSize,
    maxLeverage: spec.leverageFilter.maxLeverage,
  },
  currentPrice: price,
};
```

If equity < $20, suggest depositing more. If there's already a position on this symbol, warn the user.

## Present the trade proposal

Before executing, show the user exactly what you'll do:

> **Long LINKUSDT at ~$9.36**
> - Size: 5.3 LINK (~$50 notional, 2x leverage)
> - Margin used: ~$25 (50% of equity)
> - Stop loss: $9.17 (-2%)
> - Take profit: $9.69 (+3.5%)
> - Risk/reward: 1.75:1
>
> Should I place this?

**Be clear about margin vs notional.** "Size: X LINK (~$Y notional, Zx leverage)" so the user knows both the position value and how much margin is used.

## Execute the trade

Only after user confirms:

```typescript
const symbol = "LINKUSDT";
const side = "Buy"; // or "Sell" for short
const leverage = 2;
const equityFraction = 0.25; // 25% of equity
const slPercent = 0.02; // 2% stop
const tpPercent = 0.035; // 3.5% target

// Get specs and price
const [instruments, ticker, balance] = await Promise.all([
  eterna.getInstruments(symbol),
  eterna.getTickers(symbol),
  eterna.getBalance(),
]);

const spec = instruments.list[0];
const minQty = parseFloat(spec.lotSizeFilter.minOrderQty);
const qtyStep = parseFloat(spec.lotSizeFilter.qtyStep);
const tickSize = parseFloat(spec.priceFilter.tickSize);
const price = parseFloat(ticker.list[0].lastPrice);
const equity = parseFloat(balance.list[0].totalEquity);

// Set leverage first
await eterna.setLeverage({ symbol, leverage });

// Calculate qty — round DOWN to qtyStep
const positionValue = equity * equityFraction * leverage;
let qty = Math.floor((positionValue / price) / qtyStep) * qtyStep;
if (qty < minQty) qty = minQty;

// Calculate TP/SL — round to tickSize
let sl, tp;
if (side === "Buy") {
  sl = Math.floor((price * (1 - slPercent)) / tickSize) * tickSize;
  tp = Math.ceil((price * (1 + tpPercent)) / tickSize) * tickSize;
} else {
  sl = Math.ceil((price * (1 + slPercent)) / tickSize) * tickSize;
  tp = Math.floor((price * (1 - tpPercent)) / tickSize) * tickSize;
}

const order = await eterna.placeOrder({
  symbol, side, orderType: "Market",
  qty: qty.toString(), leverage,
  stopLoss: sl.toString(), takeProfit: tp.toString(),
});

return {
  orderId: order.orderId, symbol, side,
  qty: qty.toString(),
  estimatedEntry: price.toString(),
  stopLoss: sl.toString(), takeProfit: tp.toString(),
  leverage,
  marginUsed: (qty * price / leverage).toFixed(2) + " USDT",
  notionalValue: (qty * price).toFixed(2) + " USDT",
};
```

## Confirm execution

After placing, verify the position:

```typescript
const positions = await eterna.getPositions("LINKUSDT");
return positions.list.filter((p) => parseFloat(p.size) > 0).map((p) => ({
  symbol: p.symbol, side: p.side, size: p.size,
  entryPrice: p.avgPrice, markPrice: p.markPrice,
  unrealisedPnl: p.unrealisedPnl,
  leverage: p.leverage, takeProfit: p.takeProfit, stopLoss: p.stopLoss,
}));
```

## Rules

- **First trade for new users:** max 2-3x leverage, 20-30% of equity, always set TP and SL.
- **Always round:** qty down to `qtyStep`, prices to `tickSize`.
- **Always set leverage** before placing the order — it's per-symbol.
- **Never skip confirmation** — present the proposal, wait for explicit yes.

## SDK methods used

| Method | Returns |
|--------|---------|
| `eterna.getBalance()` | `{ list: [{ totalEquity, totalAvailableBalance, coin: [...] }] }` |
| `eterna.getPositions(symbol?)` | `{ list: [{ symbol, side, size, avgPrice, markPrice, unrealisedPnl, leverage, takeProfit, stopLoss }] }` |
| `eterna.getInstruments(symbol?)` | `{ list: [{ lotSizeFilter: { minOrderQty, qtyStep }, priceFilter: { tickSize }, leverageFilter: { maxLeverage } }] }` |
| `eterna.getTickers(symbol?)` | `{ list: [{ lastPrice, ... }] }` — strings |
| `eterna.setLeverage({ symbol, leverage })` | `{}` — must call before placeOrder |
| `eterna.placeOrder({ symbol, side, orderType, qty, leverage?, stopLoss?, takeProfit?, price?, reduceOnly? })` | `{ orderId, orderLinkId }` |
| `eterna.getOrders(symbol?)` | `{ list: [{ symbol, orderId, side, orderType, qty, price, orderStatus }] }` |
