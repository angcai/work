# Pricing an NVIDIA-Backstopped Data Centre Lease: A Credit-Only Framework

**Scope note.** This memo prices **credit risk only** — the probability that contracted cash flow stops, and what is recovered when it does. Operating risk, liquidity, complexity, capital charges and portfolio concentration are deliberately excluded and listed at the end as separate layers to add. All numbers are illustrative and must be recalibrated against live market data before use.

**Structure.** Deal is a project financing of a data centre. NVIDIA provides a **minimum revenue guarantee** (take-or-pay backstop) on GPU capacity. Debt is sized by DSCR sculpting against **CFADS** (cash available for debt service, i.e. post-cost), senior at 1.20x and mezzanine at 1.00x.

---

## 0. The framing question, and why the answer is "no, but"

The instinct is: NVIDIA guarantees the revenue, NVIDIA is rated AA-/Aa1 and trades around 72bp on 5-year CDS, therefore this is NVIDIA risk and should price near 72bp.

That instinct is right about the *credit component* and wrong about the *instrument*. The distinction that matters:

| | P(no payment \| guarantor solvent) | Is it guarantor credit? |
|---|---|---|
| **Financial guarantee** of debt service | ≈ 0 by construction | Yes |
| **Revenue guarantee / take-or-pay offtake** | > 0 by construction | No |

A take-or-pay offtake is a commercial contract with conditions, defences, set-off rights and performance triggers. A financial guarantee is a promise to pay debt service regardless. The first has a non-zero conditional non-payment probability *because of what it is*, not because anyone expects bad faith.

**The structural tell:** if this were genuinely NVIDIA credit, you would not sculpt to a DSCR at all. You would size to the guarantor's credit and price off its curve. The existence of a 1.20x coverage test is the lenders' own written acknowledgement that they are taking project risk.

So the whole question reduces to: **what is `p`, and what does it cost?** The rest of this memo is a framework for answering that in three nested cases.

---

## 1. The pricing engine

Reduced-form hazard model. For a par instrument, the fair credit spread is approximately:

```
s  ≈  Σ_k  λ_k × LGD_k
```

where `λ_k` is the risk-neutral hazard rate of loss event *k* and `LGD_k` is loss given *that specific event*. Carrying LGD per event rather than as a single number is the single most important modelling choice here — see §5 on correlation.

Calibration of the hazards:

```
λ_NVDA  = CDS_NVDA / (1 − R_NVDA)          # from traded CDS
λ_SOV   = Sovereign spread / (1 − R_SOV)   # from traded sovereign debt
λ_B     = −ln(1 − p) / T                   # from p, a cumulative estimate over T years
```

Base calibration used throughout (refresh before use):

| Input | Value | Source / note |
|---|---|---|
| NVIDIA 5y CDS | 72bp | ICE, Aug 2026 |
| NVIDIA recovery | 40% | senior unsecured convention |
| λ_NVDA | 1.20%/yr | derived |
| Sovereign spread proxy | 100bp | per instruction |
| Sovereign recovery | 40% | convention; sovereign recoveries vary widely |
| λ_SOV | 1.67%/yr | derived |
| Tenor T | 5 years | assumed |
| LGD, NVIDIA default | 55% | first-lien on DC + GPUs, but collateral impaired in that scenario |
| LGD, standalone non-payment | 50% | *entirely a documentation variable* — see §2 |
| LGD, sovereign/country event | 70% | expropriation is near-total; T&C is delay not loss |

For implementation with sculpted amortisation, use the full expected-loss form rather than the closed form above:

```
Survival:   S(t) = exp(−Λ(t)),  Λ(t) = Σ λ_k · t
EL         = Σ_t [S(t−1) − S(t)] · LGD_t · EAD(t) · DF(t)
Spread solves:  PV(s · EAD) = EL
```

`EAD(t)` is the sculpted outstanding balance schedule, which is *not* straight-line and back-loads exposure. This matters: sculpting to a flat DSCR against a flat guaranteed revenue produces slow early amortisation and a large late balance, so exposure is concentrated in the years where the guarantee is most likely to have expired.

---

## 2. Case 1 — NVIDIA only

Two competing loss events:

- **Event A:** NVIDIA defaults. λ_A = 1.20%/yr.
- **Event B:** NVIDIA remains solvent but the lease is not paid. λ_B = −ln(1−p)/5.

### Results

| p (cumulative, 5y) | λ_B | A contribution | B contribution | **Fair credit spread** |
|---|---|---|---|---|
| 0% | 0.00% | 66bp | 0bp | **66bp** |
| 2% | 0.40% | 66bp | 20bp | **86bp** |
| 5% | 1.03% | 66bp | 51bp | **117bp** |
| 10% | 2.11% | 66bp | 105bp | **171bp** |
| 20% | 4.46% | 66bp | 223bp | **289bp** |

### Three readings

**At p = 0, the answer is roughly "yes, it is NVIDIA credit" — and slightly better.** 66bp versus 72bp CDS. The lease claim prices *inside* the bond because LGD is lower: the lenders hold first-lien security over the facility, the GPUs, and the power contracts, whereas bondholders hold unsecured paper. Same probability of default, better recovery, tighter spread. This is worth stating explicitly because it is the correct answer to the original question and it is counterintuitive.

**The sensitivity is close to linear and easy to carry in your head:**

```
ds/dp ≈ LGD_B / T = 50% / 5 = ~10bp per 1pp of cumulative p
```

So `p` is not a rounding error. Ten percentage points of conditional non-payment probability is worth more than NVIDIA's entire standalone credit spread. **Past roughly p = 6%, the guarantor's rating stops being the main driver of price.** You are no longer pricing an AA credit; you are pricing a contract.

**`p` is not a market observable. It is an output of the documentation, not an input from the market.** No amount of credit analysis produces `p`. It is set by: whether the obligation is hell-or-high-water; whether defences and set-off are waived; whether payment is conditional on availability, SLA or delivery milestones; whether there is a direct agreement with lenders; whether the obligor is NVIDIA Corp (the rated bond issuer) or a subsidiary; whether the guarantee is capped; and whether the backstop tenor covers the debt tenor. **The legal team sets `p`. The credit team prices it.** If the model wants p = 10%, the correct first response is to redraft, not to reprice — see §6.

---

## 3. Correlation — where it actually lives

Three distinct correlations, pulling in different directions. Conflating them is the most common error in this structure.

**ρ₁ — between event A and event B (event-to-event).** Both are driven by AI compute demand, so positive. Under competing risks, positive correlation means the events *overlap*, so total probability of non-payment is slightly **less** than the naive sum. This is second-order and it works in your favour. Using λ_A + λ_B (the independence assumption) is therefore **conservative**. Good: keep it as the base case.

**ρ₂ — between the loss event and recovery (wrong-way risk).** This is first-order and it works against you. Ask *why* event B occurs:

- **B1 — valid defence.** The project underperformed: missed availability, power not delivered, force majeure. NVIDIA legitimately owes nothing. This is not guarantor risk at all — it is operating risk, and it should be underwritten by looking at the sponsor's track record, grid interconnection, cooling technology and construction schedule. Recovery is poor because the asset failed for a reason.
- **B2 — strategic non-payment or dispute.** NVIDIA is solvent but the contracted rate is far above the spot value of compute, creating an incentive to litigate, renegotiate or delay. This occurs precisely when GPU rental prices have collapsed — which is precisely when the facility cannot be re-let, when the residual-value support is being called on across every deal simultaneously, and when the secondary market for the collateral is at its thinnest.

**The correlation that matters is not in the PD. It is in the LGD.** This is why the model must carry LGD per event rather than a blended number. A single LGD assumption silently prices B2 at the unconditional recovery, which is exactly wrong.

**ρ₃ — across the portfolio (systematic).** Every deal in this programme shares one guarantor and one demand curve. Diversification does not work. This does not change expected loss on a single deal but it substantially raises the capital and required return above expected loss at book level. It belongs in the layers of §7, not in the credit spread, but it must not be forgotten.

---

## 4. Case 2 — adding country risk

Replace the single-name hazard with a **first-to-default** hazard across {NVIDIA default, country event}. Nests correctly: set λ_SOV = 0 and Case 2 collapses to Case 1.

### First define what a "country event" is for this cash flow

The 100bp sovereign spread is a proxy for *sovereign default*. That is correlated with, but not the same as, the risks that stop this project's cash flow. Split it:

- **λ_SOV,cash — events that stop payment:** transfer and convertibility restriction, expropriation, contract frustration, change in law or licensing, political violence.
- **λ_SOV,asset — events that impair recovery only:** enforceability of security, local court risk, asset seizure delay. **These belong in LGD, not in the hazard.**

The critical structural question is **where the cash flows**. If NVIDIA pays into an offshore USD account controlled by the lenders, transfer and convertibility risk is largely mitigated and λ_SOV,cash is materially below the sovereign spread. If payment routes through an onshore entity, you take the full T&C risk. **Do not apply the sovereign spread without first tracing the payment path.**

Also assess whether political risk insurance or multilateral cover (e.g. MIGA) is available and priced, since that converts a large part of λ_SOV into a fixed premium.

### First-to-default composition

Under independence, competing Poisson hazards superpose exactly:

```
λ_FTD = λ_NVDA + λ_SOV
```

Positive correlation between the two *reduces* λ_FTD (joint events overlap). NVIDIA default and sovereign default have largely different drivers, though both are sensitive to global risk appetite — plausible ρ of 0.2–0.35, which shaves roughly 5–10% off λ_FTD. **Independence is the conservative choice; use it as base and treat correlation as upside.**

### Results

| p | NVIDIA leg | Country leg | Standalone non-pay | **Fair credit spread** |
|---|---|---|---|---|
| 0% | 66bp | 117bp | 0bp | **183bp** |
| 5% | 66bp | 117bp | 51bp | **234bp** |
| 10% | 66bp | 117bp | 105bp | **288bp** |

At a 100bp proxy, the country leg is **larger than NVIDIA's entire credit contribution**. The guarantor's AA rating buys much less than it appears once the asset sits in an emerging-market jurisdiction. If the offshore-account structure holds, most of this should come back — which makes the account structure, not the guarantee, the highest-value negotiating point in Case 2.

---

## 5. Case 3 — the non-IG offtaker upside

Premise: additional cash flow from a non-investment-grade offtaker above the backstop level, shared 50/50 with NVIDIA. Intuition: more cash, safer debt.

**Directionally correct, but worth far less than it looks — and under one common structure, worth less than zero.** Three tests, in order of importance:

### Test 1: Does the cash arrive in the states where you default? (Correlation)

No. A non-IG offtaker pays when compute demand is strong. That is exactly the state of the world in which the debt was never going to default. In the states that matter — demand collapse, NVIDIA disputing the backstop, spot rates below opex — the non-IG offtaker is the *first* counterparty to fail.

**Cash flow that is negatively correlated with your default states has near-zero credit value.** This is the same error as counting an equity cushion that only exists when the equity is in the money. Model it as a **reduction in LGD**, never as a reduction in PD, and haircut it heavily for the states in which it is absent.

### Test 2: Does it reach the lenders, or the sponsor? (Waterfall)

Under a sculpted structure, CFADS above the DSCR target goes somewhere specific. **The credit value of Case 3 is determined entirely by the waterfall, not by the cash flow.**

| Waterfall treatment | Credit value |
|---|---|
| 100% cash sweep to prepayment, or trapped in DSRA | Real. Reduces balance and balloon → lowers LGD. |
| Distributable to equity after DSCR test | **Zero.** The lenders see the cash pass by. |
| Included in the CFADS used for debt sizing | **Negative.** See below. |

### Test 3: Is it being used to justify more leverage?

This is the trap. If the non-IG revenue is included in the CFADS that the 1.20x and 1.00x sculpt is run against, the model reports a better DSCR and permits a larger loan — meaning you have just re-levered the deal against your weakest counterparty. The reported coverage improves while the actual coverage deteriorates.

**Hard rule: sculpt on guaranteed CFADS only. Non-IG revenue may enter the sweep, never the sizing.**

### One more question to resolve

Where does NVIDIA's 50% sit in the waterfall? If the share is taken **before** debt service, it is a cash leakage that reduces CFADS and *worsens* coverage. If it is a residual split **after** the sweep, it is harmless. This is a first-order question and it is not usually obvious from a term sheet.

### Indicative quantification (Case 2, p = 10%, base 288bp)

| Scenario | Effect | Spread |
|---|---|---|
| Full sweep to prepayment; ~5pp LGD reduction across legs | Real de-levering | **~263bp** (−25bp) |
| Distributable to equity | None | **288bp** (0bp) |
| Included in debt sizing | Higher EAD, weaker coverage | **>288bp** (worse) |

The best realistic case is roughly 25bp of benefit — around one-tenth of the value that the headline "50% additional cash flow" suggests. **The upside is real economically and nearly worthless as credit support**, and it is worth being explicit about that distinction, because it is the sponsor's strongest argument for tighter pricing.

---

## 6. The calibration trap: risk-neutral versus real-world

**This is the largest technical error available in this framework and it should be flagged to anyone implementing it.**

λ_NVDA and λ_SOV are derived from traded spreads, so they are **risk-neutral** hazards — they already embed the market's risk premium. λ_B is derived from `p`, which is an analyst's **real-world** estimate of how likely the counterparty is to dispute the contract.

Mixing them understates the B leg. For investment-grade credits, the risk-neutral to real-world PD ratio typically runs **2–4x**. A real-world `p` of 10% may correspond to a risk-neutral `p` of 20–30%.

Under that adjustment, the B contribution at p = 10% moves from ~105bp to roughly **210–315bp**, and Case 2 total moves from 288bp to roughly **395–500bp**.

Two options, both defensible; pick one and state it:
1. Apply a risk-premium multiplier to λ_B to put it on a risk-neutral footing.
2. Convert everything to real-world PDs and add an explicit risk premium at the end.

Do not silently mix. If the memo is going to a committee, state the convention on the cover page.

---

## 7. Layers to add on top of the credit spread

The credit spread above is a floor, not a price. Sequence for building up:

1. **Operating / cost risk.** The guarantee covers revenue; the DSCR test consumes CFADS. Power, maintenance, tax and insurance sit in between. At 1.20x senior, if opex is 35% of revenue, an opex overrun of roughly 37% eliminates the entire cushion. **The senior lender is long NVIDIA credit and short an opex put.** Quantify separately with an opex stress.
2. **Tenor mismatch and refinancing.** Backstop tenor versus debt tenor. Any uncovered tail is merchant risk and belongs in a different model. Sculpting back-loads the balance, so the balloon sits in the least-protected years.
3. **Liquidity, complexity and capital charge.** Private credit, unrated, cross-border, small lender pool. Materially larger than the entire credit spread in most cases.
4. **Portfolio correlation (ρ₃).** No diversification benefit across a book of deals sharing one guarantor and one demand curve. Raises required return above expected loss at book level.
5. **Mezzanine, separately.** A 1.00x sculpt means zero coverage cushion — any timing mismatch, working-capital swing or tax true-up breaches immediately. Behind the senior in the waterfall and subject to standstill, effective LGD approaches 100% in stress. **Mezzanine should not be priced off the guarantor's curve at all.** It should be priced off merchant residual value and structural subordination, with an equity kicker. The mezzanine tranche is taking precisely the risk the guarantor has said it will not cover.

---

## 8. What to check before the model is worth running

The following inputs dominate every output above and none of them are market data:

1. Is the backstop hell-or-high-water, with defences and set-off waived?
2. Is the obligor NVIDIA Corp (the rated bond issuer) or a subsidiary? Pari passu with the bonds?
3. Is there a cap, and does it apply per project or across the programme?
4. Is there a direct agreement with lenders, with step-in and cure rights?
5. Does the backstop tenor cover the debt tenor plus a tail?
6. Is payment conditional on availability, SLA or delivery milestones? *(This alone sets `p`.)*
7. Does payment route to an offshore lender-controlled account? *(This alone sets λ_SOV,cash.)*
8. Where does NVIDIA's revenue share sit in the waterfall — before or after debt service?
9. Is excess cash swept to prepayment, or distributable?
10. Is the DSCR sculpt run on guaranteed CFADS only?

**Summary judgement.** At p = 0 with no country risk, the instinct is right: this prices at or slightly inside NVIDIA's curve on pure credit. Everything above that — and it can easily be 200–400bp — is bought with `p` and with jurisdiction, both of which are set in the documents rather than observed in the market. The correct use of this model is not to find a number. It is to identify which clause is worth the most basis points.
