---
name: market_scan
description: Live market scanning, technical analysis, and trade idea generation using Eterna CLI
metadata.openclaw.requires.bins:
  - name: eterna
    install: npm install -g @eterna-hybrid-exchange/cli
    postInstall: eterna login
---

# Market Scan Skill

Scan live markets and generate trade ideas using the `eterna` CLI.

## Quick market briefing

Run this to show the user what's happening right now:

```typescript
const [tickers, btcRsi, btcMacd, btcBb, ethRsi, ethMacd] = await Promise.all([
  eterna.getTickers(),
  eterna.getRsi("BTCUSDT", "1h"),
  eterna.getMacd("BTCUSDT", "1h"),
  eterna.getBollingerBands("BTCUSDT", "1h"),
  eterna.getRsi("ETHUSDT", "1h"),
  eterna.getMacd("ETHUSDT", "1h"),
]);

const pairs = tickers.list;
const btc = pairs.find((t) => t.symbol === "BTCUSDT");
const eth = pairs.find((t) => t.symbol === "ETHUSDT");

const sorted = pairs
  .filter((t) => parseFloat(t.turnover24h) > 5_000_000)
  .sort((a, b) => Math.abs(parseFloat(b.price24hPcnt)) - Math.abs(parseFloat(a.price24hPcnt)))
  .slice(0, 5);

return {
  btc: {
    price: btc.lastPrice,
    change24h: (parseFloat(btc.price24hPcnt) * 100).toFixed(2) + "%",
    rsi: btcRsi.value.toFixed(1),
    macdSignal: btcMacd.valueMACDHist > 0 ? "bullish" : "bearish",
    bbWidth: (((btcBb.valueUpperBand - btcBb.valueLowerBand) / btcBb.valueMiddleBand) * 100).toFixed(2) + "%",
  },
  eth: {
    price: eth.lastPrice,
    change24h: (parseFloat(eth.price24hPcnt) * 100).toFixed(2) + "%",
    rsi: ethRsi.value.toFixed(1),
    macdSignal: ethMacd.valueMACDHist > 0 ? "bullish" : "bearish",
  },
  topMovers: sorted.map((t) => ({
    symbol: t.symbol,
    price: t.lastPrice,
    change: (parseFloat(t.price24hPcnt) * 100).toFixed(2) + "%",
    volume: "$" + (parseFloat(t.turnover24h) / 1_000_000).toFixed(1) + "M",
  })),
};
```

**Present as a market briefing, not raw data.** Example tone:

> BTC at $94,200 — up 1.3%. RSI 62 on the hourly, MACD flipping bullish. Bollinger Bands tight (1.8%) — a move is brewing.
> SUI is the big mover — up 8.4% with $340M volume. ETH flat at +0.2%.

**Important:** Filter top movers to `turnover24h > 5_000_000` to avoid obscure low-volume tokens.

## Trade idea scan

When the user wants trade ideas, scan high-volume pairs with TA:

```typescript
const tickers = await eterna.getTickers();
const pairs = tickers.list
  .filter((t) => parseFloat(t.turnover24h) > 5_000_000)
  .sort((a, b) => Math.abs(parseFloat(b.price24hPcnt)) - Math.abs(parseFloat(a.price24hPcnt)))
  .slice(0, 8);

const analyses = await Promise.all(
  pairs.map(async (t) => {
    const [rsi, macd, bb] = await Promise.all([
      eterna.getRsi(t.symbol, "1h"),
      eterna.getMacd(t.symbol, "1h"),
      eterna.getBollingerBands(t.symbol, "1h"),
    ]);
    const price = parseFloat(t.lastPrice);
    const change = parseFloat(t.price24hPcnt) * 100;
    const bbPos = (price - bb.valueLowerBand) / (bb.valueUpperBand - bb.valueLowerBand);

    let score = 0;
    let direction = "";

    if (change > 0.3 && rsi.value < 70 && macd.valueMACDHist > 0) {
      score = change + (70 - rsi.value) / 20;
      direction = "long";
    }
    if (change < -0.3 && rsi.value > 30 && macd.valueMACDHist < 0) {
      score = Math.abs(change) + (rsi.value - 30) / 20;
      direction = "short";
    }
    if (change < -5 && rsi.value < 35 && bbPos < 0.2) {
      score = Math.abs(change) * 1.5;
      direction = "long (mean reversion)";
    }
    if (change > 5 && rsi.value > 65 && bbPos > 0.8) {
      score = change * 1.5;
      direction = "short (mean reversion)";
    }

    return {
      symbol: t.symbol, price: t.lastPrice,
      change24h: change.toFixed(2) + "%",
      rsi: rsi.value.toFixed(1),
      macd: macd.valueMACDHist > 0 ? "bullish" : "bearish",
      bbPosition: (bbPos * 100).toFixed(0) + "%",
      fundingRate: (parseFloat(t.fundingRate) * 100).toFixed(4) + "%",
      score, direction,
    };
  }),
);

const candidates = analyses.filter((a) => a.score > 0).sort((a, b) => b.score - a.score);
return { topSetups: candidates.slice(0, 3), noSignal: analyses.filter((a) => a.score === 0).map((a) => a.symbol) };
```

**Present trade ideas with reasoning:** signal, confirmation, entry/stop/target, risk/reward. If nothing looks good, say so — "market's choppy, I wouldn't trade right now" builds more trust than a forced idea.

## Deep-dive on a symbol

When the user asks about a specific symbol:

```typescript
const symbol = "BTCUSDT"; // replace

const [ticker, ob, instruments] = await Promise.all([
  eterna.getTickers(symbol),
  eterna.getOrderbook(symbol, 50),
  eterna.getInstruments(symbol),
]);

const [rsi1h, rsi4h, rsi1d] = await Promise.all([
  eterna.getRsi(symbol, "1h"), eterna.getRsi(symbol, "4h"), eterna.getRsi(symbol, "1d"),
]);
const [macd1h, macd4h] = await Promise.all([
  eterna.getMacd(symbol, "1h"), eterna.getMacd(symbol, "4h"),
]);
const [bb1h, vwap] = await Promise.all([
  eterna.getBollingerBands(symbol, "1h"), eterna.getVwap(symbol, "1h"),
]);

const t = ticker.list[0];
const price = parseFloat(t.lastPrice);
const totalBids = ob.b.reduce((s, [, q]) => s + parseFloat(q), 0);
const totalAsks = ob.a.reduce((s, [, q]) => s + parseFloat(q), 0);

return {
  price: t.lastPrice, change24h: (parseFloat(t.price24hPcnt) * 100).toFixed(2) + "%",
  volume24h: "$" + (parseFloat(t.turnover24h) / 1_000_000).toFixed(1) + "M",
  fundingRate: (parseFloat(t.fundingRate) * 100).toFixed(4) + "%",
  rsi: { "1h": rsi1h.value.toFixed(1), "4h": rsi4h.value.toFixed(1), "1d": rsi1d.value.toFixed(1) },
  macd: {
    "1h": { signal: macd1h.valueMACDHist > 0 ? "bullish" : "bearish", hist: macd1h.valueMACDHist.toFixed(2) },
    "4h": { signal: macd4h.valueMACDHist > 0 ? "bullish" : "bearish", hist: macd4h.valueMACDHist.toFixed(2) },
  },
  vwap: vwap.value.toFixed(2),
  bollingerBands: {
    upper: bb1h.valueUpperBand.toFixed(2), mid: bb1h.valueMiddleBand.toFixed(2), lower: bb1h.valueLowerBand.toFixed(2),
    width: (((bb1h.valueUpperBand - bb1h.valueLowerBand) / bb1h.valueMiddleBand) * 100).toFixed(2) + "%",
  },
  orderbook: {
    bidVolume: totalBids.toFixed(2), askVolume: totalAsks.toFixed(2),
    imbalance: (totalBids / totalAsks).toFixed(3),
    spread: (parseFloat(ob.a[0][0]) - parseFloat(ob.b[0][0])).toFixed(2),
  },
};
```

Synthesize — identify confluence (multiple indicators agreeing) and conflicts (RSI overbought but MACD still bullish). Lead with the actionable insight.

## SDK methods used

| Method | Returns |
|--------|---------|
| `eterna.getTickers(symbol?)` | `{ list: [{ symbol, lastPrice, price24hPcnt, turnover24h, fundingRate, ... }] }` — all values are strings |
| `eterna.getOrderbook(symbol, limit?)` | `{ b: [[price, qty], ...], a: [[price, qty], ...] }` — strings |
| `eterna.getInstruments(symbol?)` | `{ list: [{ lotSizeFilter, priceFilter, leverageFilter }] }` |
| `eterna.getRsi(symbol, interval, period?)` | `{ value: number }` — 0-100 |
| `eterna.getMacd(symbol, interval, ...)` | `{ valueMACD, valueMACDSignal, valueMACDHist: number }` |
| `eterna.getBollingerBands(symbol, interval, ...)` | `{ valueUpperBand, valueMiddleBand, valueLowerBand: number }` |
| `eterna.getVwap(symbol, interval)` | `{ value: number }` |
| `eterna.getEma(symbol, interval, period?)` | `{ value: number }` |
| `eterna.getSma(symbol, interval, period?)` | `{ value: number }` |

Valid intervals: `1m`, `5m`, `15m`, `30m`, `1h`, `2h`, `4h`, `1d`, `1w`. TA methods return numbers (no parseFloat needed). Bybit ticker values are strings (parseFloat needed).
