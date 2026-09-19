# Spout UX & Product Feedback Report

> **9-day real-usage test of RWA-collateralized borrowing on Solana**
> Submitted for the [Superteam x Spout Finance Beta Intelligence Challenge](https://earn.superteam.fun/listing/product-feedback-spout-finance)
> By [@0xinaids](https://x.com/0xinaids) — September 9–17, 2026

---

## Overview

This report documents a 9-day hands-on test of [Spout Finance](https://spout.finance) — a Solana-based DeFi brokerage that lets users borrow stablecoins at 0% interest against tokenized US equities, funded by a covered-call strategy on the collateral pool instead of borrower interest.

Testing was done with a diversified real position (6 assets, varying volatility), no guided tutorial, full documentation review before touching the app, DevTools inspection, on-chain verification via Solscan, and Lighthouse audits on both desktop and mobile.

---

## Proof of Usage

| Action | Asset | Amount | Status |
|---|---|---|---|
| Buy | GOOG | $2.99 | Completed |
| Buy | BSOL, MSTR, AAPL, PFE, GS | $5+5+5+5+3 | Completed |
| Sell (partial) | GOOG, PFE | — | Stuck at `placeSellOrder`, no `fulfillSellOrder` |

**Wallet:** `DHG4p1tKiXuQS2oYUMAnxR1P4YDgzGdkQfzJZfYoRnNV` — [Solscan devnet ↗](https://solscan.io/account/DHG4p1tKiXuQS2oYUMAnxR1P4YDgzGdkQfzJZfYoRnNV?cluster=devnet)

Portfolio monitored for 9 consecutive days: from $2.99 (initial fill) to a peak of $25.94 (Sep 17), with LTV/Available Borrowing Power recalculating correctly in real time throughout.

---

## Findings Summary

### Critical

| # | Finding |
|---|---|
| FP-APP-16 | Sell orders marked "Failed" but stuck mid-execution on-chain — `placeSellOrder` with no matching `fulfillSellOrder` |
| FP-APP-20 | Entire portfolio shows "No Holdings" / "No Metrics" despite data existing (chart still shows historical peak) |
| FP-MOBILE-1 | Site returns zero content on mobile under Slow 4G throttling (NO_FCP) |
| FP-APP-5 | Phantom Wallet fully blocks any transaction — "this dApp may be malicious" |
| FP-APP-12 | Raw account error + systematic 500s on `/vault/deposit` and `/vault/borrow` |
| FP-APP-15 | Avg Cost / P&L wrong for AAPL — three mismatched numbers for the same position |
| FP-DOC-2 | Liquidation fee documented as 5% flat, shown as 8.8% in the worked example |
| FP-APP-4/6 | Public FAQ (schema.org, Google-indexed) claims KYC "enforced wallet-level"; zero KYC step observed in testing |

### High

| # | Finding |
|---|---|
| FP-APP-1 | "Borrow Cost" column contradicts the "0% Interest. Always." banner on the same screen |
| FP-APP-14 | Devnet environment not persistently flagged — confused even an experienced tester |
| FP-DOC-1 | Yield targets diverge across docs pages (7%/25% vs 9%/32%) |
| FP-DOC-3 | "No losing your upside" contradicts the real covered-call assignment mechanic |
| FP-APP-21 | Asset tickers in Open Orders replaced with garbled address fragments (regression) |

### Medium

| # | Finding |
|---|---|
| FP-APP-13 | Borrow panel opens pre-selected on an asset the user doesn't own |
| FP-DOC-4 | Undocumented term "LP reserve" in distribution flow |
| FP-DOC-5 | Insurance Fund circuit breaker has no public numeric threshold |
| FP-APP-7 | "Executing" status shown while market is closed (confirmed across 6 orders) |
| FP-APP-17 | Sell confirmation modal reuses "Your purchase has been confirmed" copy |
| FP-APP-19 | No "my holdings only" filter in the Trade/Sell table |
| FP-LANDING-1 | "2%" flash during landing page load before settling on the correct "0%" |

### Low

| # | Finding |
|---|---|
| FP-APP-8 | Share rounding in the post-purchase modal (resolved correctly in Portfolio view) |
| FP-APP-10 | Inefficient RPC polling (~150ms) instead of WebSocket subscription |
| FP-APP-11 | Missing SEO meta description |

---

## Main Suggestion — Reconcile "0% Interest" Banner with the Real Leverage Cost

The trade screen banner reads "0% Interest on borrowing. Always." directly above a "Borrow Cost" column showing 0.06%–0.86%/yr per asset — an unexplained contradiction on the same screen.

**Proposed fix:**

[![Current vs Proposed Mockup](https://raw.githubusercontent.com/xinaids/spout-ux-report/main/assets/screenshots/26-mockup-current-final.png)](https://raw.githubusercontent.com/xinaids/spout-ux-report/main/assets/screenshots/26-mockup-current-final.png)
[![Proposed](https://raw.githubusercontent.com/xinaids/spout-ux-report/main/assets/screenshots/27-mockup-proposed-final.png)](https://raw.githubusercontent.com/xinaids/spout-ux-report/main/assets/screenshots/27-mockup-proposed-final.png)

Rename the column to "Est. Leverage Cost," add an info icon, and bridge it with one line of copy: *"0% applies to your principal. Leverage above 1.0x carries a variable cost by asset."*

---

## What Works

- Excellent desktop performance — Lighthouse 100/100/100/91
- No leaked keys or secrets (verified via Network/Sources tab inspection)
- LTV and Available Borrowing Power recalculate correctly and in real time as market value changes
- Transparent closed-market behavior — "fills at 9:30 AM ET" honored exactly as promised
- Full Transaction History with CSV export
- Well-designed loss waterfall with concrete, documented numbers
- Earnings-week call skip per asset — a risk consideration few protocols implement
- Verifiable FinCEN MSB registration
- Lender/protocol fee split (80/20) consistent across the public FAQ and docs

---

## Links

- 📄 [Full Report](https://github.com/xinaids/spout-ux-report/blob/main/report.md)
- 🐦 [X Thread](https://x.com/0xinaids/status/2098174491742339135)
- 🎨 [Visual Mockups](https://github.com/xinaids/spout-ux-report/tree/main/assets/screenshots)
- 🏆 [Bounty Listing](https://earn.superteam.fun/listing/product-feedback-spout-finance)
