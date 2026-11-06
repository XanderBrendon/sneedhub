# Lock & liquidity: Sneed Lock and ICPSwap positions

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Scope: `pages/LockWizard.jsx`, `Lock.jsx`, `pages/Sneedlock.jsx`, `pages/SneedlockInfo.jsx`, `TokenLock.jsx`, `PositionLock.jsx`, `TokenLocksOverview.jsx`, `Dashboard.jsx`, `LockModal.jsx`, `LockPositionModal.jsx`, `TransferTokenLockModal.jsx`, `pages/Position.jsx`, `pages/Positions.jsx`, `PositionCard.jsx`, `AddLiquidityModal.jsx`, `SendLiquidityPositionModal.jsx`, `components/{LockDissolvePanel,LockGas,LockNftPanel,LockUntil,SneedLockAssetView,PositionCardModal,PositionTransferGuide,PositionTransferPanel}.jsx`, `utils/{lockNft,lockTime,positionRecipient,PositionUtils,icpswapPoolMatch,icpswapPoolVerify,sneedLockLists,sneedLockActive,sneedLockDisplay,sneedLockNftModel}.js`, `hooks/useLocksCache.js`. I also read the wallet handlers these call (`Wallet.jsx`) and the Locks tab of `TokenCard.jsx`.

Terms used below:
- **Backend / Sneed Lock subaccount**: the account `{ owner: sneed_lock, subaccount: principalToSubAccount(me) }`. The UI calls it "Deposited", "backend wallet", "Sneed wallet ICP", "vault" or "free balance in Sneed Lock".
- **Frontend wallet**: the user's own principal, main account.
- **Lock fee**: ICP paid into the payment subaccount (`getPaymentSubaccount`) before `create_lock` or `create_position_lock`.

---

## 1. Sneed Lock overview / marketing page
- **Routes & params**: `/sneedlock`. Public. Shows a "My Locks" link only when logged in.
- **Purpose**: explains Sneed Lock and links to the tools.
- **Information shown**: static copy only.
  - Sections: "What is SneedLock?", "Revolutionary Features" (Fee Claiming from Locked LPs, Liquid Locking™, Trustless & Immutable), "Use Cases" (Team Token Distribution, LP Commitment Proof, Secondary Markets), and "Getting Started" (5 steps).
  - No live data.
- **Actions** (all links):
  - Hero: "Lock Wizard" → `/lock_wizard`, "Open Wallet" → `/wallet`, "Documentation" → `/help/sneedlock`.
  - Footer: "Lock Wizard", "Open Wallet", "My Locks" → `/sneedlock_info?owner=<me>`, "All Locks" → `/sneedlock_info`.
- **Entry points**:
  - Header Sneed Lock menu "Overview".
  - Products page, DaoInfo drill-down, revenueData "Sneed Lock" node.
  - The Header highlights "Overview" for `/tokenlock`, `/positionlock` and `/lock/*`.
- **Redesign notes**:
  - The copy is stale against v2. It says locks "cannot be canceled, modified … — enforced at protocol level", but v2 has extend, dissolve delay, increase delay and stop dissolving.
  - Step 2, "Register Tokens or LP Positions", describes the old wallet flow; the wizard now registers things automatically.
  - Pure marketing that could merge into the help or landing pages.

## 2. Create a lock: Lock Wizard
- **Routes & params**: `/lock_wizard`.
  - `?type=token|position` jumps to step 2 with that type selected.
  - `&token=<ledgerId>` preselects the token, for type=token only. For type=position the `token` param is **ignored**, even though Token Studio, tokensafety fixes and `/position` all pass it.
  - **Login required**: without login the page only says "Please log in to access the Lock Wizard".
  - **Premium-affected**: the lock fee is lower.
- **Purpose**: guided 3-step creation of a token lock or an ICPSwap LP position lock. Steps are labelled "Type", "Select", "Lock". Completed step circles are clickable to go back.
- **Step 1, Type**: option cards "Lock Tokens" (ICRC-1 for vesting, commitment proof, trust) and "Lock Liquidity Position" ("while still earning fees"), then "Continue".
- **Step 2, token path**:
  - Filter toggle: "All known tokens" / "Tokens in my wallet (N)". The list comes from backend `get_ledger_canister_ids`.
  - `TokenSelector` with allowCustom: "Search any token, or add one by ledger id".
  - Choosing a token **registers it in the user's wallet** (`register_ledger_canister_id`), then loads metadata, the frontend balance, the Sneed Lock subaccount balance and `get_summed_locks`.
  - Shows "Available to lock: X" (wallet + backend − locked) and "· Y already locked", plus a "Check again" button.
  - If nothing is available: a warning card "Your Sneed wallet holds no X yet", the wallet principal box with "Copy", and instructions to send tokens, then "Check again".
  - Otherwise: "Want to lock more than this? Send…" with a compact principal box.
  - Buttons: "Back", "Continue" (needs funds > 0).
- **Step 2, position path**:
  - Filter toggle: "Any ICPSwap pool" / "Pools in my wallet (N)".
  - "Any" uses `SwapPoolPicker` (modes "Select Token Pair" / "Enter Pool Id").
  - "Wallet" lists registered pools with "N positions available to lock" or "No positions to lock".
  - Choosing a pool **registers the pool and both tokens in the wallet**. It then lists lockable positions: positions the user owns at ICPSwap, plus positions claimed in Sneed Lock without an active lock (tagged "already in SneedLock"). Each row shows pair, #id and token amounts.
  - If there are none: a warning card "No X/Y position in your Sneed wallet yet", the `PositionTransferGuide` (5 steps to transfer on ICPSwap), "Watch for it to arrive here" (polls every 15 s for up to 10 min, auto-selects a single arrival, with "Stop" and "Watch again"), and "Check again".
  - If there are some: "Choose the position to lock", "Check again", an expander "Have another X/Y position on ICPSwap? How to move it here", and a link "No position yet? Create one here, in your Sneed wallet, then come back to lock it" → `/position?swap=<pool>`.
- **Step 3, Configure & Lock**:
  - Summary of the asset: token symbol and "Available", or pair #id and amounts.
  - Token only: "Amount to Lock:" with "MAX". MAX = backend free balance + wallet minus one deposit fee; for ICP it also reserves the lock fee plus a tx fee.
  - "Lock Until (TZ)": datetime-local, optional when a delay is set.
  - "Dissolve delay in days (optional)", with an NNS-neuron explanation. With a delay and no date there is no lock period.
  - **Fee panel**:
    - "Lock fee" badge: PriceTierBadge (admin/partner special price), PremiumBadge, or "standard". The amount shows as "FREE" or `x ICP`.
    - Plan rows: "Already deposited", "From your Sneed wallet ICP" (the backend subaccount, used first), "From your ICP wallet", "Ledger transfer fees".
    - Status lines: "Fee already deposited. Nothing more to pay."
    - Shortfall: "Not enough ICP for the fee… You need X more" with the principal box and "Check again". On load failure: "Retry".
  - Irreversibility note: "A lock is irreversible…".
  - "Back"; "Create Lock" is disabled when the fee plan is insufficient.
- **Confirmation**: `ConfirmationModal` with a full sentence covering what, until when, the delay story, the fee plan, and "THIS CANNOT BE UNDONE…".
- **Progress overlay**: phases "Paying the lock fee", "Moving X to SneedLock" or "Transferring the position to SneedLock", then "Creating the lock". If the fee was paid but the lock failed, it says the deposit is kept for the retry.
- **Success**: "🎉 Lock Created Successfully!" with the unlock/delay summary; "Create Another Lock" and "View Lock #N" (→ `/lock/N`, or "View My Locks" → `/sneedlock_info?owner=me`).
- **Calls**:
  - sneed_lock: `getEffectiveLockFee(me, isPosition)` (fallback `get_lock_fees_icp`), special prices, `getPaymentSubaccount`, `getPaymentBalance`, `get_summed_locks`.
  - Fee payment: `transfer_tokens` (backend ICP → payment subaccount) and/or an ICP ledger `icrc1_transfer`.
  - Token lock: a token ledger `icrc1_transfer` to the Sneed Lock subaccount for any shortfall, then `create_lock(amount, ledger, lockEnd, [delay])`.
  - Position lock:
    - Pre-check `verifyIcpswapPool` against the ICPSwap factory `getPools`. An unverified pool is refused **before** the position moves.
    - If the user holds the position directly: `claim_position`, then ICPSwap `transferPosition(me → sneed_lock)`.
    - Then `create_position_lock(swap, dex=1, id, lockEnd, token0, token1, [delay])`.
  - Backend: `register_ledger_canister_id`, `register_swap_canister_id`, `get_swap_canister_ids`.
  - sneed_lock `get_claimed_positions_for_principal`.
- **Entry points**:
  - Header "Sneed Lock" default path and "Create Lock".
  - `/sneedlock` CTAs, `/help/sneedlock`, `/help/wallet`.
  - Token Studio ("Lock {sym} in Sneed Lock", "Lock an LP position").
  - tokensafety fixes ("Lock tokens", "Lock the LP position").
  - `/position` "Lock in Sneed Lock" tab-link; architecture map.
- **Shared building blocks**: `TokenSelector`, `SwapPoolPicker`/`SwapPoolSummary`/`describeSwapPool`, `PositionTransferGuide` + `WalletPrincipalBox` + `TransferArrivalWatch`, `ConfirmationModal`, `PriceTierBadge`, `PremiumBadge`/`usePremiumStatus`, `specialPrices`, `icrcTx`, `icpswapPoolVerify`.
- **Gating/state**: login; Premium and special-price tiers change the fee; no feature flag.
- **Redesign notes**:
  - This is the only creation UI that supports a **dissolve delay**, Sneed-wallet ICP for fees, and a client-side pool verification.
  - The wallet's quick-lock modals (section 3) duplicate creation without these.
  - Side effect: it silently registers tokens and pools into the wallet.
  - "Sneed wallet" means two things here: the logged-in principal in step 2, and the Sneed Lock subaccount in the fee rows.

## 3. Quick lock from the wallet (LockModal, LockPositionModal)
- **Routes & params**: opened from `/wallet` token cards (TokenCard Locks tab "Lock", shown when `token.available > 0`) and position cards (PositionCard Locks tab "Lock", always shown when buttons are visible). LockModal also opens from `PrincipalBox`, the header quick wallet. Login required. Premium-affected.
- **Purpose**: create a token lock, or create **or extend** a position lock, without the wizard.
- **Information shown**:
  - Token: "Your {sym} Balance", "Amount:" with "MAX", "Expiration (TZ)".
  - Fee block: "Lock Fee: FREE / x ICP" + PriceTierBadge, "Payment Deposited:", "Your ICP Wallet:", "Fee already deposited", or "Fee will be paid automatically when you click 'Pay & Lock'".
  - Position modal: the same fee block plus "Expiration".
- **Actions**:
  - "💰 Pay & Lock" / "🔒 Add Lock", or "Close". A `ConfirmationModal` lists the amount, the date and the fee.
  - Progress overlay: "Processing Payment" → "Depositing Funds" / "Funds Ready" → "Creating Lock" → "🔒 Lock Created!" → "Done".
  - Position overlay: "Checking Position…" → "Transferring Position" / "Position Ready" → "Creating Lock".
  - Calls: an ICP ledger transfer to the payment subaccount (from the **frontend wallet only**). Then the wallet `handleAddLock` (token `icrc1_transfer` to backend + `create_lock`, **no delay arg**) or `handleAddLockPosition`.
  - `handleAddLockPosition`: if the user holds the position directly, `claim_position` + `transferPosition` + `create_position_lock(…, [])`; if it is already in Sneed Lock and `isLocked`, **`update_position_lock(swap, id, expiry)`** (an extension); otherwise `create_position_lock`.
- **Shared building blocks**: `ConfirmationModal`, `PriceTierBadge`, `specialPrices`, `icrcTx`.
- **Gating/state**: login (`ensureSession`).
- **Redesign notes**:
  - These duplicate the wizard with fewer features: no dissolve delay, no "THIS CANNOT BE UNDONE" warning, fees only from the frontend ICP wallet, and no client pool verification before the position moves.
  - The position "Lock" button on an already-locked position actually **extends** it via the legacy `update_position_lock`, under a heading that reads "Add New Lock".
  - A second extension path exists on `/lock/:id` ("Extend the lock period", v2 `extend_lock`).
  - Token locks cannot be extended from the wallet at all.

## 4. One lock's page (`/lock/:id`)
- **Routes & params**: `/lock/:id`, where id is the numeric lock id. Public view; actions are owner/holder-gated.
- **Purpose**: the canonical page for one lock, token or position. Title "Sneed Lock 2.0 / Lock Details".
- **Information shown**:
  - Token lock: a `TokenCard` in lock mode (balance tab hidden, Locks tab open). It shows logo, symbol and locked amount + USD; the lock's Lock ID, Amount (+USD), Expires (or "until dissolved" / "<date> at the soonest"), Duration countdown (or "≥ …"), Gas, Age and Owner (name/nickname, copy, context menu); and the Locks tab total.
  - Position lock:
    - Banners: "Decimals unknown: … raw base units" and "Unverified pool. <reason>… not counted in any TVL".
    - `PositionCard`, expanded, buttons hidden, unclaimed fees shown.
    - Card header: pair #id, an "unverified pool" pill, USD total incl. fees, token amounts, a lock icon and an unclaimed-fees $ chip.
    - Status badge: "Deposited (Locked)" / "NOT locked: Sneed Lock does not hold this position" / "NOT locked: the owner holds this position directly" / "Deposited (Unlocked)".
    - Card tabs: Balance (Total, Liquidity, Unclaimed Fees, all with USD); Locks (Lock ID, Lock Expires, Duration, Age, "SneedLock Owner", "ICPSwap Owner" plus a pool-side ownership verification badge "Held by Sneed Lock" / "NOT locked…" / "Not held by Sneed Lock or the lock owner", with copy); Info (Swap Canister, Position ID, Token 0/1 and their ledgers, with copy).
  - Below the card: the dissolve panel (section 5) and the NFT panel (section 6).
  - Errors: "Lock with ID N not found", "Position with ID N not found in swap canister".
- **Actions**: see sections 5–7. The page itself has none: no claim fees, no Sneedex listing, no add liquidity.
- **Calls**:
  - sneed_lock: `get_lock_by_id`.
  - For a token lock: ledger metadata, symbol, decimals and the price service.
  - For a position lock: pool `metadata`, token meta, pool verification, a page-walk of `getUserPositionWithTokenAmount` (the whole pool), `getUserByPositionId`, and `getUserPositionIdsByPrincipal` (lock owner, then sneed_lock) for the ownership check.
- **Entry points**:
  - "View" / "🔗 View" next to a Lock ID in TokenCard and PositionCard.
  - Wizard success "View Lock #N".
  - `SneedLockAssetView` ("Sneed Lock #N", "See the lock", "Open Sneed Lock #N") on Sneedex offers, the NFT item page and the feed.
  - ICRC-3 browser `lockId` fields.
- **Shared building blocks**: `TokenCard`, `PositionCard`, `LockDissolvePanel`, `LockNftPanel`, `LockGas`, `LockUntil`/`LockAge`/`LockTimeLeftAtLeast`, `icpswapPoolVerify`, `sneedLockDisplay`.
- **Gating/state**:
  - Panels render nothing against a pre-v2 canister.
  - Mobile: a single column on purpose ("side by side they were squeezed on phones").
- **Redesign notes**:
  - Gas appears **twice** for token locks: the card's "Gas:" row and the NFT panel.
  - Transfer is labelled "Send" here; the wallet calls it "Transfer Ownership".
  - Sell on Sneedex, claim fees and add liquidity for this lock are only in the wallet card.
  - "View" links use `<a href>`, which reloads the full page.

## 5. Lock time management: dissolve delay and extension (LockDissolvePanel)
- **Routes & params**: embedded in `/lock/:id`. Public view; actions need the lock owner (`st.owner === me`) while the lock is still locked.
- **Purpose**: v2 time rules, behaving like an NNS neuron.
- **Information shown**:
  - Card "Lock period and dissolve delay" with an explanation.
  - Rows: Status (`describeLockTime`, e.g. "until <date>", "dissolving, free <date>", "free <date> at the soonest (lock period until …, then a Xd dissolve delay)", "unlocked"); "Lock period ends" (+ "(over)"); "Dissolve delay"; "Free at" / "Dissolved at"; "Free at the soonest"; "Created" (date + age, or "before Sneed Lock recorded creation times"); "Limits" (min/max lock period + delay from `get_lock_limits`).
- **Actions** (each shows a "…" busy line and a done/error message; no confirmation dialog):
  - "Start dissolving" (`start_dissolving`). Needs delay > 0, not dissolving, and the lock period over. A tooltip explains when it is disabled.
  - "Stop dissolving" (`stop_dissolving`).
  - Hours input + "Add to the dissolve delay" / "Give it a dissolve delay" (`increase_dissolve_delay(id, ns)`).
  - datetime input + "Extend the lock period" (`extend_lock(id, newEndNs)`). Disabled while dissolving ("Stop dissolving first") and needs a date after the current end.
  - Non-owners see "Only the lock's owner can change its dissolve state."
- **Shared building blocks**: `utils/lockTime` (`describeLockTime`, `earliestUnlockNs`, `invalidateLockState`).
- **Redesign notes**:
  - Extension and delay controls exist only here.
  - Typing an hours-based delay here vs. a days-based delay in the wizard is an inconsistency.
  - There is no confirmation dialog, although increasing the delay and extending are irreversible.

## 6. Lock as an ICRC-7 NFT: view and send (LockNftPanel, TransferTokenLockModal, SendLiquidityPositionModal backend mode)
- **Routes & params**:
  - `/lock/:id`: the panel; holder-gated send.
  - Wallet token-card Locks tab "Transfer Ownership" opens `TransferTokenLockModal`.
  - Wallet position card "Transfer" (locked, Sneed-Lock-held) opens `SendLiquidityPositionModal` with `isBackendTransfer`.
  - Login required for sending.
- **Purpose**: "Liquid Locking", meaning the lock changes hands while the assets stay locked.
- **Information shown**:
  - Panel: "NFT #id", NFT image (`icrc7:image`), an explanation naming the canister id ("Whoever holds it owns the lock…"), "Held by" principal and "Subaccount" hex.
  - Token locks only: the gas widget.
  - TransferTokenLockModal: Amount, Expires, "Transaction Fee" (1 or "(2 tx fees)" when the backend free balance < fee, with "1 tx fee will be sent from your wallet first"), and "⚠️ Wallet Compatibility Warning" (ICRC-7).
  - SendLiquidityPositionModal: the same warning for locked positions.
- **Actions**:
  - Panel: "Send the NFT" with `PrincipalInput` "Recipient principal", "Recipient subaccount (optional, 64 hex characters)", and "Send". Calls `icrc7_transfer`.
    - Token: no Sneed fee; the ledger fee comes from the free balance, else the gas tank.
    - Position: free.
    - If gas is needed: "This lock needs gas to move: add some below…".
  - Wallet token modal: recipient + subaccount, a confirm dialog with the fee explanation, then "Transfer" / "Cancel". Calls `sendLockNft` (`icrc7_owner_of` + `icrc7_transfer`), falling back to `transfer_token_lock_ownership` against a pre-NFT canister (no subaccounts). May first send 1 fee to the backend subaccount.
  - Wallet position modal "🔄 Transfer": `positionNftId` (`get_position_nft`) then `sendLockNft`, falling back to `transfer_position_ownership`. Recipient refusal rules (section 13) apply.
- **Shared building blocks**: `PrincipalInput`, `utils/lockNft` (`sendLockNft`, `parseSubaccountHex`, `positionNftId`), `LockGas`, `ConfirmationModal`, `LockUntil`.
- **Redesign notes**:
  - The same ICRC-7 send has 3 UIs with 3 labels: "Send the NFT"/"Send" on the lock page, "Transfer Ownership" on the token card, "Transfer" on the position card.
  - Their fee explanations differ: the lock page mentions gas, the token modal mentions a "backend subaccount pre-send" and does not mention gas.
  - NFTs are also visible and sendable through the generic NFT pages (`/nft`, `/nft_item`), which is another agent's slice.

## 7. Lock gas tank (LockGas / AddGasDialog)
- **Routes & params**: token locks only. A row in TokenCard Locks tab ("Gas:"), in the `/lock/:id` NFT panel, and on `/nft_item` for Sneed Lock NFTs. Public view; "Add gas" from the wallet needs login.
- **Purpose**: prepaid ledger fees so a token-lock NFT can move even when its holder has no free balance.
- **Information shown**: "Gas X SYM · about N transfers" (amber icon when 0). The dialog shows "Now: X, about N transfers", the fee per transfer, and the wallet balance.
- **Actions**:
  - "Add gas" opens a dialog: presets "1 transfer / 5 transfers / 20 transfers" (2 fees each) or "Or an amount in SYM", then "Add gas". Calls a ledger `icrc1_transfer` from the frontend wallet to the lock's gas account (`get_lock_gas`).
  - Logged out: the dialog shows the gas account and subaccount for a manual send from any wallet.
  - Anyone can add gas to any lock.
- **Redesign notes**: a hidden concept. It is only visible as a small row; there is no explanation outside the dialog.

## 8. Browse all locks and TVL (`/sneedlock_info`)
- **Routes & params**: `/sneedlock_info?owner=<principal or text>&ledger=<ledger or text>&tab=expired`. All filters sync to the URL. Public; name lookup of owners only when logged in.
- **Purpose**: the global lock explorer, grouped by token.
- **Information shown**:
  - Hero "Total Value Locked" (≈ in the chosen denomination, for the current tab).
  - Filters: "My Locks" (logged in), `PrincipalInput` "Filter by owner..." (matches principal substring, name or nickname), `TokenSelector` "Select token..." **or** `PrincipalInput` "Filter by ledger...", and "Clear" buttons.
  - Tabs "Active" (count badge) and "Expired" (lazy-loaded, count badge).
  - Per-token rows: logo, symbol, "N locks • a token, b position", total amount + ≈USD, and "(raw units, decimals unknown)" when relevant.
  - Expanded row:
    - "Token Lock #<lockId>" cards: Amount + ≈USD, Expires (date / "Expired" / "until dissolved…" / "unknown"), Owner (short `PrincipalDisplay`).
    - "Position Lock #<positionId>" cards: "Token 1" / "Token 2" amounts + USD, Expires, Owner, and "Position data unavailable…" when the pool cannot be read.
  - "Active/Expired Locks Summary": Token Locks $, Position Locks $, Grand Total $.
- **Actions**: filter, switch tab, expand rows. Lock cards link (`<a href>`) to `/tokenlock?ledger=…&locks=<id>` and `/positionlock?swap=…&positions=<positionId>`.
- **Calls**:
  - sneed_lock `get_lock_states_page`, through `getActive*/getExpired*Locks` (legacy queries as fallback).
  - Ledger metadata for every token.
  - Per pool, `getUserPositionWithTokenAmount` over the whole pool.
  - Price service; backend `getPrincipalDisplayInfo` per owner.
- **Entry points**:
  - Header "All Locks".
  - Hub card "Sneed Lock" (CTA "Create Lock" goes here, not to the wizard).
  - Me page "My Locks" (`?owner=me`).
  - `/sneedlock` buttons, governance registry, health check "Lock state readable", wizard "View My Locks", TokenLock/PositionLock "View All Locks".
- **Shared building blocks**: `PrincipalInput`, `TokenSelector`, `PrincipalDisplay`, `LockUntil`, `sneedLockLists`, `sneedLockDisplay`, `WhitelistTokensContext`, `DenominationContext`.
- **Redesign notes**:
  - TVL here counts position locks in **unverified pools**; there is no factory check, unlike `/lock`, `/positionlock` and `/tokenlocksoverview`.
  - The hero keeps the label "Total Value Locked" on the Expired tab.
  - Lock cards link to legacy viewers, not `/lock/:id`.
  - Position cards are titled by position id; token cards by lock id.
  - A position lock is listed under both of its tokens.
  - The owner filter cannot match names when logged out.

## 9. Per-token lock overview (`/tokenlocksoverview`)
- **Routes & params**: `/tokenlocksoverview?ledger=<ledgerId>`. Public. Without `ledger` the page spins forever: the entry form is commented out.
- **Purpose**: everything locked for one token.
- **Information shown**:
  - Token name + symbol + logo.
  - "Total Value Locked" $: active token locks plus position locks in factory-verified pools.
  - "Total Supply" (+USD).
  - "Percentage Locked: **Coming Soon**" (a placeholder).
  - "Token Locks": one TokenCard (lock mode, buttons hidden) listing **all** locks, expired and archived included.
  - "Position Locks": one PositionCard per **active** position lock (buttons hidden, unclaimed fees hidden), with an unverified-pool pill.
- **Actions**: none. The cards' View links go to `/lock/:id`.
- **Entry points**:
  - tokensafety checks (`tokenLocks`, and `locks` when there are no ids).
  - `/help/sneedlock` links `/tokenlocksoverview` with no param, which is broken: it tells the user to "enter a token's ledger canister id".
  - Not in the header.
- **Redesign notes**:
  - Half-built: commented-out form and a "Coming Soon" metric.
  - The TokenCard Locks "Total" includes expired locks while TVL does not.
  - It overlaps `/sneedlock_info?ledger=` and `/tokenlock?ledger=`.

## 10. Shareable token-lock view (`/tokenlock`)
- **Routes & params**: `/tokenlock?ledger=<ledgerId>&locks=<id,id,…>` (`locks` optional; without it, every lock of the ledger). Public. Without `ledger` it spins forever.
- **Purpose**: a public page for one token's locks, a "Link" target.
- **Information shown**: hero "Token Lock – Viewing SYM lock details"; one TokenCard in lock mode with Lock ID, Amount, Expires, Duration, Gas, Age, Owner (names) and Total; "View All Locks" → `/sneedlock_info`.
- **Actions**: none beyond the card's View, gas and Help links.
- **Entry points**:
  - The TokenCard Locks tab "Link" button (wallet; opens a new tab).
  - SneedlockInfo token-lock cards.
  - `SneedLockAssetView` "Ledger" row on Sneedex offers.
  - tokensafety `locks` links.
- **Redesign notes**:
  - Legacy predecessor of `/lock/:id`. It lists expired and archived locks without a status marker; the header total counts active locks only.
  - The Header classifies it under Sneed Lock "Overview".

## 11. Shareable position-lock view (`/positionlock`)
- **Routes & params**: `/positionlock?swap=<pool>&positions=<id,id,…>` (both required). Public.
- **Purpose**: a public page for a locked LP position.
- **Information shown**: hero "Position Lock – Viewing X/Y position lock details"; the decimals and unverified-pool banners; PositionCard (lock mode, unclaimed fees hidden) with location badge, Locks tab (Lock ID, expiry, duration, age, SneedLock owner, ICPSwap owner + ownership verification) and Info tab; "View All Locks".
- **Actions**: none.
- **Entry points**:
  - The PositionCard Locks tab "Link" (wallet; new tab).
  - SneedlockInfo position cards.
  - tokensafety `positionLocks` (can pass several ids).
- **Redesign notes**:
  - It accepts a comma list but **renders only `positions[0]`**.
  - It walks the whole pool's positions.
  - Superseded by `/lock/:id`.
  - The position lock's lock id is not in the URL.

## 12. Legacy lock list (`/dashboard`)
- **Routes & params**: `/dashboard`. Fetches **only when logged in**; logged out, the spinner never stops.
- **Purpose**: "SneedLock Dashboard", a flat list of all token locks and position locks.
- **Information shown**:
  - "Lock #n" with Token, Amount and Unlock Date.
  - "Position Lock #n" with Swap Canister, Position ID, Unlock Date and token0/token1 amounts.
- **Actions**: "View Details" → `/tokenlock?id=<index>` and `/positionlock?id=<index>`. Neither route reads `id`, so both links lead to an empty page that never stops spinning.
- **Redesign notes**:
  - **Broken and dead.** It reads the v2 tuple `[owner, ledger, {…}]` as the old `[ledger, amount, expiry]`: it shows the owner principal as the "token" and formats the ledger principal as the amount. Position amounts look the positions up in the caller's own ICPSwap list.
  - It still has hard-coded dark styling.
  - `/help/sneedlock` ("All locks list") and `/help/dao` ("an older Sneed Lock list…") still link to it.
  - The Header does not list it.
  - Candidate for removal or a redirect to `/sneedlock_info`.

## 13. Manage locked / deposited positions in the wallet (PositionCard full mode)
- **Routes & params**: `/wallet` → Liquidity tab, one `PositionCard` per position in registered pools. Also `PositionCardModal` ("Open in dialog" ⤢, and in `PrincipalBox`, the header quick wallet, with refresh + Send only). Login required.
- **Purpose**: every action on a position the user holds directly, or holds through Sneed Lock (locked or not).
- **Information shown**:
  - Card header: pair #id, "unverified pool" pill, USD total, both amounts, an "In range" / "Out of range" pill (when the pool tick is known), lock icon (only when effectively locked), unclaimed-fees $ chip.
  - Location badge: "In Your Wallet" / "Deposited (Locked)" / "Deposited (Unlocked)" / "NOT locked…".
  - Balance tab: Total, Liquidity, Unclaimed Fees, and "Swap Canister Balance" (the frontend user's unused balance in the pool).
  - Locks tab: count 0/1, Lock ID + View, Lock Expires, Duration (live countdown in the last hour), Age, SneedLock Owner, ICPSwap Owner + verification.
  - Info tab: ids with copy.
- **Actions** (by state):
  - Header icons: ↑ "Send" / "Transfer", 🌊 "Manage" → `/position?swap&id`, ⤢ "Open in dialog", ⟳ "Refresh position data".
  - Action row: "ICPSwap" (external position page), "Manage" / "View" → `/position`, "Send" (not locked), "Transfer" (locked, Sneed-Lock-held), "Sell on Sneedex" → `/sneedex_create?lock=<lockId>` (locked).
  - Directly held: "Claim Fees" → ICPSwap `claim`, then withdraw (`handleWithdrawPositionRewards`, also registers tokens). Swap Canister Balance "Withdraw" (window.confirm) → pool `withdraw`.
  - Locked in Sneed Lock:
    - "Claim Fees": `claim_position_fees`, returning a request id. The button shows a status trail ("🎯 Claiming rewards...", "⏳ Waiting for the pool to deliver...", "✅ …") and "Request #N". The fees land in the backend subaccount, then `FeeForwarding` moves them to the frontend wallet.
    - "Add Liquidity" (active lock only) opens `AddLiquidityModal`.
  - Deposited but unlocked:
    - "Claim Fees" (window.confirm "This will withdraw the position to your wallet and claim the fees") → `transfer_position(me)` + `claim`.
    - "Withdraw to Wallet" (ConfirmationModal) → `transfer_position(me, swap, id)`. Also shown for expired locks.
  - Locks tab: "Link" → `/positionlock?…` (new tab); "Lock" opens `LockPositionModal` (create, or extend when locked; section 3).
  - "Send" (directly held) → `SendLiquidityPositionModal` "📤 Send" → ICPSwap `transferPosition`.
  - "Transfer" (locked) → NFT send (section 6).
  - Unlocked-held, non-transfer branch: `transfer_position(recipient)`.
  - Every recipient goes through `positionRecipientRefusal`: Sneed Lock's own canister, the anonymous principal, the management canister, and any ICPSwap pool from the factory list are refused, with an explanation.
- **AddLiquidityModal** "+ Add Liquidity to Locked Position":
  - Shows per-token "wallet: X · backend wallet: Y (+Z locked)". Inputs stay linked at the position's ratio; a side the position cannot use reads "Not needed: the position is currently out of range on this side".
  - "MAX — add as much as both wallets allow"; a fee breakdown per token ("N transaction fees … all from your backend wallet / moved from your wallet"); "Add Liquidity"; a result view with any "The pool did not add liquidity: …" message; "Done".
  - Calls: top up the backend from the frontend (`icrc1_transfer`), then sneed_lock `add_liquidity_to_locked_position`.
- **Shared building blocks**: `PositionCard`, `PositionCardModal`, `SendLiquidityPositionModal`, `LockPositionModal`, `AddLiquidityModal`, `ConfirmationModal`, `positionRecipient`, `icpswapPoolVerify`, `FeeForwarding`, `LockUntil`.
- **Wallet integration points (covered by the wallet agent)**:
  - Liquidity tab header: "Scan" (scan all ICPSwap pools for the user's positions), "Move a position here" (toggles `PositionTransferPanel`), "+ Add Swap Pair".
  - `EmptyPositionCard` for registered pools without positions, with an embedded `PositionTransferGuide`.
  - Token card Locks tab: "Link", "Lock", per-lock "🔗 View", "Transfer Ownership", "Sell on Sneedex", "Gas"/"Add gas", "Help" → `/help/sneedlock`.
  - Token card "Deposited" balance = the Sneed Lock subaccount minus locked; expired token locks free up into it; it has a "Withdraw" button.
  - `useLocksCache` (IndexedDB `sneed_locks_cache`) caches the user's token locks per principal for an instant Locks tab. It stores only `lock_id`, `amount` and `expiry`, dropping owner and active.
  - Header "collectibles" badge → `ConsolidateModal` (`useCollectiblesNotifications`) collects LP fees across direct, locked and unlocked Sneed Lock positions.
  - Pockets: `sneed_lock` hosted-pocket adapter (deposited balance).
- **Redesign notes**:
  - Several position capabilities exist **only** here: claim fees on locked positions, add liquidity to locked positions, Sell on Sneedex, extend via "Lock".
  - `/position` explicitly sends the user back here ("Fees on locked positions are collected from the wallet card").
  - Two different "Claim Fees" semantics share one label; the unlocked-deposited one also withdraws the position.
  - It uses `window.confirm`/`alert` instead of the app dialog.

## 14. One position's page: view (`/position?swap=&id=`)
- **Routes & params**: `/position?swap=<pool>&id=<positionId>`. Also accepts `&position=<id>` as an alias for `id`. `/managelp` mounts the same component; the code comment calls it a redirect, but it is not one. Public view.
- **Purpose**: complete information for any ICPSwap position. Page title "Position #N", with a "browse positions" link → `/positions?swap=`.
- **Information shown**:
  - Pool header: logos, "X/Y fee 0.30%", pool id, "ICPSwap" external link, "Price p Y per X (q X per Y), tick t", "Refresh".
  - Position: "Range" (prices, "full range (0 to ∞)") with "in range, earning fees" / "out of range, earning nothing"; "Holds" (amounts + value); "Uncollected fees" (+ value); "Owner": "you (Sneed wallet)", "🔒 locked in Sneed Lock, until <date>", "someone else" or "…"; raw liquidity.
  - Logged in: "In the pool, not in a position: …" banner when the user's unused pool balance > 0.
  - "Log in with your Sneed wallet to act on this position."
  - Foreign position: "Is this your position?" + `PositionTransferGuide` + "Watch for it to arrive here".
  - Sneed Lock holds it for the user without an active lock: "Sneed Lock holds this position for you, and it is not locked."
  - Locked: "Locked positions cannot be changed until the lock ends… Fees on locked positions are collected from the wallet card."
  - Footer "How it works" explainer.
- **Calls**:
  - Pool (as the user, host `icp-api.io`): `metadata`, `getUserPosition`, `getUserPositionIdsByPrincipal`, `getUserUnusedBalance`.
  - `WalletContext.liquidityPositions` to confirm ownership.
  - sneed_lock: lists via `getSwapPositionLocks`, `get_claimed_positions_for_principal`.
  - Ledger balances; token standards (`getTokenInfo`); price service.
- **Entry points**:
  - PositionCard "Manage"/"View" and the 🌊 icon.
  - `/positions` rows.
  - `SneedLockAssetView` "Position" rows (Sneedex, NFT item).
  - `/help/wallet`.
  - The Header "Positions" item highlights for `/position` and `/managelp`.
- **Gating/state**: actions only when `owner === 'me'`.
- **Redesign notes**:
  - This is the richest position view.
  - A position held by Sneed Lock shows as owner "sneedlock". Locked positions get no claim-fees or add-liquidity here even though Sneed Lock supports both.

## 15. One position's page: owner actions
- **Routes & params**: `/position?swap&id`. Login, and the position must be held directly by the user. "Withdraw to my wallet" also works when Sneed Lock holds it without a lock.
- **Purpose**: full ICPSwap position management run as the user against the pool canister.
- **Actions** (tab buttons):
  - "Add": linked amount inputs ("in wallet X"), "Deposit and add".
    - Steps: `requireQuiet` (refuses if an unused balance exists), deposit each token (ICRC-2 `approve` + `depositFrom`, or `transfer` to the pool subaccount + `deposit`), `increaseLiquidity`, withdraw leftovers.
  - "Remove": slider "How much: N%" with an estimate, "Remove N% and withdraw" / "Remove everything". Calls `decreaseLiquidity` + `withdraw`.
  - "Collect fees": "Uncollected: …", "Collect and withdraw". Calls `claim` + `withdraw`.
  - "Move range": range presets "±10% / ±25% / ±50% / Full range / Custom" (Lower price, Upper price), range description incl. ticks and spacing, "Remove, re-mint, withdraw leftovers".
    - Steps: decrease all, claim, `mint` new, withdraw. **The id changes** and the page navigates to the new one.
  - "Transfer": `PrincipalInput` "Recipient principal or name", "Transfer position". Calls ICPSwap `transferPosition(me → to)`.
  - "Lock in Sneed Lock" → `/lock_wizard?type=position&token=<token0>`.
  - "Withdraw to wallet" (unused-balance banner) → pool `withdraw` per token.
  - "Withdraw to my wallet" (Sneed Lock custody, unlocked) → sneed_lock `transfer_position(me, pool, id)`.
- **Feedback**: a step log panel with timestamps, busy, done and error. On failure: "anything left in the pool shows under 'In the pool' and can be withdrawn".
- **Fees**: two ledger fees per token deposited (stated in the UI).
- **Redesign notes**:
  - **"Transfer" here has no recipient guard.** It does not use `positionRecipientRefusal`, unlike the wallet "Send", so the user can send a position straight to Sneed Lock and get it stuck.
  - The "Lock in Sneed Lock" link loses the pool and position: the wizard ignores `token` for positions.
  - No confirmation dialogs on destructive steps (remove, move, transfer).
  - Overlaps the wallet card's Send, Claim Fees and Withdraw.

## 16. Create a new position (`/position?swap=<pool>`)
- **Routes & params**: `/position?swap=<pool>` with no `id`. Login required to act.
- **Purpose**: mint a new ICPSwap position owned by the Sneed wallet principal.
- **Information shown**: the pool header (as section 14); "The position is minted to your Sneed wallet principal…"; range picker; linked amount inputs with wallet balances and a fee note.
- **Actions**: "New position" (tab label), "Deposit and mint". Steps: deposit both tokens, `mint`, withdraw leftovers; then navigate to `/position?swap&id=<new>`.
- **Entry points**:
  - `/positions` "New position".
  - Wizard "No position yet? Create one here…".
  - Token Studio "Create or grow a position".
  - Pool chosen on `/position`.
- **Redesign notes**: a pool's existence is shown only by the picker; there is no pool stats panel here. Those live on `/positions?swap`.

## 17. Create an ICPSwap pool (`/position`, no params)
- **Routes & params**: `/position`. Login needed to create.
- **Purpose**: pick a pool, or create a new ICPSwap pool for a pair.
- **Information shown**:
  - "Which pool?" with `SwapPoolPicker` (choosing a pool navigates to `?swap=`).
  - "No pool for the pair yet? Create one", explaining the PasscodeManager cost.
  - After the check: "ICPSwap orders the pair as A / B (standards). Passcode price: X ICP plus ledger fees"; a suggested price note.
- **Actions**:
  - Two `TokenSelector`s ("First token (pick, or a ledger id)", "Second token").
  - "Check the pair": SwapCalculator `sortToken`, `getTokenInfo`, factory `getPool`. If a pool exists: "A pool exists already: open it".
  - "Starting price: B per one A" input; "Suggest" (USD ratio from the price service).
  - "Buy the passcode and create the pool":
    - PasscodeManager `metadata` / `balanceOf`; if short, ICP `approve` + `depositFrom`.
    - Then `requestPasscode`, then factory `createPool(token0, token1, fee 3000, sqrtPriceX96)` (via SwapCalculator `getSqrtPriceX96`).
    - Then navigates to `?swap=<newPool>`.
  - Fee: the passcode price (1 ICP in production) + ICP fees.
  - Step log panel.
- **Redesign notes**:
  - **Buried**: pool creation is reachable only by opening `/position` with no params.
  - It sits with liquidity management but is a DEX/token-launch capability. It may belong near Token Studio or Swap.

## 18. Browse positions (`/positions`)
- **Routes & params**: `/positions` (tab "Mine" by default when logged in) and `/positions?swap=<pool>` (tab "A pool"). Public; "Mine" needs login.
- **Purpose**: list your positions, or every position in a pool, the way `/neurons` lists neurons.
- **Information shown**:
  - Title "Liquidity positions".
  - "Mine": from `WalletContext.liquidityPositions`, so only pools registered in the wallet. Per pool: logos, pair, "whole pool" link. Per position: #id link, amounts, value, "in range" / "out of range" pill, "🔒 locked" or "held by Sneed Lock". Empty: "No pools in your wallet yet…".
  - "A pool":
    - Header: pair, "price p Y per X · N positions ever, M with liquidity".
    - Totals: "In positions" (amounts + $), "Uncollected fees", "Positions" (with liquidity, in range), "Yours".
    - Table columns: Position (#id + "yours"), Range, Holds, Value, Uncollected fees, Status ("empty" / "in range" / "out of range").
- **Actions**: "Mine" / "A pool" tabs; "New position" → `/position[?swap]`; `SwapPoolPicker`; "only with liquidity" checkbox; "Other pool"; refresh; row links to `/position`.
- **Calls**: pool `metadata`, `getUserPositionWithTokenAmount` paged (adaptive page size, **capped at 3000**), `getUserPositionIdsByPrincipal`; price service.
- **Entry points**: Header Sneed Hub → "Positions"; `/position` "browse positions"; `/help/wallet`.
- **Redesign notes**:
  - Placed under "Sneed Hub" (explorer), not with Sneed Lock or the wallet.
  - "Yours" in the pool table misses your positions held by Sneed Lock: they belong to the Sneed Lock canister at ICPSwap.
  - The pool table has no lock status, unlike "Mine".
  - "Mine" duplicates the wallet Liquidity tab.

## 19. Move a position from ICPSwap into the Sneed wallet
- **Routes & params**: no page of its own. Embedded in:
  - the Lock Wizard step 2 (no positions, or the expander);
  - `/position` ("Is this your position?");
  - the wallet Liquidity tab ("Move a position here" → `PositionTransferPanel`, and `EmptyPositionCard`);
  - `/help/sneedlock` (`PositionTransferPanel` titled "Move a position to your Sneed wallet now").
  - Login is needed for the watch.
- **Purpose**: a position created on ICPSwap belongs to the ICPSwap login; this explains how to transfer it to the Sneed Hub principal.
- **Information shown**:
  - 5 steps: open ICPSwap Liquidity (link app.icpswap.com/liquidity), find the pair and open "Details", "Transfer position", paste the Sneed wallet principal (copy box), confirm.
  - Watch states: idle / watching (with check count) / arrived (ids) / timeout ("Nothing new arrived in ten minutes…").
- **Actions**: "Copy" principal; "Watch for it to arrive here" (polls `getUserPositionIdsByPrincipal` every 15 s, up to 10 min); "Stop"; "Watch again".
  - The panel's pool picker also **registers the pool in the wallet** (`register_swap_canister_id`).
- **Shared building blocks**: `PositionTransferGuide`, `WalletPrincipalBox`, `TransferArrivalWatch`, `PositionTransferPanel`, `SwapPoolPicker`.
- **Redesign notes**:
  - The same onboarding appears in 4 places with 3 separate polling implementations (wizard, `/position`, panel).
  - It is a prerequisite for both locking and managing. A redesign could make it one "Import position" flow.

## 20. Sneed Lock asset/NFT summaries outside Sneed Lock (SneedLockAssetView)
- **Routes & params**:
  - Sneedex offer list and offer pages (`SneedLockAssetIcon/Title/Details`, `SneedLockNftsCompact`, `useSneedLockNfts` + `lockUsdMap` for offer value estimates).
  - Hub (offer values).
  - Feed items (icon + title).
  - `/nft_item` and `NftAssetView`, for the Sneed Lock collection (`SneedLockNftSummary` + `LockGas`).
  - `SneedexCreate` uses `SneedLockAssetIcon`, `sneedLockAssetText` and `fmtUnits`.
  - Public.
- **Purpose**: a uniform read-only description of what a lock holds, its value and its state.
- **Information shown**:
  - Title text "1,000 SNEED locked until …" / "Locked LP position #12 in X/Y until …" plus "Sneed Lock #N".
  - Details card: Lock, Amount (+$), Ledger (→ `/tokenlock`), Pool, Position (→ `/position`), Unlocks (date + days left, or the until-dissolved story), Age, "Held by: Sneed Lock, ownership escrowed in Sneedex".
  - NFT summary:
    - Header: logos, amount or "X/Y LP position #id" with total amounts.
    - "Worth $ / value unknown".
    - Status line: "Locked until … (N days left)" / "Dissolving: unlocks …" / "Locked until dissolved: …" / "Unlocked since …".
    - "Open Sneed Lock #N"; "Details" (Lock, Ledger, Amount, Pool, Position, In the position (+ out of range), Unclaimed fees, Value per token, Dissolve delay).
    - "This lock is no longer in Sneed Lock…" when it is gone.
- **Calls**: anonymous `get_lock_state`, ledger metadata, pool `metadata` + `getUserPosition`, price service; values from `sneedLockNftModel` (v3 math). Cached for 60 s.
- **Redesign notes**:
  - The most accurate lock-value model (`positionLockValue`, which includes fees) lives here and in `/position`. The Sneed Lock pages compute value their own way (`getPositionTVL`, SneedlockInfo amounts).
  - The "Ledger" link points to legacy `/tokenlock`.

## 21. Shared lock/position card views (TokenCard lock mode, PositionCard lock mode)
- **Where**: `/lock/:id`, `/tokenlock`, `/positionlock`, `/tokenlocksoverview`, `/wallet`, `PositionCardModal`, `TokenCardModal`.
- **Modes**: `hideButtons`, `hideAvailable` (hides the Balance tab), `hideUnclaimedFees`, `defaultExpanded`, `defaultLocksExpanded`, `currentLockId` (suppresses the self "View" link).
- **Information exposed**: listed in sections 4 and 13. The wallet-specific balance and neuron tabs belong to the wallet slice.
- **Redesign notes**:
  - One 5.6k-line TokenCard and one 1.3k-line PositionCard serve both as the wallet's action surface and as public read-only lock viewers. Their props decide what each page shows.
  - The lock status shown differs per page: TokenCard has no active/expired marker, and PositionCard derives one from `positionLockStatus`.

---

## Slice observations

### (a) Entities and where they can be viewed and acted on
- **Token lock** (lock id; owner; ledger; amount; expiry; v2 lock_end, dissolve_delay, dissolving_until, created; active/archived):
  - Views: `/lock/:id`, `/tokenlock?ledger&locks`, `/tokenlocksoverview?ledger`, `/sneedlock_info` (Active/Expired), wallet token card Locks tab, `/dashboard` (broken), Sneedex/NFT summaries.
  - Actions:
    - create: wizard, or wallet LockModal (no delay);
    - start/stop dissolving, increase delay, extend: `/lock/:id` only;
    - transfer: `/lock/:id` NFT panel or wallet "Transfer Ownership";
    - add gas: anywhere LockGas renders;
    - Sell on Sneedex: wallet only;
    - after expiry, tokens become the deposited balance, withdrawn in the wallet.
- **Position lock** (lock id; owner; pool; position id; token0/1; expiry; v2 fields; NFT id from `get_position_nft`):
  - Views: `/lock/:id`, `/positionlock?swap&positions`, `/tokenlocksoverview`, `/sneedlock_info`, wallet position card, `/position` (owner line only), `/positions` "Mine" (🔒 locked), Sneedex/NFT summaries.
  - Actions:
    - create: wizard, or wallet LockPositionModal;
    - extend: wallet "Lock" (`update_position_lock`) or `/lock/:id` (`extend_lock`);
    - dissolve controls: `/lock/:id`;
    - transfer as NFT: `/lock/:id` or wallet "Transfer";
    - claim fees and add liquidity: wallet only (+ header Consolidate for fees);
    - Sell on Sneedex: wallet only;
    - withdraw after expiry: wallet "Withdraw to Wallet" or `/position` "Withdraw to my wallet".
- **Lock NFT (ICRC-7, Sneed Lock collection)** (token id = lock id for token locks, first lock id for positions; holder account incl. subaccount; image):
  - Views: `/lock/:id` panel, `/nft_item`, `/nft`, Sneedex.
  - Actions: send (`icrc7_transfer`); list on Sneedex.
- **Lock gas tank** (per token lock; ledger account; balance; ≈ transfers): view in TokenCard, `/lock/:id`, `/nft_item`; add gas (anyone).
- **Lock fee / payment balance** (ICP; tier regular/premium/admin/partner; payment subaccount; deposited amount kept across failed attempts): viewed and paid only inside the creation flows.
- **Deposited balance / Sneed Lock subaccount** (per token; total − locked = free): wallet token card (other slice), wizard "Available to lock", AddLiquidityModal "backend wallet", TransferTokenLockModal fee logic, Pockets adapter.
- **Claimed position** (`get_claimed_positions_for_principal`; Sneed Lock custody of a position, with or without an active lock): appears as "already in SneedLock" in the wizard, "Deposited (Unlocked)" in the wallet, "held by Sneed Lock" on `/positions`, and the custody banner on `/position`.
- **ICPSwap position** (id, liquidity, ticks, amounts, tokensOwed, owner):
  - Views: `/position`, `/positions` (Mine/pool), wallet card, `/positionlock`, `/lock/:id`.
  - Actions:
    - on `/position`: create, add, remove, collect, move range, transfer;
    - in the wallet: send, claim, withdraw unused;
    - import from ICPSwap: guide in 4 places.
- **Pool** (pool id, token pair, fee tier, price/tick, factory-verified flag; per-user unused balance):
  - Views: `/position` header, `/positions?swap` stats, SwapPoolPicker summaries, "unverified pool" pills and banners.
  - Actions: create pool (`/position`); register in the wallet (wizard, transfer panel, "+ Add Swap Pair"); withdraw unused balance (`/position`, wallet "Swap Canister Balance Withdraw").
- **TVL / aggregates**: `/sneedlock_info` hero and summary (all tokens), `/tokenlocksoverview` (one token), `/positions?swap` totals (pool-wide, not lock-related).

### (b) Overlaps with other areas
- **Wallet**: every lock and position action ultimately lives on wallet cards (token Locks tab, Liquidity tab) and their modals. The Sneed Lock pages are mostly read-only viewers of the same cards. Locked-position fee claims, add liquidity and Sell on Sneedex are wallet-only.
- **Header quick wallet (PrincipalBox)**: LockModal (token lock), PositionCardModal (view + Send), SendLiquidityPositionModal.
- **Header collectibles / ConsolidateModal**: bulk fee collection for direct, locked and unlocked positions (`useCollectiblesNotifications`), separate from the per-card "Claim Fees".
- **Sneedex**: "Sell on Sneedex" (`/sneedex_create?lock=`), offer lists and detail pages render Sneed Lock assets and values (`SneedLockAssetView`); escrowed locks show "ownership escrowed in Sneedex".
- **NFT pages**: `/nft`, `/nft_item` and the NFT browser show Sneed Lock NFTs, with gas and summary. NFT sends there duplicate the lock-page "Send the NFT".
- **Token Studio / token safety**: Token Studio links to the wizard and `/position`. Safety checks deep-link to `/tokenlocksoverview`, `/tokenlock` and `/positionlock` (legacy viewers), and their "fixes" link to the wizard.
- **Swap / DEX**: pool creation and position management on `/position` and `/positions` are generic ICPSwap features, not Sneed Lock, but they share navigation weight with the Hub explorer ("Positions" in Sneed Hub).
- **Explorer (Sneed Hub)**: `/positions` sits beside `/neurons` and `/transactions`; `/sneedlock_info` is effectively an explorer too, but under Sneed Lock.
- **Premium**: lock-fee discounts in 3 creation UIs; special prices (admin/partner) via `PriceTierBadge`.
- **Help**: `/help/sneedlock` embeds `PositionTransferPanel` and links to legacy `/dashboard` and broken `/tokenlocksoverview`. `/help/wallet` documents `/position` and `/positions`.
- **Admin**: `/admin/sneedlock` (claim queues, logs, payment stats, ICPSwap pool balance recovery, settings, admins, Premium integration, ICP lock fees, fee recipient). Out of slice.
- **Me / Hub / DAO**: Me "My Locks" → `/sneedlock_info?owner=me`; Hub card "Sneed Lock" ("Create Lock" CTA goes to the explorer); DaoInfo, Products and revenue → `/sneedlock`.

### (c) Dead, hidden, half-built or confusing
- **Dead/broken**:
  - `/dashboard`: misreads the v2 tuples, links to `?id=` routes that do nothing, and spins forever when logged out. Help pages still link to it.
  - `/tokenlocksoverview` without `?ledger`: endless spinner; the input form is commented out, the help copy describes the missing form, and "Percentage Locked: Coming Soon" is a placeholder.
  - `/tokenlock` and `/positionlock` without their required params: endless spinners.
  - `/positionlock` shows only the first of several `positions`.
- **Legacy routes**:
  - `/tokenlock` and `/positionlock` predate `/lock/:id` and stay as "Link" share targets and SneedlockInfo link targets.
  - `/managelp` is an alias of `/position` (not a redirect, despite the comment).
  - The `?position=` param is an undocumented alias for `?id=`.
- **Hidden capabilities**:
  - Pool creation: only `/position` with no params.
  - Gas tanks: a small row plus a dialog.
  - Dissolve delay: wizard input; the controls are only on `/lock/:id`.
  - Extending a position lock: the wallet "Lock" button on a locked position.
  - Add liquidity to a locked position: wallet Balance tab only.
  - Withdrawing a deposited-but-unlocked position: hidden behind a "Claim Fees" that also withdraws.
- **Duplicated flows that differ in features and safety**:
  - Lock creation: wizard vs. wallet modals. The modals lack delay, irreversibility copy, Sneed-wallet ICP for fees, and pool pre-verification.
  - Lock extension: `update_position_lock` vs. `extend_lock`.
  - Lock transfer: 3 UIs and labels.
  - Position transfer: the `/position` "Transfer" has **no** stuck-recipient guard; the wallet "Send" has one.
  - Fee claim: wallet card vs. header Consolidate vs. `/position` "Collect fees" (unlocked only).
  - ICPSwap import guide: 4 placements with 3 polling implementations.
- **Inconsistent numbers**:
  - TVL on `/sneedlock_info` includes unverified pools; `/tokenlocksoverview` and the lock pages exclude or flag them.
  - The `/sneedlock_info` hero says "Total Value Locked" on the Expired tab.
  - The TokenCard Locks "Total" counts expired and archived locks on `/tokenlock` and `/tokenlocksoverview`, while the card header counts active only.
  - Position value: `getPositionTVL` (pool-reported amounts) vs. `positionLockValue` (tick math) vs. SneedlockInfo per-side amounts.
- **Confusing naming**:
  - The Sneed Lock subaccount goes by 5 names: "Deposited", "backend wallet", "Sneed wallet ICP", "vault", "free balance in Sneed Lock".
  - "Sneed wallet" also means the logged-in principal.
  - SneedlockInfo titles position locks "Position Lock #<positionId>", not the lock id.
  - The wallet "Lock" modal says "Add New Lock" when it extends.
  - The `/sneedlock` copy says locks can't be modified, but v2 allows extending and dissolving.
- **Navigation**:
  - Sneed Lock is split across the Header "Sneed Lock" menu (wizard, explorer, overview), "Sneed Hub" ("Positions") and the wallet.
  - `/tokenlocksoverview`, `/dashboard`, `/position` and `/positions` are not in the Sneed Lock menu.
  - The Hub "Create Lock" CTA opens the explorer, not the wizard.
  - The `/position` "Lock in Sneed Lock" link drops the pool and position context.
- **Performance/robustness**:
  - `/lock/:id`, `/positionlock`, `/tokenlocksoverview` and `/sneedlock_info` page through **every** position in a pool (`getUserPositionWithTokenAmount`) to find one, while `/position` uses `getUserPosition(id)` directly.
  - `/positions` caps at 3000 rows.
- **Destructive actions without confirmation**: `/position` remove, move range and transfer; `/lock/:id` extend and increase delay (irreversible).
- **Old dialogs**: `window.confirm`/`alert` remain in PositionCard (claim fees on unlocked deposited, withdraw swap balance, error alerts).
