# 249532524_Ustunkal_Lacin_Code
Code for: Market Anomalies during Geopolitical and Geoeconomic Conflict: Evidence from Russia

Due to the limitation of the dissertation upload section, this GitHub page will have all the necessary files required to run my code successfully, so please download the Excel file as well as all the CSV files before running the code

Notes about the code:
1. Change your filepath as necessary: I recommend using your desktop folder
2. Due to the API being a $99 per month feature from Twelvedata.com (required to import MOEX equities), I have not featured the data collection code, as my membership has expired due to the fact that the University of Bath does not have an API budget for students, and I can't keep paying the subscription fee.
3. Other than GitHub, I didn't know how else to submit the code. I tried contacting the university, but no one got back to me.
4. Finally, the code has two aspects which are worth mentioning: 1) some things may seem like redundancies; however, after my numerous attempts, I realised that the notebook requires most of them in order to run successfully, as the length made computation times rather long. 2) Some of the steps are directly taken from other attempts, so the reason they don't line up 1-X is due to this.

The following is an outline of how the code works (scroll down for details on each, after line 76):

STEP 1 — Load & validate MOEX wide Excel

STEP 1B — Per-ticker CLOSE plots

CELL A — Apply PLZL 10-for-1 split (create *_pa)

CELL B — Plot all stocks (uses *_close_pa if available)

FIXED 2B-1 — Reshape LONG + trading flags

2B-2 — Returns/features + sector/market indices (EW & VW)

3A — Align events to trading days

3B — Build estimation/event windows + coverage

3C-1 — Map reopen day0 + halt jump

3C-2 — Reopen event-time panel (k = −5…+10)

4A — Announcement event study (Market Model + Patell/BMP-like)

4B — Reopen-day CAAR

P4B — Reopen CAAR plots

P4C — Lollipop: halt jump by ticker

T4A — Announcement endpoints + BH-FDR

T4B — Reopen endpoints + BH-FDR

4A′ — Two-factor announcement AR/CAR

P4A′ — Sector CAAR plots with coverage badges

5A — Volatility features & sector VR panels

5B — Announcement volatility tests + BH-FDR + plots

5C — Reopen sector volatility (k = −5…+10)

5D — Sector volatility breaks (ruptures)

5D+ — Outliers & winsorized breaks (enhanced)

5E — Liquidity event tests (Amihud/Roll)

5F — GARCH(1,1) variance persistence (NW-HAC)

T6A — Date-clustered pooled CAR tests

6B — Calendar anomalies during conflict (weekday shift)

T7A — Calendar-time cohort portfolios (monthly) + NW α

T7B — Calendar-time portfolio regressions (monthly, HAC)

T10A — Rolling connectedness (Diebold–Yilmaz GFEVD)

T11A — GPR/EPU vs connectedness & events

T12A — Romano–Wolf step-down (FWER control)




01) STEP 1 — Load & validate MOEX wide Excel

Purpose: Ingests master wide-format Excel and sanity-check it per ticker.
Inputs: MOEX_FULL_CALENDAR_26_1998_2025.xlsx (wide OHLCV), optional events_master.csv (only sanity messaging).
Outputs:

moex_validation_report.csv (per-ticker listing start, last valid date, non-positive prices, negative/zero volume counts, missing OHLC, etc.)

moex_wide_clean.parquet (primary) with CSV fallback.
Ops: Standardizes column names to TICKER_{open,high,low,close,volume}, coerces numerics, sorts, de-dupes by date, computes listing period, flags invalid values, prints a summary, and saves a cleaned wide file.

02) STEP 1B — Per-ticker CLOSE plots

Purpose: Visual QC: one PNG per ticker showing CLOSE from first valid to last valid date.
Input: moex_wide_clean.parquet
Output: /moex_close_plots/{TICKER}_close.png
Ops: For each ticker: find valid span, plot CLOSE with nice date formatting, save PNG.


03) CELL A — Apply PLZL 10-for-1 split (2025-03-25)

Purpose: Create post-adjusted (*_pa) OHLC columns for PLZL if the vendor hasn’t already adjusted.
Input: moex_wide_clean.parquet
Outputs: moex_wide_clean_ca.parquet, moex_actions_ledger.csv, plots in /moex_actions_plots
Ops: Detects split date; checks raw pre/post price ratio to avoid double adjusting; if needed, scales PLZL OHLC by 1/10 from split onward; logs action to a ledger; makes before/after plots.


04) CELL B — Plot all stocks (prefers adjusted columns)

Purpose: One chart per ticker using _close_pa if present (PLZL), otherwise _close.
Input: moex_wide_clean_ca.parquet
Output: /moex_close_plots_adjusted/{TICKER}.png
Ops: Plots and saves per-ticker time series; skips tickers lacking data.


05) FIXED 2B-1 — Reshape to LONG + trading-status flags

Purpose: Convert wide OHLCV to a long panel and attach robust trading flags.
Inputs: moex_wide_clean_ca.parquet (or raw clean if CA missing)
Outputs: ohlcv_long.parquet, qc_long_flags_report.csv
Ops: Picks _pa over raw where available; builds is_listed, is_trading_day/valid_ret style flags; outputs a QC summary.


06) 2B-2 — Returns & features + sector/market indices (EW & VW-proxy)

Purpose: Compute per-stock daily returns and build sector equal-weight and “VW-proxy” (dollar-volume-weighted) returns; create market proxies.
Input: ohlcv_long.parquet
Outputs:

features_basic.parquet (ret, dollar volume, flags, etc.)

sector_returns.parquet (EW), sector_returns_vw_proxy.parquet

market_proxy.parquet (EW), market_proxy_vw_proxy.parquet

QC: qc_returns_summary.csv, qc_extreme_returns.csv
Ops: Cleans/guards, computes returns, aggregates to sectors (EW and VW-proxy), builds EW/VW market series, records extremes.


07) 3A — Load events & align to trading days

Purpose: Snap event dates to the next trading day if they fall on non-trading days.
Inputs: events_master.csv, calendar from moex_wide_clean_ca.parquet
Outputs: events_clean_aligned.csv, events_shift_log.csv
Ops: Parses/cleans events, aligns to calendar, prints shifted rows, saves aligned list and a shift log.


08) 3B — Estimation/event/post windows + coverage

Purpose: Build estimation windows (primary/fallback), event windows, and a coverage matrix per (ticker, event).
Inputs: events_clean_aligned.csv, features_basic.parquet
Outputs: event_windows_spec.csv, coverage_matrix.csv, coverage_summary_by_event.csv
Ops: For each eligible pair, selects primary (e.g., −120…−20) or fallback pre-window, counts valid returns in event windows, sets use_for_eventstudy.


09) 3C-1 — Detect 2022 halt, map reopen day0, compute halt jump

Purpose: Identify the exchange halt gap, map each ticker’s reopen date, and measure the reopen jump.
Input: features_basic.parquet
Outputs: reopen_map.csv, halt_jump_summary_by_sector.csv, market_trading_counts_2022.csv
Ops: Finds first traded day post-halt per ticker, computes log jump vs pre-halt, summarizes by sector.


10) 3C-2 — Reopen-anchored event-time panel (k = −5…+10)

Purpose: Build an event-time panel around each ticker’s reopen day.
Inputs: features_basic.parquet, reopen_map.csv
Outputs: reopen_event_panel.parquet, reopen_panel_coverage.csv
Ops: For each ticker, extracts k-window returns and validity flags; saves coverage stats.


11) 4A — Announcement-day event study (Market Model + Patell/BMP-like)

Purpose: Compute AR/CAR around aligned announcements using a market model; aggregate CAAR by sector; report cross-sectional stats per k.
Inputs: features_basic.parquet, market_proxy.parquet, sector_returns.parquet, events_clean_aligned.csv, coverage_matrix.csv, event_windows_spec.csv
Outputs:

event_AR_timeseries.parquet (AR & standardized AR)

event_CAR_by_ticker.csv

event_CAAR_by_sector.csv (+ BMP-like per-k stats), event_summary_tests.csv
Ops: OLS on estimation window to get α,β; predict on event window to form AR; standardize (Patell-style variance); cumulate to CAR/CAAR; compute Patell/BMP-like k-level tests.


12) 4B — Reopen-day CAAR (k = −5…+10)

Purpose: Compute sector/overall CAAR around reopen using the k-panel.
Input: reopen_event_panel.parquet
Outputs: reopen_CAAR_overall.csv, reopen_CAAR_by_sector.csv, reopen_endpoints_tests.csv
Ops: Mean per k (sector and ALL), endpoint t-tests for selected ks, saves tables.


13) P4B — Reopen CAAR plots

Purpose: Plot overall and by-sector CAAR paths.
Inputs: reopen_CAAR_overall.csv, reopen_CAAR_by_sector.csv
Output: /plots_reopen_CAAR/*.png
Ops: Line charts with zero line, legends, saved PNGs.


14) P4C — Lollipop of halt jumps

Purpose: Visualize per-ticker reopen jump magnitudes.
Input: reopen_map.csv
Outputs: /plots_halt_jump/halt_jump_lollipop.png, halt_jump_sorted.csv
Ops: Sorted lollipop; labels extremes; saves plot and sorted table.


Purpose: Multiple-testing-aware endpoint significance at chosen ks.
Input: event_AR_timeseries.parquet
Output: event_endpoints_announcement_FDR.csv
Ops: Build per-event/per-sector endpoint stats, compute p-values, apply BH-FDR, flag 10%/5%.


16) T4B — Reopen endpoints with BH-FDR (k ∈ {0,+3,+10})

Purpose: Same as T4A but for reopen k-endpoints.
Input: reopen_event_panel.parquet
Output: reopen_endpoints_FDR.csv
Ops: t-tests by sector at specified ks; BH-FDR; significance flags.


17) 4A′ — Two-factor robustness (Market + Sector)

Purpose: Re-estimate announcement AR/CAR with ret = α + β_mkt·mkt + β_sec·sector_eq.
Inputs: features_basic.parquet, market_proxy.parquet, sector_returns.parquet, events_clean_aligned.csv, coverage_matrix.csv, event_windows_spec.csv
Outputs: event_AR2_timeseries.parquet, event_CAAR_by_sector_2factor.csv
Ops: Estimation OLS with two regressors; Patell-style prediction variance; recompute AR/CAAR and save.


18) P4A′ — Sector CAAR plots with coverage badges

Purpose: Plot sector CAAR (k = −5…+5) and annotate legend/coverage counts.
Inputs: event_AR_timeseries.parquet, coverage_summary_by_event.csv
Output: /plots_event_CAAR_badges/*_sector_CAAR_badges.png
Ops: For each event, plot sector CAAR with N used/total tickers shown.


19) 5A — Volatility features & sector volatility panels

Purpose: Build per-stock volatility proxies and sector-level variance series.
Input: features_basic.parquet
Outputs: features_vol.parquet, sector_vol_eq.parquet (multi-index: AR_eq, VR_eq, VR_roll21, VR_vw, N_eq)
Ops: Per-stock r², EW sector variance, 21-day rolling variance, VW-proxy variance, and counts; pivot to wide multi-index.


20) 5B — Announcement-day volatility tests (BH-FDR) + plots

Purpose: Test pre/post abnormal volatility around announcements by sector, with FDR control.
Inputs: features_vol.parquet, events_clean_aligned.csv, coverage_matrix.csv, event_windows_spec.csv
Outputs: event_vol_endpoints_FDR.csv, plots in /plots_event_volatility
Ops: Build event-time sector volatility paths (VR minus pre mean), run endpoint tests, apply BH-FDR, and plot k-paths.


21) 5C — Reopen-day sector volatility (k = −5…+10)

Purpose: Abnormal volatility around reopen by sector.
Input: reopen_event_panel.parquet
Outputs: reopen_volatility_by_sector.csv, /plots_reopen_volatility/reopen_abnVOL_by_sector.png
Ops: Compute pre mean VR, get abnormal VR per k, save table and plot.


22) 5D — Sector volatility structural breaks (ruptures)

Purpose: Locate breakpoints in sector volatility levels.
Input: sector_vol_eq.parquet
Outputs: sector_vol_breaks.csv, plots in /plots_sector_vol_breaks
Ops: If ruptures available, run a variance change algorithm with sensible min_size/penalty, mark break dates on charts, save results; otherwise record that the lib isn’t installed.


23) 5D+ — Outliers & winsorized break detection (enhanced)

Purpose: Diagnose extreme r² outliers and compute winsorized sector VR; re-do break detection.
Inputs: features_vol.parquet, sector_vol_eq.parquet
Outputs:

Diagnostics: outliers_top_r2_by_sector.csv, winsor_thresholds_by_ticker.csv, winsor_impact_by_ticker.csv

Panels/Breaks: sector_vol_eq_winsor.parquet, sector_vol_breaks_winsor.csv
Ops: Identify top r² outliers per sector, set winsor thresholds, rebuild winsorized sector VR, and run break finding again (plots analogous).


24) 5E — Liquidity event tests (Amihud & Roll proxies)

Purpose: Event-window tests for liquidity proxies with BH-FDR and plots.
Inputs: features_basic.parquet (Amihud/dollar-vol), events_clean_aligned.csv, coverage_matrix.csv
Outputs: event_liquidity_endpoints_FDR.csv, plots in /plots_event_liquidity
Ops: Build pre/event/post windows for liquidity metrics, compute endpoint statistics, BH-FDR, and draw event-time paths.


25) 5F — GARCH(1,1) persistence (pre vs post variance, HAC t-tests)

Purpose: Measure variance shifts post-event via GARCH(1,1) variance series per sector, with HAC inference.
Inputs: sector_returns.parquet, events_clean_aligned.csv, coverage_matrix.csv
Output: garch_persistence_by_sector.csv, optional console top hits; plots dir created.
Ops: For each event×sector, fit ConstantMean+GARCH(1,1) (via arch); fallback to 21-day rolling variance if fitting fails. Run NW-HAC regression of log variance on a post dummy, report α+β, t-stat, p-value, and whether fallback occurred.


26) T6A — Date-clustered pooled tests of sector CARs

Purpose: Cluster by event date to avoid cross-sectional overstatement; test per sector and overall.
Input: event_CAR_by_ticker.csv
Outputs: clustered_sector_CAR_means.csv, clustered_sector_tests.csv, clustered_overall_tests.csv
Ops: Average CARs within each (label, date, sector, window); then across dates: compute t/p and BH-FDR; also an overall (pooled across sectors via within-date averaging).


27) 6B — Calendar anomalies during conflict (weekday shift)

Purpose: Test if weekday effects (Mon/Tue/Thu/Fri vs Wed) shift from pre [−60,−6] to post [0,+60] around events.
Inputs: features_basic.parquet, events_clean_aligned.csv
Outputs: calendar_anomalies_weekday_shift.csv, /plots_calendar_anomalies/*.png
Ops: Build pre/post masks over the calendar; estimate HAC OLS with weekday dummies; BH-FDR across weekdays and windows; plot coefficient shifts.


28) T7A — Calendar-time cohort portfolios (monthly) + NW alpha tests

Purpose: Classic calendar-time post-event persistence test using monthly EW portfolios of event cohorts.
Inputs: features_basic.parquet, market_proxy.parquet, events_clean_aligned.csv, coverage_matrix.csv
Outputs:

calendar_time_portfolios.parquet (monthly returns per portfolio)

calendar_time_alphas.csv (+ optionally calendar_time_alphas_bytype.csv)
Ops: Form monthly post-event portfolios for months +1…+6, compute monthly returns and market, run Newey–West α tests (maxlags=5 due to overlap).


29) T7B — Calendar-time portfolio regressions (monthly, HAC)

Purpose: Sector-level monthly regressions with/without event-window dummies, HAC SEs.
Inputs: features_basic.parquet, market_proxy.parquet, events_clean_aligned.csv
Outputs: ct_monthly_sector_returns.parquet, ct_monthly_factors.parquet, ct_event_dummies.parquet, ct_regressions_summary.csv, plots in /plots_calendar_time
Ops: Build monthly sector returns and a monthly market; optionally add event-window dummy cohorts; estimate OLS with Newey–West; report α, β, γ (dummy), t, R²; make bar plots.


30) T10A — Rolling Diebold–Yilmaz connectedness (GFEVD)

Purpose: Compute time-varying total/directional connectedness across sectors using a rolling VAR and generalized FEVD.
Input: sector_returns.parquet
Outputs:

connectedness_total.csv (TCI over time)

connectedness_directional.csv (to/from/net by sector)

connectedness_matrix_last.csv (last window’s GFEVD matrix)

/plots_connectedness/tci.png, heatmap_last.png
Ops: Choose VAR lag by AIC (cap), roll windows, compute GFEVD shares (order-invariant); summarize TCI and directional flows; plot TCI and last-window heatmap.


31) T11A — GPR/EPU vs connectedness & event activity

Purpose: Put the connectedness in macro context and check co-movement.
Inputs: connectedness_total.csv, events_clean_aligned.csv, optional GPR.csv, EPU.csv
Outputs: Monthly panel/correlations CSVs; overlays in /plots_context
Ops: Aggregate TCI to monthly mean; count events per month; load/parse GPR/EPU monthly series; z-score, correlate, and plot overlays (TCI vs GPR/EPU; events vs TCI vs GPR).


32) T12A — Romano–Wolf step-down on date-clustered sector CARs

Purpose: Strong FWER control across sectors per window as a complement to BH-FDR.
Input: clustered_sector_CAR_means.csv
Outputs: One CSV per window (rw_stepdown_results_{window}.csv) and a combined rw_stepdown_overview.csv
Ops: For windows {ev_−5_+5, ev_0_+3, ev_0_+10}, compute observed t, run Romano–Wolf step-down with resampling over dates, enforce monotone p-values, and flag 10%/5% significance.
