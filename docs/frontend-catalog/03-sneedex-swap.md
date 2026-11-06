# Trade: Sneedex marketplace, escrow and swap

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Source root: `src/app_sneeddao_frontend/src`. Main files: `pages/Sneedex.jsx`, `pages/SneedexOffers.jsx`, `pages/SneedexOffer.jsx` (6968 lines), `pages/SneedexCreate.jsx` (5882 lines), `pages/SneedexMy.jsx`, `pages/Swap.jsx`, `components/SwapWidget.jsx`, `components/SwapModal.jsx`, `components/OfferCard.jsx`, `components/ExtEscrowConfirm.jsx`, `components/NftAssetView.jsx`, `components/SneedLockAssetView.jsx`, `utils/SneedexUtils.js`, `utils/SneedexExt.js`, `services/dex/*`, `pages/ToolsEscrow.jsx`, `pages/EscrowSwap.jsx`.

Canister shorthand: **SX** = Sneedex canister (`declarations/sneedex`). Header section "Sneedex" menu: Swap (`/swap`), Marketplace (`/sneedex_offers`, section default), Create Offer, My Sneedex, About (`/sneedex`). Header quick-link icons (wide-screen row and narrow-screen hamburger) also have "Go to Sneedex Marketplace" and "Swap" — both drawn with the same exchange icon.

Shared data model used everywhere in this slice (`utils/SneedexUtils.js`):
- Offer states: Draft, PendingEscrow ("Pending Escrow"), Active, Completed, Expired, Cancelled, Claimed, Reclaimed. "Ran out" = Active but past expiration (the canister's timer hasn't processed it yet).
- Bid states: Pending, Won, Lost, Refunded, ClaimedBySeller ("Claimed").
- Asset variants: `Canister` (kind 0 = App/"Generic Canister", 1 = ICP Staking Bot, 2 = Trading Bot; optional title ≤100 chars, description ≤4000; controllers snapshot; cached total stake), `SNSNeuron` (governance id, neuron id hex, hotkeys snapshot, cached stake), `ICRC1Token` (ledger, amount), `ICRC7NFT` (collection, token ids), `EXTNFT` (collection, token indices), `SneedLock` (lock id + Token{ledger, amount, expiry} | Position{swap pool, position id, expiry}). Each asset entry carries an `escrowed` flag.
- Offer fields: id, creator, price token ledger, min bid (opt), buyout (opt), expiration (opt), approved bidders (opt → private/OTC), min bid increment as a multiple of the price token's fee, public note, note to buyer, fee rate bps (locked at creation), created/activated timestamps.
- Error variants mapped to text (NotAuthorized, BidTooLow{minimum}, InsufficientFunds, CannotCancelWithBids, OfferMustHaveBuyoutOrExpiration, …).
- Every monetary figure has a USD (or user-chosen denomination via `DenominationContext`) estimate from `PriceService` (ICPSwap pools).

---

## 1. Sneedex About / landing

- **Routes & params**: `/sneedex` — public; "Create Offer" and "My Offers & Bids" buttons only when logged in; admin card only for SX admins (`getConfig().admins`).
- **Purpose**: marketing/explainer page for Sneedex with live market stats.
- **Information shown**:
  - Live stats (`SX.getMarketStats`, `getMarketplaceFeeRate`): Active offers, Total offers, Completed, Bids (total), Fee (%).
  - Static text: "What is Sneedex?", "Supported Assets" (Apps, SNS Neurons, ICP Staking Bots, ICRC1 Tokens), "How It Works" (Create an Offer → Escrow Your Assets → Receive Bids → Atomic Settlement), "Key Features" (Trustless Escrow, Atomic Transfers, Asset Bundling, Flexible Pricing), "Ready to Trade?" CTA.
- **Actions**: "Browse Marketplace", "Create Offer" [login], "My Offers & Bids" [login], "Explore Marketplace" (CTA); admin: "Admin Settings — Manage fees, admins, and configuration" → `/admin/sneedex`.
- **Entry points & cross-links**: Header Sneedex › About; Help pages (`/help/sneedex`, HelpSnsJailbreak, HelpForum) link `/sneedex`; `utils/revenueData.js` revenue table links `/sneedex`. Breadcrumb Home › Sneedex.
- **Shared building blocks**: `createSneedexActor`, `formatFeeRate`.
- **Gating/state**: none beyond login/admin.
- **Redesign notes**: asset list is stale — omits Trading Bots, NFTs (ICRC-7 + EXT) and Sneed Locks that the create wizard supports. Only place that shows the global market stats; `/products` (other slice) also shows Sneedex stats.

## 2. Marketplace browse & filter

- **Routes & params**: `/sneedex_offers` — public. No URL params (tab, filters, sort, page are all local state; not shareable). "Private (OTC)" tab requires login.
- **Purpose**: browse/search all offers (public or private-to-me) with rich filters, USD valuation and "good deal" detection.
- **Information shown**:
  - Hero buttons: "Swap" (→ `/swap`), "Create Offer" [login], "Jailbreak Wizard" (→ `/tools/sns_jailbreak`).
  - Tabs: "Public Offers" / "Private (OTC)" (private = offers where viewer is creator or approved bidder; `getOfferFeed{public_only:false, viewer}`).
  - Asset pills (multi-select; none = all): All, Neurons (SNS neurons + ICP staking bots), Tokens, NFTs (ICRC-7 + EXT), Sneed Locks, Apps & bots (other canisters incl. trading bots).
  - Sort: Newest First, Ending Soon, Highest Bid, Lowest Price.
  - State toggles: Active, Sold (Completed/Claimed), Expired (Expired/Cancelled/Reclaimed/ran-out). Default Active+Sold.
  - "Filters" panel (badge shows count): Payment Token (bid currency); Price Range (min bid or buyout, in token); Asset Token (token ledger; SNS neurons match their SNS ledger; ICP staking bots match ICP); Asset Amount (token amount / neuron stake / bot ICP); Price in USD; Est. Value in USD; Value/Price Ratio (min %, "100% = value ≥ price"); Search by Offer ID; Search by Seller (`PrincipalInput`, name search). "Clear All".
  - Offer card (inline, 12 per page): `#id`, optional single-canister title, lock icon + approved-bidder count for private offers, state badge, corner banner SOLD/EXPIRED/CANCELLED; asset badges with tooltips per type (staking bot ICP total, trading bot, app, SNS neuron stake + SNS logo/symbol, token amount + logo, Sneed Lock title/compact NFT, EXT "Card #n"/"n cards", ICRC-7 NFT icon); "Min Bid"/"Min Next Bid" (highest + increment, capped at buyout) with USD; "Buyout"/"Buyout Only" with USD; "Est. Value" (tokens, SNS neuron stake, staking bot ICP, Sneed Lock NFT values) with "🔥 GOOD DEAL" badge when value > current effective price; "Min increment"; Seller (avatar + `PrincipalDisplay`); time remaining; highest bid with USD and bidder.
  - Pagination (top/bottom); "Load older offers" / "Look in older offers" (feed is paged 100 offers per canister page, newest first).
  - Empty states ("No Public Offers… Be the first to create one!", private explanation, "Connect your wallet to view private offers…").
- **Actions**:
  - Click card → `/sneedex_offer/:id`.
  - Filters/sort/tabs/toggles (client-side over loaded pages).
  - Hidden write: for up to 10 loaded offers that ran out, the page silently calls `SX.processExpiration(id)` (permissionless) and reloads — the viewer's identity performs canister updates just by browsing.
- **Canister calls**: `SX.getOfferFeed` (fallback `getPublicOffers` / `getPrivateOffersFor` if feed fails), `getOfferView` per offer (bids, highest bid), `getCachedNeuronManagerInfoQuery`/`getNeuronManagerInfo` (staking bot totals), SNS governance `get_neuron` (stake), SNS ledger metadata, Sneed Lock NFT values (`useSneedLockNfts`), ICPSwap prices.
- **Entry points & cross-links**: Header Sneedex › Marketplace (section default), header quick-link icon, Hub "Active marketplace offers" card "Browse All", Products page, Liquid Staking page, Swap page breadcrumb, offer page "back" link, My Sneedex "Browse"/empty bids state, many Help pages.
- **Shared building blocks**: `TokenSelector`, `PrincipalInput`, `PrincipalDisplay` (context menu: Send Message, Set/Edit Nickname, View Profile, Copy Principal ID), `BigAvatar`, `NftAssetIcon`/`ExtBadge`, `SneedLockAssetIcon/Title/NftsCompact`, `useTokenMetadata`, `WhitelistTokensContext`, `DenominationContext`, `PriceService`.
- **Gating/state**: login for private tab; responsive grid (`minmax(380px)`), no other mobile logic.
- **Redesign notes**:
  - Offer cards are rendered inline here, duplicating `components/OfferCard.jsx` (used only by Hub).
  - "Sneed Locks" pill only matches legacy `SneedLock` assets; locks listed today are ICRC-7 NFTs of the Sneed Lock collection and fall under "NFTs".
  - Filters are client-side over already-loaded pages — older offers are invisible to filters until "Load older offers".
  - Jailbreak Wizard button is an unrelated tool placed on the marketplace (relevant only to SNS neuron sellers).
  - Filter count badge omits USD/ratio filters.

## 3. Offer page — overview and asset inspection (anyone)

- **Routes & params**: `/sneedex_offer/:id` — public read (anonymous actor); actions need login. No other params.
- **Purpose**: single offer: what is being sold, verifiable escrow state, pricing, bids, notes.
- **Information shown**:
  - Header: "Offer #id", state badge, corner ribbon (SOLD/EXPIRED/CANCELLED…), Seller (avatar + name), refresh button; live countdown (seconds when < 1 h).
  - Assets card: total estimated USD ("Estimating…"/"Updating with live data…"); aggregate escrow verdict: "n of m assets escrowed" / "All n assets fully escrowed" / "k of n assets have escrow issues" / "All assets delivered to buyer" / "All assets returned to seller".
  - Per asset (expand "Info"/"Less"):
    - App canister (`SX.getCanisterInfo` when escrowed): description (show more), Cycles, Memory, Daily Burn (idle), Controllers list (marks SNEEDEX, "Fully escrowed" if Sneedex is sole controller), Module Hash, refresh; link "View Details →" `/canister?id=`.
    - ICP Staking Bot: Version, "Official version verified", neuron count, WASM Status (Official vX.Y.Z / "Unknown WASM", via sneedapp factory app-version hashes + module hash from SX or backend `get_canister_info`), "Managed ICP Neurons" with grand totals (Staked, Maturity, Staked Maturity) and per neuron: id, state, total ICP, hotkeys ("No hotkeys"), Stake, Maturity, Staked Maturity, Dissolve Delay, Age, Voting Power, dissolving time; Botkeys section (bot `listHotkeyPrincipals` + permissions; "No third-party principals have access…"/"Botkey info not available (older bot version)"). Escrow considered verified only if Sneedex is sole controller AND no botkeys AND no neuron hotkeys.
    - Trading Bot: WASM Status (via `SX.verifyTradingBot` + hash).
    - SNS neuron (live `get_neuron` + nervous-system params): SNS logo/symbol, neuron id (`NeuronDisplay`), Staked Amount, Voting Power (`VotingPowerCalculator`), Total Maturity, Total Value, Neuron Age, Dissolve Delay / Time Until Dissolved, Maturity Breakdown (Available/Staked/Disbursing), Created, Auto-stake Maturity, Accumulated Fees, Source NNS Neuron, Hotkeys/Permissions list (marks SNEEDEX); links "SNS Hub →" (`/hub?sns=`) and "Dashboard ↗" (ICP dashboard).
    - ICRC1 token: Name, Symbol, Decimals, Transfer Fee, Escrowed Amount, Escrow Account (Sneedex principal + offer escrow subaccount), Ledger Canister (dashboard link).
    - ICRC-7 NFT / EXT NFT (`NftAssetDetails`): images/3D model, names, attributes, "yours" marker, ICRC-37 approvals note, EXT note, "Browse collection".
    - Sneed Lock (`SneedLockAssetDetails`): what the lock holds, value, expiry, "See the lock".
  - Public Note (everyone); Note to Buyer (only creator and winning bidder — "This note is only visible to you and the winning bidder").
  - Bid History (count; rank 👑/#n, bidder, date, state badge, amount + USD; zero-amount reserved bids hidden).
  - Pricing card: SOLD overlay, Buyout Price, Min Next Bid / Minimum Bid (USD), Est. Asset Value (+ good-deal marker), Min Bid Increment, Current Highest Bid / Winning Bid (USD), Time Remaining, Status, "Private Offer — n approved bidders"; creator-only "Approved Bidders" list.
  - Offer Details: Created, Activated, Price Token, Ledger, Marketplace Fee (% "deducted from winning bid").
- **Actions (anyone)**: refresh; expand/collapse assets; per-asset refresh; principal context menus (message/nickname/profile/copy).
- **Entry points & cross-links**: marketplace cards, My Sneedex rows, Hub `OfferCard`, Feed auction items (`/sneedex_offer/<id>` from Feed/FeedItemCard), SwapWidget ("View offer", "View auction #id", ads), create wizard success/overlay "View Offer".
- **Shared building blocks**: `NeuronDisplay`, `NftAssetView`, `SneedLockAssetView`, `PrincipalDisplay`, `BigAvatar`, `InfoModal`, `ConfirmationModal`, `SwapModal`, `VotingPowerCalculator`, `WalletContext`/`NeuronsContext`.
- **Gating/state**: mobile (≤900 px) reorders cards: Assets → Pricing/actions → Bid history → Details; tighter padding on phones.
- **Redesign notes**: offer page USD estimate covers tokens, SNS neurons and staking bots but not Sneed Locks/NFTs (marketplace does include lock values) — numbers can differ between list and detail. Asset verification panels are a substantial "due diligence" capability buried behind per-asset "Info" toggles. The only way to contact a seller is the principal context menu.

## 4. Offer page — buyer: bid, buyout, pending-bid escrow, claim

- **Routes & params**: `/sneedex_offer/:id` [login]. Bid UI shows when Active, not past expiry, no pending bid. (No front-end check for approved-bidder membership or for being the creator — SX rejects.)
- **Purpose**: place auction bids or buy out; finish an interrupted payment; claim won assets.
- **Information shown**: "Min: …", wallet balance of price token (refresh), insufficient-balance hint ("You need at least X more (≈ $)"), USD conversion of entered bid, warning when bid ≥ buyout ("You will pay exactly <buyout> and win immediately"), progress text during auto-pay ("Please wait, do not close this page"), pending-bid panel (Bid ID, Bid Amount, Escrow Balance polled every 5 s; expired warning "This offer has expired! … withdraw any funds"), manual payment details (canister id + subaccount hex).
- **Actions**:
  - "Buy <SYMBOL>" → opens `SwapModal` ICP → price token, pre-filled to the deficit (or price token is ICP → open input).
  - Bid input: token/USD mode toggle, "Min" fill, "+1% +5% +10% +25% +50% +100%" increments.
  - "Place Bid": `SX.reserveBid` → `getBidEscrowSubaccount` → ICRC-1 transfer (amount + ledger fee) from wallet to SX bid escrow → `SX.confirmBid(bidId, amount)`. If a later step fails, a pending bid is kept.
  - "⚡ Instant Buyout" (inside bid area) / "⚡ Instant Buyout for X SYMBOL" (buyout-only offers): same 3-step flow at buyout price; on success "Buyout successful! You now own the assets."
  - Pending bid: "Pay X SYMBOL from Wallet" (direct transfer) → "✓ Confirm Bid" (`confirmBid`); "💸 Withdraw Funds" (`appPrompt` amount; `SX.withdrawBidEscrow`; one ledger fee); "Cancel" (clears local state only); "Or pay manually via CLI/wallet…".
  - "🎁 Claim Your Assets" (offer Completed, viewer is highest bidder): `SX.claimAssets`, then refreshes neurons cache.
- **Fees**: price-token ledger fee per transfer; marketplace fee is taken from the seller's proceeds, not added to the bid.
- **Entry points**: offer page only (claim/refund also in My Sneedex §9; buyouts also in SwapWidget §11).
- **Gating/state**: login; disabled after expiry.
- **Redesign notes**: pending-bid state lives only in page memory — leaving the page mid-flow leaves funds in a bid escrow recoverable via My Sneedex (only bids with `tokens_escrowed`) or the wallet's Sneedex pockets. Manual payment block labels the Sneedex canister as "App canister id:". Buyout logic is implemented twice (here and SwapWidget).

## 5. Offer page — seller lifecycle (draft → escrow → active → settle)

- **Routes & params**: `/sneedex_offer/:id` [login, creator only].
- **Purpose**: continue an offer the wizard didn't finish, manage a live offer, collect payment.
- **Information shown**: "📋 How to Escrow Assets" box (PendingEscrow; Apps: add `<SX id>` as controller; SNS neurons: add Sneedex hotkey; ICRC1: transfer to escrow subaccount); per-asset pre-check (Draft/PendingEscrow): app controller check (management `canister_status`), SNS ManagePrincipals check, token balance ≥ amount + fee, EXT note; "Ready"/"Cannot auto-escrow"; pending count.
- **Actions**:
  - Draft: "📋 Finalize Assets" (`SX.finalizeAssets`) ("Step 1: … lock in your asset list"); "Cancel Offer".
  - PendingEscrow: per asset "🔒 Verify & Escrow": App → management `update_settings` adds SX as controller → `SX.escrowCanister`; SNS neuron → `SX.escrowSNSNeuron` (user must add hotkey first — page does not add it); ICRC1 → transfer to offer escrow subaccount → `SX.escrowICRC1Tokens`; EXT → `ExtEscrowConfirm` → card transfers → `SX.escrowEXTNFTs`. Then "🚀 Activate Offer" (`SX.activateOffer`) when all escrowed; "Cancel Offer".
  - Active: "Accept Highest Bid (X SYMBOL)" (`SX.acceptBid`, when bids exist); "Cancel Offer" (ConfirmationModal: "…n bids that will be refunded"; `SX.cancelOffer`).
  - Completed: "💰 Claim Payment" (`SX.claimWinningBid`).
- **Entry points**: wizard success/progress overlay "View Offer"; My Sneedex rows.
- **Redesign notes**: ICRC-7 NFT and Sneed Lock assets cannot be escrowed from this page (no handler; shows "Cannot auto-escrow") — if the wizard's auto-escrow fails for them, recovery is unclear. The SNS-neuron path here needs the user to add the hotkey manually (wizard does it automatically; Jailbreak Wizard is suggested elsewhere). Cancel-with-bids copy says bids are refunded while `CannotCancelWithBids` error text exists — confusing.

## 6. Expiration processing / finalize auction (anyone)

- **Routes & params**: `/sneedex_offer/:id`, `/sneedex_my` [login]; implicit on `/sneedex_offers` (any viewer).
- **Purpose**: push an expired Active offer to its outcome.
- **Information shown**: "This offer has expired!" + "The auction can now be finalized. The highest bidder (X) will win." or no-bids text.
- **Actions**: "⚡ Finalize Auction" (has bids) / "⚡ Process Expiration" (no bids) → `SX.processExpiration`; results "Auction completed! The highest bidder has won." / "Offer expired with no bids. Assets are being returned to the seller." Marketplace auto-calls it silently (§2).
- **Redesign notes**: same action in three places, one of them invisible.

## 7. Create Offer wizard — step 1 "Configure Pricing"

- **Routes & params**: `/sneedex_create` [login; anonymous sees "Connect Your Wallet"]. Prefill params: `?nft=<collection>:<id>[,<id>]` (+ `&standard=EXT`), `?lock=<lockId>` (opens NFT tab with the Sneed Lock collection and that lock id). Premium/partner/admin pricing affects fees.
- **Purpose**: set price token, auction/buyout terms, expiry, privacy, notes; preview fee.
- **Information shown**: progress bar (Configure Pricing / Add Assets / Review & Create); selected price token + USD price ("Price not available"), custom token info; suggested min increment (targets SX `getMinIncrementSettings` USD range; warning when outside range); "Sneedex Marketplace Fee: X%" (+ `PriceTierBadge` admin/partner, "👑 PREMIUM", "(Regular: Y%)", non-premium upsell "👑 Premium members pay only Z% →" `/premium`) "…This rate is locked when the offer is created."
- **Actions / inputs**: "Price Token" (`TokenSelector`, custom ledger allowed; default ICP); "Minimum Bid Price" (optional; blank = buyout-only); "Buyout Price" (optional); "Set an expiration date" + "Expires in" days/hours/min (default 7 d); "Min Bid Increment"; "Private Offer (OTC)" → "Approved Bidders" (`PrincipalInput` with name search, add/remove); "Public Note" (everyone); "Note to Buyer" (winner only). "Next".
- **Validation**: min bid or buyout required; min bid > 0; no expiration ⇒ buyout required; expiry ≥ 1 min; min bid ≤ buyout; valid ledger.
- **Shared building blocks**: `TokenSelector`, `PrincipalInput`, `PriceTierBadge` (also in lock/jailbreak/bot-mint flows), `usePremiumStatus`, `utils/specialPrices` (`fetchSpecialPricesFor`, `effectivePrice`).

## 8. Create Offer wizard — step 2 "Add Assets" (per type)

- **Purpose**: assemble a bundle of assets, each verified as sellable by the user.
- **Asset type tabs**: "App", "ICP Staking Bot", "Trading Bot", "SNS Neuron", "NFT", "ICRC1 Token". (A "Sneed Lock" tab exists in code but is commented out; locks are sold as Sneed Lock NFTs via the NFT tab.)
- **Per type**:
  - App: pick from "📦 Registered Apps" (backend canister groups) / "💼 Wallet Apps" (backend `get_tracked_canisters`) or manual canister id; controller check ("You are a controller - can escrow automatically" / "You are not a controller… Only app canisters you control can be added"); optional "Title" and "Description"; tip linking `/apps`.
  - ICP Staking Bot / Trading Bot: pick from user's bots (wallet context bot entries classified by WASM hash) or manual id; "Verify as Staking Bot" / "Verify as Trading Bot" (`SX.verifyICPNeuronManager` / `verifyTradingBot` + module hash vs sneedapp official versions: "Official WASM (vX.Y.Z)" / "Unknown WASM hash"); staking bot shows its neurons and totals.
  - SNS Neuron: "Select SNS" → "Select Your Neuron" (neurons where user is hotkey) or manual hex id; permission check (needs ManagePrincipals; "Use the Jailbreak Wizard to add Sneedex as a hotkey" → `/tools/sns_jailbreak`); Staked + Est. Value.
  - NFT: "Your collections" chips (Sneed Lock first, saved NFT-wallet collections, saved EXT collections marked EXT), or typed collection id/name ("ICRC-7 or EXT canister"); "Pick the tokens to sell" grid or ids text; ownership check (ICRC-7 owner / EXT bearer).
  - ICRC1 Token: "Select Token", "Your balance", "Use Max" (balance − fee), "Amount", USD estimate; balance check (amount + fee).
  - (Hidden legacy "Your Sneed Locks" list with link "Create one in Sneed Lock" → `/sneedlock`.)
- **Actions**: add / "Edit asset" / "Remove asset" / "Cancel"; per-asset "Ready"/"Not ready" with message; "Total Estimated Value"; duplicate prevention.
- **Canister calls**: management `canister_status`, SX verify*, backend `get_canister_info`, sneedapp `getApps/getAppVersions`, SNS governance `list_neurons/get_neuron`, ICRC-7 `icrc7_owner_of`, EXT `bearer`, ledgers `icrc1_balance_of/fee`, Sneed Lock reads.
- **Redesign notes**: the wizard is the only place that lists "my sellable things" across apps, bots, neurons, NFTs and tokens — effectively an asset inventory duplicated from wallet/apps/neurons pages.

## 9. Create Offer wizard — step 3 "Review & Create", fees, auto-escrow, step 4

- **Information shown**: review of Price Token, Minimum Bid, Buyout Price, Expiration, Offer Visibility (🔒 Private with approved-bidder list / 🌐 Public), Min Bid Increment, "Sneedex Cut" (+ tier badges); per-asset expanded details (app id/type/title/description; SNS neuron id/governance/staked; token ledger/amount/decimals; Jailbreak link for failed neuron checks; edit); Total Estimated Value; verification summary ("✅ All assets ready!… Create and finalize… Escrow all assets… Activate" or "🚫 Cannot create offer yet." with per-type requirements); Offer Creation Fee box (Fee in ICP, strikethrough regular price for premium, partner/admin badge, `/premium` link, wallet ICP balance, "⚠️ Insufficient funds", "Pending deposit: X ICP" from earlier failed attempts).
- **Actions**:
  - "← Back".
  - "🚀 Create Offer" (disabled unless all assets ready — **admins bypass** — and ICP covers fee + 0.0001): progress overlay steps: "Paying X ICP fee..." (ICRC-1 transfer to SX creation-payment subaccount, skipped if already deposited) → "Creating offer..." (`SX.createOffer`) → "Adding assets..." (`addAsset` each) → "Finalizing offer..." (`finalizeAssets`) → "Escrowing assets..." → "Activating offer..." (`activateOffer`).
  - Auto-escrow per type: App (add SX controller → `escrowCanister`); SNS neuron (governance `manage_neuron` AddNeuronPermissions all perms 1–10 for SX → `escrowSNSNeuron`, then refresh hotkey neuron caches); ICRC1 (transfer → `escrowICRC1Tokens`); EXT (confirm → transfers → `escrowEXTNFTs`); ICRC-7 (ICRC-37 approve → `escrowICRC7NFTsByApproval`, else "canary" self-subaccount transfer check then transfer to escrow subaccount → verify); legacy SneedLock (fund fees in Sneed Lock wallet for seller and SX, `transfer_token_lock_ownership`/`transfer_position_ownership` → `escrowSneedLock`).
  - "↩️ Withdraw" pending creation deposit (`SX.withdrawOfferCreationPayment`).
  - Progress overlay "Close" / "View Offer"; step 4 success "View Offer →", "My Offers".
- **Fees**: ICP creation fee (regular vs premium vs admin/partner special price; 0 when regular fee is 0) + ICP ledger fee; auction cut locked at creation (regular/premium/special).
- **Redesign notes**: step 4 "Next Steps" text (manual escrow instructions) is legacy when auto-escrow succeeded; only relevant to admin-bypassed or failed offers. Creation-fee deposit recovery exists here, on the wallet Pockets page, but not on My Sneedex.

## 10. My Sneedex (my offers and bids)

- **Routes & params**: `/sneedex_my` [login; anonymous sees "Connect Your Wallet"]. No URL params (tab not linkable).
- **Purpose**: dashboard of offers I created and bids I placed, with inline settle actions.
- **Information shown**:
  - Tabs: "My Offers (n)" / "My Bids (n)"; refresh; "Create Offer"; "Browse".
  - Offers filter: All, Active, Completed, Draft (incl. PendingEscrow), Cancelled, Expired, Reclaimed, Claimed (with counts); paginated ("Page x of y (n offers)").
  - Offer row: banner, asset-type icons (incl. ICRC-7 NFT, EXT NFT, Sneed Lock), Highest Bid (+USD), Total Bids, Time Left, Created.
  - Bids filter: All, Pending, Won, Lost, Refunded; bid row: Your Bid (+USD), Escrow Balance (ledger read for bids with tokens escrowed), Placed, excess warning "You have X excess SYMBOL in escrow".
- **Actions**:
  - Offers: "Claim Payment" (Completed; `claimWinningBid`), "Accept Highest Bid" (Active with bids; `acceptBid`), "Cancel Offer" (Active/Draft/PendingEscrow not expired; confirm; `cancelOffer`), "Finalize Auction"/"Process Expiration" (Active past expiry; `processExpiration`).
  - Bids: "Withdraw" excess (`withdrawBidEscrow`, max = escrow − (bid + fee) − fee; `appPrompt`), "Claim Assets" (Won; `claimAssets`), "💸 Refund Bid" (Lost with tokens escrowed; `refundBid` — only here).
  - Row click → offer page.
- **Canister calls**: `getOffersByCreator`, `getBidsByBidder`, `getOfferView` per offer, ledger balances of bid escrow subaccounts.
- **Entry points**: Header › My Sneedex; About page; wizard success "My Offers".
- **Redesign notes**: duplicates most offer-page seller/buyer settle actions; "Refund Bid" exists only here; no escrow/activate for PendingEscrow offers (must open offer page). Wallet Pockets uses `getBidsByBidderAll` while this page uses `getBidsByBidder` — the two views of "my bids" may differ.

## 11. Token swap (`/swap`) and SwapWidget

- **Routes & params**: `/swap?input=<ledgerId>&output=<ledgerId>` — page public, but the aggregator only initialises with an identity: anonymous visitors get no token info, quotes, spot prices or auction matches; the button reads "Connect Wallet" (disabled, does not open login). Defaults ICP → SNEED; token changes rewrite URL params.
- **Purpose**: best-rate token swap ("Sneedex Swap — Compare quotes across multiple DEXes…").
- **Information shown**: "You pay" / "You receive" boxes with balances (privacy-masked when privacy mode on), expected output ("Fetching quotes..."), spot price per DEX ("1 A = x B"), quotes list (cards with BEST / SPLIT / AUCTION / SPLIT TRADE / Multi-hop badges, expected output + USD, fee %, price impact, "Min:" output, advantage "+x (y%) more than next best"), split slider (dormant), "Sneedex Auctions (n)" ads (collapsible, show 3/all, each with output, buyout price, rate, USD, link), selected-quote details (Rate, Price impact, Slippage tolerance, Network fees, Standard ICRC1/ICRC2; explanatory notes for auction/split/split-trade), progress panels, result ("Received: …", per-leg failures, "Split swap partially failed", "View auction #id"). Quotes auto-refresh every 15 s.
- **Actions**: token selectors (custom tokens allowed), amount, "MAX", flip, slippage settings (0.5/1/2/5 % or custom; saved in localStorage and to backend `setMySettings({swap_slippage_tolerance})`, broadcast to other widgets), pick a quote card, swap button (labels: "Select Tokens", "Select Different Tokens", "Enter Amount", "Loading...", "No Quotes", "Swap", "Split Swap", "Buy from Auction", "Split Trade", "Swapping...").
  - DEX swap (ICPSwap pool): ICRC-2 → check allowance / `icrc2_approve` → `depositFromAndSwap`; ICRC-1 → transfer to pool subaccount → `depositAndSwap`.
  - "Buy from Auction" (§12) / "Split Trade" (buyouts + ICPSwap remainder in parallel).
  - After swap: refresh balances, `onSwapComplete`, auto-register output token in wallet (`ensureTokenRegistered`).
- **Canister calls**: ICPSwap factory/pools (`services/dex/dexes/ICPSwapDex.js`), ledgers, SX `getOfferFeed` (Active, public, first 50) for auctions.
- **Entry points**: Header › Sneedex › Swap (first item); header quick-link "Swap"; marketplace hero "Swap"; Products page; `/help/tools`. Breadcrumb Home › Sneedex › Swap.
- **Gating/state**: login effectively required; mobile CSS only.
- **Redesign notes**: KongSwap has shut down — only ICPSwap is registered, so "compare across DEXes", split swap, split slider and the 50/50 auto-search are dormant code; multi-hop is not implemented ("future"). The page is in practice "ICPSwap direct pair + Sneedex buyouts". Help/skill text still mentions KongSwap. Auction matching only scans the newest 50 active public offers.

## 12. Sneedex buyouts inside the swap

- **Routes & params**: within SwapWidget (`/swap`, any `SwapModal`) [login].
- **Purpose**: when an active public offer sells only the output token for the input token with a buyout price, treat it as a quote.
- **Information shown**: AUCTION quote card ("Buyout: X", "Rate: 1 A = r B", "Exact amount (no slippage)", "View offer"); SPLIT TRADE card (stack of buyouts + swap remainder); ads section for all matching auctions even if worse than the swap.
- **Actions**: "Buy from Auction" → `SX.reserveBid` → `getBidEscrowSubaccount` → ICRC-1 transfer → `confirmBid` at buyout price; "Split Trade" runs buyouts and the swap leg in parallel (partial fills possible).
- **Qualifying rule**: buyout + one input fee ≤ input amount and rate better than best swap rate; expired offers skipped client-side.
- **Redesign notes**: hidden marketplace entry point — users can buy Sneedex token offers without seeing the offer page (notes, seller). Duplicates offer-page buyout code.

## 13. Buy/Sell via SwapModal (entry points across the app)

- **Component**: `components/SwapModal.jsx` = SwapWidget in a portal overlay (props: initialInput, initialOutput, initialOutputAmount, onSwapComplete).
- **Where it opens from**:
  - `TokenCard.jsx` (used in `/wallet`, quick wallet `PrincipalBox`, `TokenCardModal`, Lock/TokenLock pages, TokenLocksOverview): "Buy" (ICP → token; for ICP itself: open with ICP as output), "Sell" (token → ICP; only if available balance > 0), plus a compact exchange icon on the collapsed card (title "Buy <SYMBOL>", or "Swap" for ICP).
  - `SneedexOffer.jsx`: "Buy <SYMBOL>" next to balance in the bid/buyout area (pre-fills deficit as target output; widget back-solves input amount from USD/spot prices and refines).
- **Redesign notes**: three swap surfaces (page, wallet modal, offer modal) share one widget — good; but `/swap` page vs modal differ only in URL syncing.

## 14. EXT NFT escrow confirmation (shared modal)

- **Component**: `useExtEscrowConfirm()` (`components/ExtEscrowConfirm.jsx`) + `escrowExtAsset()` (`utils/SneedexExt.js`).
- **Used by**: create wizard auto-escrow and offer page "Verify & Escrow" for EXT assets.
- **Information shown**: "Send this card / n cards to escrow?", card numbers, collection name/id, offer #, escrow account id (read from SX `getOfferEscrowAccountId`, cross-checked locally against `getOfferEscrowSubaccount` — aborts if mismatch), notes (public EXT transfer; only Sneedex can move it out; "Already in escrow (not sent again)").
- **Actions**: "Cancel" / "Send to escrow" → EXT `transfer` per card (arrival verified via `bearer`) → `SX.escrowEXTNFTs`. Safe to retry; errors if a card is held elsewhere.

## 15. Escrow Service placeholder

- **Routes & params**: `/tools/escrow` — `ComingSoon title="Escrow Service"`; public.
- **Entry points**: only `/help/tools` text. Not in Tools menu or `/tools/main`.
- **Redesign notes**: name collides with Sneedex (which already is an escrow service). Dead placeholder.

## 16. OpenChat escrow swap lookup

- **Routes & params**: `/tools/escrow/swap?swap_id=<n>&accepting_principal=<principal>` — public (uses identity if present); auto-looks-up when `swap_id` is numeric; params mirror the inputs.
- **Purpose**: read-only lookup of an OpenChat escrow swap (`external/oc_escrow` `lookup_swap`).
- **Information shown**: "Escrow Swap Lookup"; status badge (Open/Accepted/Completed/Cancelled/Expired); token exchange (amount0/token0 + deposit address, amount1/token1 + deposit address); Created By, Offered By, Created At, Expires At, Visibility (Public/Private), Restricted To, Location (OpenChat Message / External); Accepted By/At, Completed By/At, Cancelled At.
- **Actions**: "Swap ID *", "Accepting Principal (Optional)", "Lookup Swap".
- **Entry points**: only `/help/tools` text. Not in any menu.
- **Redesign notes**: unrelated to Sneedex despite the "escrow"/"swap" naming; URL-only tool; could live with other inspector tools (ICRC-3 browser, account tool).

## 17. Sneedex surfaces and entry points elsewhere in the app

- "Sell on Sneedex" links (all → `/sneedex_create` with prefill):
  - `TokenCard.jsx` Locks tab, per token lock → `?lock=<lockId>` ("Create a Sneedex offer with this lock as the asset").
  - `PositionCard.jsx`, locked LP position → `?lock=<lockId>`.
  - `NftCard.jsx` (NFT wallet `/nft` grid) tag icon → `?nft=<collection>:<id>` (+`&standard=EXT`).
  - `pages/NftItem.jsx` (`/nft_item`) "Sell on Sneedex" → `?nft=…`.
  - No "Sell on Sneedex" exists for plain ICRC-1 tokens, SNS neurons, staking bots, trading bots or apps — those must be picked inside the wizard (skill doc's "on tokens" refers to token locks).
- "Create Offer"/"Browse" links: `pages/LiquidStaking.jsx` (Sneedex buttons + help), Hub "Active marketplace offers" card (5 newest active offers as `OfferCard variant="hub"`, "Browse All", "Create offer"), `pages/Products.jsx` (Sneedex + Sneedex Swap product cards with stats), many Help pages.
- Feed (`pages/Feed.jsx`, `FeedItemCard`): Sneedex auctions mixed into the forum feed (`getOfferFeed` + `getOfferView`), linking to `/sneedex_offer/:id`.
- Wallet Pockets (`/wallet/pockets`, `utils/pockets/adapters/sneedex.js`): "Sneedex offer payment" pocket (creation-fee deposit) and "Sneedex bid <id>" pockets, hidden unless withdrawable or "Show Sneedex escrows (n)" ticked; withdraw (`withdrawOfferCreationPayment` / `withdrawBidEscrow`) and deposit from wallet.
- Header neuron refresh tooltip mentions "Sneedex claim" (neurons arrive after claiming).
- Admin: `/admin/sneedex` (other slice) — fees, premium cut, creation fee, min-increment settings, fee recipients, expiration timer, asset types, misplaced-NFT return, payment logs.

## 18. SwapPoolPicker (name only — not a swap feature)

- `components/SwapPoolPicker.jsx`: picks/validates an ICPSwap liquidity pool (by two tokens via factory or by pool id); `describeSwapPool()` used by Sneed Lock asset views and the create wizard. Used by LockWizard, Position, Positions, PositionTransferPanel, AddSwapCanisterModal. Belongs with LP/Sneed Lock, not token swapping.

## Slice observations

**(a) Entities, views and actions**

- **Offer** — views: marketplace card (inline), Hub `OfferCard`, Feed item, My Sneedex row, offer page, swap AUCTION card/ads. Actions: create (wizard), finalize, escrow per asset, activate, accept highest bid, cancel, claim payment, process expiration/finalize auction (anyone), filter/sort/search.
- **Bid** — views: offer page bid history, pending-bid panel, My Sneedex bids tab, wallet Sneedex pockets. Actions: place (reserve → pay → confirm), buyout, confirm pending, pay from wallet, withdraw from escrow (pending or excess), refund (lost; My only), claim assets (won).
- **Escrowed asset, by type**:
  - App canister — view: controllers, cycles, memory, burn, module hash. Escrow: add Sneedex controller.
  - ICP Staking Bot — view: version, WASM verification, neurons (stake/maturity/VP/dissolve/age), neuron hotkeys, botkeys. Escrow: controller.
  - Trading Bot — view: WASM verification. Escrow: controller.
  - SNS neuron — view: full neuron stats, permissions. Escrow: Sneedex hotkey with all permissions (auto in wizard, manual on offer page).
  - ICRC-1 token — view: metadata, amount, escrow account. Escrow: transfer to offer subaccount.
  - ICRC-7 NFT (incl. Sneed Lock NFTs) — view: `NftAssetDetails` / Sneed Lock value. Escrow: ICRC-37 approval or transfer (wizard only).
  - EXT NFT — view: `NftAssetDetails` (EXT). Escrow: transfer to escrow account id with confirm modal.
  - Legacy `SneedLock` asset — view only (`SneedLockAssetDetails`). Creation path commented out.
- **Creation-fee deposit** (ICP subaccount on Sneedex) — view: wizard review "Pending deposit", wallet pocket. Action: withdraw (wizard, pockets).
- **Market stats / fee config** — view: About page, wizard fee box, offer details; admin edit elsewhere.
- **Swap pair / quote / route** — view: quotes list, spot prices, selected-quote details; action: swap, split swap (dormant), buy from auction, split trade; slippage setting (persisted to backend user settings).
- **ICPSwap pool** — used for swap quoting/execution and as the USD price source everywhere; picker component lives in LP flows.
- **OpenChat escrow swap** — lookup view only.

**(b) Overlaps with other areas**

- Settle actions (accept, cancel, claim payment, claim assets, process expiration, withdraw) exist on both the offer page and My Sneedex; the wallet Pockets page duplicates bid-escrow/creation-fee withdrawals.
- Buyout flow implemented twice (offer page and SwapWidget).
- Offer cards rendered three ways (marketplace inline, `OfferCard` on Hub, Feed card).
- Wizard rebuilds inventories already shown in Wallet (tokens, NFTs, locks), Apps/canisters (canister groups, tracked canisters), bots (wallet bot entries) and SNS neurons (hotkey neurons).
- Asset inspection (canister status, staking bot neurons, SNS neuron details) overlaps `/canister`, `/icp_neuron_manager/:id`, `/neuron` pages.
- SNS neuron selling relies on the Jailbreak Wizard (Tools) — linked from marketplace hero and wizard.
- Swap is filed under the Sneedex section while its main entry points are wallet "Buy"/"Sell"; Swap page breadcrumb says Sneedex. Header's location-change handler omits `/swap`, so in-app navigation to `/swap` doesn't re-highlight the Sneedex section.
- Premium (/premium) and partner/admin special prices affect Sneedex creation fee and cut; only the wizard shows them.
- `PriceService`/ICPSwap pool pricing is shared with every USD figure in the app.

**(c) Dead, hidden, half-built or confusing**

- KongSwap removed: split swaps, split slider, auto split search, "compare across DEXes" are dormant; multi-hop badge exists but no multi-hop routing. Help/skill still mention KongSwap.
- `/swap` is useless anonymously (no quotes, no prices; disabled "Connect Wallet" button that doesn't log in).
- Sneed Lock asset type tab commented out; legacy lock UI/escrow code still in wizard; "Sneed Locks" marketplace pill doesn't match current lock offers (they're ICRC-7 NFTs).
- Offer page can't escrow ICRC-7 NFTs or Sneed Locks for PendingEscrow offers; escrow instructions (offer page box, wizard step 4) mention only apps/neurons/tokens.
- Admins can create offers with unverified assets (bypass) — leaves offers in PendingEscrow.
- Marketplace silently performs `processExpiration` writes as the viewer.
- About page lists 4 asset types (missing Trading Bots, NFTs, locks).
- `/tools/escrow` ComingSoon and `/tools/escrow/swap` (OpenChat lookup) are URL-only and named confusingly next to Sneedex.
- Pending bids survive only in page memory; manual-payment block mislabels Sneedex as "App canister id".
- Bid UI shown to the seller on own offer and to non-approved users on private offers (rejected server-side).
- Cancel-with-bids confirmation says bids are refunded; an error variant "Cannot cancel offer with existing bids" also exists.
- USD estimates differ between marketplace (includes lock values) and offer page (excludes locks/NFTs).
- No shareable URLs for marketplace filters/tabs or My Sneedex tabs.
- Both header quick links (marketplace and swap) use the same exchange icon.
- `getOfferFeed` failure falls back to legacy `getPublicOffers`/`getPrivateOffersFor`.
