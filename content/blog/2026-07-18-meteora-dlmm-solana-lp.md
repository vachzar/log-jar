+++
title = "Meteora DLMM — Yield Farming di Solana"
date = 2026-07-18
description = "Setup bot yield farming Meteora DLMM pool USDC/USDT di Solana — wallet, Jupiter swap, LP position, web dashboard, cron monitoring."
[taxonomies]
tags = ["solana", "defi", "dlmm", "meteora", "yield-farming", "liquidity"]
+++

**Dynamic Liquidity Market Maker (DLMM)** adalah protokol AMM generasi terbaru di Solana dari **Meteora**. Bedanya sama AMM klasik (Uniswap, Raydium) yang nyebarin liquidity di kurva harga *infinite*, DLMM pakai sistem **bin-based** — liquidity hanya terkonsentrasi di rentang harga tertentu yang lo tentuin sendiri.

Ini bikin **capital efficiency** jauh lebih tinggi. Lo bisa dapetin yield 1-3% per bulan di pool stablecoin dengan modal kecil ($50-$100), sesuatu yang mustahil di AMM tradisional karena liquidity lo bakal tersebar terlalu tipis.

<!-- more -->

## Kenapa Solana?

- **Gas fee murah** — ~$0.01-0.02 per transaksi, cocok buat bot/automation
- **Finality cepat** — confirm dalam detik
- **Ekosistem DeFi mature** — Jupiter, Meteora, Kamino, Orca all mature
- **Liquidity dalem** — pool stablecoin USDC/USDT punya volume jutaan dolar

## Kenapa USDC & USDT?

Pool **USDC/USDT** adalah stablecoin pair — kedua token dipatok ke USD. Ini berarti **minimal impermanent loss (IL)**. Cocok buat:

- Yield konsisten tanpa risiko harga
- **Spot strategy** dengan range sempit (±8 bin = ±0.08%)
- Fee accumulation dari swap volume (arbitrageur nyari arbitrase USDC/USDT)

SOL dipake buat **gas fee** — setiap transaksi butuh ~0.001-0.005 SOL. Cukup sisain 0.1-0.5 SOL.

## The Setup

### Wallet
- Address: `5VdJzgqRcFaFjs9DNTxHYbxrjd8y3bsCGHX68LNfBjqz`
- Keypair: `~/.config/solana/id.json`
- Fund via Indodax (support Solana network) → swap via Jupiter

### Pool
- Pool: `ARwi1S4DaiTG5DX7S4M4ZsrXqpMD1MrTmbu9ue2tpmEq`
- Token X: USDC (`EPjFWdd5...`)
- Token Y: USDT (`Es9vMFrza...`)
- Active bin: #6 (price ~$1.0006)
- Bin step: 1 (0.01% per bin)

### Strategy: Spot ±8 bin
- Range: #-2 sampai #14
- Capital efficiency tinggi — liquidity cuma di range sempit deket harga pasar
- Auto rebalance kalo harga keluar range

### Position
- Position: `DzEreXQnYmjkiA8GtzZ7JhtdW5XCvvqUzWfLbcvc8uuM`
- Deposited: ~$39.61 USDC + $30.24 USDT
- Status: ✅ IN RANGE

## Scripts (`~/meteora-bot/`)

| Script | Fungsi |
|--------|--------|
| `monitor.js` | Cek wallet, pool, position, fees |
| `create-position.js` | First-time LP deposit |
| `rebalance.js` | Remove + create ulang posisi |
| `swap-usdc-to-usdt.js` | Swap USDC→USDT via Jupiter |

## Dashboard Web

Flask app di port 5050 — monitoring realtime:
- Wallet balance (SOL, USDC, USDT)
- Pool info (active bin, price)
- Position (range, in/out, fees)
- Bin range visualization
- Total value estimasi dalam IDR

Auto-start via systemd user service.

## Key Learnings

### 1. DNS Block
`quote-api.jup.ag` gak resolve (AdGuard di CasaOS). Solusi: pake `@jup-ag/api` npm package.

### 2. Borsh BN.js Bug
Node.js 22 + nested `@coral-xyz/borsh` di DLMM SDK throw `src.toArrayLike is not a function`. Fix: patch `BNLayout.encode` dengan guard konversi manual.

### 3. SDK Pattern
`initializePositionAndAddLiquidityByStrategy` return **Transaction** (bukan `{txId}`). Parameter `user` pake PublicKey, `totalXAmount`/`totalYAmount` harus BN.js.

### 4. Cron Monitoring
Setup cron tiap 6 jam via Hermes — notif Telegram kalo position out of range.

## Total Investasi

| Item | USD | IDR (≈) |
|------|-----|---------|
| LP Position | ~$69.85 | ~Rp1.255.000 |
| SOL (gas) | ~$31.57 (0.42 SOL) | ~Rp567.000 |
| **Total** | **~$101.42** | **~Rp1.822.000** |

## Referensi

- [Meteora DLMM Docs](https://docs.meteora.ag/)
- [Jupiter Swap API](https://station.jup.ag/docs/apis/swap-api)
- Pool: `ARwi1S4DaiTG5DX7S4M4ZsrXqpMD1MrTmbu9ue2tpmEq`
- Wallet: `5VdJzgqRcFaFjs9DNTxHYbxrjd8y3bsCGHX68LNfBjqz`
