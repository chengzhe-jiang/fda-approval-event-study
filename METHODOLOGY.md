# Methodology

## Research question
How do the stocks of small and mid-cap biotech companies react to FDA approval of a novel drug?

## Data
- **Approvals:** FDA CDER, *Compilation of CDER New Molecular Entity (NME) Drug and New Biologic Approvals, 1985–2025* (2026 release), filtered to approvals from 2015 through 2025 (511 drugs).
- **Validation:** per-year counts match FDA's published totals. For 2021–2025, counts also match FDA's yearly "Novel Drug Approvals" web pages exactly (238 drugs).
- **Prices:** daily prices from Yahoo Finance via `yfinance`.
- **Company data:** SEC EDGAR (company identifiers, filings, shares outstanding).

## Screening
1. **First pass (keyword filter):** applicants were flagged as large-cap, not US-listed, or not a public company using keyword matching on the applicant name. Every excluded drug keeps a written reason in `data/processed/nme_2015_2025_screened.csv`. 226 drugs remained for review.
2. **Second pass:** each remaining drug was checked against the rules below, by hand for an initial sample and with an automated screen for the rest (see Automation).

## Inclusion rules
An event is included only if all of the following hold:
- The company that owned the drug at approval is publicly traded on a US exchange.
- Market cap on day -1 is between $300M and $10B.
- At least 250 trading days of price history exist before day 0.
- No confounding event falls within ±5 trading days of day 0.

**Subsidiaries:** If another company owned the FDA applicant at approval, the parent company's stock is used, and all rules apply to the parent.

## Event timing (day 0)
Day 0 is the first trading day on which the market could react to the announcement, based on the company's press release time, not the FDA approval date.
- Before 9:30 AM ET → pre-market, day 0 = that day
- 9:30 AM to 4:00 PM ET → intraday, day 0 = that day
- After 4:00 PM ET → after-close, day 0 = next trading day
- Weekends and market holidays → day 0 = next trading day
- Half trading days (1:00 PM close) → 1:00 PM is treated as the close

## Market cap
Closing market cap on day -1, the last close before the announcement. Using a later date would let the event's own price reaction affect which events are included.
- Hand-researched rows: companiesmarketcap.com, closest date on or before day -1. Values within 15% of a cutoff were calculated as day -1 closing price × shares outstanding from the latest 10-Q.
- Automated rows: day -1 closing price (corrected for later stock splits) × the latest shares outstanding reported to SEC before the event.

## Confounding events
**Counts:** any company announcement of new regulatory, clinical, or financial information within ±5 trading days of day 0, including approvals or rejections in any country, new trial results, earnings, stock offerings, and major deals or acquisitions. The date that matters is when the market could react (an after-close release counts toward the next trading day).

**Does not count:**
- Announcements about the same approval (conference-call scheduling, the 8-K reporting it).
- Journal publications or conference presentations of results the company had already announced.
- Contract terms triggered automatically by the approval (milestone payments, royalty payments, warrant exercises), when the terms were disclosed earlier or in the same release.

Exclusions follow these written rules only, never expectations about how a stock will react.

## Automation
After an initial set of events was researched by hand, the same rules were applied to the remaining candidates in `notebooks/02_automate_research.ipynb`. Hand-researched values always take priority. The automated screen is validated against the hand-researched sample before use (see Step 7 of the notebook)
- **Day 0** for automated rows is the first trading day on or after the FDA approval date. This can be one day early for after-close announcements; the main [-1, +1] window captures the reaction either way.
- **Confounding events** for automated rows are identified from SEC filings within ±5 trading days: 8-K Items 1.01, 2.01, 2.02, 3.02, and 5.02, plus 424B prospectus filings. Flagged events are reviewed by hand.

## Known limitations
- **Survivorship bias:** companies acquired or delisted since approval often lack price data and drop out, so the sample over-represents companies that survived.
- **Press-release-only news:** confounding news released without an SEC filing may be missed in automated rows.
- **Multi-product companies:** for companies with several marketed products (e.g., Ionis, Innoviva, Jazz), one approval is a smaller share of total value, which may weaken the measured reaction.

## Event study design (planned)
- **Model:** market model, estimating each stock's alpha and beta by regression on the S&P 500, with XBI (biotech ETF) as a robustness check.
- **Estimation window:** trading days -250 to -31 before day 0.
- **Event windows:** [-1, +1] (main) and [0, +5].
- **Testing:** cumulative abnormal returns (CARs) averaged across events, with a t-test of whether the mean differs from zero.
- **Cross-sectional regression:** CARs on market cap, FDA designations (priority review, breakthrough, orphan, fast track, accelerated approval), whether approval came before the PDUFA date, and pre-event price run-up.