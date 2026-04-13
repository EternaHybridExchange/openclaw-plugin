# Eterna Trading — Openclaw Plugin & Skill

Let your openclaw AI agent trade on [Eterna](https://eterna.exchange) using the `eterna` CLI.

## Prerequisites

Install and authenticate the Eterna CLI:

```shell
npm install -g @eterna-hybrid-exchange/cli
eterna login
```

## Installation

**As a plugin** (recommended — more versatile):

```shell
openclaw plugins install @eterna-hybrid-exchange/openclaw-plugin
```

**As a skill** (for users who prefer skills):

```shell
openclaw skills install @eterna-hybrid-exchange/eterna-trading-skill
```

## Skills included

### `eterna_trading` — always active

Teaches the agent to use the `eterna` CLI for trading operations:

- `eterna balance` — check account balance
- `eterna positions` — view open positions
- `eterna orders` — view active orders
- `eterna execute <file>` — execute TypeScript trading code
- `eterna sdk --search <query>` — browse SDK method docs

### `onboarding` — explicit invocation only

A guided 5-phase onboarding flow for new end users (Discovery → Trust Building → First Deposit → First Trade → Preferences). Activate with `/eterna-trading:onboarding` or by asking the agent to onboard you.

Not suitable for fully autonomous trading agents — it assumes a human user is present.

## Releasing a new version

> **First release only:** npm OIDC trusted publishing must be configured before tagging. In the npm UI, go to each package's settings and enable "Granular access tokens" with the GitHub repo as the trusted publisher. If OIDC is not yet configured, add a `NODE_AUTH_TOKEN` secret to the GitHub repo instead and add `env: { NODE_AUTH_TOKEN: ${{ secrets.NODE_AUTH_TOKEN }} }` to the publish steps in `.github/workflows/release.yml`.

1. Update `version` in `package.json` and `packages/skill/package.json` to the same value
2. Commit: `git commit -m "chore: bump version to X.Y.Z"`
3. Tag and push: `git tag vX.Y.Z && git push origin vX.Y.Z`
4. GitHub Actions verifies version consistency and publishes both packages automatically

## License

MIT
