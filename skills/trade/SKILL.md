---
name: eterna_trading
description: Trade crypto on Eterna using the eterna CLI
metadata.openclaw.requires.bins: [eterna]
---

# Eterna Trading Skill

You have access to the `eterna` CLI to execute trading operations on Eterna.

## Prerequisites

Verify the user is authenticated:

```shell
eterna whoami
```

If not logged in, instruct the user to run `eterna login`.

## Available Commands

- **Check balance**: `eterna balance`
- **View positions**: `eterna positions`
- **View orders**: `eterna orders`
- **Execute trading code**: `eterna execute <file>`
- **Browse SDK methods**: `eterna sdk --search <query> --detail summary`

## Executing Complex Operations

For operations like placing orders, setting leverage, or closing positions,
write TypeScript using the built-in `eterna.*` methods, save to a `.ts` file,
and execute via `eterna execute`:

```typescript
// save as /tmp/trade.ts, then run: eterna execute /tmp/trade.ts
const result = await eterna.placeOrder({
  symbol: "BTCUSDT",
  side: "Buy",
  orderType: "Market",
  qty: "0.001",
});
console.log(result);
```

You can also pipe code directly via stdin:

```shell
eterna execute - << 'EOF'
const balance = await eterna.getBalance();
console.log(balance);
EOF
```

## Discovering SDK Methods

When you need to look up what methods are available or check exact parameter signatures:

```shell
eterna sdk --search "place order" --detail full
eterna sdk --search "technical analysis" --detail list
eterna sdk --detail full   # browse all methods
```
