# Eterna Trading — Openclaw Plugin

Let your openclaw AI agent trade on [Eterna](https://eterna.exchange) using the `eterna` CLI.

## Prerequisites

Install and authenticate the Eterna CLI:

```shell
npm install -g @eterna-hybrid-exchange/cli
eterna login
```

## Installation

**As a plugin** (recommended):

```shell
openclaw plugins install @eterna-hybrid-exchange/openclaw-plugin
```

**As a skill** (for users who prefer skills):

```shell
openclaw skills install @eterna-hybrid-exchange/eterna-trading-skill
```

## Skills included

### `eterna_trading` — always active (router)

Detects user state on first message and routes to the right skill. New users get an automatic market scan and guided onboarding flow. Returning traders see their positions.

### `market_scan` — market analysis and trade ideas

Live market briefings, TA scanning, deep-dives on specific symbols, and trade idea generation with entry/stop/target levels.

### `deposit` — deposit and fund trading account

Guides through deposit address, chain selection, deposit monitoring via `getDepositRecords()`, transfer from Funding to Trading wallet, and balance confirmation.

### `withdraw` — withdraw funds

Check withdrawable balance, submit withdrawal, and track status.

### `open_position` — place trades

Pre-trade checks (balance, existing positions, instrument specs), trade proposals with clear margin/notional breakdown, and execution with proper TP/SL.

### `close_position` — close positions and manage orders

Close positions, cancel orders, and modify TP/SL on existing positions.

## Releasing a new version

> **First release only:** npm OIDC trusted publishing must be configured before tagging. In the npm UI, go to each package's settings and enable "Granular access tokens" with the GitHub repo as the trusted publisher. If OIDC is not yet configured, add a `NODE_AUTH_TOKEN` secret to the GitHub repo instead and add `env: { NODE_AUTH_TOKEN: ${{ secrets.NODE_AUTH_TOKEN }} }` to the publish steps in `.github/workflows/release.yml`.

1. Update `version` in `package.json` and `packages/skill/package.json` to the same value
2. Commit: `git commit -m "chore: bump version to X.Y.Z"`
3. Tag and push: `git tag vX.Y.Z && git push origin vX.Y.Z`
4. GitHub Actions verifies version consistency and publishes both packages automatically

## License

MIT
