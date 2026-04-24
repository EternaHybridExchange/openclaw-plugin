---
name: close_position
description: Close positions and cancel orders on Eterna
metadata.openclaw.requires.bins:
  - name: eterna
    install: npm install -g @eterna-hybrid-exchange/cli
    postInstall: eterna login
---

# Close Position Skill

Close positions and cancel orders.

## Show current positions

```typescript
const [positions, orders] = await Promise.all([
  eterna.getPositions(),
  eterna.getOrders(),
]);

return {
  positions: positions.list.filter((p) => parseFloat(p.size) > 0).map((p) => ({
    symbol: p.symbol, side: p.side, size: p.size,
    entryPrice: p.avgPrice, markPrice: p.markPrice,
    unrealisedPnl: p.unrealisedPnl, leverage: p.leverage,
  })),
  activeOrders: orders.list.map((o) => ({
    symbol: o.symbol, orderId: o.orderId, side: o.side,
    type: o.orderType, qty: o.qty, price: o.price, status: o.orderStatus,
  })),
};
```

## Close a position

**Confirm with user before closing.** Show current P&L so they know what they're locking in.

```typescript
const result = await eterna.closePosition("LINKUSDT");
return result;
```

Returns `{ orderId, closedSize, side, entryPrice, markPrice, unrealisedPnl }`.

## Cancel orders

Single order:

```typescript
const result = await eterna.cancelOrder("LINKUSDT", "orderId123");
return result;
```

All orders on a symbol:

```typescript
const result = await eterna.cancelAllOrders("LINKUSDT");
return result;
```

All orders across all symbols:

```typescript
const result = await eterna.cancelAllOrders();
return result;
```

## Modify TP/SL on existing position

```typescript
await eterna.setTradingStop({
  symbol: "LINKUSDT",
  takeProfit: "9.80",
  stopLoss: "9.10",
});
```

## SDK methods used

| Method | Returns |
|--------|---------|
| `eterna.getPositions(symbol?)` | `{ list: [{ symbol, side, size, avgPrice, markPrice, unrealisedPnl, leverage, takeProfit, stopLoss }] }` |
| `eterna.getOrders(symbol?)` | `{ list: [{ symbol, orderId, side, orderType, qty, price, orderStatus }] }` |
| `eterna.closePosition(symbol)` | `{ orderId, closedSize, side, entryPrice, markPrice, unrealisedPnl }` |
| `eterna.cancelOrder(symbol, orderId)` | `{ orderId, orderLinkId }` |
| `eterna.cancelAllOrders(symbol?)` | `{ list: [{ orderId }] }` |
| `eterna.setTradingStop({ symbol, takeProfit?, stopLoss?, trailingStop?, ... })` | `{}` |
