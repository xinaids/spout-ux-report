# Spout Finance — Raw Notes (Beta Intelligence Challenge)

Status: Day 0 — full docs sweep done, awaiting first app session.
Beta code obtained: [09/09/2026]
Test wallet: DHG4p1tKiXuQS2oYUMAnxR1P4YDgzGdkQfzJZfYoRnNV

---

## Reviewer profile (for Executive Summary)

- Full-stack Solana dev (Rust/Anchor), Superteam Brazil bounty hunter
- Previously tested DeFi lending/borrow (2nd place, Zodial — Superteam Germany)
- Approach: no official tutorial on the first session, document real hesitation

---

## Pre-loaded findings from the documentation sweep (to validate against the app)

### FP-DOC-1 — Target yield diverges across docs pages

**What happened:** `/introduction` and `/how-lending-works` show Senior ~9% / Junior ~32% APY. `/what` shows Senior ~7% / Junior ~25%+. The detailed math on `/lending-tranches` confirms 9%/32.8% as the real numbers (7% is just the "priority yield" before the excess split).

**Why it caused friction:** A user who only reads `/what` (the overview page) forms an expectation of returns lower than the real one — or, worse, if they only read the landing page, higher than what the more detailed educational page suggests. Inconsistent cost/yield disclosure is exactly the kind of finding that won 1st place for Minkhanov on Zodial.

**Severity:** High (candidate for Critical if the app UI also uses the 7% number)

**Suggested improvement:** Standardize every APY mention to use the blended number (9%/32%), with a clear note that "priority yield" (7%) is just the Senior floor before the excess split.

**TODO:** check which number appears on the app's deposit screen.

---

### FP-DOC-2 — Liquidation fee: 5% flat (marketing) vs 8.8% in the worked example

**What happened:** `/liquidation` and `/fee-structure` claim a flat 5% fee, comparing favorably to the "10-15% DeFi standard." But Dave's scenario in `/scenarios` uses "NVDA's liquidation fee (8.8%, its per-asset buffer)" — language that suggests a variable per-asset fee, not flat.

**Why it caused friction:** If the real fee varies with asset volatility (which makes sense from a risk standpoint), the "5% flat, better than the market" marketing claim may be hiding that volatile assets (NVDA, MSTR) pay nearly double the advertised rate.

**Severity:** Critical (directly affects the decision of how much collateral to bring and which asset to choose)

**Suggested improvement:** The fees page should show the fee per asset (as it already does for LTV/cycle in the `/supported-collateral` table), not just a generic flat number.

**TODO:** simulate/observe a liquidation (or the liquidation preview screen) on at least 2 assets of different volatility (e.g., NVDA/MSTR vs PFE/GLD) and compare the shown fee.

---

### FP-DOC-3 — "Keep every share, no losing your upside" vs the real assignment mechanic

**What happened:** Landing page / `/introduction`: "No selling, no taxable event, no losing your upside." But `/covered-call-strategy` and `/options-assignment` make clear that upon assignment, shares ARE sold at the strike — the upside above the strike for that cycle is sacrificed. The FAQ calls this a "bounded outcome," but the landing page claim ignores this scenario entirely.

**Why it caused friction:** A user who only reads the landing page (any product's entry funnel) forms an expectation of zero upside risk that isn't true. This is "leverage/risk framing" — exactly the angle that separated 1st from 2nd place on Zodial.

**Severity:** High

**Suggested improvement:** The landing page should swap "no losing your upside" for something like "no losing your upside below the strike," or link directly to the assignment FAQ.

**TODO:** check whether the app, in the lock/borrow flow, warns about assignment risk BEFORE the user confirms the transaction, or only afterward in the docs.

---

### FP-DOC-4 — "LP reserve" mentioned with no explanation anywhere

**What happened:** `/distribution` mentions a deduction for "protocol fee, insurance fund contribution, and the LP reserve" before distributing to the lender. But `/settlement-flow` (the more detailed page for the same flow) only lists protocol fee + insurance fund + senior priority + excess split — no "LP reserve." The term doesn't appear in the glossary.

**Why it caused friction:** An undocumented technical term that directly affects how much the lender receives. Zero explanation of size, purpose, or accrual rule.

**Severity:** Medium

**Suggested improvement:** Add "LP reserve" to the glossary and reconcile both pages to describe the same settlement flow identically.

**TODO:** check the distribution screen in the app — does the term appear there? At what value?

---

### FP-DOC-5 — Circuit breaker with no public threshold

**What happened:** `/circuit-breakers`: "If the fund draws down past a defined threshold, new cycles for affected assets pause." The number is never revealed on any page.

**Why it caused friction:** This is an important structural safety mechanism (what happens when the Insurance Fund has already drained significantly), but the user has no way to gauge how close the protocol is to this limit at any given time.

**Severity:** Medium

**Suggested improvement:** Publish the numeric threshold (e.g., "pauses when the fund falls below X% of target") and show it in the fund transparency UI.

**TODO:** check whether the app shows the Insurance Fund's balance/history (docs say it should: "current fund balance, target level, contribution rate, and drawdown history are visible in the app").

---

## Financial Safety Analysis angles (to explore during testing)

1. **Weekend/overnight gap risk**: `/oracles` says price updates at "reduced frequency" outside US market hours. Stocks have weekend/after-hours gaps. If NVDA opens 15% down on a Monday, does the Health Factor only react once the market opens? Is there any protection?
2. **30% tax withholding for non-US** (`/tax`) — relevant for a BR audience, a specific content angle for Superteam Brazil.
3. **BSOL as collateral** — Solana exposure via a stock/ETF, a natural angle to pull in a crypto-native audience (the docs' "Onchain Native" persona).
4. **Real sensitivity of the liquidation fee** — test with at least 2 assets of different volatility.

---

## FP-APP-10 — Technical recon (DevTools): no key leaks, but inefficient RPC polling

**What happened:** Inspecting the Network tab (Fetch/XHR) on the Trade screen shows direct browser calls to the public RPC `api.devnet.solana.com`, with no backend proxy. Mostly repeated `getTokenAccountBalance` calls for the same two accounts (`8KTa1mJHsy6UfswXP8HxQBVzcH4jgRkwh3hHqLDLLUFf`, `3rAWFGFUitzCXCouzU3fdwdYCaCBfSVCX3VYogyGeMc8`) plus `getAccountInfo` and `getMultipleAccounts`, all repeating every ~140-160ms.

**Security (positive):** No API key or secret visible in these headers/payloads — makes sense, since the public devnet RPC requires no auth. No leak here. Sources tab (searching for 'sk_', 'api_key', 'secret', etc. in the JS bundle) not yet checked.

**Performance (technical finding, non-critical):** The pattern is repeated active polling instead of WebSocket subscription (`onAccountChange`/`accountSubscribe` from `@solana/web3.js`). Works fine on devnet under low load, but won't scale — on mainnet, a public RPC would rate-limit quickly at this request volume; even with a dedicated RPC (Helius, likely what the product will use in production), constant polling wastes RPC credits compared to subscriptions.

**Severity:** Low/Info — not a bug, an architecture optimization suggestion.

**TODO:** still need to check the Sources tab (secret search in the bundle) and Application tab (localStorage/cookies) — pending.

---

## FP-APP-11 — Lighthouse baseline (desktop, /buy) — excellent, and Sources tab with no obvious secrets

**Lighthouse scores (desktop, beta.spout.finance/buy, 09/09/2026 8:39 PM GMT-3):**
- Performance: 100 (FCP 0.2s, LCP 0.6s, TBT 20ms, CLS 0, SI 0.4s)
- Accessibility: 100
- Best Practices: 100
- SEO: 91 (only issue: no meta description)
- Agentic Browsing: 2/2

**Why this matters:** A result well above the DeFi average — most protocols tested before (Hobba, Zodial) had Performance scores in the 60-80 range. Worth citing as a strong "what works" point in the report; few competitors will likely run this audit, so it's a technical-rigor differentiator.

**One real improvement point:** SEO 91 due to missing `<meta name="description">` — an easy quick win to report.

## FP-APP-14 — Devnet vs. real money signaling isn't clear enough in the UI

**What happened:** After a full day of testing, the genuine question "did I put in real money?" came up — despite the initial "Wallet verified for devnet" toast and the "DEVNET TEST FUNDS" section in the wallet panel, at no subsequent point in the UI (Trade, Borrow, Portfolio screens) is there a persistent, visible indicator of "you're in test mode/devnet" while navigating and operating.

**Why it caused friction:** If an experienced Solana developer — who literally configures devnet/mainnet infra for a living — had a moment of "wait, is this real?", that's a strong signal that the environment indicator is insufficient for any less technical user. The "verified for devnet" toast appears once and disappears — there's no persistent badge, different theme color, or persistent label (like "TESTNET" in the header) reminding the user across every subsequent screen.

**Severity:** High — this is about trust and financial clarity, the core of any DeFi product. Easily fixable, with a large impact on the user's psychological sense of safety.

**Suggested improvement:** A persistent header badge (e.g., "DEVNET" in yellow/orange, always visible) while the beta runs on testnet, not just a toast that disappears. This kind of flag is standard in test financial products (sandboxed digital banks, exchange demo modes, etc.).

---

## FP-APP-12 — Confirmed as the report's #1 technical finding: systematic 500 on /deposit and /borrow

**What happened:** On the `/borrow` screen, with the GOOG position already active, the borrow panel displayed the raw message: `CollateralType: unexpected length 213 (expected 165, or 149 pre-migration)`.

**Why this is the report's strongest technical finding:** This isn't a UI error message — it's literally an **on-chain account struct deserialization error** (a classic Anchor/Borsh pattern: the parser expects an account with a specific byte size — 165 bytes in the current layout, or 149 in the "pre-migration" layout — and received 213 bytes, matching neither). This suggests one of two serious scenarios:
1. **Incomplete schema migration**: the program was updated (account layout migration), but old/new accounts with inconsistent sizes are being read by the same parser, and the "graceful fallback" to the pre-migration layout (149 bytes) doesn't cover this case (213 bytes).
2. **Unhandled error leaking to the client**: even if this is a known/expected backend case, exposing this internal error string (which reveals program implementation details: field names like `CollateralType`, exact struct sizes) directly in the interface is bad error-handling practice — it gives debugging info to any attacker mapping the program's account structure, and scares/confuses the end user who has no idea what it means.

**Severity:** Critical — both from a functional angle (the borrow flow may be broken for this specific account/collateral) and a security angle (leaking internal implementation details, the kind of finding sought in an audit — directly aligned with "SVS-8" and other prior Anchor account review work).

**Suggested improvement:** (1) Handle this error client-side with a human message ("Could not load your collateral position — try again or contact support"), never expose the parser's raw error string. (2) On the backend/program side, investigate why this specific account (this user's GOOG collateral account, created in the last 24h) is coming back with 213 bytes — if it's a newly created account, it shouldn't have this old-schema migration issue.

**ROOT CAUSE CONFIRMED (Console, screenshot 17):** two API calls failed with **status 500**:
- `api/vault/deposit?us...YoRnNV&ticker=XOM:1`
- `api/vault/borrow?use...YoRnNV&ticker=XOM:1`

Both for the `ticker=XOM` parameter — the same wrong asset identified in FP-APP-13 (the panel opens defaulted to XOM, an asset the user holds no position in). This connects both findings to a single root cause: **the frontend tries to preload vault data (deposit/borrow) for the default ticker (XOM) before the user even selects an asset; since no position/vault exists for that user+XOM pair, the endpoint returns 500; and the client, while trying to process this error response as if it were account data, generates the raw deserialization error that leaks onto the screen ("CollateralType: unexpected length...").**

This is a MUCH simpler and less alarming explanation than the original "broken schema migration" hypothesis — it's not an on-chain account layout bug, it's an HTTP error-handling error on the client: a 500 (probably a "vault not found" case wrongly categorized as a server error instead of a 404) being processed as if it were a valid account payload.

**Revised severity:** still High (no longer "Critical blockchain-level technical bug," but still a real error-handling bug that leaks internal details and can confuse/scare users), downgraded from "possible on-chain data corruption" to "HTTP error-handling UX bug + endpoint returning the wrong status code (500 instead of 404 for 'vault doesn't exist')."

**Suggested improvement:** (1) Backend: return 404 (not 500) when the vault doesn't exist for that ticker/user — 500 implies a server error, not "resource not found," which is semantically different and cheaper to handle client-side. (2) Frontend: never initialize the borrow panel with an API call for a ticker the user doesn't own — load empty/neutral until the user selects a real asset (fixes this and FP-APP-13 at the same time). (3) Never let a parsing exception leak as raw text into the UI — always have a human fallback.

**CRITICAL UPDATE — not specific to the wrong ticker (screenshot 18):** with GOOG correctly selected (the asset the user actually owns), the SAME endpoints keep returning 500: `borrow?userAddress=DHG4p1...&ticker=G...` and `deposit?userAddress=DHG4p1...&ticker=G...`. So the problem is systematic in these two endpoints, not specific to the wrong default (XOM) — this rules out the earlier "simple root cause" hypothesis and reopens the question.

**Serious side effect observed now:** despite the persistent 500, most of the panel works correctly with locally/client-side computed data — Health Factor 6.23 (green, healthy), correct "0.009170935 GOOG / $3" holdings, functional LTV slider. BUT a warning banner appeared at the top: **"Borrowing $0.48 exceeds the $0.00 this position supports."** — a message directly contradicting the rest of the screen (healthy Health Factor, $1.50 Max Borrow confirmed yesterday in the Portfolio). The most likely hypothesis: this specific warning IS fed by the (failed) response of these 500 endpoints — when the borrow-capacity call fails, the client seems to "fail closed," treating capacity as $0.00, but generates an incorrect and alarming alert instead of simply not blocking or using the already-correct locally computed value (proven correct by the Health Factor).

**Severity:** Critical (reinforced) — a real, reproducible failure of two central endpoints (`/deposit`, `/borrow`) in the product's most important flow, with a direct, visible UI effect (an incorrect, alarming warning that could make a user abandon a legitimate, safe operation).

**Reproduction confirmed across multiple amounts:** the same incorrect warning appeared for both a $0.08 and a $0.48 borrow attempt — confirms this isn't an edge case tied to a specific value, it's the endpoint failure state (`/deposit`, `/borrow` returning 500) propagating to any borrow attempt, regardless of amount. Further reinforces that this is a systematic flow bug, not an isolated edge case.

**FINAL CONFIRMATION:** with GOOG correctly selected (no longer XOM by mistake), Health Factor shows "1.00 / 37.40" (healthy) and "Est. borrower cost/yr: $0.00" — but the raw error **"CollateralType: unexpected length 213 (expected 165, or 149 pre-migration)"** still appears, alongside "You receive $0.08" and an active "Borrow $0.08 USDC" button. This definitively proves the error doesn't depend on the selected ticker (it's not the XOM-default bug) — it's a real, persistent failure in the account deserialization layer, coexisting with the 500s on the `/api/vault/deposit` and `/api/vault/borrow` endpoints. The most likely cause now: the user's collateral account has 213 bytes, and neither the current layout (165) nor the legacy one (149 pre-migration) match — suggesting a third account format not covered by the current parser, possibly introduced by a more recent feature (like Leverage itself) that wasn't correctly migrated for this specific endpoint.

---

## FP-APP-13 — Borrow panel opens on the wrong default asset ("Borrow against XOM" without owning XOM)

**What happened:** Entering `/borrow` with an active position only in GOOG (Position Value $2.99, Max. Borrow $1.50 — correctly shown in the table), the action panel on the right opened pre-selected as "Borrow against **XOM**" — an asset the user holds no position in (row shows $0.00 / 0 shares). The panel also shows "Health Factor: 1.00 / ∞" and "Borrow $0.00 USDC" in this state, which make sense for empty collateral, but the wrong asset being pre-selected is confusing — the user would logically expect the panel to open already showing the asset they actually own (GOOG), or at least a neutral/empty state, not a random asset with no position.

**Severity:** Medium — doesn't block the flow (clicking the GOOG row presumably fixes it), but it's a confusing first impression right at the entry point of the product's most important screen (borrow is the core value prop).

**TODO:** click the GOOG row in the table and confirm the panel correctly switches to "Borrow against GOOG" with the right values.

---

## Section 8 — Senior Analysis (draft, to refine before final submission)

*What would make Spout unbeatable — structural points beyond individual findings.*

**1. The conversion funnel breaks before the user even decides to buy.**
Not an isolated FP — it's a pattern. The "0% Interest. Always." banner contradicts the "Borrow Cost" column on the same screen (FP-APP-1), and Phantom physically blocks the transaction with "may be malicious" (FP-APP-5). A new user hits both warning signs *before* any informed decision about the product itself. Fixing internal UX bugs doesn't matter if the funnel already broke at the front door.

**2. The product treats "infrastructure error" and "business error" as the same thing.**
The raw account deserialization error (FP-APP-12) and the incorrect "$0.00 capacity" warning were both born from the same place: when an API call fails (500), the client doesn't distinguish "I couldn't fetch the data" from "you don't have capacity." This is symptomatic of an error-handling layer not designed with UX in mind — only the happy path. In fintech, a poorly communicated error state is a product failure, not just polish.

**3. The test environment (devnet) doesn't announce itself — a trust problem, not just a labeling one.**
FP-APP-14 isn't about "forgot to add a badge." It's about the fact that a financial product that will eventually handle real capital risk trains the user, from beta onward, not to pay attention to which network they're operating on. A habit formed now is a habit carried into launch.

**4. Documentation and the product tell two different stories about the same mechanism.**
We saw this repeatedly: target yield (7% vs 9%), liquidation fee (5% flat vs 8.8% in the example), "no losing your upside" vs the real assignment mechanic. Individually each looks like a typo. Together, they form a pattern: the marketing/docs layer was written before (or separately from) the final implementation, and nobody reconciled the two afterward. This is solvable with a process, not by a dev fixing each instance.

**5. The product's most interesting structural risk (RWA + closed market) is its least communicated one.**
Nothing in the product warns what happens to a user's Health Factor if a stock's price drops 15% on a Monday-morning gap, with the oracle running at reduced frequency over the weekend (finding from the docs sweep, `/oracles`). This is the most native and differentiating risk of Spout compared to pure-crypto DeFi — and it's precisely the least explained, both in the docs and the UI.

**6. The product hasn't decided whether it's "DeFi-native" or "regulated fintech" in how it communicates risk.**
It mixes "0% interest, no margin calls" language (reassuring, traditional fintech tone) with the real mechanics of covered calls and assignment (real, options-like risk). Mature DeFi-native protocols (Kamino, MarginFi) tend to expose risk mechanics more rawly and assumedly — the user knows they're in DeFi. Spout tries to soften, with fintech language, a product that structurally still carries derivative risk.

**TODO:** validate point 6 with a direct (feature-by-feature) comparison against Kamino or MarginFi before finalizing — haven't done that comparison yet.

---

## Section 6 — One-Sentence Test

*"Spout Finance lets you borrow stablecoins at 0% interest using tokenized stocks as collateral — funded by covered calls on those same assets — but still communicates liquidation and leverage risk as if it were traditional fintech, not the derivatives product it structurally is."*

---

## FP-APP-4/6 — ESCALATED: public FAQ (schema.org) claims mandatory KYC, directly contradicting the observed absence

**What happened:** Inspecting the source HTML of `spout.finance` (the SSR/fallback content served before React hydration — what Google, crawlers, and screen readers actually index), there's a `FAQPage` block in JSON-LD with public, structured answers. Two of them are explicit about compliance:

- *"How do I get started?"* → **"Connect a supported Solana wallet, complete a one-time KYC verification, and you can deposit equities..."**
- *"Is Spout compliant with US regulations?"* → **"Spout uses a regulated US broker-dealer for custody and options execution, enforces wallet-level KYC on all tokenized asset holders, and is structured to comply with applicable US securities laws."**

This is a public, indexable claim, specifically structured for SEO/AI crawlers — not just any marketing text, it's data that Google and AI assistants will cite as fact about the product.

**Why this raises the severity:** Previously (FP-APP-4/6) we treated the absence of KYC as possibly acceptable — "it's devnet, makes sense to skip verification." But this new evidence changes the framing: the site **publicly and structurally claims** that KYC is enforced "wallet-level" on "all tokenized asset holders," with no caveat for "except in beta/devnet." A user, auditor, or institutional partner reading this FAQ (or an AI citing it) forms a factually incorrect belief about the product's current state. This is different from "confusing UX" — it's a regulatory compliance claim not verifiable on the real product, the kind of thing security/compliance auditors flag as Critical.

**Severity:** **Critical** (upgrade from Medium/High) — a public, structured regulatory compliance claim (KYC "enforced... on all tokenized asset holders") not observable in the real tested flow, in any environment (not even a "KYC required in production, skipped in beta" notice appears).

**Suggested improvement:** Either (a) implement the KYC gate even in beta/devnet — at least a simulated flow, to validate enforcement before mainnet — or (b) add an explicit caveat to the public FAQ: "KYC enforcement is active on mainnet; the current beta on devnet does not require it." Leaving the public claim uncaveated while the real product doesn't comply is the kind of gap that can become a real regulatory problem, not just a UX one.

---

## Note — useful cross-confirmations (source HTML)

- **Lender/protocol fee split confirmed:** the public FAQ says "distributes 80% of the collected premium to lenders" — matches exactly the 20% protocol fee already documented in `/fee-structure` (100% - 20% = 80%). **Real consistency, no contradiction** — good sign, worth citing as "what works" (numbers matching across different sources).
- **Quantified average borrower cost:** the FAQ says "Historically this cost averages around 0.5% annualized across the portfolio" for the borrower's assignment risk — this could be the real explanation behind the Trade table's "Borrow Cost" (FP-APP-1/5), since the observed values (0.54%, 0.86%, 1.12%, 0.41%, 1.35%) hover around that average. Still not explained inside the app via a tooltip, but the public data exists — reinforces the recommendation to bring this explanation into the product, not just the external FAQ.
- **Minor inconsistency:** the public FAQ promises "double-digit APY across 11 assets" for lenders, but the documented Senior tranche is ~9% (not double-digit). Only the Junior (~32%) fulfills that promise. Low-severity finding — worth mentioning in passing.
- **200% collateralization** mentioned in the FAQ is consistent with a 50% LTV (50% LTV = 2x collateral = 200%) — just another way of expressing the same number, not an inconsistency.

---

**FP-LANDING-1 — "2% Interest" flash during load before settling on "0% Interest"**

**What happened:** While the landing page loads, the "The Cost of Borrowing" section briefly displays **"2% Interest — What the ultra-wealthy pay"** before settling on the correct final value: **"0% Interest — What you pay with Spout"** (confirmed by the user — the correct value does appear, there was just an intermediate state captured in screenshot 19). Likely an animated counter (count-up/count-down) or a hydration placeholder rendering a transient value before the real one settles.

**Why it caused friction:** Even being transient, a flash of incorrect content on the hero section — exactly the product's central claim — is a real risk: (1) any screenshot/screen-recording taken at that instant captures the wrong version (as happened here), (2) on slow connections or weaker devices, this intermediate state could last long enough to be read as a real claim, (3) it's the kind of "flash of incorrect content" that accessibility tools and screen readers can literally capture before the correction.

**Severity:** Medium — not a permanently wrong data bug (the final value is correct: "0% Interest — What you pay with Spout"), but a polish failure in a high-visibility spot, with a real risk of miscapture (as this very case demonstrates).

**Suggested improvement:** If it's an animated counter, start the count from a neutral value (e.g., "—%") instead of showing "2%" as an intermediate frame, or use a fade-in instead of a visible count. If it's a hydration state, ensure the initial SSR value is already correct (0%) before JS takes over.

---

**Total findings: 26** (5 documentation + 1 landing page + 1 mobile + 19 app), broken down by severity:
- **Critical: 8** — FP-APP-16 (sell marked "Failed" but actually executed — the most severe in the report), FP-MOBILE-1 (site doesn't render on mobile/Slow 4G), FP-APP-5 (total Phantom block, buy and sell), FP-APP-12 (deserialization error + systematic 500s), FP-APP-15 (wrong Avg Cost/P&L for AAPL), FP-DOC-2 (inconsistent liquidation fee), FP-APP-4/6 (publicly promised KYC, absent in practice), related FP-APP-13/9
- **High: 5** — FP-APP-1 (Borrow Cost vs "0% Always"), FP-APP-14 (devnet not flagged), FP-DOC-1 (inconsistent yield), FP-DOC-3 ("no losing upside" vs assignment)
- **Medium: 6** — FP-APP-13 (panel opens on wrong asset), FP-DOC-4/5 (LP reserve, circuit breaker with no threshold), FP-APP-7 (incorrect "Executing" status), FP-LANDING-1 ("2%" flash during hero load)
- **Low: 4** — FP-APP-8 (share rounding, resolved), FP-APP-10/11 (inefficient polling, SEO)

**The core thesis in 2 sentences:** the product has a genuinely differentiated financial model (RWA as collateral, yield via covered calls) and a backend with at least one real, serious bug (account deserialization), but the communication layer — marketing, docs, error messages — has not been reconciled with the implementation in any of those three places. The result is a product that scares users on first contact (Phantom) and then confuses them mid-flow (contradictory messages), even when the underlying mechanism works correctly.

**The 3 fixes that would move the needle immediately:**
1. Resolve the Phantom security flag (direct outreach to Phantom/Blowfish for allowlisting) — the highest, cheapest-to-remove conversion barrier
2. Fix the `/api/vault/deposit` and `/api/vault/borrow` endpoints returning 500 (technical finding #1) — affects the perceived reliability of the core feature
3. Reconciliation audit between docs/marketing and the real UI (yield, liquidation fee, "no losing upside") — a single review pass would resolve 3-4 findings at once

---

*Kamino chosen as the reference because it's Solana's largest money market, structurally similar (peer-to-pool, LTV + liquidation threshold + oracle-based), but purely crypto-native — highlighting where Spout's RWA model diverges.*

| Dimension | Spout | Kamino Lend |
|---|---|---|
| **Max LTV** | 50% flat, same for all assets | 70-80% typical, varies by asset (higher for blue-chip) |
| **Liquidation penalty** | Documented as 5% flat, but the worked example shows 8.8% (see FP-DOC-2) — **inconsistency not present at Kamino** | 2-10%, **explicitly variable and documented as such**: starts at 2% for fast liquidators, rises to 10% as LTV worsens |
| **Liquidation type** | Not documented whether total or partial | **Soft liquidation**: closes only the necessary fraction of the debt (e.g., 20%), softening the impact for the borrower |
| **Interest rate model** | Fixed 0% for the borrower, externally subsidized via covered calls | Floating, utilization curve (kink) — classic DeFi-native model, no external subsidy |
| **Oracle sources** | 1 oracle, reduced frequency outside US market hours (see `/oracles`) | **Pyth + Switchboard cross-referenced**, explicit redundancy |
| **Liquidation risk transparency** | Real fee depends on the asset but isn't communicated upfront to the user | Kamino documents the full range (2-10%) and the logic of how the penalty rises, publicly, before the user even enters the position |

**What this reveals (connects to Senior Analysis, point 6):** Kamino publicly assumes liquidation is part of the game and documents the mechanic with concrete, deliberately variable numbers. Spout communicates a "flat, low" fee (5%) that practice contradicts (8.8%) — not because the variable model is wrong (makes sense given volatile stocks carry more risk), but because the communication hasn't kept up with the implementation. A user coming from Kamino/MarginFi would enter Spout expecting the same level of quantitative transparency about liquidation risk, and wouldn't find it.

**Note on RWA vs pure crypto:** worth noting that Spout's more conservative LTV (50% vs 70-80%) makes sense given the additional RWA risk (T+1/T+2 settlement, closed weekend markets, gap risk) — this is a genuinely good design choice, not a criticism. The criticism is only about the communication of the liquidation mechanic, not the parameter itself.

---

## What works well (for Section 5 — What Works, to avoid an all-critical report)

- Well-designed loss waterfall, documented with concrete numbers (Insurance Fund → Junior → Senior)
- `/liquidation-example` is detailed and educational, above the DeFi average
- Earnings skip for individual stocks (no calls opened during earnings) — a risk consideration few protocols think to implement
- FinCEN registration as an MSB is verifiable, reinforces regulatory legitimacy
- Fee model is simple to understand at the macro level (0% borrow, 20% protocol fee on premium)

---

## FP-APP-1 — "0% Interest. Always." vs a "Borrow Cost" column with values > 0% on the same Trade screen

**What happened:** The banner at the top of the Trade screen prominently reads: "0% Interest on borrowing. Always, no matter the market conditions." A few inches below, the asset table has a column called "Borrow Cost" with real, non-zero values per asset: GS 0.86%/yr, XOM 0.80%/yr, MSTR 0.58%/yr, IBIT 0.55%/yr, GOOG 0.54%/yr, AAPL 0.37%/yr, PFE 0.07%/yr, GLD 0.06%/yr — and only NVDA, BSOL, and SMCI show 0.00%/yr.

**Why it caused friction:** This directly contradicts the product's central claim ("0% interest, always") on the very screen where the user decides what to buy. If "Borrow Cost" is actually a fee charged on the loan, the entire site's marketing headline is wrong/misleading. If it's something else (e.g., a risk metric disguised as "cost," or an asset's implicit covered-call spread), the field name is terrible and confuses any new user.

**Severity:** Critical — it's the product's claim #1 apparently being contradicted on the first screen any visitor sees, without even connecting a wallet.

**Suggested improvement:** Either (a) rename "Borrow Cost" to something that doesn't clash with "interest" (e.g., "Assignment Probability Cost" or "Est. Opportunity Cost"), with a tooltip explaining what it really is, or (b) if it really is disguised interest, fix the banner.

**RESOLVED (partially) via FP-APP-2:** the Leverage tooltip (screenshot 02) literally says: *"At 2.0x you put up half and borrow half at 0% interest. Higher leverage means more upside but also more **borrower cost**."* — meaning the product itself uses the phrase "borrower cost" in the same sentence that reaffirms "0% interest." This confirms that "Borrow Cost" in the Trade table isn't interest in the traditional sense, but also confirms there IS a real cost to taking leverage, which the "0% Interest. Always." banner simply doesn't communicate. The contradiction isn't an isolated copy bug — it's a consistent tension between the marketing headline and the real technical explanation, repeated in at least 2 places in the UI (banner vs column, banner vs tooltip).

**Still open:** the "Borrow Cost" column in the Trade table has no tooltip of its own (not tested yet) — we don't know whether it's literally the leverage's "borrower cost," or a different metric (e.g., that asset's implicit covered-call opportunity cost). Needs confirmation before closing the analysis.

---

## FP-APP-2 — "Leverage" slider (1.0x–2.0x) in the Buy flow doesn't appear on any documentation page

**What happened:** On the purchase panel (BUY/SELL NVDA), there's a "Leverage" slider marked 1.0x / 1.25x / 1.5x / 2.0x, alongside an info icon (not yet clicked). None of the 31 `/docs` pages mention a "leverage" mechanism at the time of purchase — the docs only describe implicit leverage via lock + borrow (50% LTV) after you already own the spAsset.

**Why it caused friction:** It's a real risk feature (up to 2x leverage on a simple purchase) with no prior explanation anywhere in the official docs. A user could enable 2.0x without understanding the mechanism behind it (probably synthetic margin buying via the protocol itself?) — this is buy-side leverage, distinct from the borrow-side LTV the docs cover extensively.

**Severity:** Critical — undocumented financial risk is the kind of finding that weighs most heavily in a lending/borrow UX bounty.

**Suggested improvement:** Document the leverage mechanism in the Buy flow with the same depth the docs give to borrowing (Health Factor, liquidation, etc.), or link the "i" tooltip to a dedicated docs page.

**PRIORITY TODO:** click the "i" icon next to "Leverage" and document exactly what appears. Test moving the slider to 1.25x and see if any warning, health factor preview, or screen change appears.

---

## FP-APP-3 — "Borrow" tab — REVISED: not a bug, it's an empty state (downgraded from Critical)

**What happened:** The `/borrow` page isn't empty due to a bug — it's an intentional empty state: "Trade stocks, unlock 0% borrowing. Every $10,000 of eligible stock lets you borrow up to $5,000 USDC at 0% interest, without selling anything." + an "Explore Stocks" button leading back to `/trade` (screenshot 03).

**Why it's actually decent UX:** Correctly confirms the 50% LTV (matches the docs: $5,000/$10,000 = 50%). Directs a collateral-less user to the right flow. Not friction — this is onboarding working as intended.

**Severity:** downgraded to Low/Info — worth citing in the report as an example of "what works" (well-guided empty states), not as a problem.

**New TODO:** buy a small position in NVDA or another asset first, then go back to `/borrow` to see the real lock+borrow screen, and then test the documented flow (Health Factor, LTV, etc.)

---

## FP-APP-4 — No KYC step anywhere in the connect flow (contradicts /security-and-compliance)

**What happened:** Full documented connect flow (screenshots 05-09): "Log in or sign up" (Privy, email or external wallet) → "Select your wallet" (Phantom/Solflare/Backpack/Jupiter/WalletConnect) → Phantom approval → toast "Wallet verified for devnet — you can place test orders now." → wallet panel shows devnet balance ($34.73 test USDC) already credited, no need to request from the faucet. At no point in this flow did any KYC, identity verification, or regulatory terms acceptance screen appear.

**Why it caused friction / matters:** `/security-and-compliance` in the docs states the protocol uses "Token-2022 transfer hooks" for on-chain KYC enforcement. If that's real, the enforcement should block or at least warn BEFORE letting the user get "verified for devnet" and cleared for test orders. It could be that (a) KYC is only required on mainnet and the devnet beta skips it on purpose (reasonable, but not explained anywhere in the UI), or (b) the enforcement actually only happens at the moment of transaction (e.g., tries to buy and IS blocked there). Both hypotheses need testing before reporting this as a bug.

**Severity:** High — it's a central compliance claim of the product (cited as part of the entire regulatory case) not observable in the real flow tested so far.

**TODO:** try a real purchase now that the wallet is connected and see if KYC appears at that moment (e.g., an "verify your identity" modal when confirming the order). If the order goes through with no verification, document it as a strong finding for the Senior Analysis.

**Positive note:** the connect UX itself is good — devnet auto-activated with test funds already available, no friction of manually requesting from the faucet to start (though the faucet link is also available if more is needed). This makes testing MUCH easier, worth citing as "what works."

---

- "Held 1:1 at Alpaca Securities · Reserves 100.2%" — good, this is the Proof of Reserve promised in the docs, visible with a real number (100.2%, slightly above 1:1, probably due to rounding or a buffer).
- Guided tour available ("Take a tour") — not yet clicked; UX-testing best practice recommends testing WITHOUT the tour first to capture real friction, and THEN comparing with what the tour explains (or fails to explain).
- "Market: Open" with a live clock (2:29:45) — good sign of market-hours transparency, relevant to the "what happens outside hours" angle (FP-DOC gap risks).
- Not every asset has a visible Market Cap (BSOL, GLD, IBIT show "—") — possible data gap, check whether this affects any calculation or is purely cosmetic.
- **Market: Closed now** (screenshot 04) — the Trade screen remains fully navigable and the Buy panel appears active even with the market closed. TODO: actually try submitting a buy order with the market closed (without a connected wallet, it's not yet possible to see whether the button changes from "Connect Wallet" to something like "Market Closed" or stays the same). Directly relevant to the weekend/overnight gap risk angle (see Financial Safety section above) — if the UI makes it seem like you can operate normally outside hours with no warning, that's friction.

---

## FP-APP-5 — Phantom shows a red "dApp may be malicious" + "domain is new" alert on a legitimate purchase

**What happened:** Upon confirming the GOOG purchase, Phantom Wallet displayed two stacked warnings: a red banner "This dApp may be malicious. Don't proceed unless you're sure it's safe." and a yellow banner "This domain is new. Only proceed if you trust this site." The confirm button shows "Confirm (unsafe)" instead of Phantom's default text.

**Why it caused friction:** This is the worst possible kind of friction in a financial flow — the user's own security software is saying "don't trust this" at the exact moment of conversion. For a new user with no prior context (e.g., someone clicking an ad or coming from content), this alert alone is enough to abandon the purchase. It's probably a false positive from Phantom's blocklist due to the domain's age (`beta.spout.finance` being a new/beta subdomain), but the effect on the user is the same regardless of the cause.

**Severity:** Critical — this is a conversion dealbreaker that neither the docs nor the product's own UI mention or prepare the user for.

**Suggested improvement:** (1) Submit the domain for allowlisting/security review with Phantom, Solflare, Backpack, etc. before public launch (a process usually exists via the wallets' own forms). (2) Until that's resolved, add an in-product notice in the purchase flow like "your wallet may show a new-domain alert — this is expected during beta, here's how to verify it's safe" — so the product gets ahead of the fear instead of leaving the user alone with a red alert.

**TODO:** test whether the same alert appears with Solflare/Backpack (could be a Phantom-specific blocklist) — if it's just one wallet, it's even easier to report/resolve with them directly.

## FP-APP-5 — ESCALATED: Phantom escalates from "warning" to a full "Request blocked," confirmed across multiple assets

**Critical update:** what started as a red warning (GOOG, screenshot 10) and then a simulation error (BSOL, screenshot 13) has now escalated to a **full block** on a PFE purchase attempt (screenshot 14): a full red-screen "Request blocked" / "For your safety, Phantom has blocked this request." The only way to proceed is the low-visibility "Continue anyway (unsafe)" link — most users will simply close and give up here.

**Why this changes the severity and framing:** This is no longer an isolated single-asset problem (BSOL) — it's happening on GOOG, BSOL, and PFE, meaning the **entire dApp** (`beta.spout.finance`) is flagged in Phantom's security blocklist, not a specific transaction or asset. This is the most critical and most actionable finding in the entire report: **any new Phantom user (the most popular Solana wallet) trying to buy any asset in the beta will hit this full block before even completing their first purchase.**

**Severity:** Critical, priority #1 of the report — this is a top-of-funnel conversion blocker, affects 100% of Phantom users, and is quickly resolvable (an allowlist/false-positive report process directly with Phantom's security team, usually via a form or direct contact).

**New, positive data captured in this same screenshot:** the PFE purchase panel now shows a cost breakdown we hadn't seen before: "Amount / Stocks borrowed: $0.00 / 0", "Total amount / Stocks owned: $3.00 / 0.11", "**Est. borrower cost/yr: $0.00**" (in green) and "Your cost today: $3.00." This is a partial answer to FP-APP-1 (Borrow Cost) — at Leverage 1.0x (no leverage), the "Est. borrower cost/yr" is $0.00, which suggests the table's "Borrow Cost" only applies when there's actual leverage/borrow involved. Still need to test with leverage > 1.0x to confirm the number rises and matches the "Borrow Cost" shown in the table for that asset.

**Spout team's response (Telegram, 09-10/09/2026):** "we're currently in beta and the product is still being tested, as the audit hasn't been completed yet. also, you don't have to interact with real money. all you need to do is request testnet USDC from the Solana Devnet faucet and interact with the product from there. so for now, it might look that way, but we're 100% legit and actively working toward the mainnet launch."

**Analysis of the response:** the team confirms they're aware of the alert and explains the context (pre-audit beta, no real-fund risk). This is useful to ease the "is this a scam?" concern, but **doesn't resolve the root cause of FP-APP-5**: Phantom's block isn't about the project's legitimacy — it's about the wallet's security blocklist age/reputation, which is normally resolved through an allowlist/report process directly with Phantom's team (or Blowfish/other detection firms the wallets use), independent of whether the smart contract audit is complete. Worth reinforcing this in the report as a specific follow-up: "this is resolvable today, even before the audit is done, and fixes a real conversion loss."

**Value for the report:** this exchange itself is good proof of proactive behavior (methodology/adversarial self-review section) — I reported the finding to the sponsor before submitting, and have the timestamp and response documented.

---

**What happened:** The GOOG purchase completed end-to-end — Phantom confirm → "Order placed" → "Your purchase has been confirmed" — with no KYC/identity verification screen at any point in the flow, including the exact moment of the on-chain transaction.

**Why it matters:** Resolves the open question from FP-APP-4: at least on devnet, there's no visible KYC enforcement anywhere in the purchase flow, despite `/security-and-compliance` describing Token-2022 transfer hooks for this. Reinforces the hypothesis that compliance gating only exists on mainnet and the beta skips it on purpose, but this isn't communicated anywhere in the UI.

**Severity:** Medium (downgraded from High since this is likely expected devnet behavior, but still worth reporting the lack of communication about it)

---

## FP-APP-7 — Inconsistent messaging about order status with the market closed

**What happened:** The confirmation modal clearly says "Market is closed. Your order fills at 9:30 AM ET, 10 Sep" (screenshot 11) — great transparency. But the "Open Orders" table right after shows Status = **"Executing"** (screenshot 12), which suggests something is happening now, not that it's queued waiting for market open.

**Why it caused friction:** Small but real — "Executing" and "fills at 9:30 AM ET tomorrow" communicate different things. A user who only looks at the Open Orders table (without recalling the modal) might think the order is being processed right now and get confused why it doesn't reflect in their balance.

**Severity:** Low — easy quick win (swap the label for "Queued" or "Pending market open").

**Suggested improvement:** The status label should change to "Queued" or "Scheduled" when the market is closed, reserving "Executing" for when the order is actually being processed.

---

## FP-APP-8 — Total paid doesn't exactly match the displayed shares value (rounding)

**What happened:** "You own 0.01 GOOG" / "Bought at $328.39" / "Total paid $3.00." But 0.01 × $328.39 = $3.28, not $3.00. The user probably typed "$3" as Amount and the app calculated the real fractional shares (≈0.00913) but displayed it rounded to "0.01" on the confirmation screen.

**Why it caused friction:** This is a display-precision issue, not a calculation bug (the $3.00 total paid is probably correct, it's the share count that's misleadingly rounded). A user who trusts the shown "0.01 GOOG" and later goes to sell might be surprised the real balance is smaller.

**Severity:** Low/Medium — not a real financial bug, it's a display issue, but in a financial product any number imprecision breeds distrust.

**RESOLVED/UPDATED — Portfolio confirms the real data is correct:** the Portfolio screen (09/10/2026, market open) shows the position with full precision: **0.009171 GOOG**, price $326.14, market value $2.99. This confirms the calculation was always correct — the problem was only the rounded display in the post-purchase confirmation modal ("0.01 GOOG"). In the Trade table, "Shares owned" now shows "< 0.01" for that position, which is a correct, honest display (avoids the false precision of an exact "0.01"). **Definitively downgraded to Low** — only the post-purchase confirmation modal needs more decimal places; the rest of the product already handles this well.

---

## FP-APP-9 — BSOL: Phantom fails to simulate the transaction ("Could not simulate the results of this request")

**What happened:** Attempting to buy BSOL (Bitwise Solana Staking ETF), Phantom showed the same "dApp may be malicious"/"new domain" warnings from FP-APP-5, **plus** a third, new and more serious red alert: "Could not simulate the results of this request." The confirm button remains available as "Confirm (unsafe)" — not yet confirmed, awaiting.

**Why it's different/worse than FP-APP-5:** The "new domain" warning is reputational (domain age in the blocklist). "Could not simulate" is a technical signal — Phantom tries to run the transaction in a simulated environment before signing, to predict the outcome (which tokens leave, which come in) and show the user. When that simulation fails, it usually means one of two things: (a) the transaction would revert/fail on-chain anyway, or (b) the instruction uses something the simulator can't process well (e.g., complex CPI, state dependency that only exists at the exact moment of execution). Both scenarios deserve investigation before confirming.

**Severity:** Critical — if the cause is (a), it's a real bug that would waste the user's gas on a transaction doomed to fail. If (b), it's still a serious friction: the user loses exactly the preview that should give them confidence to sign.

**PRIORITY TODO:** decision to make when confirming — (1) if I confirm and the tx succeeds on-chain, document it as a simulation false alarm (still reportable as UX friction). (2) if it fails on-chain, it's concrete proof of a real bug, with an error hash to document. Also compare whether this error is specific to BSOL (maybe because it's an ETF/staking token with different logic than a regular stock) or happens with any asset — GOOG already confirmed without this specific simulation error (only the reputational warnings), so it's quite possible this is BSOL-specific.

**Note:** not yet confirmed — awaiting a decision to proceed or cancel before recording the final result.

---

- Transparency about the closed market: clearly warns BEFORE confirming that the order only executes at the next open (9:30 AM ET) — better than many traditional brokerage products that hide this.
- Purchase flow is fast: from the Trade screen to "purchase confirmed" in a few clicks, no unnecessary redirects.
- The "Order placed — tx [hash]" toast with an implicit link to the explorer is a good on-chain transparency practice the docs promise and the UI actually delivers.

---

- [x] Landing page → Trade screen loads directly, WITHOUT requiring connect to see prices/data — good zero-friction discovery
- [ ] Wallet connect: what was clear/confusing? (not yet clicked)
- [x] Onboarding: KYC didn't appear at any point of connect (see FP-APP-4) — check if it appears at order time
- [x] First screen post-connect: wallet panel shows devnet balance, clear and direct — good test onboarding
- [x] First real hesitation captured: "Borrow Cost" != "0% interest" (see FP-APP-1 above)
- [x] Second real hesitation captured: "Leverage" slider with no explanation (see FP-APP-2 above)
- [ ] Screenshot of this screen saved in assets/screenshots/

---

## Confirmation: GOOG order filled correctly (09/10/2026, market open)

- Order placed yesterday (market closed) → today at 9:30 AM ET the order executed normally, exactly as promised in the modal ("Market is closed. Your order fills at 9:30 AM ET, 10 Sep") — **promise fulfilled, no surprises**. Good point for "What Works": the future-fill communication was accurate.
- **Portfolio Overview:** Total Equity $2.99, Total Borrowed $0.00, Net Worth $2.99, **Available Borrowing Power $1.50**
- Confirms the 50% LTV: $2.99 × 50% = $1.495 ≈ $1.50 — matches the docs exactly (`/how-borrowing-works`, `/health-factor`). Great sign of consistency between documentation and real implementation.
- "My Holdings" shows the position with "Active" status and full share precision (0.009171) — see the FP-APP-8 update above.
- "Positions" (bottom section, referring to borrow positions) shows "No Positions — Deposit collateral to open a position" — the natural next step now: go to `/borrow`, that GOOG position should already appear as available collateral.

---

## GOOG position monitoring over time

**Day 1 (09/10, fill):** 0.009171 GOOG @ ~$326.14, value $2.99, LTV available $1.50
**Day 2 (09/12, Friday night):** same position (0.009171 GOOG), price rose to $335.38, value $3.08, **Unrealized P&L: +$0.09 (+2.87%)**, Available Borrowing Power rose proportionally to $1.54 (50% of $3.08)

**Observation:** "Available Borrowing Power" recalculated correctly and in real time as the position's market value changed — confirms the 50% LTV is dynamic (recalculated over the current value, not locked at entry value), consistent with the Health Factor docs. Good sign of the system's mathematical correctness, even with the already-documented API bugs (FP-APP-12) — suggests the bugs are in the display/error-communication layer, not the core portfolio calculation.

**Tabs explored (Day 2):**
- **Transaction History**: clean, complete record — "Yesterday / 10:30 AM / Bought GOOG / 0.009171 shares at $326.03 / +0.009171 GOOG / $2.99 / Fees: -- / Completed." Has an "Export CSV" button, good sign of a product designed with user-side auditing/accounting in mind.
- **Activity**: "No Activity — Your open orders will appear here," a clean empty state, consistent with the already-good pattern seen in `/borrow`. Correctly empty (never confirmed the $0.08/$0.48 borrow attempts).
- **Minor precision note:** the purchase price shows as $326.03 here, but earlier screens (confirmation modal, portfolio) showed $326.14 and $326.22 at different moments — probably just reflects the exact price at each query's timestamp (market moved between the purchase and the views), not a bug, but worth mentioning as a precision note in the report in case the final on-chain-recorded price diverges from what appears in the table.

---

## Diversification session — 5 new positions opened with the market closed (09/11/2026)

Orders placed deliberately with the market closed, to (a) reinforce FP-APP-7 with more samples and (b) diversify volatility for the week's monitoring:

| Asset | Amount | Displayed status | Placed |
|---|---|---|---|
| BSOL | $5.00 | Executing | Sep 11 |
| MSTR | $5.00 | Executing | Sep 11 |
| AAPL | $5.00 | Executing | Sep 11 |
| PFE | $5.00 | Executing | Sep 11 |
| GS | $3.00 | Executing | Sep 11 |

**Confirms FP-APP-7 at scale:** all 5 orders show "Executing" (yellow) even with the market closed — not an isolated GOOG case, it's the system's default behavior for any order placed outside market hours. Reinforces the recommendation: swap to "Queued"/"Pending Market Open" when `Market: Closed`.

**Good volatility coverage for the week's monitoring:** MSTR and BSOL (more volatile, crypto-adjacent) vs PFE/AAPL/GS (more stable) — will allow comparing the real Borrow Cost by volatility once positions fill Monday morning (9:30 AM ET) and each one's Health Factor over the week.

---

## Competitive intelligence — another tester's thread (@blessedboy32, 09/11)

**Strong independent confirmation:** they also found "CollateralType: unexpected length 213..." in NVDA's borrow flow, and describe the same symptom we saw: "UI still shows a max borrow amount. Click it and it says the position supports $0.00." — this is a **second independent source confirming FP-APP-12**, strongly reinforcing that finding's credibility in the final report (can cite as "corroborated by other independent testers in the same beta cohort").

**3 new bugs, not yet tested by us — TODO validate:**
1. **Sell confirmation modal reuses the buy template**: "Sell confirmation modal still says 'Your purchase has been confirmed' (reused buy template)" — a copy/template bug not swapped between buy/sell flows.
2. **Blank fields in Portfolio**: "Portfolio shows Total Equity but Net Worth + Available Borrowing Power stay blank" — different from what we saw in our test (where those fields appeared correctly), could be a specific state (maybe after a partial sale, or a different asset).
3. **"Max sell" leaves residual dust**: "Max sell leaves dust instead of fully closing the position" — suggests the Sell "Max" button doesn't sell 100% of the position, some fraction is left over.

**Their UX friction angle, worth considering:**
- "Heavy gating (email + passcode) slows real testing" — onboarding friction we may not have felt as much (we used Phantom directly, without Privy's email flow). Worth testing the email+passcode flow too, if we haven't already.
- "High cognitive load — locking into weekly options cycles, health factor, potential assignment" — a qualitative view that reinforces our Senior Analysis (point 6, about mixing reassuring fintech language with real derivative mechanics).

**How to use this in the report:** no need (and shouldn't) name the tester — but worth noting that technical finding #1 was corroborated externally, and testing the 3 new bugs (partial/full sale of a small position, e.g., selling a fraction of GOOG) before finalizing the report, since we have an active position large enough to reproduce.

---

## FP-APP-15 — CRITICAL: Incorrect AAPL Avg Cost, causing a massive error in displayed P&L

**What happened:** Monday, market open, the 5 new positions filled (screenshot 36). Validating each row's math (shares × avg cost should match the original order value, ~$5 or $3):

| Asset | Shares | Avg Cost | Calculated Cost Basis | Order value | Match? |
|---|---|---|---|---|---|
| MSTR | 0.038121 | $130.90 | $4.99 | $5.00 | ✅ |
| BSOL | 0.35822 | $13.93 | $4.99 | $5.00 | ✅ |
| PFE | 0.177013 | $28.19 | $4.99 | $5.00 | ✅ |
| GOOG | 0.009171 | $326.03 | $2.99 | $2.99 | ✅ |
| GS | 0.00296 | $1,010.00 | $2.99 | $3.00 | ✅ |
| **AAPL** | **0.014912** | **$334.62** | **$4.99** | **$5.00** | ✅ (the order itself matches) |

The AAPL order matches the original intent mathematically ($5 → 0.014912 shares at $334.62). **The problem is that $334.62 isn't anywhere close to AAPL's real price at the time of purchase** — the current price shown in the same row is $228.50, a 46% difference. If Avg Cost were correct (near $228), the P&L would be close to zero (the market barely moved since the order). But with Avg Cost at $334.62, the real calculable loss is **-$1.58 (-31.7%)** — yet the screen shows **"$-0.02 (-0.50%)"**, a completely different, equally wrong number (matching neither the wrong Avg Cost nor a correct one).

**Why this is critical:** It's a double error — (1) the Avg Cost recorded for AAPL is wrong by a huge margin (the execution price data used seemed to come from another asset/time period, "$334" isn't a plausible recent AAPL price), and (2) the displayed P&L isn't even consistent with the wrong Avg Cost — it's a third number, suggesting the screen's P&L calculation uses a different data source than the one that populated Avg Cost. This is the kind of bug that, in real production with real money, would show the user a completely fictional loss/gain.

**Severity:** Critical — financial data integrity is the most sensitive possible category in a product like this; an incorrectly displayed P&L could literally lead to wrong sell/hold decisions.

**TODO:** check whether this error is specific to AAPL (maybe a price collision with another asset on the backend, given that $334 doesn't correspond to anything obvious) or happens with any asset under certain conditions. Compare against AAPL's on-chain transaction hash if possible, to confirm whether the real execution price (on-chain) matches $228 (correct) or $334 (what's being displayed).

---

## FP-APP-16 — CRITICAL: Sell orders marked "Failed" but ACTUALLY EXECUTED (shares really debited)

**What happened:** Two sell orders (PFE and GOOG, 09/15) appear in the "Open Orders" table with **"Failed"** status, each with an odd partial-execution detail (e.g., "Failed 0.036519942 @ $27.43"). Comparing holdings before and after these orders:

| Asset | Shares before | Shares after | Difference | Matches the shown "Failed"? |
|---|---|---|---|---|
| GOOG | 0.009171 | 0.003316 | 0.005855 | ✅ matches "0.005854972" |
| PFE | 0.177013 | 0.140493 | 0.036520 | ✅ matches "0.036519942" |

**Both sales actually happened — shares were debited exactly by the amount recorded in the "Failed" order — but the displayed status says it failed.**

**Why this is the test's most severe finding:** this is worse than the 500s or the deserialization error, because those at least clearly communicated something was wrong. Here, the system **executes the transaction and lies about the result**. Real consequences: (1) a user seeing "Failed" might try to sell again, potentially selling more than intended; (2) the user's real balance diverges from what they believe they have, with no warning; (3) in a real production scenario with real money, this is the kind of bug that generates support disputes and irreversible loss of trust.

**Severity:** Critical — the highest in the entire report. Likely root cause: the same class of problem as FP-APP-12 (a communication error between the layer that processes the on-chain transaction and the layer that updates the UI status) — the transaction probably succeeded on the blockchain, but the confirmation/callback call that should mark it "Completed" failed or timed out, leaving the status "frozen" at "Failed" as an error default.

**CRITICAL UPDATE — confirmed: USDC was NOT credited (worse than it initially seemed):** checking the wallet's USDC balance before and after the two "Failed" orders: it stays at **$8.73** in both readings, with no increase. If the sales had actually settled (shares → USDC), the balance should have risen by roughly $3 (≈$0.99 from PFE + ≈$2.00 from GOOG, based on market values at the time). This changes the finding's severity: it's not just "incorrect status on a transaction that succeeded" — it's **shares debited from holdings with no corresponding USDC appearing in the wallet**. Two hypotheses, both serious:
1. The sell leg of the swap (shares → SOL/USDC) executed on-chain (which is why holdings dropped), but the credit leg genuinely failed, leaving the user with a reduced position and nothing received in exchange — real value loss.
2. The share count shown in Holdings is decremented optimistically/locally as soon as the order is submitted, before on-chain confirmation — and in that case the UI is lying about the real balance in the opposite direction (showing less than you actually have), which is also serious, just in a different sense (suggests selling again something already "spent" only on screen).

**RESOLVED via Solscan devnet — root cause confirmed, reclassification needed:** inspecting the wallet's on-chain history (`solscan.io/account/DHG4p1...RnNV?cluster=devnet`), both sell orders appear as a `placeSellOrder` instruction (timestamps matching exactly the "Failed" orders' times), but **there's no `fulfillSellOrder` or equivalent settlement instruction following it** in the wallet's most recent transactions. So: the order was genuinely placed on-chain (which is why holdings drop — shares get reserved/locked in the pending order), but the fulfillment step (which would return USDC and finalize the sale) never happened.

**Reclassification:** this isn't "shares lost with no compensation" (hypothesis 1) — it's an order that got **stuck in a pending/unfulfilled state, and the UI incorrectly labels that state as "Failed"** (terminal, implying nothing happened) when it should show something like "Pending Fulfillment" or "Awaiting Settlement." The "Cancel" button visible in the Open Orders table would likely return the shares — but that hasn't been tested yet.

**Bonus relevant finding (connects to FP-APP-4/6):** in the same wallet's older purchase history, there's a repeated pattern of `adminThaw` followed by `fulfillBuyOrderFreezeGated` — this is the real implementation of the Token-2022 freeze/thaw mechanism the docs (`/security-and-compliance`) describe as KYC enforcement. Confirms the compliance architecture exists on-chain (tokens are minted "frozen" and an `adminThaw` releases them), but on devnet this thaw seems to happen automatically/with no visible real KYC gate — consistent with our observation that no KYC step appeared in the UI.

**Revised severity:** stays Critical — not because of fund loss, but because (a) the "Failed" label is factually incorrect and misleads the user about the real state of their position, and (b) shares become effectively unavailable (neither in the wallet to sell again, nor converted into USDC) until the user discovers they need to cancel manually — if the cancel even resolves it.

**TODO:** test clicking "Cancel" on one of these stuck orders and confirm whether the shares return to holdings.

---

## FP-APP-17 — Sell confirmation modal reuses the "Your purchase has been confirmed" copy (template bug, independently confirmed)

**What happened:** Upon confirming a GOOG and a PFE sale, the success modal shows **"Your purchase has been confirmed"** — even though it was a sale, not a purchase (screenshots 44, 45; shown in the popup even after clicking SELL). This confirms exactly what the other independent tester (@blessedboy32) reported publicly: "Sell confirmation modal still says 'Your purchase has been confirmed' (reused buy template)."

**Severity:** Medium/High — not financially dangerous on its own, but confusing (a user selling might be alarmed thinking they bought by mistake), and combined with FP-APP-16 above, adds another layer of contradictory signals exactly in the product's most sensitive flow (moving money).

**Suggested improvement:** Create a dedicated sell confirmation modal ("Your sale has been confirmed" + appropriate icon/copy), don't reuse the buy one.

---

## FP-APP-18 — Phantom also blocks SELL, not just BUY

**What happened:** The same Phantom "Request blocked" alert (FP-APP-5) appeared on a **sell** attempt for GOOG (screenshot 43), confirming that the security block covers any dApp transaction, not just purchases. This widens the scope of the report's #1 finding — the problem isn't specific to the buy flow, it's any interaction with the contract.

**Severity:** already covered by FP-APP-5's Critical rating, this is just a confirmation of broader scope — worth mentioning in the final text.

---

## FP-APP-19 — No "my holdings only" filter in the Trade/Sell table

**What happened:** The "All Types" dropdown only filters by Stocks/ETFs (screenshot 40), there's no "My Holdings" or "Owned only" option to quickly see only the assets the user already owns. With a 6+ asset portfolio, finding your own holdings requires scrolling the entire table and manually checking the "Shares owned" column.

**Severity:** Medium — real usability friction, more noticeable the more assets a user accumulates (as in our case, after the week's diversification).

**Suggested improvement:** Add "My Holdings" as an option in the "All Types" dropdown, or a quick toggle above the table.

---

## Note — Earn confirmed as not yet available (not our testing gap)

The Earn tab shows "Earn is coming soon. Lending vaults are on the way. In the meantime, you can trade tokenized stocks or borrow against the ones you already hold, at 0% interest." (screenshot 46) — confirms what the bounty brief already warned ("lending market rolling out soon"). We didn't test the lender side because **it doesn't exist yet in beta**, not due to a gap on our end. Worth mentioning in the report to make clear that "DeFi/tokenization analysis" coverage was limited to the borrower side by product scope, not by test omission.

---

## FP-MOBILE-1 — CRITICAL: page completely fails on mobile with Slow 4G throttling (NO_FCP)

**What happened:** Running PageSpeed Insights in Mobile mode (emulated Moto G Power, Slow 4G throttling) on the `beta.spout.finance` home page (screenshot/doc 9, captured 09/15), **the page didn't render any content**: "The page did not paint any content... (NO_FCP)" — an error across ALL metrics (FCP, LCP, TBT, CLS, Speed Index) and in nearly the entire Accessibility, Best Practices, and SEO audit (a generalized "Error!," because Lighthouse couldn't even load the page to analyze it).

**Why this is critical:** The earlier desktop Lighthouse run (FP-APP-11) gave 100/100/100/91 — excellent. This mobile result is the complete opposite: total rendering failure under realistic network conditions (Slow 4G is Google's default test scenario for a common mobile connection, not an extreme case). This suggests a serious bundle-size/performance issue that only manifests under limited bandwidth, consistent with the already-documented inefficient polling pattern (FP-APP-10) and the heavily fragmented JS bundle across dozens of chunks (seen in the Sources tab, FP-APP-11), which may be competing severely for limited bandwidth.

**Severity:** Critical — a significant share of any real product's traffic (DeFi included) comes from mobile, and "the page doesn't load" is the worst possible outcome, worse than any individual UX bug.

**TODO:** manually reproduce on a real mobile device with network throttling (Chrome DevTools mobile emulation + Network throttling) to visually confirm what happens — white screen? infinite loading? — and capture a screenshot/video as more direct evidence than just the Lighthouse report.

---

**Final checkpoint (09/17):** Portfolio $21.26, Unrealized P&L -$0.05 (-0.18%), Available Borrowing Power $10.63, Total Borrowed $0.00. The 1W chart shows a peak of $25.94 on 09/17 at 6 PM, with a slight decline through the time of reading — real variation captured across the entire week, from the initial fill to now. **Position monitoring ends here — moving on to final report assembly.**

---

## FP-APP-20 — CRITICAL: The entire portfolio zeroes out ("No Holdings," "No Metrics") despite the positions existing

**What happened:** In a later session (09/19), the Portfolio screen shows "No Holdings — Trade tokenized stocks to build your portfolio" and "No Metrics — Build your portfolio to see your total equity, borrowed amount, and net worth" — as if the account never had any position. Yet the "Portfolio Overview" chart at the same moment still shows the historical peak "$27.35, Sep 19, 12:00 AM," proving the data existed and was recorded. The 6 diversified positions (GOOG, MSTR, BSOL, PFE, GS, AAPL) simply no longer appear.

**Why this is critical:** It's the same failure pattern already seen in FP-APP-12/16 (a data API failing and the UI falling into an incorrect empty state, instead of showing an error or cached data), except now it hits the product's most important view for the user — "how much do I have." Unlike an isolated bug on a secondary screen, this one hides the entire portfolio.

**Severity:** Critical — there's no indication of a real liquidation (Total Borrowed has always been $0, no margin-call risk), so this is almost certainly a display/fetch bug, not real loss — but the user experience is indistinguishable from "my money is gone" until proven otherwise.

**UPDATE — confirmed as persistent, not one-off:** checked again on 09/21 (2 days after the first record): "No Holdings" and "No Metrics" are still there, now with the chart showing an updated peak of $28.08 (09/21, 6 PM) and Unrealized P&L +$2.35 (+9.06%) — meaning the system keeps tracking and correctly updating the position's historical value in the background, but the holdings/metrics screen simply never goes back to showing data. This isn't a one-off network glitch — it's a permanently broken state for this specific account, for at least 2 days. Reinforces the Critical severity: a real user in this situation has no way, inside the product, to see what they own.

---

## FP-APP-21 — Corrupted asset labels in the Open Orders table (regression)

**What happened:** The same two stuck orders from FP-APP-16 (PFE and GOOG, seen on prior days as "PFECLzi…UbBA" and "GOOG7zo3…H3nV" — ticker + address) now appear as **"EG3r…CLzi…UbBA"** and **"6a2y…7zo3…H3nV"** — the ticker has completely disappeared, leaving only illegible on-chain address fragments.

**Why this matters:** It's a regression — the same screen got worse over the course of testing, not better. Combined with FP-APP-20, it suggests a broader failure in the layer that resolves asset metadata (name/ticker) from an on-chain address, not isolated to these two specific screens.

**Severity:** Medium/High — doesn't hide money, but destroys the readability of a screen that was already showing an incorrect status ("Failed").

---

## On-chain transaction log (fill in as tested)

| Action | Amount | Asset | Transaction | Date |
|---|---|---|---|---|
| Buy | $3.00 | GOOG (0.01 shares, rounded) | 4ZReviwT... (confirm full hash on explorer) | 09/09/2026, market closed, fills 9:30 AM ET 09/10 |

---

## Loose ideas / Day 0 observations

-