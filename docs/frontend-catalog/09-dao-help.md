# Sneed DAO pages, help, developer docs, and the game

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Source root: `src/app_sneeddao_frontend/src`. All pages render `<Header />` (except `/outrun`). Header menus covering this slice:
- **Sneed DAO** menu (default `/dao`): DAO `/dao` · Dashboard `/dao_info` · Health `/health` · Custody `/guardian` · Tokenomics `/rll_info` · Loop ledger `/rll_ledger` · Revenue `/revenue` · Rewards `/rll` · Products `/products` · Partners `/partners` · Thank-you wall `/donate` · Feedback `/feedback` · Mr Sneed `/mr-sneed` · Diamond Club `/diamond-club` · Lore `/lore` · Discord (external) · X (external) · Projects `/projects` · Disclaimer `/disclaimer`.
- **Help** menu (default `/help`): Help `/help` · API `/api` · Architecture `/architecture`. `/help/*` and `/api/*` also highlight it.
- Global: the header's heart pill **Donate** (`components/DonateButton.jsx`) on every page.
- Not in any menu: `/outrun`, `/doc`, `/tokenomics`, `/tokenomics_info`, `/me_info`, `/help/canister-manager`, hidden `/api/<slug>` pages.

---

## DAO introduction — `/dao`
- **Routes & params**: `/dao`, no params. Public.
- **Purpose**: static "about Sneed DAO" landing page.
- **Information shown** (all hard-coded): About Sneed DAO (SNS-1 legacy, "Blank Canvas"); "What Makes Sneed DAO Special" (Blank Canvas Model / Community Governance / 100% On-Chain); "Web3 vs Traditional DAOs" comparison; "How Sneed DAO Works" 3 steps (Proposal Submission → DAO Decision → DAO Action); "Join the Community"; "Further Links".
- **Actions**: external links only: ICPSwap, Sonic, ICLight, ICPCoins (Web3); CoinMarketCap, SNS Dashboard (Web2).
- **Entry points**: Sneed DAO menu default; HelpDao links it.
- **Cross-links**: none internal (no link to dashboard, proposals, forum, revenue).
- **Gating/state**: none.
- **Redesign notes**: pure marketing copy with zero live data and no internal links; overlaps Lore (origin story) and HelpDao "What Sneed DAO is". External link list partly dated (Sonic, ICLight). Could merge with Lore/DAO overview.

## DAO dashboard — `/dao_info`
- **Routes & params**: `/dao_info`. Public. Auto-refresh every 60 s; prices every 5 min.
- **Purpose**: summary metrics for the Sneed DAO with drill-down links.
- **Information shown**:
  - **DAO Metrics** (from `rll.get_neuron_statistics()` + SNS governance `list_neurons` + off-chain `sns-api.internetcomputer.org/.../proposals/count`): Total Neurons; Active Neurons; Dissolve State ×3 (Not Dissolving / Dissolving / Dissolved, each count + SNEED stake); Total Voting Power (min/max); Hotkeys (total, multi-hotkey neurons); Total Proposals.
  - **Tokenomics**: SNEED token card (logo, name, Symbol, Decimals, Fee from ledger metadata); SNEED Price (denomination + ICP); "FDV" (price × total supply, USD + ICP); Total Supply; Total Staked (+% of supply); Total Rewards Distributed per token with USD and total (`rll.get_total_distributions`).
  - **Partners**: logo tiles from `backend.get_partners()` (tooltip = name + description).
  - **Products**: two hard-coded cards: SneedLock, SwapRunner.
- **Actions**: links only: "Drill down" → `/neurons`; "View proposals" → `/proposals`; "Drill down" → `/rll_info`; "View rewards" → `/rll`; "View all" → `/partners`; partner tile click → partner's first link (new tab) or `/partners`; "View all" → `/products`; "Learn more" → `/sneedlock`; "Visit site" → swaprunner.com.
- **Entry points**: Sneed DAO menu "Dashboard"; Hub token-stats tiles (`/dao_info` twice in Hub.jsx); HelpDao.
- **Shared building blocks**: `priceService`, `calculateTotalAssetsValue` (TokenUtils), `WhitelistTokensContext`, `DenominationContext`.
- **Gating/state**: none; grids collapse to 1 column ≤900 px.
- **Redesign notes**:
  - Fetches but never renders: RLL balance reconciliation, total-assets (treasury ICP/SNEED value), event statistics, latest distribution round/time, DeFi-canister (`ok64y-…`) token balances. Treasury value is computed and thrown away.
  - Products card list (2 items) is stale vs `/products` (8 items) and `/revenue` PRODUCTS (5 items) — three different product lists.
  - Same SNEED supply/staked/price numbers also computed independently by Hub, `/rll_info` and `/health` → candidates for one shared "DAO stats" source.
  - Calls `agent.fetchRootKey()` against mainnet `ic0.app` (should only be done locally).
  - Proposal count comes from an off-chain API, not governance.

## DAO health diagnostics — `/health`
- **Routes & params**: `/health`. Public, anonymous reads only (agent per canister at `https://icp-api.io`, always mainnet even on staging).
- **Purpose**: live pass/warn/fail check tree of the whole DAO; a group shows the worst status below it.
- **Information shown** (tree root "Sneed DAO", built in `health/checks.js`; leaves show summary, expandable JSON-ish details, an "open" link):
  - **Canisters** (dynamic, from SNS root canister summary): per DAO canister **Cycles** (warn < 2 T, fail < 1 T); **Running and controlled by the DAO** (running, code installed, no controllers besides SNS root); **Memory** (% of memory allocation / Wasm limit / IC default 3 GiB; warn 60 %, fail 80 %). Empty canisters optionally ignored.
  - **DAO control and code**: Hub canisters controlled by the DAO alone and registered with SNS root (backend, Sneed Lock, RLL, Premium, SMS, Forum, Sneedex, Sneedapp, sGLDT, DeFi, 6 naming canisters); Vector pylons controlled by an SNS DAO (Neutrinite, ICPCoins); "Running the Wasm of their latest upgrade proposal" (module hash vs executed proposal).
  - **Services responding**: App frontend (`/index.html` fetch time); "Token list refreshed daily" (backend `get_token_list_status`); pings of App backend, Sneed Lock, Rewards, Forum, SMS, Premium, Sneedex, Sneedapp, sGLDT, ICP staking bot, ICP neuron manager, SNEED ledger, SNEED index, SNS governance; live Custody (guardian) deployments and their children's cycles; "Sneedex expiration timer" (timer/worker running, overdue active offers).
  - **SNS core**: governance in normal mode; voting rewards on schedule; recent proposals executed without failure (30-day window); SNEED ledger tip certified; SNEED index in sync (lag ≤100 blocks); decentralization swap committed.
  - **Revenue**: DAO ICP neuron locked (≥720 days); one leaf per governance-controlled vector node (ICP neuron vector, splitters, exchange/buyback: active, not frozen, billing, last operation, pickup of hidden-pylon deliveries); "SNEED buyback delivering" (last buy age, 24 h buys); Treasury holds ICP (≥1 ICP); Treasury holds SNEED; DeFi canister holds SNEED; SNEED/ICP LP positions held by DeFi canister (expects #24, #25, #26).
  - **Rewards** (RLL): main loop running; next run scheduled/not overdue; last cycle completed in time; tokens distributed in last period; import pipeline idle/progressing; reward claims succeeding (pending >1 h, failures in 24 h); reward tokens configured.
  - **Sneed Lock**: lock state readable; activity-log tip certified; log spilling to archives; each ICRC-3 archive (cycles, parent, SNS registration).
  - Header line: overall badge ("all systems go" / "needs attention" / "problems found"), counts (passed / warnings / failed / skipped / pending), run progress and duration.
- **Actions** (buttons): "Run again"; "Auto-refresh every 5 min" (toggle); "Problems only" (filter); "Expand all"; "Expand problems"; "Collapse"; "Thresholds" → panel of 18 thresholds (cycles warn/fail, ignore empty canisters, memory %, archive cycles, RLL slack/periods, neuron dissolve days, vector activity days, buyback slack/max age, pickup hours, index lag, proposal window, min treasury ICP, call timeout) with "Save and re-run" / "Defaults" / "Cancel" — saved in localStorage only ("Saved in this browser only.").
- **Entry points**: Sneed DAO menu; Architecture page footer and "Cycles" service; HelpDao.
- **Cross-links out**: `/rll`, `/sneedlock_info`, `/tools/icrc3?canister=…`, `/guardian?deployment=…`, IC dashboard canister/neuron/account pages, NNS dapp project/proposals, ICPCoins vectors, ICPSwap position.
- **Shared building blocks**: `health/runner.js` + `health/TreeView.jsx` (Node/Badge/STATUS, also used by Token Safety / Token Studio, admin SneedPad), `health/agent.js` (anonymous actors, `readControllers` via read_state — also used by Architecture, rllLedger, Follow the Money, tokensafety), `customStyles` reused by NftItem, NftBrowser, Positions, Position, Guardian, FollowMoney, RllLedger.
- **Gating/state**: none; only "live" guardians are checked.
- **Redesign notes**: the most complete "DAO state" view (treasury, neuron, vectors, LP, RLL, Lock) but presented only as test results; the same facts are shown as numbers on `/rll_info`, `/rll_ledger`, `/dao_info`. Custody checks link to `/guardian`, which is "coming soon" for non-admins.

## Tokenomics / Recursive Liquidity Loop explorer — `/rll_info`
- **Routes & params**: `/rll_info?tab=diagram|components` (default Total assets tab). Public. File: root `RLLInfo.jsx` (4,582 lines).
- **Purpose**: explain and show live state of the DAO's revenue loop (2-year ICP neuron → ICP splitter → SNEED buyback → SNEED splitter → burn / treasury / member rewards) and the DAO's total assets.
- **Information shown**:
  - **Hero "loop, stage by stage"**: 2-year ICP neuron (stake) → "maturity" → ICP splitter (other destinations %) → % to buyback → SNEED buyback (ICP per buy) → SNEED splitter (n destinations) → fan of Burned / Treasury / Member rewards with live % (read from vectors). Link "See what actually moved: the loop ledger." → `/rll_ledger`.
  - **Price strip**: SNEED in ICP, SNEED in user denomination, ICP; "Prices from ICPSwap".
  - **Tab "Total assets"**: "Total Assets Overview" tiles FDV, Circulating market cap, NAV, TVL (each with InfoTip). Expand (▸/▾) for: **Total Value** (GAV + ICP equiv., FDV/GAV, NAV + ICP equiv., P/NAV, NGAR ratio); **Market Cap & Supply** (FDV, Circ. Market Cap, TVL, TVL/MCap, Total Supply, Circulating Supply, Total Staked, Total in LPs); **ICP Assets** (Treasury, ICP Neuron, ICP/SNEED LP + unclaimed LP rewards, DeFi Canister, Member Rewards, Buyback Vector, Other Pools: ICP/CLOWN; Total ICP); **SNEED Assets** (Treasury, ICP/SNEED LP + unclaimed, DeFi Canister, Member Rewards; Total SNEED); **Other Tokens** (DeFi Canister balances, Member Rewards balances per RLL known token; Total Value); **Other Positions** (ICP/CLOWN Position #168: current amounts, unclaimed rewards, USD; Total Value). Each line links to IC dashboard / ICPSwap / vector UIs.
  - **Tab "Diagram"** ("The loop as a diagram"): ReactFlow graph of nodes 1–18 (neuron, vectors, treasury, DeFi canister, burn, member rewards canister, Revenue, ICPSwap LP, Products, and product nodes Sneedapp/Sneed Lock/Sneedex/Premium/SNS Jailbreak), animated token particles (burn/distribution effects), legend ICP/SNEED/mixed revenue. Hover/tap opens a pinned popover per node/edge: description, inputs/outputs, live balances, NNS neuron status (stake, voting power, age, dissolve delay, state), vector status/destinations with accounts, buyback exchange status (rate, next buy, buy amount, balance), LP totals, RLL "Token Balances and Reconciliation" (all-time distributed, currently claimable, server balance per token), plus `RevenueParts` (loop-ledger 90-day lines per node/edge with link to `/rll_ledger#<section>`; product earnings with link to `/revenue`).
  - **Tab "Components"**: expandable groups Core Infrastructure / Token Management / Revenue, each item with description, inputs, outputs, live details, external link; "Token flows" groups ICP Flows / SNEED Flows / Revenue Flows (token, %, "live" tag).
  - Bottom: `RllActivityCard` (latest distributions & claims).
- **Data sources**: NNS governance `get_neuron_info`; ICP/SNEED ledgers (treasury = SNS governance + treasury subaccount, DeFi canister); vector pylons `icrc55_get_nodes` (Neutrinite nodes 1/6/45; ICPCoins exchange node 18); ICPSwap pools `getUserPositionsByPrincipal(DeFi)` (SNEED/ICP, ICP/CLOWN); RLL `get_known_tokens`, `get_total_distributions`, `balance_reconciliation_from_balances`, `get_wallet_known_tokens(DeFi)`, `get_neuron_statistics`; SNEED total supply; priceService.
- **Actions**: tabs "Total assets" / "Diagram" / "Components"; stage buttons in the hero jump to the Diagram with that node's card open; expand/collapse sections and items; InfoTip "What is this?" popovers; diagram zoom/fit controls (scroll-zoom off), tap to pin card, tap pane to close. All read-only.
- **Entry points**: Sneed DAO menu "Tokenomics"; `/dao_info` "Drill down"; Hub tokenomics tile; `/rll` intro link "Recursive Liquidity Loop"; `/revenue` footer "tokenomics diagram"; `/rll_ledger` intro; HelpDao.
- **Shared building blocks**: `components/RevenueTooltips.jsx` (LedgerSummary/ProductSummary/ProductsSummary), `utils/revenueData` + `revenueStore` (shared cache with `/revenue`, `/rll_ledger`), `RllActivityCard`, ReactFlow (also Architecture).
- **Gating/state**: none; mobile tap-to-pin popovers, pinch/zoom buttons.
- **Redesign notes**: route is named "rll_info" but labelled "Tokenomics" in the menu; it mixes three concerns (treasury/asset accounting, tokenomics ratios, loop mechanics). The Total-assets accounting is the only treasury/NAV view in the app and is buried behind expanders. Dead imports (PrincipalBox, headerStyles, logout). Hard-coded position ids (#168) and pools.

## Revenue loop ledger — `/rll_ledger`
- **Routes & params**: `/rll_ledger?period=30|90|365|all` (default 90) `#neuron|#icpSplitter|#buyback|#sneedSplitter|#lpFees` (auto-scrolls to section). Public.
- **Purpose**: "the books" — what actually moved through each loop stage, read from ICP and SNEED index canisters.
- **Information shown**: per section (1 · The 2-year ICP neuron; 2 · The ICP splitter; 3 · The SNEED buyback vector; 4 · The SNEED splitter; 5 · LP fees): tiles per flow line (total, value, tx count, "since … only" when capped); in/out totals; Per day; buyback "Average price paid" (ICP/SNEED, all-time only, vs market); "Where it went" actual share vs configured share [%] from live vector splits; per-day bar chart (in/out); transaction table (When, Flow, Amount, Counterparty with named accounts, tx link) collapsed to 3 rows. Neuron section adds tiles Stake, Dissolve delay (locked/dissolving), Age, Voting power, Neuron id link. Footer explains where each flow is measured.
  - Flow lines: maturity minted by NNS; stake deposits; splitter deposits (product revenue); to buyback; to ICP treasury; ICP handed to buyback; SNEED bought; SNEED from buyback; Burned; to SNEED treasury; to RLL rewards; ICP/SNEED fees claimed from SNEED/ICP pool; ICP fees from ICP/CLOWN pool; ICP/SNEED sent on to splitters.
- **Actions**: period buttons "30 days" / "90 days" / "1 year" / "All time"; section chips (anchor nav); per-section refresh (title "Re-read from the index"); show all / show latest rows toggle; row `#id` → `/transaction?ledger=…&id=…` (+`&sns=` for SNEED); external "the vector" / "the neuron".
- **Entry points**: Sneed DAO menu "Loop ledger"; `/rll_info` hero link and diagram cards; `/revenue` "In the loop ledger" and source cards (`?period=all#neuron`, `#lpFees`); HelpDao.
- **Shared building blocks**: `utils/rllLedger.js` (account map, index readers, SECTIONS), revenue cache, `PrincipalDisplay`, recharts, `health/TreeView` styles.
- **Gating/state**: rows capped (MAX_ROWS 6000) per account — "since … only" notes.
- **Redesign notes**: best "audit trail" view; overlaps `/revenue` "Into the loop" card (same icpSplitter section) and `/rll_info` diagram cards (same lines over 90 days).

## Revenue — `/revenue`
- **Routes & params**: `/revenue?period=30|90|365|all` (period only affects "Into the loop"); anchors `#neuron`, `#lp`, `#products`, `#<productId>`. Public.
- **Purpose**: what the DAO earns, by source, all time.
- **Information shown**:
  - **Revenue, all time** (sum of all sources, value at today's prices) + share bar (2-year neuron / ICPSwap LP / Products).
  - Source cards: **2-year ICP neuron** (maturity total, payouts, since) and **ICPSwap LP** (SNEED/ICP pool ICP+SNEED, ICP/CLOWN pool ICP "its CLOWN not counted", first/last claim) — from loop ledger, all time.
  - **Earned by the products, all time** + share bar; per product card (canister lifetime counters): **Sneedapp** (DAO cut per app with "n minted" and link to app page; Bots minted); **Sneed Lock** (token locks, LP position locks); **Sneedex** (offer creation fees "n paid", cut of completed sales per token "n sales"; Offers/Completed/Bids); **Sneed Premium** (memberships paid in ICP "n payments"; Claimed with voting power; Promo codes); **SNS Jailbreak** (regular fee, premium fee, payments; Scripts made; Users). Hidden-total notice when a canister keeps totals admin-only ("Its canister still shows its revenue totals to admins only…").
  - **Into the loop** (period): Deposits (product fees and more), Neuron maturity, Together; stacked per-day bar chart.
  - `RllActivityCard`.
- **Actions**: refresh (title "Read the ledger and the products again"); "Try again" per card; period buttons; card/part links (`/sneedapp`, `/sneedlock`, `/sneedex`, `/premium`, `/tools/sns_jailbreak`, `/sneedapp/app/<id>`, `/rll_ledger…`); footer links to `/rll_info`, `/rll_ledger`.
- **Entry points**: Sneed DAO menu; RevenueTooltips product cards on `/rll_info`; Architecture service "Revenue"; HelpDao.
- **Shared building blocks**: `utils/revenueData.js` (PRODUCTS, SOURCES, product loaders: sneedapp `getAllAppRevenueStats/getDaoRevenueStats`, sneed_lock `get_lock_fee_stats`, sneedex `getPaymentStats/getMarketStats`, premium `getPaymentStats`, backend `get_jailbreak_payment_stats`), `revenueStore` cache (10-min TTL, shared across pages), `RllActivityCard`.
- **Redesign notes**: overlaps `/rll_info` (product nodes, revenue node), `/rll_ledger` (same data), `/products` (different product set, usage stats rather than revenue). Revenue numbers per product are not shown on the products' own pages.

## Voting rewards (public RLL page + claim) — `/rll`
- **Routes & params**: `/rll`. Public; "Your rewards" section needs login. File: root `RLL.jsx`.
- **Purpose**: the RLL rewards canister: stats, the signed-in user's claimable rewards and claim, distribution history.
- **Information shown**:
  - Intro (rewards shared daily to Sneed neuron owners by voting power cast; links `/rll_info`, `/help/dao#rewards`).
  - **Statistics**: Distributed, all time (per token, value at today's prices); tiles Distributions, Owners rewarded (credits), Claims paid (by n principals), Last 24 hours; Claimed all time; Distributed in last 24 h. Transitional banner "Statistics are being fixed (proposal #444)…".
  - **Your rewards** (login): n neurons (of n owners), "your votes so far weigh X % of a distribution"; Claimable per token with value; "not above the fee yet"; tabs **Received (n)** (credits: amount, value, owner when several, date, proposal range) and **Your claims (n)** (status Paid/Failed/Pending, amount, "ledger tx N" link, error, date, seq). Empty state links `/help/neurons`.
  - **Distributions** (paged, 20/page, newest first): amount, value, proposal range, timestamp; total count.
  - `RllActivityCard`; footer with canister id and `/api/rll` link.
- **Actions**: "Sign in"; "Claim" per token → ConfirmationModal "Claim X? It is sent to your Sneed Hub wallet, less the ledger fee of Y." → `rll.claim_full_balance_of_hotkey(ledger, fee)` (ICRC-1 transfer from RLL to user; disabled when amount ≤ fee); toast via root `Notification` (success with ledger block / errors InsufficientFunds, BadFee, GenericError); tabs; Pager; refresh buttons ("Read the statistics again", "Read your rewards again", "Read the distributions again"); "+N more"/"fewer" token lists.
- **Data**: `fetchUserNeuronsForSns` (Sneed governance), `rll.balances_of_hotkey_neurons`, `get_hotkey_voting_power`, `get_claim_events_for_hotkey`, `get_user_specific_distribution_events` per owner, paged `get_distribution_events_page` (fallback unpaged), `get_event_statistics`.
- **Entry points**: Sneed DAO menu labelled **"Rewards"**; `/dao_info` "View rewards"; `/health` RLL leaves; RLLInfo member-rewards node link (absolute `https://app.sneeddao.com/rll`); HelpDao.
- **Shared building blocks**: `utils/rllApi.js` (also `/admin/rll`), `Pager`, `ConfirmationModal`, `PrincipalDisplay`/naming, revenueData token meta/prices.
- **Redesign notes**: **duplicates `/rewards`** (Sneed Me menu "Rewards", `pages/Rewards.jsx`, 1,199 lines) which uses the same `balances_of_hotkey_neurons` / `claim_full_balance_of_hotkey` / `get_claim_events_for_hotkey` calls with different UI ("Sneed Rewards", "Unclaimed Rewards", "Total Claimed", "Claim History"). Two menu items both named "Rewards" in different sections. HelpDao's step-by-step points to `/rewards`, then describes `/rll` too. Admin side is also split: `/admin/rll` and `/admin/rewards`.

## RLL activity log card (shared) — `components/RllActivityCard.jsx`
- **Where**: `/rll`, `/rll_info`, `/revenue`.
- **Information shown**: "Recent activity" from RLL ICRC-3 log: latest 5 distributions and 5 paid claims (amount, value, who, when, block #, claim's ledger tx); "All N blocks" link.
- **Actions**: refresh; links to `/tools/icrc3?canister=<rll>&block=<n>`, `/transaction?id=…&ledger=…`.
- **State**: "The activity log arrives with the next RLL upgrade." when the canister lacks ICRC-3.

## Products overview — `/products`
- **Routes & params**: `/products`. Public (uses identity if present).
- **Purpose**: marketing overview of DAO products with some live stats.
- **Information shown**: hero "Our Products". Cards: **Sneedex Swap** (DEX aggregator features); **Sneedex** (Active Offers, Total Offers, Completed, Total Bids — `getMarketStats`); **SwapRunner** (Total Swaps, Split Swaps, Kong Swaps, ICPSwap Swaps, Registered Users, Active Traders — swaprunner backend); **Sneedapp**; **Trading Bot**; **ICP Staking Bot**; **Liquid Staking**; **SneedLock** (Token Locks, Position Locks, Active Users, Token Locks Value, Pos. Locks Value, Total Value Locked — all locks via `getAllTokenLocks/getAllPositionLocks`, per-pool position amounts via ICPSwap). Refresh every 5 min, animated counters.
- **Actions** (links): "Swap Tokens" → `/swap`; "Explore Sneedex" → `/sneedex_offers`; "Visit SwapRunner" → swaprunner.com; "Browse Apps" → `/sneedapp`; "Learn More" → `/sneedapp/app/sneed-trading-bot`; "Learn More" → `/sneedapp/app/sneed-icp-staking-bot`; "Start Liquid Staking" → `/liquid_staking`; "Launch SneedLock" → `/sneedlock`.
- **Entry points**: Sneed DAO menu; `/dao_info` "View all"; HelpDao.
- **Redesign notes**: USD value tiles always 0 — `fetchConversionRates` is stubbed ("Rate fetching disabled"); unused Neutrinite import; heavy console logging; loads every lock on page view. Missing products that `/revenue` counts (Premium, SNS Jailbreak) and newer ones (Token Studio, Custody, Forum, SMS, Notifications). Overlaps Hub "Featured Products", `/projects` "Products" group, `/dao_info` Products card.

## Partners — `/partners`
- **Routes & params**: `/partners`. Public.
- **Purpose**: DAO-curated partner list.
- **Information shown**: per partner: logo (fallback initial), name, description, link buttons (title + url) — `backend.get_partners()`. Empty state "No partners yet".
- **Actions**: external partner links (new tab).
- **Entry points**: Sneed DAO menu; `/dao_info` Partners tiles/"View all". Admin editing at `/admin/partners`.
- **Redesign notes**: no relation shown to "partner prices" (`SpecialPricesCard`, admin-only, per-canister partner discount lists with public labels) — the public never sees which partners get discounts.

## Projects — `/projects`
- **Routes & params**: `/projects`. Public.
- **Purpose**: DAO-curated ecosystem list grouped by type.
- **Information shown**: sections **Products**, **Projects**, **Community Forks** (sorted by admin index then name); card: logo, name, type badge, links, description — `backend.get_projects()`.
- **Actions**: external links.
- **Entry points**: Sneed DAO menu (listed after Discord/X, near Disclaimer). Admin at `/admin/projects`.
- **Redesign notes**: "Products" group is a third, admin-maintained product list, separate from `/products` (hard-coded) and `/revenue` (code constant). Merge candidates: Products + Projects + Partners = "Ecosystem".

## Donations: donate dialog + thank-you wall — header **Donate**, `/donate`
- **Routes & params**: `/donate`. Wall public; donating from wallet needs login.
- **Purpose**: tip "the people who build Sneed Hub" (a backend-configured donation account, not the DAO treasury) and show a public thank-you wall.
- **Information shown**: `/donate` hero "Thank you"; stats Donors, Gifts, Given (value); per-token totals chips; donor cards (avatar, name/principal, date, amount + value, optional quoted message), 48 per page.
- **Actions**:
  - Header pill **Donate** (label hidden on narrow screens) or `/donate` **Donate** → dialog "Support Sneed Hub".
  - Logged out: copy rows "Principal (any ICRC-1 token)", "Subaccount", "ICP account ID (exchanges)"; "Log in to donate"; note "Only donations made here while logged in go on the wall."
  - Logged in: Token (TokenSelector, default ICP, shows Balance), Amount presets ($5/$20/$50/$100 converted to token) or "Other amount", Message (optional, 200 chars, "shown on the wall") → "Continue" → confirm (To, Network fee, Your message, public notice) → "Back" / "Send donation" → ICRC-1 transfer (memo "Sneed Hub donation") + `backend.register_donation(ledger, block, message)`; then automatic Sneed SMS from donor to recipient (best effort, uses donor's SMS allowance) → done "Thank you!" with "See the wall" / "Close".
  - Recovery: sent-but-unregistered donations kept in localStorage → banner "A donation you sent is not on the wall yet." + "Add it".
  - `/donate`: "Show more".
- **Entry points**: header heart (global); Sneed DAO menu "Thank-you wall"; HelpDao; admin `/admin/donations` (set account, hide donations).
- **Shared building blocks**: `utils/donations.js` (own IDL), `TokenSelector`, `BigAvatar`, `PrincipalDisplay`, revenueData prices, `icrc1Transfer`.
- **Gating/state**: "Donations are not open yet." when no account set; admin can hide donations.
- **Redesign notes**: menu label "Thank-you wall" vs page title "Thank you" vs dialog "Support Sneed Hub". Side-effect SMS is not disclosed in the confirm step. Donations go to builders, distinct from DAO revenue — worth making explicit next to Revenue/Treasury.

## Feedback (bug / idea / private security report) — `/feedback`
- **Routes & params**: `/feedback?kind=bug|idea&from=<path>` (from defaults to same-site referrer). Reading recent reports public; posting needs login.
- **Purpose**: turn a bug report or idea into a public Sneed-forum thread and ping maintainers by SMS; or send a private security report by SMS.
- **Information shown**: intro; form; "Recent reports" (last 10 threads from configured bug/idea topics: kind icon, title, author, date, reply count).
- **Actions**: tabs "Report a bug" / "Suggest an idea"; "Log in"; title (100 chars), Details (MentionTextarea with @-mentions; 3,500 public / 1,700 private chars); bug only: checkbox "Attach these details…" (Page, App build from `/version.json`, Browser UA, Screen) and checkbox "This is a security issue: send it privately…"; submit "Post bug report" / "Post idea report" / "Send privately" → forum `create_thread(topic, "Bug: …"/"Idea: …", body)` + SMS `send_message` to recipients (+ SMS to @-mentioned users); security → SMS only. Success: "Open the thread" / "Your messages" (`/sms`) / "Send another".
- **Entry points**: Sneed DAO menu; HelpForum/HelpDao links; Diamond Club error text ("Report it on the feedback page").
- **Shared building blocks**: `utils/feedback.js`, `MentionTextarea`, `utils/mentions.smsMentioned`, forum + SMS canisters.
- **Gating/state**: "Feedback is not open yet." (old backend); per-kind disabled if no topic set; private reports disabled if no recipients. Config at `/admin/feedback`.
- **Redesign notes**: no global "report a problem" entry from pages (relies on referrer); feedback is really a forum + SMS feature living under DAO.

## Mr Sneed (DAO mascot's on-chain memory & governance) — `/mr-sneed`
- **Routes & params**: `/mr-sneed?path=&kind=&status=active|superseded|removed|all&q=&memory=<id>`. Public reads; "about me", removal and proposing need login (and a Sneed neuron with SubmitProposal).
- **Purpose**: inspect Mr Sneed's canister (identity, DAO-set personality, rule frame, governance, memories) and propose changes to Sneed DAO.
- **Information shown**: facts: Principal, Neuron (→ `/neuron?neuronid=…`), Canister (dashboard), Governed by (DAO / operator until handover), Log (→ `/tools/icrc3?canister=…`), May write memories (writers), Limits (writes/day, max pinned, max active, max bytes), Memories stats (active/total/pinned/bytes). **Personality** (version, set date, set-by proposal link, SHA-256, text; "Earlier versions (n)"). **"The frame: rules in his code"** (prompt frame + hash). **What Mr Sneed remembers about you** (login). **Memories** browser: folder tree with counts, kind chips (note, decision, lesson, project, reference, member; "Any kind"), status select, search ("Search titles, folders and tags"), paged list; each memory: title, pinned, kind detail, status (removed by/reason, superseded → link), text, links (proposal/commit/thread/URL), author/date, "Link" permalink.
- **Actions**: reload; search/clear; filter by folder/kind/status; "Remove" (memory about me) → ConfirmationModal → `mr_sneed.forget_about_me(id)` (direct, no proposal); "Propose a change" (personality), "Propose new writers", "Propose new limits", per-memory "Propose removing" → ProposeDialog ("Propose a new personality" / "Propose who may write memories" / "Propose new memory limits" / "Propose removing a memory"): edit args, canister validator preview ("What voters will see"), Proposal summary, "From your neuron" picker → SNS governance `manage_neuron(MakeProposal ExecuteGenericNervousSystemFunction)`; result link to `/proposal?proposalid=…`. Warns about rejection fee; blocked with notice if DAO hasn't registered the function.
- **Entry points**: Sneed DAO menu; Lore "Meet him"; HelpProposals ("the one place Sneed Hub files proposals itself"); HelpDao; SKILL.md.
- **Shared building blocks**: `utils/mrSneed.js`, `PrincipalInput`, `ConfirmationModal`, naming.
- **Gating/state**: "Mr Sneed's canister is not deployed on this network."; proposal types may be unregistered.
- **Redesign notes**: the only generic "submit an SNS proposal" UI in the app — hidden inside a mascot page. Governance actions mixed with a memory browser.

## Diamond Club EXT → ICRC-7 migration — `/diamond-club`
- **Routes & params**: `/diamond-club`. Public status; swapping needs login (or proof-by-payment from another wallet).
- **Purpose**: one-way 1:1 migration of Sneed Diamond Club NFTs from the old EXT canister to `sneed_nft` (ICRC-7, on-chain images).
- **Information shown**: user's principal (receiver) and **Swap address**; old cards in the Sneed Hub wallet (EXT); ICRC-7 cards held (→ `/nft`); swap progress tiles Swapped / Retired by proof / Still on EXT; retired cards grid; "My proofs"; card lookup result.
- **Actions**: "Sign in"; refresh; "Swap all N" / per-card "Swap" (EXT transfer from Sneed Hub wallet to swap address, then `claim_swap`) with appConfirm; manual path for other wallets (send to swap address, then "Claim"; auto-claims when it sees waiting cards); **Proof by payment**: pay small ICP (account ID) or SNEED (ICRC-1) to proof address from the holding wallet → "I've sent it" (polls index) / search → enter paying wallet principal → claim (`claim_by_proof`); Advanced: "Proof ledger", block index, "Check this payment", "Claim by proof", proof-address details (owner, subaccount, account identifier); "Check card" (is an EXT card retired?); "Retry refund" on proofs.
- **Entry points**: Sneed DAO menu; NftCard / NftCollectionCard banners for EXT cards (`extSwapPage`).
- **Shared building blocks**: `utils/sneedNft.js`, `utils/ExtUtils.js`, `components/convert/ConvertUi` (Card, Notice, Advanced, CopyValue, QrCode), `appAlert/appConfirm`.
- **Gating/state**: "The swap is not open yet." / collection not on this network.
- **Redesign notes**: an NFT-wallet utility filed under the DAO menu; belongs with NFTs (`/nft`) or a "Migrations/Convert" area (the app also has `/convert` for ck-assets). API doc `sneed_nft` exists but is hidden.

## Lore — `/lore`
- **Routes & params**: `/lore`. Public.
- **Purpose**: origin story.
- **Information shown**: The name; The dot (SNS-1 letter, Sneed Vision Letter — external canister sites); The road so far (dated timeline Nov 2022 → Now); The old token (pre-SNS SNEED ledger `r7cp6-…`, blackholed; live remaining supply via `icrc1_total_supply`; converter `5ceyv-…`); Join us; Mr Sneed today.
- **Actions**: external: the two letters, converter app (external frontend), dashboards for old ledger and converter, Discord, X; internal: `/forum` ("The Sneed forum"), `/mr-sneed` ("Meet him").
- **Redesign notes**: old-SNEED conversion happens on an external site, not in-app. Overlaps `/dao` copy.

## Disclaimer — `/disclaimer`
- **Routes & params**: `/disclaimer`. Public. Static legal text.
- **Sections**: IMPORTANT NOTICE - NO RESPONSIBILITY DISCLAIMER; Complete Disclaimer of Liability; Absolute Non-Responsibility Statement; High-Risk Activity Warning; User Responsibility; No Financial Advice; Indemnification; Final Warning.
- **Entry points**: Sneed DAO menu (last item), HelpDao. Not linked from login, footer or any transaction flow.

## Placeholders, legacy and dead files in this slice
- `/tokenomics`, `/tokenomics_info`, `/me_info`: 7-line pages rendering `ComingSoon` ("Coming Soon", hard-coded dark colors). **Not redirects.** Not in menus; HelpDao says live tokenomics are on `/rll_info`.
- `/doc` (`src/Doc.jsx`): "Documentation coming soon!" placeholder; not in menus.
- `src/Banner.jsx` (+Banner.css): old Sneed Lock logo banner; **imported nowhere** (dead).
- `src/Notification.jsx`: tiny auto-dismiss toast (success/error, 3 s); used only by `/rll` and `/rewards` (rest of app uses other dialogs).
- `/guardian` ("Custody" in Sneed DAO menu) and `/custody` (Tools menu): ComingSoonPage — admins see preview, others "coming soon". Same feature reachable from two menus with different routes.
- `/dashboard`: older Sneed Lock list (HelpDao documents it as legacy); not in menus.
- `components/SpecialPricesCard.jsx`: **admin-only** ("Admin & partner prices": admin price + partner list with per-fee overrides/expiry) on admin pages of Sneed Lock, Sneedapp, SneedMon, SNS Jailbreak and Guardian; not DAO-facing. Public effect only through lower prices in LockModal/LockPositionModal/pad prices.

## Help index — `/help`
- **Routes & params**: `/help`. Public. No search.
- **Information shown**: topic cards (icon, title, description) in groups:
  - Getting started: Sneed Hub on Your Phone `/help/app`; Understanding Your Wallet `/help/wallet`; Names, Profiles and Settings `/help/names`.
  - Sneed DAO and governance: Sneed DAO `/help/dao`; Proposals and Voting `/help/proposals`; Understanding SNS Neurons `/help/neurons`; Liquid Staking `/help/liquid-staking`.
  - Community: Forum, Feed and Tips `/help/forum`; Sneed SMS Messages `/help/sms`; Notifications `/help/notifications`; Sneed Monitor `/help/monitor`.
  - Locking and trading: Understanding Sneedlock `/help/sneedlock`; Sneedex Marketplace `/help/sneedex`; SNS Jailbreak `/help/sns_jailbreak`.
  - Apps and bots: Sneedapp App Store `/help/sneedapp`; App Manager `/help/dapp-manager`; ICP Staking Bot App Canisters `/help/icp-neuron-manager`; Sneed Trading Bot `/help/trading_bot`.
  - Tools: Token Studio and Token Safety `/help/token-studio`; Tools and Explorers `/help/tools`.
  - "For AI agents and developers" / "Ask your AI agent": link to `/llms.txt`, `/api` ("Open the API docs"), `/architecture` ("Open the architecture map").
- **Entry points**: Help menu; SKILL.md; links from many pages' help icons.
- **Redesign notes**: no help topic for Swap, Premium (inside HelpDao), Hub, Diamond Club, Custody, Convert, Pockets, Drive; route naming mixes `-` and `_` (`sns_jailbreak`, `trading_bot` vs `icp-neuron-manager`, `token-studio`); `/help/canister-manager` kept as alias of `/help/dapp-manager`.

## Help topic pages — `/help/*`
All static (no actions beyond links) unless noted. HelpLayout (`HelpPage/Section/Tip/Warning/Info/Steps/Related`) is used by 11 pages (dao, forum, liquid-staking, monitor, names, notifications, proposals, sms, sneedapp, token-studio, tools); the other 9 have their own bespoke layout.
- `/help/app` (HelpApp): install on iPhone (Share → Add to Home Screen), Android (Install app), desktop; pairing flow for iPhone/other ("Pair it with Safari/a browser", Copy link, Approve, Copy approval, Paste approval); phone notifications. Documents `/pair`, LoginModal, `/notifications`.
- `/help/wallet` (HelpWallet): Why Your Principal Matters; Finding Your Way Around; Token Management; Token Operations; Liquidity Position Management; Neuron Management; Apps: Bots and Canisters; NFTs; Tips; FAQ; Related. Shows logged-in principal with copy button. Documents `/wallet`, `/nft`.
- `/help/names`: logging in/out, installed app, principal, public names vs nicknames, moderation, nickname privacy, setting a name, verified names, address book `/names`, principal picker, avatars/NFT avatars, account linking (public), profile pages `/me` `/user` `/users`, settings, safety.
- `/help/dao`: what Sneed DAO is; the hub front page; DAO facts and figures (`/dao_info`, `/health`, `/products`, `/partners`, `/projects`, `/dashboard`, placeholder routes); Revenue and the RLL (`/revenue`, `/rll_info`, `/rll_ledger`); voting rewards (`/rewards` steps + `/rll`); Sneed Premium (`/premium` tiers, three ways in); donations and thank-you wall; feedback; Mr Sneed; Lore; Disclaimer.
- `/help/proposals`: where proposals live; SNS proposals browsing/filtering; voting with neurons; nudges; proposal detail page; ICP (NNS) proposals; canister-managed ICP neurons; following; making a proposal (only via `/mr-sneed`); rejection cost; troubleshooting.
- `/help/neurons` (bespoke): what SNS neurons are; what you can do; hotkeys; managing neurons in Sneed Hub; cross-platform; best practices; FAQ. Shows logged-in principal with copy button.
- `/help/liquid-staking`: meaning; SNS neuron creation step by step (`/sns_neuron_wizard`), fee, partial failures; ICP Staking Bot; sending/selling positions; existing neurons; SNS directory (`/sns`); safety checklist.
- `/help/forum`: forum, feed, posting, voting on posts, polls, tips (ledger-verified, one-click tip again), @-mentions (SMS from poster), notification, moderation; points to `/feedback`.
- `/help/sms`: who can read messages; sending (free), limits & Premium, mention alerts use allowance, reading, replies/threads, conversations, removing, notifications.
- `/help/notifications`: consent model, topics, levels/destinations, defaults, quiet hours/pausing, channels (SMS, push, Discord; encrypted addresses), balance & paying, Recent history.
- `/help/monitor`: Sneed Monitor (proposal watcher set up on `/notifications`): turning on, background neuron loading, prices (**live tiers from `sneed_mon.get_tiers()`**), advanced settings, daily checks, stopping/withdrawing.
- `/help/sneedlock` (bespoke): what/why; lock tokens; lock LP positions; move a position to your Sneed wallet; dissolve delays; locks as NFTs; gas for token locks; liquid locking; finding/sharing locks; fees and limits (**live lock fees from `sneed_lock.get_lock_fees_icp()`**); unlocking; security; FAQ.
- `/help/sneedex` (bespoke): what; supported asset types; creating an offer; finding offers and bidding; lifecycle; private offers (OTC); fees; FAQ.
- `/help/sns_jailbreak` (bespoke): what; how it works; running the script; after; saved scripts (`/tools/sns_jailbreak_list`); supported SNS DAOs; FAQ.
- `/help/sneedapp`: browsing apps; minting step by step; payments final; control model; your apps; versions; upgrading bots; publishers; safety.
- `/help/dapp-manager` (= `/help/canister-manager`, bespoke): App Manager overview; health status lamps; custom apps; individual app view; topping up cycles; best practices; FAQ.
- `/help/icp-neuron-manager` (bespoke): ICP staking bot: creating; neuron operations; controllers/security; Botkeys; chores; cycles; upgrading; withdraw; events; canister logs; FAQ.
- `/help/trading_bot` (bespoke, 1,448 lines): getting started; chores (Trade, Rebalance, Move Funds, Distribute Funds, Snapshot); purses; wallet & token registry; performance tab; DEX settings; circuit breaker; inter-bot events; controllers & botkeys; logs; SneedScript (AI configuration); FAQ; checklist.
- `/help/token-studio`: Token Studio checks; old addresses (`/tools/token_safety` redirect); rating gates; thresholds; follow the money in a report; Harden fixes; coming soon (hand over, index, launch); reading safely.
- `/help/tools`: Tools page; Account Tool; ICRC-3 Browser; Follow the Money (limits); Swap; Scan Wallet; Transactions; NFT browser; Escrow.
- **Redesign notes**: Premium, rewards, donations, feedback, Mr Sneed, lore all live inside HelpDao; Swap help is inside HelpTools; two visual systems for help pages.

## API docs index — `/api`
- **Routes & params**: `/api`. Public.
- **Purpose**: developer entry: canister reference docs + machine-readable resources.
- **Information shown**: intro ("Integration is permissionless", link `/architecture`); 11 listed doc cards (title, description, "Read the API doc"): Sneed Lock canister `sneed_lock`, Sneedex marketplace `sneedex`, Forum canister `sneed_sns_forum`, Sneed SMS `sneed_sms`, Naming service `sneed_names`, Sneed Notifications `sneed_notifications`, Sneed Premium `sneed_premium`, Rewards distribution (RLL) `rll`, Sneedapp factory `sneedapp`, Mr Sneed canister `mr_sneed`, Sneed Hub backend `app_sneeddao_backend`. Machine-readable panel: "Raw Markdown and Candid" (links to each `.md`), "Agent skills" (`/.well-known/agent-skills/index.json`, `sneed-lock/SKILL.md`, `/llms.txt`, MCP server on GitHub), "Source code" (GitHub repo, `src/sneed_lock/main.mo`).
- **Hidden (reachable, not listed)**: `/api/sneed_proxy`, `/api/sneed_attestations`, `/api/sneed_nft` (not deployed yet), and naming sub-canisters `sneed_pid_names`, `sneed_neuron_names`, `sneed_pid_nicknames`, `sneed_neuron_nicknames`, `sneed_profiles`, `sneed_avatars` (stubs pointing into the names doc; targets of ICRC-3 `block_types_url`).
- **Entry points**: Help menu "API"; `/help` agent card; ApiDoc back link; `/rll` footer; Architecture node "API doc"; ToolsIcrc3 link.

## API doc reader — `/api/:slug`
- **Routes & params**: `/api/<slug>` (+ `#heading` anchors). Public.
- **Information shown**: rendered `/api-docs/<slug>.md` (react-markdown + GFM) with generated table of contents ("Contents", static on narrow screens), code blocks with "Copy".
- **Actions**: pills "Raw Markdown", "Candid" (`.did`), "JS IDL" (`.did.js`), "Agent skill" (only for sneed_lock, sneedex, sneed_sns_forum, sneed_sms, sneed_names, sneed_notifications); "All API docs" back link; in-doc `/api/…` links route internally; `#` links scroll.
- **Redesign notes**: an unknown-to-`apiDocs` slug shows "No API document named …" **and** still renders the markdown if the file exists — e.g. `/api/sneed_mon` (linked from the Architecture "Sneed Monitor" service) has a published `.md` but no `apiDocs` entry.

## Architecture map — `/architecture`
- **Routes & params**: `/architecture?canister=<nodeId>` or `?service=<serviceId>`. Public. Data in `src/data/architecture.js` (8 clusters, 50 nodes, 157 edges, 29 services, 5 edge types, 10 controller kinds).
- **Purpose**: interactive map of every Sneed canister, what it connects to, which services use it.
- **Information shown**: hero counts (Sneed canisters, controlled by the DAO, services, outside dependencies, connections); **At a glance** cluster cards (Who uses it, Governance, Money and trading, Apps and bots, Community, Identity, Infrastructure, Internet Computer and partners) with service chips; **Network** ReactFlow diagram; node detail panel: Mainnet/Staging ids (copy), Controlled by (+note), Persistence (EOP/Classic), ICRC-3 log yes/no, API doc link, **live** "Controllers" and "Module hash" (read_state via `health/agent.readControllers`; "None: blackholed", "No code installed"), connections in/out with labels, "Used by" services; service panel: its canisters and route; **All canisters** list by cluster with tags (DAO, EOP, ICRC-3).
- **Actions**: search "Find a canister, service or id"; edge-type toggles (call/payment/notify/relay on; control off); outside-canister toggle; zoom in/out/"Fit everything"/clear (Esc); pick node/service (URL updates); service routes link into the app (e.g. `/revenue`, `/health`, `/mr-sneed`, `/donate`); copy ids.
- **Entry points**: Help menu "Architecture"; `/help`; `/api` intro; llms.txt.
- **Redesign notes**: doubles as an app sitemap (each service has a `route`) — useful source for a redesign IA; some routes there are odd (Trading bots → `/help/trading_bot`, Sneed Monitor → `/api/sneed_mon`, Cycles → `/health`).

## Published material for agents and integrators (static files in `public/`)
- `/llms.txt` (52 lines): Start here (sneed-hub skill, llms-full.txt); Canister API docs (14 entries incl. Sneed Mon, Sneed proxy, Sneed attestation registry marked "not deployed yet"); human API pages; architecture map; Agent skills index + 7 skills; MCP server; Help pages; source code.
- `/llms-full.txt` (625 lines): sneed-hub skill + the six integration skills concatenated (Sneed Lock, Sneedex, forum, SMS, names, notifications).
- `/.well-known/agent-skills/`: `sneed-hub`, `sneed-lock`, `sneedex`, `sneed-forum`, `sneed-sms`, `sneed-names`, `sneed-notifications` (`SKILL.md` each); `index.json` is **generated at build** (`scripts/build-api-docs.mjs`, schema 0.2.0 with sha256 digests) — not in git.
- `/api-docs/`: `.md` for app_sneeddao_backend, mr_sneed, rll, sneedapp, sneedex, sneed_lock, sneed_sns_forum, sneed_sms, sneed_names, sneed_notifications, sneed_premium, sneed_mon, sneed_proxy, sneed_attestations, sneed_nft, sneed_pid_names, sneed_neuron_names, sneed_pid_nicknames, sneed_neuron_nicknames, sneed_profiles, sneed_avatars (21); `.did`/`.did.js` copied from declarations at build (only `sneed_names.did(.js)` hand-written in git).
- `/.well-known/ic-domains` (custom-domain config).
- **Redesign notes**: no in-app page for skills/MCP beyond the `/api` panel and `/help` card; SKILL.md "Reading public state" slug list omits sneed_mon.

## Game "SNEED RUN" — `/outrun`
- **Routes & params**: `/outrun`. Public. **No Header**, not linked anywhere in the app (URL-only easter egg).
- **Purpose**: canvas pseudo-3D arcade racer (OutRun clone) "SNEED RUN — A SNEED DAO PRODUCTION".
- **Information/gameplay**: tracks "Coconut Beach" and "Desert Dusk" (themes beach/forest/desert/night), stage timer with time extension per stage, traffic cars, roadside sprites, crashes, forks into the next track, speed (km/h), time, STAGE HUD; title "PRESS ENTER TO START", countdown, "GAME OVER / TIME'S UP! / PRESS ENTER TO RETRY". No score persistence, no on-chain interaction.
- **Actions**: "← Back" (→ `/`); track buttons; "Fullscreen"; keyboard arrows/WASD, Enter; Esc exits fullscreen or navigates home; touch zones ←, GAS, BRK, START, → (mobile).
- **Files**: `pages/OutRun.jsx` + `OutRun.css`, `game/outrun/{engine,render,road,sprites,tracks,input,constants,utils}.js`, assets `/outrun-*.png`.

---

## Slice observations

**(a) Entities and where they are viewed / acted on**
- **DAO (identity/about)** — view: `/dao`, `/lore`, HelpDao, Architecture. Actions: none.
- **DAO governance metrics** (neurons, dissolve states, VP, hotkeys, proposal count) — view: `/dao_info` (via RLL `get_neuron_statistics`), Hub tiles; `/health` SNS core. Actions: none here (proposals/neurons pages elsewhere).
- **SNEED token / tokenomics** (supply, circulating, staked, price, FDV, market cap, TVL, NAV/GAV ratios) — view: `/dao_info`, `/rll_info` Total assets, Hub. Actions: none.
- **DAO treasury** (ICP + SNEED in SNS governance accounts, DeFi canister balances, other tokens) — view: `/rll_info` Total assets (ICP/SNEED/Other Tokens), `/health` Revenue leaves, `/rll_ledger` "to treasury" flows; fetched-but-hidden on `/dao_info`. Actions: none.
- **2-year ICP neuron** — view: `/rll_info` hero/diagram/assets, `/rll_ledger` neuron section, `/revenue` source card, `/health` rev:neuron. Actions: none.
- **Vectors / pylons** (neuron vector, ICP splitter, buyback exchange, SNEED splitter; split %) — view: `/rll_info` hero + diagram + components, `/rll_ledger` configured vs actual shares, `/health` vector leaves & pylon control. Actions: external links only.
- **LP positions** (SNEED/ICP #24–26, ICP/CLOWN #168) — view: `/rll_info` assets, `/revenue` LP card, `/rll_ledger` LP fees, `/health` rev:lp. Actions: none.
- **Loop flow / transaction** — view: `/rll_ledger` rows (→ `/transaction`), `/revenue` Into the loop chart, `/rll_info` diagram cards.
- **Revenue stream** (neuron maturity, LP fees, product fees per product) — view: `/revenue` (all time), `/rll_info` product nodes, `/rll_ledger`. Actions: refresh.
- **Product** — view: `/products` (8 hard-coded, usage stats), `/revenue` (5, earnings), `/dao_info` (2), `/projects` "Products" group (admin list), Hub "Featured Products", Architecture services. Actions: links into each product.
- **Reward distribution / credit / claim** (RLL) — view: `/rll` (stats, distributions, your credits & claims), `/rewards` (Sneed Me), RllActivityCard on `/rll` `/rll_info` `/revenue`, `/health` Rewards, `/rll_info` member-rewards reconciliation. Actions: Claim (on `/rll` and `/rewards`).
- **Partner** — view: `/partners`, `/dao_info` tiles. Admin: `/admin/partners`. Partner *prices* only in admin SpecialPricesCard.
- **Project** (product/project/fork) — view: `/projects`. Admin: `/admin/projects`.
- **Donation** — view: `/donate` wall, totals. Actions: Donate dialog (header + `/donate`), "Add it" retry. Admin: `/admin/donations`.
- **Feedback item** (bug/idea thread, private security SMS) — view: `/feedback` recent reports, forum `/thread`. Actions: post/send. Admin: `/admin/feedback`.
- **Mr Sneed memory / personality / writers / limits** — view: `/mr-sneed`. Actions: forget-about-me, propose personality/writers/limits/removal (SNS proposal).
- **Diamond Club card (EXT/ICRC-7)** — view: `/diamond-club`, `/nft`, NftCard banners. Actions: swap from wallet, claim, proof-by-payment, check card, retry refund.
- **Health check** — view/run: `/health`; thresholds (per browser).
- **Canister (architecture node) / service** — view: `/architecture` (live controllers/module hash), `/health`.
- **Help topic** — view: `/help`, 20 topic routes (+1 alias).
- **API doc / Candid / skill** — view: `/api`, `/api/:slug`, raw `/api-docs/*`, `/.well-known/agent-skills/*`, `/llms*.txt`.
- **Game track** — `/outrun`.

**(b) Overlaps with other areas of the app**
- Rewards claim exists twice: `/rll` (DAO menu "Rewards") and `/rewards` (Sneed Me menu "Rewards") — same canister calls, different UIs; help text points to both.
- Loop/revenue data shown on four pages (`/rll_info`, `/rll_ledger`, `/revenue`, `/health`) plus `/dao_info` and Hub; they share a cache (`revenueStore`) only for the three newer pages — `/rll_info` asset accounting, `/dao_info` and Hub each re-query supply/treasury/staking on their own.
- Product lists in four+ places with different members (`/products`, `/revenue`, `/dao_info`, `/projects`, Hub featured, Architecture services).
- Custody appears as `/guardian` (DAO menu) and `/custody` (Tools menu); `/health` links `/guardian?deployment=`.
- `/diamond-club` is an NFT-wallet migration under DAO; `/nft` and NftCard link into it; `/convert` is a separate conversion area.
- Feedback writes forum threads and SMS (Forum/SMS area); donations send SMS too.
- Mr Sneed's ProposeDialog is the only in-app "make a proposal" flow (Governance area has none).
- `health/*` infrastructure (TreeView, runner, agent) is shared with Token Studio/Token Safety, Guardian, Follow the Money, NFT and Position pages; Architecture reuses `readControllers`.
- Premium, rewards, donation, feedback, Mr Sneed and lore help all live inside `/help/dao`.

**(c) Dead, hidden, half-built or confusing**
- Dead: `src/Banner.jsx` (never imported); `RLLInfo.jsx` unused imports (PrincipalBox, headerStyles, logout); `/dao_info` computes treasury value, reconciliation, event stats and DeFi balances but renders none of them; `Products.jsx` unused Neutrinite actor import.
- Placeholders: `/tokenomics`, `/tokenomics_info`, `/me_info` (ComingSoon, not redirects, hard-coded dark styling), `/doc` ("Documentation coming soon!"); `/guardian` and `/custody` ComingSoonPage for non-admins.
- Broken-ish: `/products` USD values always 0 (price fetch stubbed); `/api/sneed_mon` shows "No API document named…" above a rendered doc; `/dao_info` uses `fetchRootKey()` on mainnet and an off-chain API for proposal count.
- Transitional states tied to canister upgrades: RLL stats banner "being fixed (proposal #444)", history "can be read after the RLL upgrade of proposal #444", activity log "arrives with the next RLL upgrade"; `/revenue` hidden product totals ("admins only … once a DAO upgrade makes them public"); Mr Sneed proposal types possibly unregistered; Diamond Club "swap is not open yet"; donations/feedback "not open yet"; llms.txt marks Sneed Mon "not deployed yet" while `/help/monitor` and the Help index present it as live.
- Hidden: `/outrun` game (no link, no header); hidden `/api/<slug>` docs (sneed_proxy, sneed_attestations, sneed_nft, six naming sub-canisters); `/help/canister-manager` alias; `/dashboard` legacy lock list; Health thresholds only in localStorage.
- Confusing naming: `/rll_info` = "Tokenomics", `/rll` = "Rewards", `/rll_ledger` = "Loop ledger"/"Revenue loop ledger"; `/donate` = "Thank-you wall"/"Thank you"/"Support Sneed Hub"; DAO menu ordering puts Projects and Disclaimer after the external Discord/X links; help route naming mixes `_` and `-`; Help pages use two different layouts.
- Mixed concerns: `/rll_info` (treasury accounting + tokenomics ratios + loop diagram + component catalogue); `/mr-sneed` (memory browser + DAO proposal tool); `/feedback` and `/diamond-club` filed under "Sneed DAO" though they are support and NFT utilities.
- Root `Notification.jsx` toast used only by `/rll` and `/rewards` — inconsistent with the app's dialog host.
- Disclaimer is only reachable from the DAO menu/HelpDao — not linked from login or any money-moving flow.
