# Solana Autonomous Trading System

Monorepo for an autonomous Solana trading system: on-chain Anchor program, Node.js backend, AI risk manager, Next.js dashboard, Telegram Mini App + bot, React Native client with Live Activities, and an independent emergency killswitch.

> **Honest warning:** Memecoin trading is adversarial (honeypots, rug pulls, MEV, sandwich attacks). Nothing in this repo is a profit guarantee. Run on devnet first, then mainnet with small capital only.

---

## Instructions for AI assistants (GitHub Copilot, ChatGPT, Claude)

Read this whole file before generating code. Follow these rules in every file you write.

### Output rules
- Write complete, working files. No `TODO`, no `...`, no placeholder functions, no missing imports.
- TypeScript strict mode everywhere. No `any`. Validate all external input with `zod`.
- Rust/Anchor: checked math only, typed custom errors, no `unwrap()` in program code.
- Add tests for every module you create.
- Build in the milestone order below. Do not skip ahead.

### Safety rules (non-negotiable)
1. **No raw private keys** in `.env`, source, logs, or git. Use the `Signer` interface in `packages/shared` backed by an encrypted keystore (dev) or KMS/HSM (prod).
2. **Hard limits** (max position size, max daily loss, max open positions, max slippage) are enforced in backend code AND in the on-chain program. The AI layer may only tighten limits, never loosen or bypass them.
3. **Fail closed.** Any RPC error, timeout, or ambiguous data in the anti-rug gate means reject the token.
4. **Simulate before send.** Every transaction is simulated first. Orders use idempotent IDs so retries never double-send.
5. **Write-ahead intent log.** Record the intended trade in the database before sending it.
6. **Killswitch is a separate process** from the trading loop. It must keep working if the trading loop hangs or crashes.
7. **Anchor account validation:** check owner, signer, PDA seeds and bump, `has_one`, and constraints on every account. Prevent account substitution. Handle rent exemption and close accounts safely.
8. **Telegram:** allowlist by user ID, validate Mini App `initData` signature server-side, rate limit, and require confirmation for destructive commands.

---

## Architecture

| Layer | Path | Responsibility |
|---|---|---|
| On-chain | `programs/vault` | Anchor program: capped vault, position limits, pause flag, authority checks |
| Backend | `apps/backend` | DEX scanner, anti-rug gate, Jupiter/Jito execution router, persistence |
| AI manager | `packages/ai` | Market regime classifier, risk sizing, position monitor, dynamic exits |
| Risk | `packages/risk` | Pure-function limit checks shared by backend and AI |
| Killswitch | `apps/killswitch` | Independent circuit-breaker daemon |
| Web | `apps/web` | Next.js App Router dashboard, `@solana/wallet-adapter` |
| Telegram | `apps/tma` | Telegram bot + Mini App web view |
| Mobile | `apps/mobile` | Expo / React Native, push notifications, iOS Live Activities |
| Shared | `packages/shared` | Types, zod schemas, constants, `Signer` interface |

### Directory tree

```
.
├── .github/workflows/ci.yml
├── .gitignore
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Anchor.toml
├── programs/vault/
├── apps/
│   ├── backend/
│   ├── killswitch/
│   ├── web/
│   ├── tma/
│   └── mobile/
└── packages/
    ├── shared/
    ├── risk/
    └── ai/
```

---

## Anti-rug gate (all checks must pass)

- Mint authority revoked
- Freeze authority revoked
- LP burned or locked
- Top-holder concentration below threshold
- Honeypot check: simulated sell succeeds
- Minimum liquidity met
- Any RPC failure means reject

## Killswitch triggers

- Daily loss cap reached
- Consecutive RPC failures above threshold
- On-chain state and database state drift
- Manual `/panic` command

On trigger: halt scanner, cancel pending orders, exit positions, lock state until a human unlocks.

---

## Milestones (build in this order)

1. Monorepo skeleton, tooling, CI
2. Anchor program + tests
3. Backend core: scanner, anti-rug gate, execution router, SQLite (WAL) persistence
4. Killswitch daemon
5. AI manager
6. Web dashboard
7. Telegram bot + Mini App
8. Mobile app + Live Activities

After each milestone: how to run it, how to test it on devnet, and what to verify before moving on.

---

## Setup

```bash
# prerequisites: Node 20+, pnpm, Rust, Solana CLI, Anchor CLI
git clone https://github.com/<your-user>/<your-repo>.git
cd <your-repo>
pnpm install

# devnet
solana config set --url devnet
solana-keygen new --outfile ~/.config/solana/dev.json
solana airdrop 2

# program
anchor build
anchor test

# apps
pnpm turbo run dev
```

Copy `.env.example` to `.env` and fill in RPC URL and bot tokens. **Never put a private key in `.env`.**

## Git workflow

```bash
git init
git add .
git commit -m "chore: initial monorepo skeleton"
git branch -M main
git remote add origin https://github.com/<your-user>/<your-repo>.git
git push -u origin main
```

Use one branch per milestone (`feat/m2-anchor-program`) and open a pull request for each.

## Defaults

- Network: devnet. Mainnet only behind an explicit `NETWORK=mainnet` flag.
- Custody: hot wallet with a hard-capped balance; profits sweep to a cold wallet.
- Database: SQLite in WAL mode, migrate to Postgres later.
