# Wallet & assets

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Scope: everything a user holds or moves through Sneed Hub, as the code implements it today (read from source; nothing was run).

Routes in this slice:

| Route | Page | Auth | Menu |
|---|---|---|---|
| `/wallet` | Wallet.jsx: tabs Tokens, Liquidity, Apps, Approvals, Statistics | login | Sneed Me → Wallet |
| `/wallet/pockets` | pages/Pockets.jsx | login | none; only "Manage pockets" in the wallet's pocket switcher |
| `/nft` | NFT wallet | login | Sneed Me → NFTs |
| `/nfts` | collection browser | public | Sneed Hub → NFTs |
| `/nft_item` | one NFT | public | none |
| `/transactions` | ledger-wide history | public | Sneed Hub → Transactions |
| `/transaction` | one block | public | none |
| `/scan_wallet` | legacy balance scan of any principal | public | none (HelpTools link) |
| `/convert` | BTC/ETH/ERC-20 ↔ ck tokens | login for actions | none; "Convert" button on ck token cards |
| `/btc.html` (+ `#send`, `#move`, `#move-tobtc`, `#credit`, `#activity`) | separate Vite entry: native Bitcoin wallet | login | none; buttons on the wallet's Bitcoin card |
| `/help/wallet` | help | public | Help |

The sections are grouped in this order:
- Wallet page and its tabs, token cards, pockets, valuation and data layer.
- Token modals and approvals: send, receipt, deposit/withdraw, wrap, collect, add token/pool, approvals, subaccount editor, lock/position transfer modals.
- NFTs.
- Transactions and Scan Wallet.
- Native Bitcoin (btc.html) and Convert.

PositionCard and the lock and position modals are summarized here only as the wallet uses them. Slice 02 (Sneed Lock & liquidity) has the full detail. Neuron pages, `/apps`, `/swap`, `/me` settings and the header quick wallet (PrincipalBox) belong to other slices; they appear here only as cross-links.

**Part A: Wallet page, token cards, pockets, valuation**

## Wallet page shell, identity bar and portfolio header (`/wallet`)
- **Routes & params**: `/wallet` (no query params read; `useLocation` imported but unused). Login required for content; logged out shows "Please log in to view and manage your wallet…" + "Click the "Connect Wallet" button in the header…". Header menu: Sneed Me → "Wallet". Help: `/help/wallet` ("Help" button in hero).
- **Purpose**: one page for every asset the user holds through Sneed Hub (tokens, neurons, locks, LP positions, bots/canisters, approvals) plus a portfolio summary.
- **Information shown**:
  - Title "Sneed Wallet", subtitle "Tokens, liquidity & locked assets".
  - Compact identity bar: the user's principal, or (toggle) the ICP account ID; copy button ("Copy principal" / "Copy account ID"); toggle link "account ID" / "principal".
  - "Total Portfolio" in the user's denomination; inline ICP price (USD); privacy toggle (masks all amounts).
  - Breakdown badges (only non-zero): Liquid (💧), Staked (neurons), Locked (Sneed Lock), Liquidity (LP), Maturity, Rewards (Sneed voting rewards), Fees (unclaimed LP fees).
  - Tab bar with live subtitles: Tokens (total value), Liquidity (LP value), Apps (ICP value in staking bots, else count), Approvals (count of ICRC-2 approvals), Statistics (grand total).
  - Session-expiry handling: send/wrap first call `ensureSession()`; when the login ran out a banner asks to log in instead of a failing call.
- **Actions**:
  - "Refresh" (title "Refresh entire wallet") → re-reads all tokens, positions, neurons, managers.
  - Copy principal / account ID; switch principal ↔ account ID.
  - Privacy toggle (`PrivacyToggle`, also keyboard shortcut; app-wide).
  - Tab switch (persisted).
  - Admin only: "Debug Report" (see below).
- **Entry points & cross-links**: Header "Wallet" (Sneed Me), header quick wallet (PrincipalBox) "open wallet" navigations, ~27 in-app links to `/wallet` (e.g. Lock wizard, position pages "Fees on locked positions are collected from the wallet card"), `/wallet/pockets` back-link.
- **Shared building blocks**: `Header`, `PendingTopUpsNotice` (unfinished ICP→cycles top-ups) and `PendingStakesNotice` (unfinished neuron stakes) rendered at the top of the page; `PrivacyToggle`; `useDenomination`; `WalletContext`.
- **Gating/state**: localStorage `activeWalletTab` (tokens|positions|dapps|approvals|stats). Card order per section saved server-side (`WalletLayoutContext` → backend `get_wallet_layout` / `set_wallet_layout`, debounced 500 ms; sections tokens/positions/apps/sneedapp) via drag-and-drop (react-dnd HTML5 backend; desktop-oriented).
- **Redesign notes**:
  - The page mixes four unrelated domains: fungible tokens, DeFi positions, canister/bot fleet management and allowances; Apps duplicates `/apps` (App Manager).
  - The principal shown here is the only "Receive" affordance for Main (no dedicated Receive dialog for Main outside pockets' Receive panel).
  - Dead code: `SectionHeader` component, `isSneedLockExpanded` (`sneedLockDisclaimerExpanded` key) and `consolidationExpanded` state are persisted but never rendered; `showDebug` is hard-coded `false` (token-card debug output unreachable).

## Collect card: claim all fees, rewards and maturity (wallet header)
- **Routes & params**: `/wallet` header, shown only when (fees + rewards + collectable maturity) ≥ the user's "collectibles threshold". Login.
- **Purpose**: one-click harvesting of everything claimable across the wallet.
- **Information shown**: "Collect" total in denomination; per-kind chips with value: fees (LP), rewards (Sneed voting rewards / RLL), maturity (only from neurons where the user has DISBURSE_MATURITY permission). Error banner "No items available to collect at this time…" (auto-clears 5 s).
- **Actions**:
  - "Collect All" → `ConsolidateModal` with every item; per-kind buttons (fees / rewards / maturity) open it pre-filtered.
  - Per item execution (`handleConsolidateItem`): direct LP position → ICPSwap `claim` + withdraw; Sneed-Lock-locked position → sneed_lock `claim_position_fees` (+ fee forwarding); unlocked deposited position → withdraw position from Sneed Lock then claim; reward → rll `claim_full_balance_of_hotkey(ledger, fee)`; maturity → SNS governance `manage_neuron DisburseMaturity 100%` to the user's default account.
- **Entry points & cross-links**: also reachable from the Header's collectibles notification (same modal via `useCollectiblesNotifications`, separate claim code).
- **Shared building blocks**: `ConsolidateModal` (detailed in "Collect fees, rewards and maturity in bulk"), `FeeForwarding`.
- **Gating/state**: threshold from localStorage `collectiblesThreshold` (set on `/me` settings; `collectiblesThresholdChanged` window event).
- **Redesign notes**: claim logic is implemented twice (Wallet.jsx and the header hook); "unlocked" fee claim silently also withdraws the position from Sneed Lock.

## Tokens tab: the token list
- **Routes & params**: `/wallet` → Tokens (always mounted, hidden when inactive so neurons keep loading). Login.
- **Purpose**: list every registered ICRC-1 token with its full balance picture.
- **Information shown**:
  - One `TokenCard` per token (see the token-card sections). Tokens = ledgers registered for the user in the Sneed backend (`get_ledger_canister_ids`) + tokens the user received as forum tips (`getTipTokensReceivedByUser`), with locks (`get_summed_locks`), RLL rewards, SNS/ICP neurons, pockets.
  - Native Bitcoin standalone card when the BTC flag is on and the wallet has no ckBTC (see "Native Bitcoin row on the wallet's Bitcoin card").
  - Scan progress (current/total/found) while scanning.
- **Actions**:
  - "Hide dust" toggle (title "Hide tokens worth less than $0.01"; counts liquid + locked + staked + maturity + rewards + neuron stake).
  - "Refresh" (tokens only).
  - "Scan" (title "Scan all whitelisted tokens to find balances"): for every whitelisted token not yet registered, `icrc1_balance_of` on the **default subaccount**, 8 in parallel; any non-zero balance is registered (`register_ledger_canister_id`) and appears.
  - "+ Add Token" → `AddLedgerCanisterModal` (any ICRC-1 ledger by canister id → backend `register_ledger_canister_id`).
  - Drag-and-drop reordering of cards (saved per user on the backend).
  - Pocket switcher above the list (see Pockets sections): choosing one pocket replaces the grid with that pocket's token list.
- **Entry points & cross-links**: default tab.
- **Shared building blocks**: `TokenCard`, `AddLedgerCanisterModal`, `useWhitelistTokens` (backend `get_whitelisted_tokens`, cached in IndexedDB `sneed_token_cache/whitelistTokens`), `WalletLayoutContext`, `PocketSwitcher`, `PocketTokenList`, `BitcoinCard`.
- **Gating/state**: localStorage `hideDust_Wallet`. Token list/metadata cached (IndexedDB `sneed_wallet_cache`, `sneed_token_cache`) so cards appear instantly before refresh.
- **Redesign notes**: token registration is a per-user server-side list (not on-chain holdings); "Scan" only checks whitelisted tokens and only Main; tokens auto-appear from tips, LP fee claims (both pair tokens get registered) and swaps — users can't tell why a token is listed.

## Token card: header and quick actions (TokenCard)
- **Routes & params**: `/wallet` Tokens tab; same component on `/lock/:id`, `/tokenlock`, `/tokenlocksoverview` (lock mode, `hideAvailable`), and inside `TokenCardModal` (header quick wallet + "Open in dialog"). Login for actions.
- **Purpose**: summary + entry to every action on one token.
- **Information shown**:
  - Logo (fallback initial), "SNS" badge for SNS tokens, name; ckBTC card titled "Bitcoin" / "ckBTC + BTC" with a network badge when native BTC is on.
  - Total value in denomination and total amount = available + locked + other pockets + (SNS/ICP) neuron stake + maturity + rewards.
  - Price per token (USD and ICP), "Loading prices...", "Price unavailable".
  - Status icons with tooltips: locks count + locked amount; ICRC-2 approvals count (`TokenApprovalsBadge`); neurons count; maturity; "disbursing (7-day vesting)" maturity; rewards.
- **Actions** (collapsed header): Buy / "Swap" icon (title "Buy <SYM>" or "Swap" for ICP) → in-card `SwapModal` (ICP → token; for ICP: output ICP); "Mint <SYM>" (only if the user is the token's minting account → `MintTokenDialog`); "Send" → `SendTokenModal`; "Open in dialog" → `TokenCardModal`; "Refresh token data" (also refreshes neurons).
- **Actions** (expanded action row): "Buy" (ICP→token swap), "Sell" (token→ICP swap) in `SwapModal`; "Mint" (minter only); `ConvertButton` ("Convert" for ckBTC/ckETH/ckERC20 → `/convert`); "Send"; "Remove" (only when the token has zero available+locked+rewards, or setting "always show remove token") → `ConfirmationModal` "You are about to unregister ledger canister …?" → backend `unregister_ledger_canister_id`.
- **Entry points & cross-links**: Swap modal (refreshes both tokens after a swap), `/convert`, MintTokenDialog (Token Studio area).
- **Shared building blocks**: `SwapModal` (same engine as `/swap`), `MintTokenDialog` + `useIsMinter`, `ConvertButton`, `TokenApprovalsBadge`, `TokenIcon`, `useDenomination`, `ConfirmationModal`.
- **Gating/state**: localStorage `alwaysShowRemoveToken` (set on `/me`; `alwaysShowRemoveTokenChanged` event). Card expanded state auto-opens when a Sneed Lock deposit balance exists.
- **Redesign notes**: Buy/Sell/Swap are hidden inside the card as a modal instead of linking to `/swap`; "Remove" disappears whenever any balance exists (no way to hide a token you hold).

## Token card Balance tab: where the token sits
- **Routes & params**: card tab "Balance" (not on lock pages). Login.
- **Purpose**: breakdown of the token across Main, Sneed Lock, other pockets, neurons and rewards; Sneed Lock deposit/withdraw.
- **Information shown**:
  - "Total"; expandable "Liquid" (+ "· in N pockets" when more than one pocket holds it).
  - With pockets (wallet): `TokenPocketBreakdown` rows — Main ("Wallet"), "Sneed Lock" (title "Tokens you hold in Sneed Lock; locked ones can't move until they unlock"; free vs locked), every other pocket (subaccounts, Sneedex escrows, ICPSwap unused balances, service balances) with host badge, amount, value, approval-shield count.
  - Without pockets (legacy, e.g. modal): "Wallet" row and "Sneed Lock" row with "free" amount.
  - "Locked", "Staked", "Maturity", "Disbursing", "Rewards" lines.
  - ckBTC card: native Bitcoin section / Bitcoin (beta) offer (see BTC sections).
  - `TokenApprovalsRow`: approvals granted on this token (only when any exist).
  - GLDT/sGLDT: wrap/unwrap buttons at the card bottom.
- **Actions**:
  - Pocket row actions (per host capability): "Send", "Move" (to another own pocket), "Receive", "Withdraw" (hosted → wallet), "Top up" (service balances), "Wrap to sGLDT"/"Unwrap to GLDT" (Main only), "Move to Main" (non-Main GLDT/sGLDT), "Remove" (pocket). All open the one `SendTokenModal` with presets (`handlePocketAction`: send / move / withdraw / topup / tomain / emptytomain / moveto).
  - "New pocket" (title "A numbered subaccount of your wallet: receive on it, approve from it") and "Move to new" (create + move from Main) inline, with naming and Receive panel in place.
  - Sneed Lock row "Withdraw" (free > 0) → `WithdrawTokenModal`; legacy rows "Deposit" (balance > fee) → `DepositTokenModal`, "Withdraw" → `WithdrawTokenModal` (sneed_lock subaccount of the user).
  - "Claim" rewards → rll `claim_full_balance_of_hotkey`.
  - Approvals row: revoke / change (see approvals sections).
  - "Wrap" (GLDT) / "Unwrap" (sGLDT) → `WrapUnwrapModal`; "Move to Main to wrap" when the token sits only in another pocket.
  - ckBTC Main row: "Move to my Bitcoin address" → `/btc.html#move-tobtc`.
- **Shared building blocks**: `PocketRows` (`TokenPocketBreakdown`, `PocketRow`, `NewPocketActions`, `ReceivePanel`), `wrapActions`, `ApprovalsList`, `BtcNativeSection`, `BtcBetaOffer`.
- **Redesign notes**: "Sneed Lock" balance is a user-owned subaccount of the Sneed Lock canister ("Deposited"/"backend wallet") used as lock staging; Deposit lives only on the legacy rows/Lock flows, Withdraw on both; pockets "Move" covers the same ground — three ways to move tokens into/out of Sneed Lock.

## Token card Locks tab (Sneed Lock token locks)
- **Routes & params**: card tab "Locks" (count in tab). Login for actions.
- **Purpose**: list and act on Sneed Lock time-locks of this token.
- **Information shown**: per lock: "Lock ID:" (link `/lock/<id>` unless it is the current lock page), "Amount:", "Expires:" (`LockUntil`; "Expired"; "until dissolved"), "Duration:" (live second countdown in the last hour; "at least" when dissolve delay), "Gas:" (`LockGas`), "Age:", "Owner:" (when known); "No locks found". "Learn about Sneed Lock" → `/help/sneedlock`.
- **Actions**:
  - "Link" → `/tokenlock?ledger=<id>&locks=<ids>` (shareable proof page).
  - "Lock" → `LockModal` (create a lock from wallet; lock fee may apply; covered by the Sneed Lock slice).
  - per lock "Transfer Ownership" → `TransferTokenLockModal` (lock NFT transfer).
  - per lock "Sell on Sneedex" → `/sneedex_create?lock=<lockId>`.
  - Gas "Add gas" (LockGas).
- **Shared building blocks**: `LockUntil`/`LockTimeLeftAtLeast`/`LockAge`, `LockGas`, `LockModal`, `TransferTokenLockModal`; locks cached in IndexedDB `sneed_locks_cache` (`useLocksCache`).
- **Redesign notes**: full lock management (dissolve/extend) lives on `/lock/:id`; the card offers create/transfer/sell only.

## Token card Neurons tab: SNS neurons
- **Routes & params**: card tab "Neurons" for SNS tokens (count; spinner while loading). Login. Neurons = those where the user's principal has permissions (owner or hotkey) on that SNS governance.
- **Purpose**: manage SNS neurons in-place (deep management on `/neuron`, covered elsewhere).
- **Information shown**: "Learn about SNS Neurons" → `/help/neurons`; "Hide empty neurons"; neuron selector (name/nickname, permission icons, state icon); selected neuron: "Neuron ID:", "Stake:", "Dissolve Delay:", "State:" (Locked/Dissolving/Dissolved), "Voting Power:" (computed from nervous-system params), "Created:", "Age:", "Maturity:", "Disbursing Maturity" list ("Arrives in:"/"Available:"/"Ready to claim"); "Automatically stake new maturity" checkbox; warning "This SNS doesn't allow managing neuron permissions. Neurons created here cannot be transferred to other wallets."
- **Actions** (neuron actions offered; permission-gated):
  - "Create New Neuron" (disabled without balance or if SNS can't grant MANAGE_PRINCIPALS) → dialog: "Stake Amount" (MAX, fee, total), optional "Set Dissolve Delay" (days, min/max from params), "Advanced" → "Neuron Nonce" (auto-found free nonce, "Check Free", subaccount balance) → "Create Neuron" (icrc1_transfer to neuron subaccount → `ClaimOrRefresh` → optional dissolve delay; journaled so "Finish staking" can resume).
  - "Manage" → `/neuron?neuronid=<hex>&sns=<root>`.
  - "Send" → "Send Neuron" dialog (adds recipient with all permissions, verifies, removes all other principals; warnings).
  - "Start Dissolving" / "Stop Dissolving"; "Set/Increase Dissolve Delay" dialog ("Confirm").
  - "Disburse to Wallet" (dissolved + DISBURSE; confirm).
  - Disburse maturity (confirm "Disburse X maturity to your wallet?") → `DisburseMaturity`.
  - "Increase Stake" dialog (MAX, fee, total → transfer to neuron subaccount + `ClaimOrRefresh`).
  - "Split Neuron" dialog (min/max split shown).
  - Toggle auto-stake maturity.
- **Gating/state**: localStorage `hideEmptyNeurons_Wallet`; neurons cached in IndexedDB `NeuronsDB` (`useNeuronsCache`), refreshed into `WalletContext` and `NeuronsContext`.
- **Redesign notes**: a parallel neuron manager to `/neuron` and `/me` "My Neurons"; the wallet is the only place to create an SNS neuron outside `/sns_neuron_wizard`.

## Token card Neurons tab: ICP (NNS) neurons
- **Routes & params**: card tab "Neurons" on the ICP token. Login. Neurons where the user is controller or hotkey (NNS governance).
- **Purpose**: basic NNS neuron management.
- **Information shown**: "Learn about ICP Neurons" → `/help/icp-neuron-manager`; "Hide empty neurons"; selector (controller first; "Ctrl"/"HK"; "You are the controller"/"You are a hotkey"); "Neuron ID:", "Stake:", "Dissolve Delay:", "State:", "Voting Power:", "Age:", "Maturity:", "Role:" (Controller/Hotkey/Viewer), "Disbursing Maturity"; "Hotkeys" list ("No hotkeys").
- **Actions** (controller only): "Start Dissolving", "Stop Dissolving", "Disburse" (dissolved), "Spawn Maturity" (`SpawnOrMergeMaturity 100%`, falls back to legacy Spawn), hotkey "Remove", add hotkey (input "Principal ID to add as hotkey").
- **Redesign notes**: no create/stake/increase for ICP neurons here (that is the ICP staking bot, Apps tab); NNS neurons show in two places (this card and staking-bot cards) with different powers.

## Token card Info tab
- **Routes & params**: card tab "Info".
- **Information shown**: "Ledger Canister:" (copy, "Copy to clipboard"), decimals, fee, name, symbol, conversion rate (USD with 6 decimals, ICP rate), standard.
- **Actions**: copy ledger id.
- **Redesign notes**: no link to Token Studio safety check, `/transactions?ledger=` or `/tools/money` from the card (Apps-tab ledger cards do have them).

## Token card in a dialog (TokenCardModal)
- **Routes & params**: modal; opened by "Open in dialog" on a wallet card, and from the header quick wallet (PrincipalBox) token list.
- **Purpose**: full TokenCard (default expanded) in an overlay with refresh and close (Esc).
- **Shared building blocks**: `TokenCard`.
- **Redesign notes**: from the quick wallet the dialog lacks deposit/withdraw/wrap handlers (buttons would throw); the same token UI exists in three containers (wallet grid, dialog, lock pages).

## Liquidity tab: ICPSwap positions in the wallet
- **Routes & params**: `/wallet` → Liquidity (`positions`). Login.
- **Purpose**: track and manage ICPSwap LP positions held directly or via Sneed Lock.
- **Information shown**: "Liquidity Positions"; one `PositionCard` per position in each registered pool (`get_swap_canister_ids`); `EmptyPositionCard` for a registered pool without positions (pair logos, refresh, "Remove", embedded `PositionTransferGuide`); scan progress. Position card details (amounts, USD, range, location "In Your Wallet"/"Deposited (Locked)"/"Deposited (Unlocked)", fees, Swap Canister Balance, lock info) are catalogued in the Sneed Lock & liquidity slice (02, §13).
- **Actions**:
  - "Refresh"; "Scan" (title "Scan all ICPSwap pools to find your liquidity positions": factory `getPools`, per unregistered pool `getUserPositionIdsByPrincipal`; pools with positions are registered).
  - "Move a position here" (toggles `PositionTransferPanel`: steps + "Watch for it to arrive here").
  - "+ Add Swap Pair" → `AddSwapCanisterModal` (register pool by canister id).
  - Empty pool "Remove" → backend `unregister_swap_canister_id`.
  - Per position (PositionCard): "Claim Fees", "Withdraw" swap balance, "Withdraw to Wallet", "Send"/"Transfer", "Lock", "Add Liquidity", "Sell on Sneedex", "Manage"/"View" → `/position`, "ICPSwap" (external), "Open in dialog" (`PositionCardModal`), refresh.
  - Drag-and-drop reordering.
- **Shared building blocks**: `PositionCard`, `PositionCardModal`, `PositionTransferPanel`/`PositionTransferGuide`, `AddSwapCanisterModal`, `SendLiquidityPositionModal`, `LockPositionModal`, `AddLiquidityModal`, `positionRecipientRefusal`.
- **Redesign notes**: Scan only finds positions owned by the principal on ICPSwap (not Sneed-Lock-held ones, which come via registered pools); positions are also on `/positions` and `/position`, but claiming fees on locked positions and adding liquidity to them exist only here.

## Apps tab: Sneedapp bots (ICP staking bots, trading bots)
- **Routes & params**: `/wallet` → Apps (`dapps`), section "Sneedapp (N)". Login.
- **Purpose**: fleet view of the user's Sneedapp-minted canisters (registered with the sneedapp factory).
- **Information shown**:
  - Banners: "N bot(s) can be upgraded to newer version(s)" ("Upgrade"), "N canister(s) low on cycles" ("Top Up"); link "Manage canister groups →" (`/apps`); "Learn how it works →" (`/help/icp-neuron-manager`).
  - Per card: bot icon (green/red validity), "You are a controller" badge, "WASM hash doesn't match any known app version" warning, name (`PrincipalDisplay`), total ICP value (stake + maturity), version (outdated warning), cycles, memory, maturity available, neuron count, chores status lamp.
  - Expanded "Info": "Canister ID", "Neurons", "App"; for token-app bots (ledger/index cards) a "Token" block: Name, Decimals, Transfer fee, Supply, Minting account, Ledger, Archives, links "Safety report", "Transactions", "Block log"; neuron list (staking bots): per neuron state, stake, value, maturity; expanded: "Neuron ID:", "Stake:", "State:", "Dissolve Delay:", "Voting Power:", "Age:", "Maturity:", "Staked Maturity:", "Auto-stake Maturity:", "Total Value:".
  - Expanded "Chores": per chore summary lamp, "Scheduler"/"Conductor"/"Task" lamps.
  - Empty: "No Sneedapp canisters found." + "Create Your First Manager →".
- **Actions**:
  - "+ Create" → `/create_icp_neuron`; "+ Add" existing manager by canister id (input "Add existing manager by canister ID" → sneedapp factory `registerCanister`).
  - "Refresh"; per card refresh ("Refresh manager data"), "Open in dialog" (`DappCardModal`).
  - "Upgrade to vX.Y.Z" (outdated) → `UpgradeBotsDialog`; banner "Upgrade" (all outdated).
  - "Send" / "Send (transfer control)" → transfer dialog (management `update_settings` controllers=[recipient] + factory `transferManager`).
  - "Manage" → `/icp_neuron_manager/<id>` or trading bot page; "Stake ICP" (link); per-neuron manage link.
  - "⚡ Top-Up" → inline "Add Cycles": "ICP amount", conversion estimate, "Top Up" (ICP transfer to the CMC + `notify_top_up`, journaled for recovery; "Converts ICP from your wallet to cycles for this canister.") ; banner "Top Up" → `TopUpCyclesDialog` (batch).
  - Chores: "▶ Start", "▶ Resume", "⏸ Pause", "⏹ Stop", "⚡ Run Now" (bot `startChore`/`resumeChore`/`pauseChore`/`stopChore`/`triggerChore`); "Full chore management →" (`/icp_neuron_manager/<id>?tab=chores`).
  - "✕ Remove" ("Remove from list (does not delete canister)", inline "Remove?" Yes/No) → factory `deregisterCanister`.
  - Drag a card to "Other Apps" (and back) to move between lists ("Moving app..." progress).
- **Shared building blocks**: `StatusLamp`, `BotIcon`, `DappCardModal`, `UpgradeBotsDialog`, `TopUpCyclesDialog`, `topUpCanisterWithIcp`, `PrincipalInput`; data from `WalletContext` (managers, versions, chore statuses, bot-log alerts, low-cycles list).
- **Gating/state**: localStorage `neuronManagersExpanded`; cycles-threshold settings from `NeuronManagerSettings` (`getNeuronManagerSettings`/`getCanisterManagerSettings`, set elsewhere).
- **Redesign notes**: near-duplicate of `/apps` (App Manager), `/canister`, `/icp_neuron_manager` and trading-bot pages; ICP neurons inside bots show here, not on the ICP token card.

## Apps tab: Other Apps (tracked canisters)
- **Routes & params**: `/wallet` → Apps, section "Other Apps". Login.
- **Purpose**: watch arbitrary canisters (cycles, memory, controller status) and act on those the user controls.
- **Information shown**: per card: "You are a controller", SNS pill, "Controller"/"Tracked", name, cycles, memory, version/neurons/chores badges for detected bots; "Incompatible Manager" card ("This canister's WASM matches an ICP staking bot version, but its interface is not responding correctly…"); expanded "Canister ID", "Cycles", "Memory" (controllers only).
- **Actions**: "+ Add" (input "Add canister by ID", Enter) → backend `register_tracked_canister`; "Refresh"; per card refresh, "Open in dialog", "Send"/"Send (transfer control)" (controllers; management `update_settings`), "View Details" → `/canister?id=`, "Manage"/"Manage Neurons" for detected bots, "⚡ Top-Up" (anyone; ICP → cycles), "Remove" ("Remove from wallet"; Yes/No) → `unregister_tracked_canister`; chore buttons for detected bots; drag to Sneedapp.
- **Gating/state**: localStorage `trackedCanistersExpanded`.
- **Redesign notes**: tracked canisters are the same list `/apps` groups; the transfer-control dialog exists twice (manager vs canister) with identical copy.

## Transfer canister control dialog (Apps tab)
- **Routes & params**: modal from Apps tab "Send" on a bot or tracked canister (controller only).
- **Information shown**: "This will transfer full control of the canister to another principal. You will lose all access."; "Canister to Transfer"; "Recipient Principal ID".
- **Actions**: recipient via `PrincipalInput` (principal or name) → "Transfer Control" (management canister `update_settings` controllers = [recipient]; bots also factory `transferManager`); "Cancel".
- **Redesign notes**: no review step beyond the warning; same capability on `/canister` (controllers edit).

## Statistics tab: portfolio analytics
- **Routes & params**: `/wallet` → Statistics (`stats`). Login.
- **Purpose**: visual breakdown of the whole portfolio.
- **Information shown**: "Portfolio Statistics"; stacked category bar (label, value, %); donut charts "Portfolio by Token", "Category Breakdown", "Liquid Tokens", "Staked (Neurons)", "Liquidity Positions", "Locked Assets", "Unclaimed Rewards & Fees"; "Detailed Breakdown" table with columns Token, Liquid, Staked, Maturity, Liquidity, Fees, Locked, Rewards, ICP Bots, Total (+ Total row); empty state "No portfolio data yet".
- **Actions**: none (read-only; values in denomination; privacy-masked).
- **Redesign notes**: the only portfolio analytics in the app; categories match the header badges; "ICP Bots" counts staking-bot neurons.

## Admin wallet debug report
- **Routes & params**: `/wallet` header "Debug Report" button, admin principals only (backend `caller_is_admin`).
- **Information shown**: text report "WALLET DEBUG REPORT": totals (grand, tokens, LP), breakdowns, per-token/position data.
- **Actions**: "Copy" ("Copied!"), close.
- **Redesign notes**: admin tooling inside a user page.

## Pocket switcher and single-pocket view (wallet)
- **Routes & params**: top of the Tokens tab; shown only when the user has a pocket besides Main and the Sneed Lock wallet. Login.
- **Purpose**: view the wallet as "All pockets" or as one pocket.
- **Information shown**: select "All pockets" / each listed pocket (name, host badge, "· empty"); "Only tokens it holds" checkbox (one pocket); "reading pockets…"; "couldn't read your saved pockets"; one pocket's token list (`PocketTokenList`: every wallet token it holds with balance/value; "This pocket holds none of your wallet's tokens yet.").
- **Actions**: pick pocket (sends then default to it as source); row actions (Send/Move/Receive/Withdraw/Top up/Wrap…); pocket "Remove" (`RemovePocketPanel`: empty → forget after confirm; holding tokens → "Move all <SYM> to Main" per token); "Manage pockets" → `/wallet/pockets`.
- **Gating/state**: localStorage `sneed_pockets_selected_v1:<principal>`, `sneed_pockets_hide_empty_v1`.

## Pockets page: manage all accounts that hold my tokens (`/wallet/pockets`)
- **Routes & params**: `/wallet/pockets` (no params). Login ("Sign in to see and manage your pockets."). Reached only via "Manage pockets" in the switcher (no header entry; header section highlighting doesn't include it).
- **Purpose**: "Every account that is yours and holds tokens: numbered subaccounts of your principal, accounts Sneed canisters hold for you, and service balances. Approve from a pocket that holds only what you want at risk…"
- **Information shown**:
  - "Your pockets": per pocket avatar, name (✓ "Remembered in your account (syncs across devices)"), host badge, "hidden"/"escrow", ICRC-1 account text, principal-shaped-subaccount hint, error "couldn't read <host>: …", per-token balances ("locked X / free Y", "X in escrow"), "Empty".
  - "Hosts": every adapter (Sneed Lock, Sneedex, ICPSwap, Sneed Monitor, Sneed Notifications, TACO, Bitcoin address when on) with kind ("service balance", "chain address · on the Bitcoin card", …) and state ("N pockets", "off", "couldn't read: …", "manage at <host>" for read-only); note "Sneed Notifications is topped up from Main only…".
  - "Check pocket #N": explanation (ICP index works on account identifiers, so ICP pockets can't be listed), results "#N: amount" / "No tokens in those pockets."; note that deep history scan "comes with the ledger history cache".
  - Notes: names are "unpublished, not secret"; "Empty, unnamed pockets stay out of the wallet's lists; names and pockets sync across your devices through the Sneed naming service."
- **Actions**:
  - "New pocket": "Name (optional)" → "Create pocket #N" (next free number) or "Exact subaccount (byte editor)" → `SubaccountEditor` → "Create with this subaccount" (registered in sneed_pid_nicknames `pocket` field).
  - Per pocket: "Receive" (ICRC-1 account copy, "Show QR"/"Hide QR", ICP account identifier "for exchanges", warning "Only wallets that understand subaccounts can pay this pocket"), "Rename" ("Name (private to you)", max 32, "Save", "Clear"), "Approvals" (per-pocket ICRC-2 list, "Check again"; revoke/change), "Hide"/"Unhide", "Remove" (confirm; "Nothing moves").
  - Header: "Show hidden" / "Show Sneedex escrows" toggles (title "Sneedex bid and offer-payment escrows…"), "Refresh" ("Reading…").
  - "Check #N", "Check #1–#20" (choose token; cheap `icrc1_balance_of`; hits are remembered).
- **Shared building blocks**: `usePockets`/`PocketsContext`, `ApprovalsList` + `useApprovalsMany` (subaccount-scoped), `SubaccountEditor` (also `/tools/account`), `ReceivePanel`, `PocketVisuals` (`PocketAvatar`, `HostBadge`, `TokenAmount`, `PocketButton`).
- **Gating/state**: localStorage `sneed_pockets_show_escrows_v1:<principal>`, `sneed_pockets_show_qr_v1` (Receive QR shown); registry mirror in localStorage (NamingService). TACO adapter behind `ctx.flags.taco` (off); Bitcoin address only with the BTC flag.
- **Redesign notes**: the only place to name/hide/forget pockets, see hosts, check ICP pockets and see per-pocket approvals; pocket send/move happen on `/wallet` instead, so management and use are split across two pages.

## Pockets engine: discovery, registry, hosts and moves (what users see)
- **Purpose**: model behind every pocket view (`utils/pockets/*`, `PocketsContext`, lazy — reads only when a component calls `usePockets()`).
- **Information shown / behaviour**:
  - Pocket kinds: wallet (Main = default subaccount, #N numbered, exact subaccounts), hosted (Sneed Lock wallet `h:sneed_lock:wallet`, Sneedex offer-payment and bid escrows, ICPSwap pool unused balances), service (Sneed Monitor prepaid ICP, Sneed Notifications message balance), chain (native BTC address, card-only).
  - Discovery per ledger: registry (sneed_pid_nicknames account entries with `pocket` field) → ICRC index-ng `list_subaccounts` where the ledger has an SNS index (found ones auto-registered) → ICP manual "check pocket #N" → hosted adapters' `listMyAccounts` (15 s timeout each; failures show "couldn't read").
  - Totals count every pocket (hidden too); Sneedex escrows are "quiet" (shown only when withdrawable or "Show escrows"); hosted dust ≤ ledger fee hidden.
  - Withdraw rules per host (e.g. "a won bid stays in escrow until the seller is paid", "X is locked").
  - Moves (`movePlan`): wallet→wallet = transfer; wallet→hosted = host deposit (Sneed Monitor: transfer + `credit_deposit`; Notifications: ICRC-2 approve + `top_up`, from Main only); hosted→wallet = host withdraw (Sneed Lock withdraw, Sneedex `withdrawBidEscrow`/`withdrawOfferCreationPayment`, ICPSwap pool `withdraw` to Main, Monitor `withdraw_balance`, Notifications `withdraw`); hosted→hosted = two steps, two fees.
- **Gating/state**: localStorage `sneed_pockets_icpswap_pools_v1` (pools the swap UI used); registry mirror; selection/hide-empty/show-escrows keys above.
- **Redesign notes**: pockets unify balances that are otherwise scattered (Sneedex escrows, Monitor/Notification balances, ICPSwap leftovers); the send executor here is a third implementation alongside Wallet.jsx and WalletContext.

## Prices, valuation and denomination
- **Routes & params**: app-wide (every value shown in the wallet, quick wallet, stats, pockets). Denomination chosen on `/me` settings ("Denomination Currency"; stored in backend user settings `denomination_token` and localStorage).
- **Information shown**: USD price per token = token/ICP price (ICPSwap pool, fee tiers 0.3%/1%/0.05%) × ICP/USD (ICPSwap ICP/ckUSDC pool `mohjv-…`); ICP price shown in wallet header; all totals converted into the denomination token (default ckUSDC = USD; ckUSDT $, ckEURC €, ckBTC ₿, ckETH Ξ, or any token) with fiat-style or significant-digit formatting; "..." when no price; "Price unavailable" on cards.
- **Actions**: change denomination (on `/me`); privacy mode toggle (masks every formatted amount as ••••, except typed inputs and review/confirm steps).
- **Gating/state**: localStorage `denominationToken` (+ `denominationTokenChanged` event across tabs), `sneed_privacy_mode_v1`, `icpswap_pools`, `icpswap_token_prices` (TTL 1 min ICP/USD, 5 min tokens); IndexedDB `sneed_token_cache` (metadata, ledger lists, whitelist).
- **Redesign notes**: prices come only from ICPSwap pools paired with ICP (tokens without an ICP pool show no value; Kong/Sneedex prices not used for valuation); denomination is chosen far from where values are read; Trading Bot has its own ICP/USD denomination switch.

## Wallet data layer (WalletContext) as seen by users
- **Purpose**: shared wallet state for `/wallet`, the header quick wallet (PrincipalBox), `/wallet/pockets`, VP bar and header notifications.
- **Information exposed**: wallet tokens (progressive load, cached), positions, SNS/NNS neurons per governance, RLL rewards, neuron managers (versions, outdated list, low-cycles list, chore statuses auto-refreshed, bot-log alerts), tracked canisters + controller status, ICP price.
- **Actions exposed**: `sendToken` (quick-wallet send), `refreshWallet`, `refreshTokenBalance`, `ensureTokenRegistered` (other flows auto-add tokens to the user's list), neuron refresh.
- **Gating/state**: IndexedDB `sneed_wallet_cache`, `NeuronsDB`, `sneed_rewards_cache`, `sneed_locks_cache`.
- **Redesign notes**: Wallet.jsx keeps a second, independent copy of most of this state and fetch logic (tokens, positions, managers, sends), so the quick wallet and `/wallet` can disagree and features differ between them (e.g. the quick wallet's position send is a no-op).

**Part B: Token modals and approvals**

## Send tokens to any account (SendTokenModal, standard "All pockets" mode)

- **Routes & params**: modal; no route of its own. Opened on `/wallet` (Wallet.jsx `openSendModal(token, opts)`) and in the header quick wallet (PrincipalBox.jsx). Login required: both callers and the modal call `ensureSession()`. On an expired session the modal shows "Your session expired. Log in again to send."; Wallet's opener refuses to open at all. Not affected by premium.
- **Purpose**: send any registered ICRC-1 token, or ICP, from the user's own balance to any principal or account. The user does not choose whether the tokens come from the wallet or from Sneed Lock: the code takes them from the free Sneed Lock balance, the wallet, or both.
- **Information shown**:
  - Header "Send {SYMBOL}" with the token logo.
  - "Available balance": the token's `available`, which is Main plus free Sneed Lock (masked in privacy mode).
  - "Fee: {fee} {SYM} per transfer · Max you can send: {max} {SYM}". Max = what each source can give after its own fee (utils/sendPlan.js `maxSendable`). Masked in privacy mode.
  - Recipient feedback, live:
    - "✓ Valid principal ID".
    - "✓ Extended address format detected" (ICRC-1 textual `principal-checksum.subhex`), with the decoded Principal and Subaccount, each with a copy button.
    - For ICP: "✓ Valid ICP Account Identifier" (64 hex with a valid CRC32) and the note "...legacy transfer to this account ID directly".
    - For ICP with a principal recipient: "ICP Account ID (for reference)", computed from principal + subaccount, copyable.
  - "Recipient's {SYM} balance": an anonymous `icrc1_balance_of` on the recipient account, debounced 500 ms. Lets the user read any account's balance.
  - Scan note, if a QR code was scanned: a warning when the code names another ledger ("This code asks for a payment on another ledger …"), and a note when the code also asks for an amount (the amount is never auto-filled).
  - DIP20 tokens: "⚠️ This token uses the DIP20 standard which does not support subaccounts." The subaccount section is hidden.
  - Review screen ("Confirm Send / Review before confirming"):
    - The amount, and "+ {fee} {SYM} fee". With two transfers it says "fees (2 transfers: Sneed Lock balance and wallet)".
    - "To" box: the recipient as a PrincipalDisplay (name/nickname, copy), plus an "ACCOUNT ID" or "+ SUBACCOUNT" chip.
    - For ICP, the computed "ICP Account ID".
    - "Show full details": copyable Principal / Subaccount / Extended / Account ID.
    - Warnings: "Account ID transfer. This will use the ICP ledger's legacy transfer method…" and "Subaccount transfer. Verify the recipient controls this subaccount."
- **Actions**:
  - Recipient field: PrincipalInput (placeholder "Enter principal ID, name, extended address, or account ID"; ICP only for the account ID part).
    - Typing searches named principals (tabs "Private" = your nicknames, "Public" = names, "All"; filters for users and canisters). Picking a result fills the principal.
    - Accepts a principal, ICRC-1 textual account, or 64-hex ICP account ID (ICP only).
    - Valid input collapses to a PrincipalDisplay with an "Edit" (pen) button and a clear button.
  - QR scan button ("Scan the recipient's QR code"): camera (BarcodeDetector or jsQR) or reading a picture. Target `icp` or `icrc`. It only fills the field; nothing is sent from a scan.
  - "Use Explicit Principal + Subaccount" (on an extended address): splits it into principal + hex subaccount.
  - "▶ Advanced: Send To Subaccount" toggle (shows a "SET" chip and a collapsed "Sending to subaccount: … Edit" row when filled):
    - Warning "Usually leave this empty".
    - Format choice "Hex String" / "Byte Array" / "Principal ID".
    - Input with a live "✓ Valid subaccount / Resolved (hex)" (copyable) or "✗ Invalid subaccount format".
    - "Convert to Extended Address String" rewrites the recipient into ICRC-1 textual form.
  - Amount input and "MAX". MAX fills `maxSendable`; for ICP account-ID sends it counts the wallet only.
  - "Review Transaction" validates the form:
    - Recipient: "Please enter a recipient address first!", "Invalid recipient address! …", "Invalid subaccount format! …".
    - Amount: "Please enter an amount first!", "Invalid amount! …", "positive amount".
    - Balance: "Insufficient available balance! The most you can send is … Each transfer pays a fee: 1 when sending from one source, 2 when the send is split…".
  - "Cancel"; on review "← Back" and "Confirm & Send" (spinner "Sending...").
  - On confirm, Wallet.handleSendToken (or WalletContext.sendToken in the quick wallet) runs `planSend`. It prefers one source (Sneed Lock first, then wallet) and splits only when needed:
    - Sneed Lock part: `sneed_lock.transfer_tokens(recipient, subaccount, ledger, amount)`.
    - Wallet part: `icrc1_transfer` to `{owner, subaccount}`.
    - ICP account ID: the ICP ledger's legacy `transfer` (memo 0, wallet only).
    - Partial failure: "The Sneed Lock part of the send went through, but the wallet part failed: …".
    - Auth expiry during the send: "Your session expired during the send. Log in again, then check your balance before retrying."
  - After a send, balances refresh (`fetchBalancesAndLocks`) and the send receipt opens.
- **Entry points & cross-links**:
  - Wallet: TokenCard "Send" button (needs `token.available > 0`) and the compact-header send icon; TokenCardModal "Send" (token detail popup on /wallet).
  - Pocket rows, through `handlePocketAction` (see the next section).
  - Header quick wallet: the token row's send icon and the TokenCardModal "Send".
  - The receipt links to `/transaction?id=&ledger=`.
- **Shared building blocks**:
  - Components: PrincipalInput (shared app-wide), QrScanButton + utils/scanParse.js, PrincipalDisplay, PocketPicker.
  - Helpers: utils/AccountParser.js (parseAccount, extended address, resolveSubaccount), utils/sendPlan.js, utils/icrcTx.js (`icrc1Transfer`, `icpLegacyTransfer`).
  - Hooks and dialogs: usePockets, appDialog (receipt).
- **Gating/state**:
  - Privacy mode (localStorage `sneed_privacy_mode_v1`) masks the available and max amounts; review amounts are never masked.
  - No feature flag. Width capped at 90vw; nothing mobile-specific beyond the QR camera.
- **Redesign notes**:
  - Three implementations of the send executor: Wallet.handleSendToken, WalletContext.sendToken, and the pockets `executeTransfer`. Each carries its own copy of the ICP legacy-transfer IDL.
  - The modal plans with pockets-model balances (`tokenWithPockets`), but Wallet re-plans from `token` fields. They can disagree if one is stale.
  - Subaccount entry here uses its own three-format input, not SubaccountEditor (six formats). See the Sub-slice notes.
  - The Main/Sneed Lock split is invisible until review. Users are never asked which source to use.
  - Nothing names the destination account type (user vs canister). "Recipient's balance" is a hidden but useful lookup.
  - There is no "send to neuron" or "top up canister" mode here. SNS neuron stake and increase are in TokenCard; canister top-ups are elsewhere.

## Send from one pocket, or move between my own pockets (SendTokenModal pockets mode)

- **Routes & params**: same modal. Props `fromPocketId`, `toPocketId`, `mode: 'send'|'move'`, `fillMax`, `onPocketTransferDone`. Opened from `/wallet` only; the quick wallet passes none of these, but still shows the From picker when the user has extra pockets. Login required.
- **Purpose**: pick which of the user's "pockets" pays, or move tokens between them. Pockets are:
  - wallet subaccounts: Main and numbered pockets #1, #2…;
  - hosted accounts: Sneed Lock wallet, Sneedex bid/payment escrows, ICPSwap pool balances, TACO;
  - service balances: Sneed Monitor and Sneed Notifications.
- **Information shown**:
  - "Can send from {pocket}: {withdrawable}" (the adapter's `rules()`), with the adapter's explanation (`why`).
  - "From:" PocketPicker. Options show name · number, host badge, free/locked balance, and 🛡 grants; the first option is "All pockets (Main + Sneed Lock)".
  - In move mode: header "Move {SYM}", a "To:" PocketPicker (excluding the source), and "Two steps through Main: N ledger fees."
  - Review: "From: {pocket}", "To your pocket: {pocket}", and "This takes N steps through Main, each paying the ledger fee." Fees = token fee × number of steps, with a note such as "fees (2 steps: withdraw, then send)".
- **Actions**:
  - Choose the From pocket. Changing it resets the amount.
  - Toggle "Move to one of my pockets" / "Send to someone else instead". The toggle shows only when the user has pockets besides Main and Sneed Lock.
  - "+ New pocket" inside the To picker creates the next numbered pocket right away (`pk.create({})`).
  - MAX = pocket withdrawable minus the fees of later steps. With `fillMax` the amount starts at MAX ("Move all {SYM} to Main").
  - Validation errors:
    - "Pick the pocket to move from/to", "Still reading what this pocket can send…".
    - "The most {pocket} can send now is …", "{pocket} pays out whole: send … (MAX)" (Sneedex payment escrows are all-or-nothing).
    - From `transferProblem`: "That is the same pocket", "That is the same account", "{host} does not let tokens out from here", "{host} can't be topped up from the wallet", "Sneed Notifications tops up from Main only".
    - "Sends to an ICP account ID come from Main: pick "All pockets" or Main."
  - Confirm runs `pk.transfer` → `executeTransfer`, which goes through the pocket adapters. Possible steps:
    - wallet→any: `icrc1_transfer`;
    - wallet→host: adapter `deposit`;
    - host→wallet: adapter `withdraw`;
    - host that pays to Main only → elsewhere: withdraw to Main, then send;
    - host→host: withdraw to Main, then deposit.
  - If a step after the first fails: "Step N of M failed (step N-1 went through, the tokens are in Main)".
  - The receipt says "Moved" or "Sent", with block links for `send` steps only.
- **Entry points & cross-links**: all through Wallet `handlePocketAction(kind, pocket, token)`:
  - Pocket rows on the token card (TokenPocketBreakdown) and the PocketTokenList shown when the PocketSwitcher selects one pocket.
  - Row actions: "Send", "Move", "Withdraw" (to Main), "Top up" (service balances), "Move to Main" (GLDT/sGLDT outside Main), "Move all {SYM} to Main" (emptying a pocket before removal), and `moveto` (right after creating a pocket on the card).
  - With the PocketSwitcher set to a pocket, a plain Send defaults to that pocket.
  - "Wrap"/"Unwrap" pocket actions open WrapUnwrapModal.
- **Shared building blocks**: PocketPicker (also used on the Pockets page and in receive/approve flows), usePockets/PocketsContext, utils/pockets/move.js, model.js, and the adapters.
- **Gating/state**: pocket UI appears only when `hasExtraPockets` or a pocket is preset. Hidden and errored pockets are excluded from From choices.
- **Redesign notes**:
  - Send, Move, Withdraw, Top up and Move to Main are all one dialog under different presets. The header and review still say "Confirm Send" / "Confirm & Send" for moves.
  - Bug (plausible): choosing "Main" explicitly sends you down the pocket path, which then refuses ICP account-ID recipients with a message telling you to pick Main. When the PocketSwitcher is set to Main, account-ID sends are blocked unless the user switches to "All pockets".
  - Multi-step fee costs only show up after choosing the To pocket.

## Send receipt (components/SendReceipt.jsx)

- **Routes & params**: an app-level alert dialog (`appAlert`), opened by `showSendReceipt()` after SendTokenModal succeeds. Only SendTokenModal uses it.
- **Purpose**: confirm what was sent and link to the ledger blocks.
- **Information shown**:
  - Title "Sent" or "Moved".
  - "{Sent|Moved} {amount} {SYM} to {recipient}". The recipient is the pocket name, extended address, account ID or principal.
  - One line per block: "Block #N", with a note "(from your Sneed Lock balance)" or "(from your wallet)" on split sends.
- **Actions**: "View transaction" link to `/transaction?id={block}&ledger={ledger}` (closes the dialog). Host withdrawals and deposits get no link.
- **Redesign notes**: the other modals (deposit, withdraw, wrap, lock transfer, position send) show no receipt or block link at all; they just close.

## Deposit tokens into the Sneed Lock "backend wallet" (DepositTokenModal)

- **Routes & params**: modal on `/wallet`. Wallet `handleDepositToBackend(token)` silently does nothing unless `token.balance > fee`. Login required.
- **Purpose**: move free wallet tokens into the user's own subaccount on the Sneed Lock canister (the "backend" / "deposited balance") so they can be locked or used by Sneed Lock flows. Sneed Lock is the only backend; there is no other target.
- **Information shown**: "↓ Deposit {SYM}", "Wallet Balance:" (Main balance), "Amount to Deposit", "Transaction Fee: {fee}".
- **Actions**:
  - "MAX" = wallet balance − fee.
  - "Deposit" validates: "Please enter an amount first!", invalid/positive checks, "Insufficient frontend balance! Remember that depositing requires 1 transaction fee.", "Frontend balance is too small to cover the transaction fee…".
  - A ConfirmationModal ("Confirm Action": "You are about to deposit X SYM from your wallet. This will cost {fee} in transaction fees.", "Confirm"/"Cancel") then calls `icrc1_transfer` to `{owner: sneed_lock, subaccount: principalToSubAccount(user)}` and refreshes balances.
  - "Cancel".
- **Entry points & cross-links**: TokenCard legacy balance breakdown, "↓ Deposit" on the wallet balance row. That breakdown shows only without pockets, i.e. inside TokenCardModal. With pockets the card has no Deposit button; depositing happens through the Move dialog (to the Sneed Lock pocket) or the Lock flows. LockModal deposits any shortfall automatically.
- **Shared building blocks**: ConfirmationModal (`doAwait=false`), StringUtils.
- **Redesign notes**:
  - Neither the dialog nor the confirm text names the destination (Sneed Lock). "frontend/backend" jargon shows up in error texts.
  - It duplicates the pockets "Move to Sneed Lock" path.
  - In the quick wallet, the TokenCardModal does not get `handleDepositToBackend`, so its Deposit button would throw (see the Sub-slice notes).

## Withdraw from the Sneed Lock backend wallet (WithdrawTokenModal)

- **Routes & params**: modal on `/wallet`. Wallet `handleWithdrawFromBackend(token)` silently does nothing unless `available_backend > fee`. Login required.
- **Purpose**: move unlocked tokens from the user's Sneed Lock subaccount back to Main.
- **Information shown**: "↑ Withdraw {SYM}", "Deposited Balance:" (`available_backend` = Sneed Lock balance − locked), "Amount to Withdraw", "Transaction Fee".
- **Actions**:
  - "MAX" = available_backend − fee.
  - "Withdraw" validates: "Insufficient backend balance! Remember that withdrawing requires 1 transaction fee.", "Backend balance is too small…".
  - Confirm ("You are about to withdraw X SYM from your deposited balance. This will cost …") calls `sneed_lock.transfer_tokens(user, [], ledger, amount)` and refreshes.
  - "Cancel".
- **Entry points & cross-links**:
  - TokenCard legacy breakdown: "↑ Withdraw" on the Sneed Lock row.
  - TokenCard pockets breakdown: "Withdraw" on the Sneed Lock row (`onWithdrawLock`).
  - TokenCard GLDT/sGLDT "Move to Main to wrap/unwrap" falls back to it when no pocket source is known.
  - The pockets "Withdraw" on other hosted pockets goes through SendTokenModal instead.
- **Redesign notes**: two different UIs withdraw from Sneed Lock (this modal, and the pockets Move/Withdraw in SendTokenModal). The labels "Deposited Balance" and "backend" are inconsistent with the "Sneed Lock" pocket name.

## Wrap GLDT → sGLDT and unwrap sGLDT → GLDT (WrapUnwrapModal)

- **Routes & params**: modal on `/wallet` (Wallet `openWrapModal` / `openUnwrapModal`, both `ensureSession`). Mode comes from the token: GLDT ledger `6c7su-kiaaa-aaaar-qaira-cai` wraps; sGLDT `i2s4q-syaaa-aaaan-qz4sq-cai` unwraps. No other tokens wrap. Login required.
- **Purpose**: convert gold-backed GLDT to staked/wrapped sGLDT and back, from the Main balance only.
- **Information shown**:
  - "🔄 Wrap GLDT to sGLDT" or "Unwrap sGLDT to GLDT".
  - "Wrapping Process": Approve call fee, Transfer call fee, "Total Cost" (2 × GLDT fee).
  - "Unwrapping Process": Transaction Fee (1 × GLDT fee), "Unwrapping Fee" (2 × GLDT fee), "Total Cost" (3 × GLDT fee).
  - Live "Expected result: X sGLDT/GLDT (costs Y GLDT in fees)".
  - No balance is shown in the modal.
- **Actions**:
  - "MAX" = the whole Main balance.
  - "Wrap"/"Unwrap" validates:
    - minimums: "Minimum wrap amount is 0.7 GLDT.", "Minimum unwrap amount is 0.4 sGLDT.";
    - "Insufficient frontend balance!", "Amount too small! Must be larger than … to cover transaction fees.";
    - "GLDT token not found. Please add GLDT to your wallet first."
  - Then ConfirmationModal "You are about to wrap/unwrap X … Expected result …".
  - Wrap:
    - Read `icrc2_allowance` (GLDT → sGLDT canister).
    - If the allowance is short, `icrc2_approve` for amount − fee with a 1 h expiry and `expected_allowance`; it is recorded for the approvals list.
    - `sgldt.deposit([], amount − 2·fee)`.
    - Auto-register sGLDT in the wallet (`register_ledger_canister_id`) if missing.
  - Unwrap: `sgldt.withdraw([], amount)`, then auto-register GLDT if missing.
  - Both tokens refresh afterwards.
- **Entry points & cross-links**:
  - TokenCard GLDT/sGLDT footer: "Wrap to sGLDT" / "Unwrap to GLDT".
  - When only other pockets hold the token: "Move to Main to wrap/unwrap" (opens the move dialog or WithdrawTokenModal) with the note "Wrapping works from Main: move your … to Main first." (utils/wrapActions.js).
  - Pocket row actions "Wrap to sGLDT" / "Unwrap to GLDT" on Main, and "Move to Main" elsewhere.
- **Shared building blocks**: ConfirmationModal, `approveAndRecord` (utils/approvals.js), wrapActions.js, pockets model.
- **Redesign notes**:
  - The "add GLDT first" error contradicts the auto-register on unwrap.
  - The fee wording in the confirm text says "GLDT equivalent".
  - No balance shown in the modal.
  - Wrap/unwrap logic lives in Wallet.jsx (not reusable from the quick wallet).

## Collect fees, rewards and maturity in bulk (ConsolidateModal)

- **Routes & params**: modal. Login required.
  - `/wallet`: the "Collect" card. It shows only when the collectible total (fees + rewards + collectable maturity, in USD) is at least the `collectiblesThreshold` setting (localStorage `collectiblesThreshold`, default $1). Buttons: "Collect All" plus per-category chips (💸 fees $, 🎁 rewards $, 🌱 maturity $).
  - The header notification icon ("N items to collect worth $X (fees, rewards, maturity)") opens it with type `all`. It shows when `notifyCollectibles` (set on /me, default on) and header notifications are enabled.
- **Purpose**: claim everything claimable across positions, Sneed rewards and SNS neurons in one pass.
- **Information shown**:
  - Title "Collect Fees" / "Collect Rewards" / "Collect Maturity" / "Collect All".
  - One row per item with icon, name, amounts and USD value. Item kinds:
    - fee items: "{A}/{B} Position #id (Direct|Locked|Unlocked)" with "x A + y B";
    - reward items: "{SYM} Rewards";
    - maturity items: "{SYM} Neuron {8-hex}...".
  - "Select All (n of m)"; "Selected Totals" with "Total Value" (masked) and per-token sums.
  - Per-item status ⏳/✅/❌ and "Error: …".
- **Actions**:
  - Toggle items or all. The action button reads "Claim Fees" / "Claim Rewards" / "Disburse Maturity" / "Collect All" and is disabled with 0 selected. "Cancel" becomes "Close" after a run.
  - Items run one after another. Per item kind:
    - Direct position fees: ICPSwap `claim` + `withdraw` of both tokens.
    - Locked position: `sneed_lock.claim_position_fees`, then best-effort forwarding to the wallet.
    - Unlocked position held by Sneed Lock: `sneed_lock.transfer_position` to the user (moves the position out of Sneed Lock), then claim + withdraw.
    - Rewards: `rll.claim_full_balance_of_hotkey(ledger, fee)`.
    - Maturity: SNS governance `manage_neuron` DisburseMaturity 100% to the user's own account.
  - Items whose amount is below the token fee are left out. Maturity counts only neurons where the user has DISBURSE_MATURITY.
- **Entry points & cross-links**: Wallet Collect card (`handleOpenConsolidateModal`); with nothing loaded it shows the error "No items available to collect at this time…", auto-cleared after 5 s. Header (Header.jsx + hooks/useCollectiblesNotifications.js, module cache, re-check at most every 30 s, refresh 500 ms after closing).
- **Shared building blocks**: Wallet claim handlers (handleWithdrawPositionRewards, handleClaimLockedPositionFees, handleClaimUnlockedDepositedPositionFees, handleClaimRewards, handleDisburseMaturity); FeeForwarding utils.
- **Gating/state**: localStorage `collectiblesThreshold`, `notifyCollectibles`; window events `collectiblesThresholdChanged`, `notifySettingChanged`.
- **Redesign notes**:
  - Two independent implementations of every claim (Wallet.jsx and useCollectiblesNotifications.js).
  - "Unlocked" fee claims withdraw the position from Sneed Lock as a side effect, and the dialog does not say so.
  - "Selected Totals" re-parses formatted description strings with `^([\d.]+)`, so amounts with thousands separators (formatAmount prints "1,234.5") or masked amounts drop out of the per-token totals.

## Register a token in the wallet (AddLedgerCanisterModal)

- **Routes & params**: modal on `/wallet`, from the Tokens tab header "+ Add Token" (next to "Scan", which scans whitelisted tokens for balances). Login required.
- **Purpose**: add an ICRC-1 ledger to the user's tracked token list, stored server-side.
- **Information shown**: "+ Add Token"; mode toggle "Select from List" / "Enter Manually"; "Select Token" (TokenSelector: whitelisted tokens from the backend, searchable, logos) or "ICRC1 Token Ledger Canister ID".
- **Actions**: "Add Token" validates "Please select or enter a ledger canister id first!", "Invalid canister id!…", and "Invalid ICRC1 ledger canister id!…" (an anonymous `icrc1_metadata` probe). Then `app_sneeddao_backend.register_ledger_canister_id` and a refresh of that token. "Cancel".
- **Entry points & cross-links**: Wallet only. The reverse action, unregister ("remove token", `unregister_ledger_canister_id` with a ConfirmationModal), sits on TokenCard. Wrap/unwrap also registers tokens automatically.
- **Shared building blocks**: TokenSelector (used app-wide), WhitelistTokensContext.
- **Redesign notes**: an error from `onSubmit` is set and the dialog closes in `finally`, so registration failures are never seen. The list does not exclude tokens already registered.

## Register an ICPSwap pool for LP tracking (AddSwapCanisterModal)

- **Routes & params**: modal on `/wallet`, from the Liquidity tab header "+ Add Swap Pair" (next to "Move a position here"). Login required.
- **Purpose**: tell the backend to track an ICPSwap pool so the user's positions in it show up.
- **Information shown**:
  - "💱 Add Swap Pool Canister". SwapPoolPicker modes "Select Token Pair" ("First token:" / "Second token:", looked up via the ICPSwap factory `getPool` at the 0.3% fee tier) or "Enter Pool Id".
  - Pool preview: both logos, "✓ A/B", token names, both ledger ids and the pool id.
  - "No ICPSwap pool found for this pair at the 0.3% fee tier. If the pool exists, enter its canister id manually."
- **Actions**: "Add Swap Pair" (disabled until the pool resolves; "Adding...") calls `app_sneeddao_backend.register_swap_canister_id`, then refreshes positions. Errors stay in the dialog. Validation: "Select both tokens of the pair, or enter the pool canister id manually.", "Still checking the pool…", "That canister is not an ICPSwap pool." "Cancel".
- **Entry points & cross-links**: Wallet only. Unregister lives on the position UI (`unregister_swap_canister_id`). SwapPoolPicker is reused in SneedexCreate, LockWizard, Positions, Position, PositionTransferPanel and SneedLockAssetView.
- **Redesign notes**: only the 0.3% tier is looked up automatically; any other tier needs the manual pool id.

## See all my ICRC-2 approvals (Wallet "Approvals" tab — components/WalletApprovals.jsx)

- **Routes & params**: `/wallet`, tab `approvals`, persisted in localStorage `activeWalletTab`. The tab subtitle shows the count. Login required. Covers the main account only.
- **Purpose**: list every live allowance the user's main account has granted, across all wallet tokens, and let them be revoked or changed.
- **Information shown**:
  - "Approvals (N)" and an explainer: "An approval lets another account or canister take up to an amount… You have N across M tokens; K without an expiry or for an unlimited amount. Revoke the ones you no longer need."
  - The ApprovalsList, with the token named on each row.
  - "How these were found (N tokens checked partly, M failed)" toggle. It shows per-token coverage:
    - "complete: the ledger lists its approvals (ICRC-103)";
    - "from the transaction history (index)" ("latest part only");
    - "Sneed canisters, pools and spenders approved here only";
    - "no approvals on this token (no ICRC-2)";
    - or "failed: …", plus discovery notes.
- **Actions**: "Refresh" (forced re-discovery of all ledgers; shows "Checking d/n"); everything in ApprovalsList (below).
- **Entry points & cross-links**: Wallet tab bar. Wallet also runs `useApprovalsMany` for all tokens at mount (for the tab count and the token-card badges).
- **Shared building blocks**: ApprovalsList, useApprovalsMany, utils/approvals.js.
- **Gating/state**: localStorage `sneed_approvals_cache_v1` (results, 30 min max age, 400 entries), `sneed_approval_spenders_v1`, `icpswap_pool_cache` (read); window event `sneed-approvals-changed`.
- **Redesign notes**: a strong security feature buried as the 4th wallet tab. Pocket and subaccount approvals are only on the Pockets page.

## Approvals on a token card (TokenApprovalsBadge, TokenApprovalsRow)

- **Routes & params**: inside TokenCard on `/wallet` and in TokenCardModal. Login required.
- **Purpose**: surface a token's approvals where the token is shown.
- **Information shown**:
  - Header badge: 🤝 count. It turns to warning colour if any approval is unlimited or has no expiry; tooltip "N approvals: others may spend this token from your account". It reads the cache only and makes no calls.
  - Balance tab row: "N approvals ▸", which expands to the ApprovalsList for that token. Hidden when there are none.
- **Actions**: expand/collapse; "Check again" (refresh icon); Revoke/Change in the expanded list.
- **Redesign notes**: each card's row auto-loads (network) while the page-level hook loads the same data. Several ways to reach the same list.

## Revoke or change an approval (ApprovalsList / ApprovalRow)

- **Routes & params**: wherever ApprovalsList renders: Wallet Approvals tab, token card row, Pockets page per-pocket "approvals" panel. Revoke and Change appear only when the approval's `from.owner` is the logged-in principal and the spender principal is known.
- **Purpose**: inspect, revoke, or re-scope an ICRC-2 allowance.
- **Information shown**:
  - Token (logo + symbol →) when listing across tokens.
  - Spender, from one of these:
    - known name (Sneedex, Sneed Lock, Forum, Sneed SMS, Notifications, Names, Nicknames, Rewards (RLL), Sneedapp, Premium, Sneed Monitor, Sneed Hub backend, Sneed Pad (ICP), CyclePay (ckUSDC), KongSwap, sGLDT (GLDT), "ICPSwap A/B pool");
    - PrincipalDisplay;
    - "sub …" for a spender subaccount;
    - "Unknown spender · account id …" (ICP index only).
  - Allowance ("Unlimited" when ≥ 10¹⁰ whole tokens or ≥ total supply) and "≈ $value".
  - "Never expires" or "Expires {date}"; "Main account" or "Subaccount …".
  - Badges: "unlimited", "no expiry", "above balance", "estimated" (inferred from history, not verified). Riskiest first, then largest.
- **Actions**:
  - "Revoke": a ConfirmationModal ("Revoke {spender}'s approval to spend … The ledger charges {fee}"), then `icrc2_approve` with amount 0 and `expected_allowance` = the shown amount.
  - "Change": "New allowance ({SYM}; 0 revokes)"; Expiry options "Keep current expiry", "In 1 hour", "In 1 day", "In 7 days", "In 30 days", "Pick a date…" (datetime, at least 1 min ahead), "Never (not recommended)". Then "Review" and a confirm ("Let {spender} spend up to … until … This replaces the current approval…") → `icrc2_approve`. "Cancel".
  - Results: "Revoked." or "Approval changed." Readable errors, e.g. "The allowance changed meanwhile…", "Not enough balance to pay the ledger fee…", "The expiry is in the past."
  - After either action the approvals reload with force.
- **Shared building blocks**: ConfirmationModal (`doAwait`), PrincipalDisplay, DenominationContext, `setApproval`, `approvalsAgent`.
- **Redesign notes**: there is no "create approval" UI. Approvals are only made inside flows: wrap, SneedPad, Studio index panel, DEX swaps, CyclePay, ckMinters, Sneed Monitor, Notifications (all through `approveAndRecord`, which also feeds the known-spender list).

## Approval discovery engine and known-spender labels (utils/approvals.js, approvalSpenders.js, hooks/useApprovals.js)

- **Routes & params**: infrastructure behind the approvals capabilities above. Discovery uses anonymous queries; changes are signed by the user.
- **Purpose / information exposed**: finds an account's live allowances per ledger, best source first:
  1. ICRC-103 `icrc103_get_allowances` (complete).
  2. An index: index-ng found through ICRC-106 or the SNS index hint, or the ICP index. It reads up to 3000 transactions and confirms each spender with `icrc2_allowance`. On ICP the spender is known only by account ID; matching uses hashed candidates, otherwise the allowance is estimated as approved minus spent.
  3. Known spenders: Sneed canisters, services, up to 12 ICPSwap pools from the swap pool cache and the user's positions, and spenders this browser approved.
- Each result carries `method`, `complete` and `notes`. A risk judgement (`approvalRisk`) is attached; at most 3 discoveries run at once; results are cached.
- **Gating/state**: localStorage `sneed_approvals_cache_v1`, `sneed_approval_spenders_v1` (≤ 50 per ledger), `icpswap_pool_cache`; event `sneed-approvals-changed` marks a ledger stale. Max age 30 min.
- **Redesign notes**: coverage is honest but technical (shown only behind "How these were found"). Approvals made outside Sneed Hub, on ledgers with no ICRC-103 and no index, are invisible.

## Approvals per pocket (Pockets page)

- **Routes & params**: `/wallet/pockets`, PocketCard panel "approvals". Login required.
- **Purpose**: the same list for one subaccount pocket, over that pocket's known ledgers.
- **Actions**: "Check again"; Revoke/Change through ApprovalsList. The empty message is "No approvals from this pocket." or "No tokens known in this pocket yet."
- **Redesign notes**: the only place subaccount approvals are visible. The Wallet tab says "Your main account only".

## Subaccount byte editor (components/SubaccountEditor.jsx)

- **Routes & params**: embedded. `/tools/account` (account/ID converter, Subaccount field) and `/wallet/pockets` (create pocket: "Exact subaccount (byte editor)" checkbox → "Create with this subaccount"; default format Number). Pockets also uses `subaccountHint`.
- **Purpose**: build a 32-byte subaccount from human input.
- **Information shown**: format chips with descriptions; live hex preview; hint ("Numbered pocket #n", "Principal-derived: some dapps use this as {principal}'s deposit address (Sneed Vault index i)", "Principal-derived ({principal}, then index n)"); "Not a valid subaccount for this format".
- **Actions**: format choices:
  - "Hex String" (right-aligned)
  - "Number" (big-endian; pocket #n = n)
  - "Text" (UTF-8, left-aligned)
  - "Byte Array" ("Start from left/right")
  - "Principal" ([len][principal][index]) with an Index sub-format Number/Hex/Bytes and "N bytes available"
  - "Sneed Vault layout" (owner principal + Index 0–65,535)
- **Redesign notes**: not reused by the subaccount fields in SendTokenModal (its own hex/bytes/principal input), TransferTokenLockModal and SendLiquidityPositionModal (raw 64-hex input), or PrincipalInput's subaccount dialog. That makes four subaccount-entry UIs with different formats and alignment rules.

## Transfer ownership of a token lock (TransferTokenLockModal)

- **Routes & params**: modal on `/wallet`, from the TokenCard lock row "Transfer Ownership" (Wallet `openTransferTokenLockModal`, `ensureSession`). Login required.
- **Purpose**: give a Sneed Lock token lock (an ICRC-7 NFT) to another account. The lock stays active.
- **Information shown**:
  - "🔄 Transfer {SYM} Lock #{id}"; Amount; Expires (date or LockUntil for until-dissolved).
  - "Transaction Fee" (1 or 2 fees). When the free Sneed Lock balance is below one fee: "(2 tx fees)" and "1 tx fee will be sent from your wallet first."
  - "⚠️ Wallet Compatibility Warning" (only ICRC-7-capable wallets).
- **Actions**:
  - Recipient: PrincipalInput "Enter recipient principal" plus an optional subaccount (64 hex).
  - "Transfer" validates the principal, the subaccount format, and whether the wallet can pay 2 fees when a pre-send is needed. Then confirm.
  - The call sequence:
    1. Optional `icrc1_transfer` of 1 fee into the Sneed Lock subaccount, polling up to 10 s for arrival.
    2. ICRC-7 NFT transfer (`sendLockNft`).
    3. Fallback `sneed_lock.transfer_token_lock_ownership` (no subaccount: "This Sneed Lock cannot send to a subaccount yet.").
  - "Cancel".
- **Redesign notes**: the modal ignores Wallet's "no fee when sending to yourself" rule, and the confirm text omits the subaccount. Selling a lock on Sneedex is a separate path.

## Send or transfer a liquidity position (SendLiquidityPositionModal)

- **Routes & params**: modal. `/wallet` (PositionCard "Send" / compact send icon / "Transfer"; PositionCardModal) and the header quick wallet (PositionCardModal "Send"). Login required.
- **Purpose**: give an ICPSwap LP position to another principal. Two modes:
  - Send: unlocked positions. Direct-owned → ICPSwap `transferPosition`; held by Sneed Lock but unlocked → `sneed_lock.transfer_position`.
  - Transfer (`isBackendTransfer`): locked positions keep their lock and are sent as an ICRC-7 NFT (`positionNftId` + `sendLockNft`, fallback `transfer_position_ownership`).
- **Information shown**: "📤 Send" or "🔄 Transfer {A/B} Position #{id}"; for locked positions the "⚠️ Wallet Compatibility Warning" (ICRC-7, possible permanent loss).
- **Actions**:
  - Recipient: PrincipalInput, plus an optional 64-hex subaccount (locked only).
  - "Send"/"Transfer" refuses Sneed Lock itself, the anonymous principal and any ICPSwap pool canister (factory pool index). Then a confirm ("…This will transfer backend ownership while keeping the position locked.").
  - "Cancel".
- **Redesign notes**: likely broken in the quick wallet. PrincipalBox calls `walletContext.sendLiquidityPosition`, which WalletContext does not define, so the handler logs an error and returns, and the modal closes as if the send succeeded. "Send" vs "Transfer" naming depends on lock state; "Sell on Sneedex" sits beside it on the card.

## Add liquidity to a locked position (AddLiquidityModal)

- **Routes & params**: modal on `/wallet`. PositionCard "Add Liquidity" appears for a position held by Sneed Lock with an active lock (expiry in the future); also from PositionCardModal on /wallet. Login required.
- **Purpose**: top up a locked ICPSwap position in its existing range without unlocking it.
- **Information shown**:
  - "+ Add Liquidity to Locked Position", with the explainer about funding order and the returned remainder.
  - In range / out of range state ("Out of range: only {SYM} is needed right now").
  - Per token: "wallet: X · backend wallet: Y (+Z locked)", read live from the ledgers and Sneed Lock (`get_summed_locks`).
  - "Fees for this configuration" per token (2 or 3 fees; "all from your backend wallet" / "X moved from your wallet, the rest from your backend wallet"), plus "Plus one fee per token if the pool returns an unused remainder."
  - Result: "Added:", "Returned to backend wallet:", "The pool did not add liquidity: …".
- **Actions**:
  - Amount inputs are linked by the position ratio. "MAX — add as much as both wallets allow".
  - "Add Liquidity": for any shortfall, `icrc1_transfer` top-ups to the Sneed Lock subaccount; then `sneed_lock.add_liquidity_to_locked_position`; the position and both tokens refresh. Close button on the result view.
- **Redesign notes**: Wallet only (the quick wallet does not wire it). It is the only place the "backend wallet" balances of both tokens show side by side.

## Lock tokens / lock a position (LockModal, LockPositionModal)

- LockModal: create a Sneed Lock token lock. It deposits any shortfall to the Sneed Lock subaccount, then `create_lock` with an expiry, for an ICP lock fee discounted for premium users. Opened from TokenCard "Lock" on /wallet and from the quick wallet's TokenCardModal. Covered in depth by the Sneed Lock slice.
- LockPositionModal: lock an ICPSwap position. Steps: `claim_position` + `transferPosition` to Sneed Lock + `create_position_lock`, or `update_position_lock` to extend. Premium-discounted fee. Opened from the PositionCard lock button on /wallet.

**Part C: NFTs**

## NFT wallet: my collections and the NFTs I hold (`/nft`)

- **Routes & params**: `/nft` (no params). **Auth: login required.** Logged out you get a "View Your NFT Collection" prompt with a "Connect Wallet" button (calls `login`). Nothing is premium-gated.
- **Purpose**: list every collection saved in this browser (ICRC-7 and EXT) with the NFTs the user's **default account** holds in each.
- **Information shown**
  - Page header "🎨 My NFTs" and the line "Your ICRC-7 and EXT NFT collections".
  - Progress while loading: "Connecting your wallet…", then "Loaded X of Y collections…". Each collection still loading gets a placeholder with its logo, name or canister id, symbol and current stage. Stages, in order: "Connecting…", "Reading collection info…", "Looking for your tokens…", "Found N of yours…", "Reading metadata x / y…", "Fetching token documents…", "Checking standards…", "Reading approvals…". For EXT: "Looking for your tokens…", "Reading N cards…".
  - A collection that fails to load shows "Could not load: {error}" with **"Retry"** and **"Remove"** buttons.
  - Collection card header: logo (🎨 fallback), name, an `EXT` badge for EXT collections, symbol (ICRC-7 only), "N NFTs" (how many the user holds), "Supply: X" (total supply).
  - Collection card body (expanded by default; clicking the header collapses it): description, "Canister ID" with a 📋 copy button, the collection-approvals panel (ICRC-37 collections, see the approvals section), the admin mint panel (admins only), the EXT swap notice (Diamond Club EXT only), then a grid of NFT cards. With no tokens it says "You own none of the {supply} tokens in this collection".
  - Empty state: "No NFT Collections Yet", "Add an ICRC-7 or EXT NFT collection to view and manage your NFTs", and a "+ Add Your First Collection" button.
- **Actions**
  - "+ Add Collection" (header) opens AddNftCollectionModal (see the next section).
  - 🔄 "Refresh collection" (card header) reloads that one collection. ICRC-7: `loadNftCollection`. EXT: `loadExtCollection` with `refresh:true`. The admin flag is kept.
  - "Remove Collection" opens a ConfirmDialog: "Remove {name}?" / "This only removes it from your view. Your NFTs stay in your wallet…", with the button "Remove" (danger). It changes localStorage only, with no canister call.
  - "Retry" / "Remove" on a collection that failed to load.
  - Per-NFT actions come from NftCard: View, Sell on Sneedex, Swap to ICRC-7, Send NFT, Approve, Revoke (see the sections below).
- **Load path (canister calls)**
  - **ICRC-7**: `icrc7_name`, `icrc7_symbol`, `icrc7_logo`, `icrc7_total_supply`, `icrc7_description`, `icrc7_max_take_value`, then `icrc7_tokens_of(account{owner:me, subaccount:[]})` paged (take ≤200, up to 200 pages), then `icrc7_token_metadata` in batches of 50.
    - It also HTTP-fetches each token's ICRC-97 JSON document (`icrc97:metadata`; `ipfs://` becomes `ipfs.io`).
    - Standards: `icrc10_supported_standards`, or probes `icrc37_max_approvals_per_token_or_collection` when ICRC-10 is missing.
    - On ICRC-37 collections it reads `icrc37_get_token_approvals` and `icrc37_get_collection_approvals`.
    - It calls the non-standard `get_admins` to detect admins.
  - **EXT**: `tokens_ext(accountId)`, falling back to `tokens`. Then `getTokensByIds`, falling back to `metadata(tid)` per token. The image route is `https://<canister>.raw.icp0.io/?tokenid=…`; known Diamond Club cards use `https://v7b6c-…icp0.io/card/{n}.jpg`.
- **Entry points & cross-links**
  - Header "Sneed Me" menu → "NFTs" (`/nft`).
  - Links into `/nft` from elsewhere:
    - NftBrowser "My NFT wallet" and "Add to my wallet" (both navigate here).
    - NftItem "Open in wallet" / "Add collection to wallet".
    - NftAvatarPicker empty state ("NFT wallet").
    - DiamondClub ("They are in your NFT wallet too").
    - Help pages: HelpWallet `#nfts`, HelpNames, HelpSneedlock ("add the Sneed Lock collection on the NFTs page").
  - Links out: `/nft_item?collection=&id=` (View), `/sneedex_create?nft=…` (Sell), `/diamond-club` (EXT swap).
- **Shared building blocks**: NftCollectionCard, NftCard, SendNftModal, AddNftCollectionModal, ConfirmDialog, PrincipalInput (used in mint and approve), NftModel, the Soulbound badge, and NftUtils / ExtUtils.
- **Gating/state**
  - **localStorage**:
    - `nft_collections`: ICRC-7 ids the user added.
    - `nft_collections_hidden`: built-in ICRC-7 collections the user removed.
    - `ext_nft_collections`: added EXT collections `{canisterId, standard, name?}`, plus `{hidden:true}` entries for removed built-ins.
  - **Built-ins**, shown to everyone unless hidden:
    - ICRC-7 "Sneed Diamond Club" `v7b6c-niaaa-aaaac-bgiiq-cai` (KNOWN_NFT_COLLECTIONS).
    - EXT "Sneed Diamond Club" `gjv36-laaaa-aaaah-adyza-cai` (KNOWN_EXT_COLLECTIONS: supply 1000, imageHost, swapPage `/diamond-club`).
  - EXT ids are kept under their own key on purpose, because other pages read `nft_collections` as ICRC-7.
  - Collections are listed in saved order: EXT first, then ICRC-7. Each one loads independently in parallel.
- **Redesign notes**
  - Only the **default subaccount** is scanned (`subaccount: []`). NFTs held in subaccounts are invisible here, although the NFT browser marks them "YOURS".
  - There is no search, sort, filter or total count across collections, and the list is per-browser (not synced to an account).
  - The Sneed Lock collection is not built in, even though every lock is an NFT. Help tells users to add it by hand; SneedexCreate lists it first.
  - Dead code in `Nft.jsx`: an unused `app_sneeddao_backend` import, and an `error` state that is never set to a message.
  - "Send NFT" only appears after the user expands a card.

## Add an NFT collection by canister id (AddNftCollectionModal)

- **Routes & params**: a modal on `/nft` only ("+ Add Collection" / "+ Add Your First Collection"). Login required. (`/nfts` "Add to my wallet" and `/nft_item` "Add collection to (my) wallet" skip this modal and write to localStorage directly.)
- **Purpose**: check that a canister is an ICRC-7 or EXT collection, preview it, then save it.
- **Information shown**
  - Input: "Canister ID" (PrincipalInput, so it can also search by name; placeholder "Collection canister id or name"). Invalid text shows "Invalid canister ID format".
  - Preview: logo, name, symbol, description, "Total Supply" ("unknown" if null).
  - Standards line:
    - ICRC-7: "Standards: ICRC-7[, ICRC-37 approvals]". If ICRC-10 is missing it adds a warning that marketplaces relying on ICRC-10 won't see the collection.
    - EXT: "Standard: EXT. EXT collections have no on-chain name…".
  - Badge: "✓ ICRC-7 Compatible" or "✓ EXT collection".
  - Error when neither standard answers: "This canister answers neither the ICRC-7 queries (icrc7_name, icrc7_total_supply, icrc7_tokens) nor the EXT ones (extensions, bearer)…".
- **Actions**
  - "Preview Collection":
    - A known EXT id goes straight to `fetchExtCollectionInfo`. That reads `getRegistry`, so supply is the registry length.
    - Anything else goes to `fetchCollectionInfo`: `icrc7_*` metadata plus `detectStandards`.
    - If that is not ICRC-7, it calls `isExtCollection` (`extensions()` or `bearer(tid 0)`).
  - "Add Collection" saves the id to localStorage (`addNftCollection` / `addExtCollection`). Adding a hidden built-in un-hides it.
  - "Cancel" closes the modal.
- **Redesign notes**: three separate "add to wallet" paths exist (this modal, the browser, the item page). Only the modal previews the collection.

## Owned-NFT card: inline details (NftCard inside the wallet)

- **Routes & params**: inside each collection card on `/nft`. Login required.
- **Purpose**: a compact tile that expands to show everything known about a token, plus the actions on it.
- **Information shown**
  - Collapsed:
    - Image, with an EXT fallback to the raw image route. Without an image it shows "no image" / "image did not load", with the URL in a tooltip.
    - Name, plus a SOULBOUND badge if the metadata has `bound_to`.
    - "{SYMBOL} #{id}", or for EXT "EXT token index N".
    - A ▶/▼ expand marker.
  - Expanded:
    - Description.
    - ICRC-37 "Approvals" list: spender principal and "until {date|never}"; "Nobody else may move this token." when empty.
    - Media: a 3D model (`.glb/.gltf` animation_url), an audio player, or a video player.
    - "Attributes" chips (from ICRC-97 JSON or the inline `attributes`).
    - "Links": every http/ipfs URL in the metadata, plus the ICRC-97 image.
    - An ICRC-97 read-error notice ("ICRC-97 document at {url} could not be read").
    - "Metadata" key/value list (image/url/logo keys left out; values formatted, blobs as hex or size).
- **Actions**
  - Always visible:
    - "View" → `/nft_item?collection=&id=`.
    - "Sell on Sneedex" → `/sneedex_create?nft={collection}:{id}[&standard=EXT]`.
    - "Swap to ICRC-7" (Diamond Club EXT only) → `/diamond-club`.
  - Expanded:
    - "Approve a spender" / "Revoke" (see the approvals section).
    - "Send NFT" opens SendNftModal (or the EXT variant).
  - Clicking the image or title toggles expand.
- **Shared building blocks**: the getters in NftUtils (`getNftImage`, `getNftName`, `getNftDescription`, `getNftAttributes`, `getNftLinks`, `getNftMedia`, `formatNftMetadata`), NftModel, the Soulbound badge, PrincipalInput, `extSwapPage`.
- **Redesign notes**: this is the third renderer of the same token facts, next to NftAssetDetails and the NftItem page. The redesign could keep just one.

## Send an ICRC-7 NFT (SendNftModal)

- **Routes & params**: a modal from NftCard "Send NFT" on `/nft`. Login required, and the user must hold the token.
- **Purpose**: transfer one ICRC-7 token to any account.
- **Information shown**: preview with image, name, "{symbol} #{id}" and collection name. Validation messages: "Invalid principal format", "Subaccount must be 64 hex characters (32 bytes)".
- **Actions**
  - Fields: "Recipient Principal *" (a **plain text input**, no name lookup), "Subaccount (optional)" (64 hex; "Leave empty for default subaccount"), "Memo (optional)" (max 32 chars, sent as UTF-8 bytes).
  - "Cancel".
  - "Send NFT" / "Sending..." calls `icrc7_transfer([{from_subaccount:[], to, token_id, memo, created_at_time:now}])`.
    - Errors are mapped: Token does not exist, Invalid recipient, Unauthorized…, Transaction too old, Created in future, Duplicate (…), GenericError message.
  - Success: a ConfirmDialog "NFT sent": "{name} now belongs to its new owner. Transaction index: N." with the button "Done". The collection then refreshes.
- **Fees/confirmations**: no review step and no fee shown. A Sneed Lock token-lock NFT sent this way pays a ledger fee from free balance or gas. That isn't explained here, and the error code 7 ("needs gas") isn't handled specially.
- **Redesign notes**: inconsistent with the EXT send dialog, which has name lookup, a review step and an irreversibility warning. It sends from the default subaccount only.

## Send an EXT NFT (ExtSendNftModal, inside SendNftModal)

- **Routes & params**: the same modal on `/nft`, chosen when `collection.standard === 'EXT'`. Login required.
- **Purpose**: send one EXT token to a principal or to an account id.
- **Information shown**
  - An `EXT` badge, image, name, collection name, "token index N".
  - Tabs: "To a principal" / "To an account id".
  - Principal mode shows the derived default account id. Account-id mode checks for "64 hex characters with a matching checksum".
  - Self-send warning: "That is the account that already holds it."
  - Review step: "Send {name} to:" with PrincipalDisplay, the principal text and the account id, plus the warning "A transfer cannot be undone…".
- **Actions**
  - "Review" (disabled when invalid or self-send), "Back", "Cancel".
  - "Confirm and send" / "Sending…" calls `transferExt`:
    1. `bearer(tid)` must equal my default account id.
    2. Then `transfer({from:{principal}, to:{principal|address}, token:tid, amount:1, memo:[], notify:false})`.
    3. Then `bearer` again to confirm the move.
  - Success dialog "NFT sent": "…The collection now shows account {aid} as its holder."
- **Redesign notes**: anonymous principals are refused. Holdings are read from the default account only.

## ICRC-37 approvals (per token and whole collection)

- **Routes & params**
  - Managed on `/nft`, in the NftCard expanded section and the NftCollectionCard panel. Login required; the user must own the NFTs.
  - Read-only on `/nft_item` for the owner.
  - Shown only when the collection supports ICRC-37 (announced via ICRC-10, or probed).
- **Purpose**: let a marketplace or another principal move one token, or any token in the collection, without taking custody.
- **Information shown**
  - Per token: each approved spender principal (`spender.owner`) and "until {localized date | never}"; "Nobody else may move this token." when empty.
  - Collection panel "Collection approvals (ICRC-37)": an explanation ("A collection approval lets a spender move any token you hold here, which is how a marketplace lists without taking custody. Per-token approvals are on each card."), then a list of spender and expiry, or "None.".
  - Item page "Approvals (ICRC-37)": "token: {PrincipalDisplay}" and "whole collection: {PrincipalDisplay}" rows (no expiry shown). Empty state: "No spender is approved for this token. Approvals are managed in the NFT wallet."
- **Actions**
  - "Approve a spender" opens a form: spender PrincipalInput ("Spender principal (a marketplace, a wallet)") and "Expires in days (empty: never)", with "Approve"/"Approving…" and "Cancel". It calls `icrc37_approve_tokens` with spender subaccount always `[]`, `from_subaccount: []`, `expires_at` in ns. Note: "An approved spender can transfer this token out of your account until the approval expires, is revoked, or the token moves."
  - "Approve a spender for the whole collection": the same form, calling `icrc37_approve_collection`.
  - "Revoke" (per spender): `icrc37_revoke_token_approvals` / `icrc37_revoke_collection_approvals`. Errors go to the "Revoke failed" dialog.
  - After each change the collection reloads.
- **Fees/confirmations**: no confirmations and no fee shown.
- **Redesign notes**
  - Approval management lives only inside the expanded wallet card and collection card. The item page is read-only and points back to the wallet.
  - Sneedex create also calls `approveToken` (ICRC-37) as part of escrow; that belongs to the Sneedex slice.

## Mint an NFT into a collection (admin only)

- **Routes & params**: `/nft`, inside a collection card. Shown only if `get_admins()` (a non-standard method used by "our demo" and "the coming launchpad") includes my principal. EXT collections never qualify.
- **Purpose**: let a collection admin create a new token.
- **Information shown**: a toggle "▶ Mint (you are an admin of this collection)", then "Minted #{id}" on success or the error text.
- **Actions**
  - Form fields:
    - "Recipient": PrincipalInput, defaults to me.
    - "Name" and "Description".
    - "Image URL (https, ipfs, or a data URL)", or an image file of up to 300 kB, stored on-chain as a data URL. Larger files get the error "image files up to 300 kB…".
    - "Attributes": rows of "Trait"/"Value" with "×", plus "+ attribute".
  - "Mint" / "Minting…" calls `mint(Account{owner, subaccount:[]}, [name, description, image, attributes(Array of Map)])` and then refreshes the collection.
- **Redesign notes**: a hidden capability tied to a custom interface. There is no launchpad page yet.

## Browse any collection (`/nfts`, NftBrowser)

- **Routes & params**: `/nfts?collection=<canister id>`. **Public** (anonymous agent). Being logged in adds "YOURS" marks, "only mine" and "Add to my wallet".
- **Purpose**: explore any ICRC-7 or EXT collection, owned or not. Each token links to its own page.
- **Information shown**
  - Title "NFTs" and "Browse any ICRC-7 or EXT collection, owned or not. Every token has its own page."
  - Collection picker:
    - PrincipalInput ("Collection canister id or name", search defaults to canisters).
    - Chips "In your wallet": saved ICRC-7 and EXT collections, built-ins included.
    - Chips "Recently browsed": localStorage, last 12, saved ones excluded.
    - Each chip shows logo, name, and symbol or an EXT badge.
  - Collection header:
    - Logo, name (or "Reading collection…" / "Unknown collection"), "(SYMBOL)" or EXT badge.
    - Canister id, "supply N", and a standards string such as "ICRC-7 · ICRC-37 · ICRC-10", or "standards unknown".
    - A "dashboard" link (dashboard.internetcomputer.org/canister/…) and the description.
    - Errors: "Could not read the collection: …" and "That is not a canister id.".
  - Status line: "N tokens listed[ so far]", "only mine (n)" (logged in), "loading…".
  - Token grid (48 per page): image (lazy), "YOURS" badge, name, "#id". "No tokens listed in this collection." when empty.
- **Actions**
  - "Browse" or Enter (or picking from PrincipalInput) navigates to `/nfts?collection=…` and pushes it to the recent list.
  - "My NFT wallet" → `/nft` (logged in).
  - "Add to my wallet" (logged in and not saved): saves to localStorage and navigates to `/nft`.
  - "Refresh": clears and reloads the first page. For EXT it also refreshes `getRegistry`.
  - "only mine" checkbox: filters **only the pages loaded so far**.
  - "Go to #" with a "Go" button: jumps to `/nft_item?collection=&id=`.
  - "Load more" / "Loading…" fetches the next page.
  - Tile click → `/nft_item`.
  - Calls:
    - ICRC-7: `icrc7_max_take_value`, `icrc7_tokens(prev,take)`, then `icrc7_token_metadata` and `icrc7_owner_of` in batches of 12, plus ICRC-97 fetch.
    - EXT: `getRegistry` sliced client-side, then `getTokensByIds`/`metadata`.
- **Entry points & cross-links**
  - Header "Sneed Hub" menu → "NFTs" (`/nfts`). `/nft_item` highlights this item (legacy mapping).
  - Also reached from: NftItem "← {collection}" and "All collections"; NftAssetTitle / NftAssetDetails "Browse collection" (Sneedex offers, feed); HelpTools `#nfts`; HelpWallet.
- **Gating/state**
  - localStorage `nft_browsed_collections`.
  - Module caches in NftAssetView for the session: `collectionCache`, `tokenCache`, `standardCache`. ExtUtils keeps `registryCache` and `metaCache`.
- **Redesign notes**
  - The header has **two menu items both labelled "NFTs"**: Hub → `/nfts` (browser) and Me → `/nft` (wallet). Confusing.
  - "Mine" here means owner principal == me (any subaccount), or EXT default account id. The wallet only lists the default subaccount.
  - The "ICRC-3" standard label is dead: `detectStandards` never returns an `icrc3` key.
  - There is no collection directory or discovery: the user must know the id or name.

## NFT item page: everything about one token (`/nft_item`)

- **Routes & params**: `/nft_item?collection=<canister>&id=<token id (EXT: index)>`. **Public.** Owner-only extras are covered in the next section. A missing or invalid param shows "Give a collection canister id and a token id.".
- **Purpose**: a permanent page per token, like `/transaction`, showing all facts the collection returns about it.
- **Information shown**
  - Breadcrumb: "← {collection name}" (→ `/nfts?collection=`) · "All collections" (→ `/nfts`).
  - Media panel:
    - A 3D model (NftModel, with the image as poster) or the image (EXT falls back to the raw route).
    - Placeholders: "This token does not exist in the collection" / "The picture could not be loaded" / "No picture in the metadata".
    - Audio or video player for `animation_url`, and an "open picture" link.
  - Title card: name, collection logo and link, EXT badge or symbol, "#id", a refresh button ("Re-read from the collection"), the description, and SoulboundNote (see the character section).
  - "Sneed Lock" card: only when collection == sneed_lock (see the Sneed Lock section).
  - "Owner" card:
    - ICRC-7: PrincipalDisplay (→ `/principal?id=`), plus the subaccount hex.
    - EXT: "account {account id}".
    - A "YOURS" badge when it is mine.
    - Unknown: "Unknown (the collection did not answer icrc7_owner_of, or the token does not exist)" or the EXT registry equivalent.
    - CarriedBy: "Carried by the character {link}", when the owner account is a character NFT's own account.
  - "Gas (Sneed Lock token lock)" card: Sneed Lock token locks only.
  - "Approvals (ICRC-37)": owner only, read-only (see the approvals section).
  - "Attributes" grid (trait_type → value).
  - CharacterPanel: when the metadata has `wu:stats` / `wu:items`.
  - "Links": "ICRC-97 document" and every metadata URL.
  - "Metadata (N fields)" with a "Show/Hide" toggle: raw key/value pairs (values over 600 characters truncated) plus the pretty-printed ICRC-97 JSON document.
  - Footer: collection id, "collection on the dashboard" link, EXT "token identifier {tid}", and the standards list.
- **Actions (all users)**
  - Refresh (re-reads the token with `refresh:true`; the collection info stays cached).
  - "Add collection to my wallet": logged in, not the owner, and not saved yet. Saves to localStorage, with no navigation.
  - External links.
- **Calls**: anonymous `icrc7_token_metadata` and `icrc7_owner_of` (or EXT registry plus metadata), the ICRC-97 fetch, collection info via `loadCollectionInfoCached`. For the owner on ICRC-37 collections, it also calls `icrc37_get_token_approvals` and `icrc37_get_collection_approvals` with the user's identity.
- **Entry points**
  - NftCard "View", NftBrowser tiles and "Go to #", NftAssetIcon/Title/Details links (Sneedex offers, feed).
  - CharacterPanel item tiles, CarriedBy and SoulboundNote links.
  - DiamondClub card tiles (`/nft_item?collection=SNEED_NFT_ID&id=n`).
- **Gating/state**: on mobile the two-column grid collapses to one column at ≤720px. No localStorage writes besides the add-to-wallet keys.
- **Redesign notes**: there is **no "Send" on the item page**; sending is only possible from the `/nft` wallet card. Approvals are view-only here.

## Owner actions on the NFT item page

- **Routes & params**: `/nft_item?...`. Shown only when `mine`: ICRC-7 owner principal == me (any subaccount), or EXT holder == my default account id.
- **Purpose**: quick owner shortcuts from a token's page.
- **Actions**
  - "Sell on Sneedex" → `/sneedex_create?nft={collection}:{id}[&standard=EXT]`.
  - "Open in wallet" (already saved) → `/nft`. "Add collection to wallet" saves to localStorage, then goes to `/nft`.
  - "Use as my avatar" / "Setting…" / "Your avatar now":
    - Requires `namingServiceEnabled` and an image.
    - Calls `set_my_nft_avatar` or `set_my_ext_nft_avatar` (sneed_pid_names) with the collection and token id, then updates NamingContext `setKnownAvatar`.
    - Errors show inline.
    - Tooltip: "Show this NFT next to your name while you own it".
  - "Use this look as the NFT's 3D model": in CharacterPanel, owner only (see the character section).
- **Redesign notes**: "mine" ignores the subaccount here, but approvals are read with `subaccount: []`.

## Sneed Lock locks as NFTs (lock summary, gas, send-as-NFT)

- **Routes & params**
  - `/nft_item?collection=<sneed_lock canister>&id=<lock id>`: public.
  - `/lock/:id`, via LockNftPanel in `Lock.jsx`: public view, holder-only send.
  - `/wallet` lock send paths: login required.
- **Purpose**: every Sneed Lock token lock and LP position lock is an ICRC-7 token in the sneed_lock canister. These views show what the lock holds and let the holder move it.
- **Information shown**
  - On `/nft_item`:
    - **SneedLockNftSummary** card "Sneed Lock":
      - Token lock: amount and symbol. LP position lock: "{T0}/{T1} LP position #id" with token amounts.
      - "Worth {USD}", or "value unknown", in the user's denomination.
      - Lock status (locked or expired, dissolve delay).
      - "Open Sneed Lock #id" → `/lock/:id`.
      - "Details" toggle: Lock #, Ledger, Amount, Pool, Position link (`/position?swap=&id=`), "In the position" (with "(out of range)"), "Unclaimed fees", "Value per token", "Dissolve delay".
    - **LockGas** card "Gas (Sneed Lock token lock)", token locks only:
      - Text: "Gas {balance} {symbol} · about N transfers".
      - Explanation: a transfer to another principal costs one ledger fee of the locked token; moves between subaccounts of the same principal are free; LP position locks move without a fee.
  - LockNftPanel on `/lock/:id`:
    - "NFT #{tid}" with its `icrc7:image`.
    - Explanation that the lock or position is an ICRC-7 NFT in the Sneed Lock collection (canister id shown), and that ICRC-7/ICRC-37 wallets and marketplaces can show, send and list it.
    - "Held by" principal, "Subaccount", and LockGas for token locks.
    - Renders nothing when the lock is not an NFT.
- **Actions**
  - "Add gas" opens AddGasDialog:
    - Presets "1/5/20 transfers" or "Or an amount in {symbol}".
    - Shows the current balance, the transfers it covers and the wallet balance.
    - "Add gas" / "Adding…" makes an `icrc1_transfer` from the user's wallet to the lock's gas account.
    - Logged out, it shows the gas account and subaccount so the user can pay manually.
    - Anyone can add gas.
  - LockNftPanel, holder only:
    - "Send the NFT": a recipient PrincipalInput and "Recipient subaccount (optional, 64 hex characters)".
    - "Send" / "Sending…" calls sneed_lock `icrc7_transfer` from the holder's actual subaccount.
    - Results: "Sent (block N)."; or "This lock needs gas to move: add some below (anyone can)…" (GenericError code 7); or mapped errors.
    - Fee notes: token lock "No Sneed fee; the ledger fee comes from your free balance in Sneed Lock, else from the lock's gas"; position "Free: no Sneed fee."
  - Wallet (`Wallet.jsx`): sending a liquidity position or token lock uses `positionNftId` (`get_position_nft`) and `sendLockNft` (an `icrc7_transfer` that supports subaccounts), falling back to native calls when sneed_lock has no ICRC-7.
  - Sneedex create `?lock=<id>` pre-fills an NFT asset in the Sneed Lock collection.
- **Shared building blocks**: SneedLockAssetView (`useSneedLockNft`, SneedLockNftSummary, also used in Sneedex offers), `sneedLockNftModel.js` (value math, also used by Hub, Position, SneedexOffers), LockGas, `lockNft.js` (`parseSubaccountHex`, also used by SendLiquidityPositionModal and TransferTokenLockModal).
- **Redesign notes**
  - The same lock NFT can be sent from three places with different UX: the `/nft` wallet card (generic, with no gas handling), LockNftPanel on `/lock/:id`, and Wallet lock cards.
  - The Sneed Lock collection is not in the NFT wallet by default.

## Game-character NFTs and soulbound items (CharacterPanel / Soulbound)

- **Routes & params**: `/nft_item`. Shown when the token's metadata has `wu:stats` or `wu:items` (principal arrays), or `bound_to`. Public; one action is owner-only.
- **Purpose**: for character NFTs following the "wu" convention (wu_chars and compatible collections), show live game data and the items linked to the character.
- **Information shown**
  - Per game root in `wu:stats`, a card "In {game name}":
    - The game's "site" link and root id. "The game did not answer." / "No facts about this character there yet." when there is no data.
    - A facts grid (labels, values, events) using the game's readable names.
    - Fact families (quests, reputation, skills…) as collapsible lists with counts. Events show "N×, last YYYY-MM-DD". `questat:` values show as dates.
    - The game's 3D "look" (a glTF document) in NftModel.
    - An inventory list (parsed from the XML doc): bags, worn slots, keychain, item names, stack ×N, "(soulbound)".
  - "Its things (NFTs)": tokens in each `wu:items` collection, grouped as "Carried by the character" (the NFT's own account) and "Kept by the owner for this character ({game})" (an owner subaccount = sha256("wu-char/"+root+"/"+charKey)). Tiles link to `/nft_item` and carry 3D and SOULBOUND marks.
  - Owner card "Carried by the character {name}": appears when this token's owner account is a character NFT's own account (`account_of(id)` on the owner canister).
  - SoulboundNote: "Soulbound — Bound to {character link | name}. It can be sent or sold like any NFT, but only that character can equip it." The character is resolved via `nft_of` on the game root.
  - SOULBOUND badge on NftCard, NftAssetDetails and CharacterPanel tiles.
- **Actions**: "Use this look as the NFT's 3D model". Owner only; calls `set_model(tokenId, [glb bytes])` on the collection. Result: "This is the NFT's 3D model now (reload to see it above)." or the error.
- **Calls**: anonymous `showcase`, `character_of`, then the shard's `get` / `get_doc` (gzip-decoded), `icrc7_tokens_of`, `icrc7_token_metadata`, `account_of`, `nft_of`.
- **Redesign notes**: a hidden, niche feature that only appears on token pages with this metadata. It uses custom canister interfaces.

## Use an NFT as profile avatar (NftAvatarPicker on `/me`, plus app-wide display)

- **Routes & params**: `/me` profile avatar editor. Login required, and gated by `namingServiceEnabled` (naming canisters exist on this network). The same action exists on `/nft_item` ("Use as my avatar").
- **Purpose**: show an NFT the user owns as their avatar everywhere.
- **Information shown**
  - "Use an NFT I own…" / "Hide my NFTs" toggle.
  - While loading: "Looking through your NFT collections…".
  - A grid per collection (name, EXT tag) of owned NFT thumbnails, with the chosen one highlighted. A thumbnail without an image is disabled ("no picture").
  - Empty state: "No NFTs found in the collections of your NFT wallet. Add a collection there, or open any NFT you own and choose "Use as my avatar"" (link to `/nft`).
  - Current avatar: "Showing an NFT (token #N). Your picture above shows if you sell it."
- **Actions**
  - Clicking a thumbnail calls `set_my_nft_avatar` (ICRC-7) or `set_my_ext_nft_avatar` (EXT) on sneed_pid_names. The canister checks ownership.
  - "Stop showing the NFT" / "Stopping…" calls `set_my_nft_avatar(null)`.
- **Display**: `NamingService.getAvatars` checks ownership again whenever an avatar is shown (`resolveNftAvatars`: ICRC-7 `icrc7_owner_of` == owner; EXT `bearer` == default account id). After a sale it falls back to the normal avatar.
- **Gating/state**: localStorage `nft_avatars_v2` caches resolved avatar URLs for 10 minutes and is cleared after a set or clear.
- **Redesign notes**
  - The picker loads every saved collection in full (`loadNftCollection`, including approvals and ICRC-97 fetches), which is heavy.
  - It only sees collections saved in this browser's NFT wallet, and only the default subaccount.

## NFT media rendering (images, 3D, audio, video)

- **Where**: NftCard, NftItem, NftAssetDetails, CharacterPanel. Public wherever those views are.
- **Information/behavior**
  - `getNftImage` order:
    1. The ICRC-97 `image`.
    2. `image` / `icrc7:image`.
    3. A walk through keys such as url, logo, thumbnail and asset, nested Map/Array up to depth 4, with Blob values turned into PNG data URLs.
    4. Any URL that looks like an image.
    - It skips `animation_url`, `external_url` and `youtube_url`. `ipfs://` becomes `https://ipfs.io/ipfs/`.
  - `getNftMedia` picks the kind from the `animation_url` extension: audio (mp3, ogg, wav, m4a, aac, flac, opus), video (mp4, webm, ogv, mov, m4v), 3D model (glb, gltf).
  - NftModel lazy-loads `@google/model-viewer`: drag to rotate, auto-rotate, zoom, the image as poster. If loading fails: "The 3D viewer did not load".
  - EXT: images come from the token's JSON metadata, falling back to `https://<canister>.raw.icp0.io/?tokenid=…`. Diamond Club EXT cards are served from the ICRC-7 canister's `/card/{n}.jpg`.
- **Redesign notes**: one shared asset-rendering layer that the redesign could keep as a single component.

## Embedded NFT views in other areas (Sneedex, Feed) and the Sneedex hand-off

- **Where**
  - NftAssetView exports:
    - `NftAssetIcon`: thumbnail from the first token's image, else the collection logo.
    - `NftAssetTitle`: "Name · Collection (SYM)", linking to `/nft_item` and `/nfts`.
    - `NftAssetDetails`: a grid per token with collection header (id, supply, "ICRC-37 approvals", "EXT standard (holders are account ids)", "Browse collection"), image, audio, SOULBOUND/3D marks, SneedLockNftSummary for lock NFTs, description, holder or owner ("yours"), attributes, links, and "Show/Hide metadata (N fields)".
    - `ExtBadge`.
  - Used by SneedexOffer (icon, title, details), SneedexOffers, Feed, FeedItemCard (icon plus "Card #n" for EXT) and ExtEscrowConfirm (ExtBadge).
  - Sneedex create reads `?nft=<collection>:<id>[,<id>]&standard=EXT` and `?lock=<id>` (login required) to pre-fill an NFT asset. Its collection picker lists Sneed Lock first, then `getSavedNftCollections()`, then EXT collections. It uses `loadNftCollection`/`loadExtCollection`, `approveToken` (ICRC-37) and `escrowExtAsset`. Details belong to the Sneedex slice.
- **Redesign notes**: the NFT wallet's localStorage list (`nft_collections`, `ext_nft_collections`) is a cross-page dependency of SneedexCreate and NftAvatarPicker.

## EXT legacy collections and the Diamond Club migration entry points

- **Where**: `/nft` (EXT cards and the collection notice), NftCard "Swap to ICRC-7", NftBrowser and NftItem (EXT reads), and `/diamond-club` (DiamondClub page, `sneedNft.js`, belongs to the DAO slice).
- **Information shown**
  - Collection notice: "These are the old EXT cards. The collection has moved to ICRC-7, with every image stored on-chain: swap your cards 1:1 (same card numbers) on the swap page" (→ `/diamond-club`).
  - EXT badges and tooltip ("the older Toniq / Entrepot standard").
  - Holders are shown as account ids, with the EXT token identifier.
- **Actions**
  - "Swap to ICRC-7" (per card) → `/diamond-club`.
  - On `/diamond-club`: swap by sending, claim, claim by proof of payment, card lookup. It links back to the NFT wallet and `/nft_item` for ICRC-7 cards.
- **Redesign notes**: both Diamond Club collections are built into the wallet, so a holder sees old and new cards side by side. EXT support (ExtUtils) is a full second backend for every NFT view.

**Part D: Transactions and Scan Wallet**

## Ledger-wide transaction history (any token's block log)

- **Routes & params**: `/transactions` (`src/pages/Transactions.jsx`). Public; no login needed. If logged in, the identity is used for the calls and for the "Follow the money" `start`.
  - `?ledger=<ledger principal>`: which token to show. Takes precedence over everything else. Also written to localStorage. Any ledger works here, including tokens that are not on the whitelist, but only through the URL (see notes).
  - `?sns=<root>`: read by SnsContext/SnsDropdown. The selected SNS's ledger becomes the default token when there is no `?ledger=`. It also sets the hero logo.
  - `?start=<n>`: a ledger index offset, read and written by TransactionList (non-embedded mode only). It opens on page `floor(start/pageSize)`. It is written back (`replace`) as you page.
- **Purpose**: browse the raw transaction log of one ICRC-1 ledger, page by page.
- **Information shown**:
  - Hero: logo of the selected SNS (from SNS governance metadata via `fetchSnsLogo`; fallback icon). Title "Transaction History" / "Browse and filter all ledger transactions".
  - "Select Token" card: a TokenSelector over the backend whitelist, with search ("Search tokens..."). The hint reads "View transaction history for any ICRC-1 token on the Internet Computer".
  - Empty state when no token is chosen: "Select a Token" / "Choose a token above to view its transaction history".
  - The TransactionList table, in ledger mode (see the TransactionList section). Total = the ledger's `log_length` ("N txs").
- **Actions**:
  - Pick a token (TokenSelector). This writes `?ledger=`; the param is removed when the pick equals the SNS ledger.
  - Change the SNS in the header SNS dropdown (`Header showSnsDropdown`). This switches the token only when there is no `?ledger=`.
  - "Follow the money for this token →" opens `/tools/money?ledger=<ledger>[&start=<my principal>]`.
  - Every TransactionList action (type pills, From/To filters, AND/OR, "Jump to" + "Go", "Export CSV", sort, Prev/Next, page size).
  - Canister calls:
    - SNS root `list_sns_canisters`, as a fallback for the ledger id when it is not in the SNS cache.
    - SNS governance metadata, for the logo.
    - Then the TransactionList ledger-mode calls.
- **Entry points**:
  - Header "Sneed Hub" menu → "Transactions". `/transaction` is highlighted under it (legacyMappings).
  - Hub feature card "Transactions" ("View Transactions"; its description claims "Track transfers, swaps, and more").
  - `/sns` quick link `/transactions?sns=<root>`.
  - Wallet canister card for a ledger/index canister ("Transactions", next to "Safety report" and "Block log").
  - Token Safety checks "Mints after genesis" and "Token" (`HUB.transactions`).
  - HelpTools "Transactions" section.
- **Cross-links out**: `/tools/money`. Per row: `/transaction`, principal profile (`/user` or `/canister`), subaccount profile.
- **Shared building blocks**: TransactionList, TokenSelector (WhitelistTokensContext, logo cache), Header + SnsDropdown, SnsUtils (`getAllSnses`, `fetchSnsLogo`).
- **Gating/state**:
  - localStorage `transactions_selected_ledger`, `selectedSnsRoot` (SnsContext), `sneed_privacy_mode_v1` (amounts masked as ••••).
  - No premium gating and no feature flags.
- **Redesign notes**:
  - The page opens on the **oldest** transactions: page 0 = index 0, sorted ascending. There is no "latest" or "last page" button; you have to page forward or "Jump to" a high index.
  - From/To and type filters only filter the **currently loaded page** (10–100 rows), not the whole ledger. A filtered-empty page also hides the Prev/Next bar (see notes).
  - The ICP ledger (and any ledger without ICRC `get_transactions`) fails silently: the error is never rendered, so the box is blank. `/transaction` handles ICP; this list does not.
  - The SNS selector and the token selector are two independent "what am I looking at" controls. The hero logo follows the SNS, even when another token is selected.
  - The localStorage persistence is effectively overridden: when there is no `?ledger=`, the SNS-ledger effect runs last and replaces the stored token.
  - A token from `?ledger=` that is not on the whitelist shows the placeholder in the selector, while its transactions are listed.
  - Switching token keeps the page index and `?start=`, because TransactionList has no `key`.
  - Help text describes "Export" as saving "the shown transactions". In ledger mode that means only the current page.
  - `handleSnsChange` is a no-op.

## Shared transaction list engine (TransactionList): table, filters, sort, paging

- **Component**: `src/components/TransactionList.jsx`, with two data modes chosen by `principalId`.
  - **Ledger mode** (no `principalId`):
    - `ledger.get_transactions({start: page*pageSize, length: pageSize})`.
    - For each `archived_transactions` range: `archive.get_transactions({start, length})`, using `callback[0]` as the archive canister.
    - Rows get `txIndex`; the total is `log_length`.
  - **Account (index) mode** (`principalId` set):
    - Needs the SNS index canister from `sns_root.list_sns_canisters` (`snsRootCanisterId`). So it works **only for the SNS token of that root**.
    - Loops `index.get_account_transactions({account:{owner, subaccount}, max_results:100, start:cursor})`, using the oldest id as the cursor, until a batch has fewer than 100 rows. The **entire** account history is loaded progressively.
    - Filtering, sorting and paging happen client-side.
  - Both modes:
    - Token decimals/symbol come from `getTokenMetadataSync` (token cache), else `icrc1_decimals` + `icrc1_symbol`; the fallback is 8 decimals.
    - The ledger id is either a prop or `list_sns_canisters().ledger`.
- **Props**:
  - `snsRootCanisterId`, `ledgerCanisterId`, `principalId`, `subaccount` (fixed account subaccount).
  - `showSubaccountFilter`, `initialSubaccountFilter` (hex or bytes), `onSubaccountFilterChange(hex|null)`.
  - `isCollapsed`, `onToggleCollapse`, `showHeader`, `embedded` (no URL state; newest first), `headerIcon`.
- **Information shown**:
  - Table columns:
    - Type badge (Transfer / Mint / Burn / Approve / Unknown).
    - ID (`#<block index>`, a link).
    - From / To: PrincipalDisplay, short form, with name/nickname/verified badge from NamingContext. For transfers only, the subaccount appears as "Sub: <16 hex>..." plus a copy button.
    - Amount, in ledger decimals and masked in privacy mode.
    - Time: relative; the full locale time is in the tooltip.
  - Approve rows show the spender in the To column. Mint has no From; burn has no To.
  - Not shown in the UI: memo, fee, `created_at_time`, expected allowance, the index's account `balance` / `oldest_tx_id`. Memo and fee are only in the CSV.
  - Status lines:
    - "Loading transactions...".
    - "Loading more transactions... (N loaded so far)", while the index loads progressively.
    - "No transactions found".
    - "Page X of Y" (shown as "Y+" while loading more).
    - "N txs (loading...)".
  - Subaccount filter (account mode with `showSubaccountFilter`): "N found" count of the principal's non-zero subaccounts (`index.list_subaccounts({owner, start: []})`, first page only).
- **Actions** (labels as in code):
  - **Type pills**: "All", "Transfer", "Mint", "Burn", "Approve".
  - **Address filters**: "Filter" row with two PrincipalInputs, "From (principal or account)..." and "To (principal or account)...".
    - Each PrincipalInput has a name-search dropdown (Private/Public/All tabs, user/canister toggles).
    - Each also has a subaccount dialog button: "Account with Subaccount" dialog with "Principal + Subaccount" / "Subaccount (hex) - optional" / "Encoded Account" / "Parsed Values" / "Cancel" / apply.
    - Matching:
      - An exact principal matches any subaccount.
      - An ICRC-1 textual account requires the subaccount to match too.
      - Otherwise a substring of the principal text, the name or the nickname.
  - **"AND" / "OR"** combine the From and To filters. **"✕ Clear"** resets both.
  - **"Jump to" [index] "Go"** (ledger mode only): jumps to the page containing that index and writes `?start=` when not embedded.
  - **Sort** by clicking the headers "Type", "ID", "From", "To", "Amount", "Time". Clicking again toggles asc/desc.
  - **"Prev" / "Next"** buttons. **"Show [10|20|50|100] per page"** select.
  - **"Export CSV"** (see the CSV section).
  - **Row click / "#id" link** → `/transaction?sns=<root>&id=<id>&ledger=<ledger>`.
  - **Principal name** → profile (`getPrincipalProfileUrl`: `/user?id=` for users, `/canister?id=` for canisters).
    - Right-click or long-press opens the context menu: "Send Message" (login), "Set/Edit Nickname" (login), "View Profile", "Copy Principal ID".
    - For transfers with a subaccount, the menu adds "Copy ICRC-1 Account".
  - **"Sub: …" on a transfer row**:
    - If the row's principal is the page's own principal and `showSubaccountFilter` is on, a click applies that subaccount as the filter and calls `onSubaccountFilterChange`. Tooltip "Filter by this subaccount".
    - Otherwise it links to the profile with `&subaccount=<hex>`. Tooltip "View transactions for this subaccount".
    - Copy icon: "Copy subaccount".
  - **Subaccount combo box** (account mode with `showSubaccountFilter`):
    - Placeholder "All (type to filter or enter hex)".
    - The dropdown offers "Main account (default subaccount)", the discovered subaccounts (filterable by hex), and `Use "<hex>" as subaccount`.
    - Enter applies hex (left-padded to 32 bytes); Esc closes; "Clear filter" (×).
- **Embeddings** (every place TransactionList is used):
  - `/transactions` (Transactions.jsx):
    - Ledger mode: `snsRootCanisterId=selectedSnsRoot`, `ledgerCanisterId=selectedLedger`, `showHeader=false`, `embedded=false`.
    - It is the only use with URL state (`start`), "Jump to", and ascending default sort.
  - `/principal` and `/user` (Principal.jsx), "Transactions" tab ("<SNS name> Transactions"):
    - Account mode: `snsRootCanisterId = ?sns || selectedSnsRoot || SNEED`, `principalId` = viewed principal, `showSubaccountFilter=true`, `initialSubaccountFilter = ?subaccount`.
    - `onSubaccountFilterChange` writes and removes `?subaccount`. A `?subaccount` in the URL auto-opens this tab, and the profile header shows a "Sub: …" chip with copy.
  - `/me` (Me.jsx), "Transactions" tab ("My <SNS name> Transactions"):
    - Account mode for the logged-in principal; `snsRootCanisterId=selectedSnsRoot`. There is no subaccount filter, so it shows the **default subaccount only**.
    - "Select a DAO to view transactions" when no SNS is selected. Login required (the Me page).
  - `/neuron` (Neuron.jsx), "Transactions" tab, next to "Details":
    - Account mode with `principalId` = SNS governance canister and `subaccount` = neuron id bytes. That is the neuron's staking account history (stake, disburse, top-ups).
    - Only rendered when the selected SNS has a governance id.
  - Not embedded anywhere else: Wallet token cards, `/canister` and SNS pages have no inline history.
- **Shared building blocks**:
  - PrincipalDisplay + PrincipalContextMenu + `getPrincipalDisplayInfoFromContext` (PrincipalUtils).
  - PrincipalInput (also used by send forms and @-mentions).
  - NamingContext.
  - StringUtils `formatAmount` (privacy-masked) and `subaccountToHex`.
  - DateUtils `getRelativeTime`.
  - useTokenCache.
  - `utils/txCsv`.
  - The `external/sns_root`, `sns_index`, `sns_archive`, `icrc1_ledger` actors.
- **Gating/state**:
  - No premium, flags or login needed (logged-in identity is used when available).
  - Reads localStorage `sneed_privacy_mode_v1` indirectly (masking).
  - Mobile (≤640px, CSS injected as `<style id="transaction-responsive-css">`): the table becomes cards showing type badge, `#id`, From/To short names, amount and relative time. Cards have **no subaccount info, links or copy**. The filter row stacks.
- **Redesign notes**:
  - Two very different products share one UI. Ledger mode is a paged raw log with page-local filters. Account mode is the full history with global filters and an "N found" subaccount filter.
  - Account history is only possible for **SNS tokens**: the index comes from the SNS root.
    - There is no account history for ICP, ckBTC or arbitrary ICRC tokens here, even though Follow the Money and approvals code can find index canisters elsewhere.
    - Principal and Me pages only show the currently selected SNS's token, one DAO at a time.
  - `showHeader`/`isCollapsed`/`onToggleCollapse`/`headerIcon` (the collapsible "Transactions ▼" header) are dead in practice: every embedding passes `showHeader={false}`.

## Single transaction explorer (one block of a ledger)

- **Routes & params**: `/transaction` (`src/pages/Transaction.jsx`). Public.
  - `?id=<block index>`: the transaction to show. Non-digits are stripped before fetching.
  - `?ledger=<ledger principal>`: takes precedence.
  - `?sns=<root>`: the fallback ledger is that SNS's ledger. Without either, the default is `selectedSnsRoot`, then SNEED. `?sns=` also changes the global SNS selection via SnsDropdown sync.
  - The ICP ledger (`ryjl3-tyaaa-aaaaa-aaaba-cai`) is special-cased.
- **Purpose**: look up and inspect one ledger transaction by index; step to neighbours.
- **Information shown**:
  - Header "Transaction Explorer" / "View detailed transaction information on the ledger".
  - Type banner with label and description: "Token transfer between accounts", "New tokens created", "Tokens permanently destroyed", "Spending allowance granted".
  - Token chip: logo (`resolveTokenLogo`), name, symbol, from `icrc1_metadata` + `icrc1_decimals`.
  - "Transaction ID" `#id`, plus "Timestamp" (relative + full locale).
  - Transfer: "Amount" + symbol, "Fee: …", "From" card, "To" card, "Spender" card when present.
  - Mint: "Minted Amount" (+), "To".
  - Burn: "Burned Amount" (−), "From", "Spender".
  - Approve: "Approved Amount", "Fee", "Expires: <time>", "From", "Spender".
  - Account cards: full PrincipalDisplay (name, copy button, context menu) plus "Subaccount: <hex>". ICP blocks show "Account ID: <hex>" (account identifier) instead.
  - Not shown: memo (ICRC memo; ICP `memo`/`icrc1_memo` — ICP memo is fetched but dropped), `created_at_time`, `expected_allowance`, the ledger canister id, the archive canister, the block hash/certificate.
- **Actions**:
  - "Enter transaction ID..." input + "Search" (or Enter). Validates the input as a BigInt ("Please enter a transaction ID" / "Invalid transaction ID format. Please enter a valid number.").
  - "Previous" (disabled at 0) / "Next" (no upper bound) step `id` ±1 and keep `ledger`.
  - Copy ID button (title "Copy ID").
  - Context menu on principals: "Send Message" (login), "Set/Edit Nickname" (login), "View Profile", "Copy Principal ID". There is no "Copy ICRC-1 Account" here.
  - Header SNS dropdown. It changes the ledger only when there is no `?ledger=`.
  - Canister calls:
    - ICRC: `ledger.get_transactions({start:id, length:1})`; if the id is archived, `archive.get_transactions` on `archived_transactions[0].callback[0]`.
    - ICP: `query_blocks({start, length:1})`, then the archive's callback method (`utils/icpBlocks.js` `fetchIcpTransaction`).
    - SNS root `list_sns_canisters`, as a fallback for the ledger.
- **Entry points**:
  - Every TransactionList row or `#id`.
  - SendReceipt "View transaction" (the dialog after any send from SendTokenModal; `transactionUrl`).
  - Rewards claim history rows (`?sns=SNEED&id&ledger`).
  - RLL page, admin RLL, RllActivityCard ("ledger tx N"), `/rll_ledger` rows.
  - Token Safety "Age" check (`/transaction?id=0&ledger=`).
  - HelpTools "Transaction Explorer".
- **Cross-links out**: only principal profile links and the context menu. There is no link back to `/transactions`, `/tools/money`, `/tools/icrc3`, or the token's Safety report.
- **Shared building blocks**: PrincipalDisplay / NamingContext, StringUtils `formatAmount` (privacy-masked), `subaccountToHex`, TokenUtils `resolveTokenLogo`, `utils/icpBlocks` (shares `getAgent` with `utils/icrc3/client`), Header + SnsDropdown.
- **Gating/state**:
  - No auth needed.
  - Privacy mode masks amounts (`sneed_privacy_mode_v1`).
  - No localStorage of its own.
- **Redesign notes**:
  - The only place that understands ICP blocks. The list page does not, so ICP history has no list view at all.
  - Once `?id=` is in the URL, typing in the search box refetches on every keystroke (the effect depends on `currentId`), before "Search" is pressed.
  - Rendering calls `BigInt(currentId)` for the Prev button, so typing a non-numeric character while a transaction is shown likely throws during render.
  - ICRC `fee`/`expires_at` are opt arrays and are tested for truthiness:
    - A transfer without a recorded fee shows "Fee: -".
    - An approval without expiry likely shows "Expires: <1970 epoch>".
  - Unused imports and state (`useNavigate`, `fetchAndCacheSnsData`, `getAllSnses`, `getFullDate`, `archiveCanisterId`).

## Transaction CSV export

- **Where**: the "Export CSV" button in every TransactionList (`/transactions`, `/principal`, `/user`, `/me`, `/neuron` tabs). Implemented by `src/utils/txCsv.js` (pure: `transactionsToCsv`, `txToCsvRow`, `csvCell`, `formatDecimal`, `downloadCsv`).
- **Information exported**: columns `time` (ISO 8601 UTC), `type`, `token` (symbol), `amount`, `fee` (exact decimals, **never masked**), `from`, `to` (ICRC-1 textual accounts incl. subaccount; the approve spender goes in `to`), `memo` (hex), `block_index`, `ledger`. RFC 4180 with CRLF and a UTF-8 BOM; formula-injection guard (a leading `'`).
- **Actions/preconditions**:
  - Disabled until token metadata is known and at least one row matches.
  - Tooltip: "Download the N loaded transactions that match the filters as CSV (exact amounts, ISO times)".
  - File name: `transactions_<SYMBOL>[_<principal first 11 chars>]_<timestamp>.csv`.
- **Scope**:
  - Account mode: every loaded history row that passes the type/From/To filters, sorted.
  - Ledger mode: only the current page (filtered).
- **Redesign notes**:
  - The only way to see memo and fee for list rows.
  - The export scope differs by mode, and that is not explained in the UI.

## Scan wallet: token balances of any principal (legacy)

- **Routes & params**: `/scan_wallet` (`src/ScanWallet.jsx`, root-level legacy file). Public; calls are anonymous. `?principal=<principal>` pre-fills the box and scans automatically on load. "Scan" rewrites the URL to `?principal=`.
- **Purpose**: show the main-account balance of every whitelisted token for an arbitrary principal.
- **Information shown**:
  - "Scan Wallet" heading and "Token Balances" list. Per token: logo (from `icrc1_metadata` via `resolveTokenLogo`), symbol, balance (`balance/10^decimals` with fixed decimals).
  - A spinner per token while loading. "Loading tokens..." while the whitelist loads.
  - Error text: "Invalid principal ID" for bad input.
- **Actions**:
  - "Enter Principal ID" input + "Scan" (validates with `Principal.fromText`).
  - "Hide empty balances" checkbox. It hides zero balances and errored tokens; tokens still loading stay visible.
  - Calls: whitelist (backend `get_whitelisted_tokens`, via WhitelistTokensContext), then for each ICRC token in parallel `icrc1_balance_of({owner, subaccount: []})` + `icrc1_metadata()`. Non-ICRC standards are skipped.
- **Entry points**: none in the nav, the Hub or other pages. Only HelpTools ("Scan Wallet" section). Reached by URL only.
- **Cross-links out**: none. No link to the principal's profile, token pages, transactions or Follow the Money.
- **Shared building blocks**: WhitelistTokensContext, TokenUtils `resolveTokenLogo`, Header, `Help.css` (`scanwallet-container`).
- **Gating/state**:
  - No auth, no localStorage.
  - **Not** privacy-masked: it uses its own formatter, so balances show during screen-share even with privacy mode on.
  - Hard-coded dark styling: it does not use the theme system.
- **Redesign notes**:
  - Fully superseded by the Principal page's "Wallet"/balances tab ("Scan Tokens"). That tab adds an auto-scan, concurrency 8, a skip list, USD values and metadata parsing, and it reads the same whitelist and main account.
  - Errors and unsupported standards display as `0.000…` instead of "Error loading balance". That text only shows when no result object exists.
  - Precision is lost for large balances (`Number`).
  - Main account only: no subaccounts, no ICP account-id view, no USD.
  - Unused imports (backend actor, `Link`, `useNavigate`, auth).

**Part E: Native Bitcoin (btc.html) and Convert**

## Native Bitcoin page shell — `btc.html` ("Sneed Wallet · Bitcoin")
- **Routes & params**: `/btc.html` — a separate Vite HTML entry (`vite.config.js` `rollupOptions.input.btc`), NOT a React-Router route; shares JS chunks with the SPA but never loads `index.html`'s gtag. Tabs live in the hash: `#send` (default), `#move` (Bitcoin → ck), `#move-tobtc` (ck → Bitcoin), `#credit`, `#activity`; anything after `&` in the hash is kept but ignored by tab parsing. `btcnet=mainnet|testnet` (query or hash) = staging developer-mode switch for headless test browsers. Tab clicks rewrite the hash with `history.replaceState`; `hashchange` is listened to.
  - Auth: login required for everything except the beta explainer (II session carries over from the SPA: same origin, same AuthClient storage). Not premium-affected. Not admin.
- **Purpose**: the only surface that signs native Bitcoin (send, sweep, speed-up, move to ckBTC) and manages signing credit and BTC activity, isolated from analytics.
- **Information shown**:
  - "← Back to wallet" link (`/wallet`), title "Sneed Wallet · Bitcoin", network badge (TESTNET / red "MAINNET · REAL BTC · developer mode" / "Beta"), red developer-mode banner.
  - Signed-out: "Sign in to send Bitcoin… same principal as in Sneed Hub, so it is the same Bitcoin address."
  - Signed-in header: **Balance** (confirmed BTC, privacy-masked), "· X in pending sends" (locked), "From <own bc1/tb1 address>", testnet4 staging warning ("test coins with no value. Send only to testnet4 addresses").
  - Not available states: prod with beta off → full beta explainer + switch (no send UI); other builds → "Native Bitcoin is not available here (<reason>)".
  - One-time "Bitcoin (beta) is on" explainer (prod, until dismissed).
  - "Advanced: Bitcoin (beta)" (prod): Network label, Bitcoin canister id, beta switch. "Advanced: network" (staging allowlisted principal): Network label, Bitcoin canister id, developer mainnet toggle.
  - Footer: no analytics/third-party scripts, refuses to run in a frame, every action public and tied to principal.
- **Actions**:
  - "Internet Identity" (II 2.0 login) / "Internet Identity 1.0".
  - Tab pills: "Send" · "Move" (only when network has a ck pair) · "Signing credit" · "Activity".
  - "Verify balance (0.313B cycles)" (0.113B signer + 2×0.1B ledger fees) → ICRC-2 approve on cycles ledger → `btc_caller_balance` on the Chain Fusion Signer (replicated) → compares with UTXO sum (re-reads once on mismatch). Shows "Verified: … the same as the coins shown here" or error.
  - "Bitcoin (beta)" switch (writes localStorage, reloads). "Developer: use Bitcoin mainnet (real BTC)" toggle (writes localStorage, strips `btcnet` from URL, reloads).
- **Entry points & cross-links**: full-page navigations (`window.location.assign`, never router push) from the SPA's Bitcoin card ("Send", "Move to ckBTC") and the ckBTC row ("Move to my Bitcoin address"); hrefs built by `btcPageHref(status, tab)` (keeps `?btcnet=mainnet` if that's how dev mode was enabled). Out: `/wallet`, mempool.space tx links. `#credit` / `#activity` are reachable only from inside btc.html (tab bar, "Top up signing credit", "Activity" buttons) or by typing the URL.
- **Shared building blocks**: `AuthProvider`, `ThemeProvider`, `DenominationProvider` only (no Naming/Wallet/Notification providers, no Header); `ConvertUi` kit (Card/Button/Notice/Advanced/Row/Pill/Steps); `useBtcWallet` (one set of actors + reads for all tabs) and `useMoves`; `readNative` from the pockets `btc_native` adapter (20 s shared cache); `PriceService`.
- **Gating/state**:
  - Feature flag `btcNativeStatus` (utils/btc/network.js): prod build (DFX_NETWORK=ic + prod frontend id `pxtkg-giaaa-aaaal-ajjzq-cai`) → off until per-browser "Bitcoin (beta)" switch, then mainnet (Beta badge). Staging build, not prod id, not prod host → Bitcoin testnet4 (`tBTC`, ckTESTBTC pair). Staging + allowlisted principal (`tksci-…-aae`, "Mr Sneed" test identity) + opt-in (toggle or `btcnet=mainnet`) → mainnet developer mode (red badges). Local/other builds → off. Network fixed once per page load.
  - Security: JS refuses to run inside a frame; `.ic-assets.json5` gives btc.html its own strict CSP (script-src 'self', connect-src IC + mempool.space only), `X-Frame-Options: DENY`, `frame-ancestors 'none'`, `camera=(self)` (QR scan), `Referrer-Policy: no-referrer`, `Cache-Control: no-cache`. No manifest/service worker (not part of the PWA).
  - Background (only while this page is open and visible): UTXO + credit refresh every 60 s; market (ICP/USD, CMC ICP/XDR, ICP balance, BTC/USD via ckBTC price on mainnet) on load/after actions; pending-record reconciliation on every UTXO read; RBF winner lookup (bitcoin canister query of recipient UTXOs) ≤ every 10 min per group; moves tick every 60 s.
  - localStorage: `sneed_btc_beta_v1`, `sneed_btc_beta_intro_seen_v1`, `sneed_btc_dev_mainnet_v1` (+ per-tab keys listed below).
- **Redesign notes**: must stay a separate HTML entry (no analytics where BTC is signed; CSP) — any redesign of the SPA can't simply absorb it as a route. Own minimal shell, no top nav; leaving = full reload back to `/wallet`. Two different "Advanced" footers depending on build. "Verify balance" is a paid action sitting in the header text block.

## Bitcoin (beta) opt-in, network badges & developer mainnet mode
- **Routes & params**: on btc.html (header, Advanced), the SPA wallet's Bitcoin card / ckBTC card, `btcnet` URL param (staging dev only). Login not needed for the switch.
- **Purpose**: per-browser opt-in to native Bitcoin on prod; visible marking of testnet / real-BTC developer mode / beta.
- **Information shown**: `BtcBetaOffer` — "Your own Bitcoin address" + Beta badge, "Receive and send real bitcoin from Sneed Wallet, next to your ckBTC. Off until you turn it on.", "What is this?" disclosure (open by default on btc.html). `BtcBetaExplainer` bullets: native BTC vs ckBTC; no seed phrase, key held by IC threshold signing via DFINITY Chain Fusion Signer, whoever acts as your principal (incl. this site while signed in) can move coins; receiving free, each send = BTC network fee + signing credit in cycles; moving via ckBTC minter (4 confirmations ≈ 40 min minus check fee; back usually within an hour minus fees); "speeding up a stuck send and sweeping come later" (stale); switch remembered per browser, turning off hides the address, coins stay. Badges: TESTNET, "MAINNET · REAL BTC · developer mode", "Beta". Red banner text for dev mode.
- **Actions**: "Bitcoin (beta)" checkbox switch ("In this browser only · the page reloads"); "Got it" (dismiss one-time intro); "Developer: use Bitcoin mainnet (real BTC)" checkbox (shows "· on from the URL (btcnet=mainnet)" when param-driven).
- **Entry points & cross-links**: btc.html; SPA Bitcoin card (standalone `BitcoinCard`) / ckBTC `TokenCard` (offer shown when beta off; switch also inside the card's Receive → Advanced).
- **Shared building blocks**: `BtcBeta.jsx`, `TestnetBadge.jsx` (`BtcNetworkBadge`, `MainnetDevBanner`), `MainnetDevToggle.jsx` — all used both on btc.html and in the SPA.
- **Gating/state**: localStorage `sneed_btc_beta_v1` ('1' on), `sneed_btc_beta_intro_seen_v1`, `sneed_btc_dev_mainnet_v1`; prod ignores principal/dev toggle/URL. Off ⇒ no address derivation, no canister call, no `btc_native` pockets adapter registered.
- **Redesign notes**: a feature flag exposed as a user setting but scattered (card, btc.html Advanced, receive panel Advanced); not in any Settings page. Explainer copy outdated vs implemented sweep/speed-up.

## Native Bitcoin row on the wallet's Bitcoin card (SPA entry point)
- **Routes & params**: `/wallet` (Wallet.jsx). Rendered inside the ckBTC `TokenCard` when the flag is on (card title becomes "Bitcoin · ckBTC + BTC" + network badge; card total adds native **mainnet** BTC via `nativeBtcForCard`), or as a standalone `BitcoinCard` ("Your native Bitcoin address. Add ckBTC to your wallet to see both on this card.") when the wallet has no ckBTC token. Login required.
- **Purpose**: receive-side view of the native BTC address and launch point for btc.html.
- **Information shown**: row "Bitcoin address" + badge; confirmed balance (privacy-masked) + USD (mainnet only, ckBTC price); "+ X pending" or "test coins, no value" / "REAL BTC · developer mode"; "couldn't read Bitcoin" / "couldn't refresh: …". Receive panel: testnet / dev-mode warnings, QR (`bitcoin:` URI), "Your <network> address (Native SegWit)" with copy, note that Send/Move open the Bitcoin page and need signing credit, explanation (threshold key, no seed phrase, anyone knowing your principal can compute it, differs from Oisy address). Advanced: dev toggle / beta switch, Network, Address type "P2WPKH (Native SegWit)", Derivation path, Public key, Signer `grghe-syaaa-aaaar-qabyq-cai`, Bitcoin canister id, "Chain tip seen by the IC", "Under 6 confirmations", "Bitcoin canister balance (cross-check)" (+ "differs from the UTXO sum; refresh"), UTXO list (txid…:vout → mempool.space, amount, confirmations, "more not read").
- **Actions**: "Receive" (toggle panel), "Show QR"/"Hide QR" (remembered), copy address, "Refresh"; "Send" → `/btc.html#send`; "Move to ckBTC" (only when a ck pair exists; disabled without confirmed balance) → `/btc.html#move`. On the ckBTC Main pocket row: "Move to my Bitcoin address" (only if the network's pair ledger == this ledger, i.e. not on staging testnet4, and balance > fee) → `/btc.html#move-tobtc`.
- **Entry points & cross-links**: Wallet card grid; `/wallet/pockets` page lists the adapter as "Bitcoin address · chain address · on the Bitcoin card · read-only".
- **Shared building blocks**: `usePockets`, `btc_native` pockets adapter (kind `chain`, caps receive-only/readOnly, pseudo-ledger `btc:mainnet`/`btc:testnet`), `PocketLine`/`PocketButton`, `ConvertUi` `CopyValue`/`QrCode`.
- **Gating/state**: `btcNativeShown()` (flag on or prod beta offer); refresh every 60 s while visible; free bitcoin-canister queries only (`bitcoin_get_utxos_query`, `bitcoin_get_balance_query`); localStorage `sneed_btc_native_show_qr_v1`.
- **Redesign notes**: ckBTC card mixes two assets (ck token + native BTC address); the same card also shows a "Convert" button (→ /convert) — two different BTC↔ckBTC paths side by side. Native BTC never sends from the SPA (pockets caps `send:false`).

## Send native Bitcoin (btc.html `#send`)
- **Routes & params**: `/btc.html#send`. Login + active flag.
- **Purpose**: sign and broadcast a P2WPKH Bitcoin transaction from the user's principal-derived address via the Chain Fusion Signer.
- **Information shown**:
  - Edit: "Spendable" (confirmed, unreserved coins), "reserved by pending sends", refresh icon; "To (Bitcoin address)" with network hint (testnet4 prefixes), validation error, address-type label, "· your own address"; QR-scan note; "Amount (BTC)" with sats hint; "Network fee" tier cards Slow/Normal/Fast/Custom (sat/vB, ~blocks 6/3/1, "≈ N sats" for the selected one); fee-source text (mempool.space, never your address) or "Fee lookups are off" / "Couldn't get fee suggestions"; custom-rate errors/warnings; plan errors (insufficient, max too small, "needs X sats, Y spendable"); summary "Network fee ≈ N sats · $ · signing ≈ N cycles · $"; dust-to-fee warning (< 1,000 sats leftover can't be change); "Reading your coins…" / "Couldn't read Bitcoin"; storage-blocked error; footer "Signed by DFINITY's Chain Fusion Signer… Bitcoin sends can't be reversed."
  - Advanced (edit): coin-control list (txid…:vout, amount, confirmations; Pill "automatic"/"manual"; note each coin costs 37B cycles to sign; reserved/unconfirmed coins not listed).
  - Review: "Review the send" + badge; To + address type; own-address warning; "Recipient gets" (amount · $); "Network fee" (rate × vsize, tier/~blocks or custom, dust note); "Leaves your address"; "Change back to you"; "Signing cost" (signer cycles for N coins + one-time address check 0.079B + balance cross-check 0.113B + ledger fees, paid from signing credit); "Balance cross-check" row (reason: >25% of balance / >1,000,000 sats / sweep; "done a moment ago"/"skipped"/cost); Advanced "the exact request" (JSON `SendBtcRequest`, "btc_caller_send … paid with CallerPaysIcrc2Cycles").
  - Sending: progress lines ("Checking your coins…", "Approving the signing fee…", "Cross-checking your balance…", "Checking your address with the signer…", "Signing and broadcasting… 10–40 seconds per coin. Keep this page open.").
  - Done: "Sent." / failed / unknown / not_sent notices; txid-drift note; "Transaction id"; "Open on mempool.space".
- **Actions**:
  - Paste address or scan QR (`QrScanButton` target btc; camera allowed only on this page); type amount or "Max" (all spendable minus fee, no change); pick fee tier or Custom sat/vB; Advanced: "Don't look up fees (enter my own). Remembered in this browser." (forces Custom); "Sweep: empty this address…"; coin-control checkboxes.
  - "Review" (needs valid plan, readable pending store, move guard, sweep target).
  - Review: "← Edit"; tick for unusual fee (fee > 10% of amount or rate > 3× Fast); real-BTC tick on prod beta ("I checked the address and amount. This sends real bitcoin and can't be undone.") or red dev-mode tick; Advanced "skip the balance cross-check" tick (with warning); primary "Send X BTC" (dev: "… on MAINNET").
  - Execution: re-read UTXOs + verify plan coins still free → `ensureCredit`: read cycles-ledger balance/allowance, `icrc2_approve` the signer on cycles ledger `um5iw-rqaaa-aaaaq-qaaba-cai` for exactly the needed cycles for 1 h (only if current allowance insufficient/expiring) → optional `btc_caller_balance` cross-check → first send per device: `btc_caller_address` check vs offline derivation (0.079B) → pending record written (`submitting`) → `btc_caller_send(req, opt CallerPaysIcrc2Cycles)` with explicit fee, expected txid precomputed.
  - Shortfall → error with "Top up signing credit" (jumps to `#credit` with the exact need). Done: "New send", "Activity", "Top up signing credit".
  - Costs: BTC network fee (sats) + signer 95B + 37B/input + 1B/output (+1 output always priced for change) ≈ 134B typical, + 0.1B cycles-ledger fee per charge and per approve.
- **Entry points & cross-links**: SPA "Send" button; btc.html default tab; links to `#credit`, `#activity`, mempool.space.
- **Shared building blocks**: `BtcSend` is reused by Move → ckBTC (`move` prop) and Sweep (`sweepTo` prop); `QrScanButton` (also in other SPA send flows); `parseBtcAddress`/`btcAddressTypeLabel` (utils/ckAddress, also /convert); utils/btc `select` (UTXO selection, vsize/fee math), `fees` (mempool.space), `credit`, `send`, `pending`, `verify`, `txid`.
- **Gating/state**: fee rates fetched from `mempool.space/api/v1/fees/recommended` (or testnet4) only while the form is open, every 60 s, unless lookups disabled; sanity bounds (≤2000 recommended, custom ≤10000, warn >500). Sends blocked if localStorage pending store unreadable. Min send 1,000 sats. localStorage: `sneed_btc_no_fee_lookup_v1`, `sneed_btc_pending_v1:<network>:<principal>`, `sneed_btc_addr_verified_v1:<network>:<principal>`. Privacy mode masks balances (`formatAmount`) but review always shows signed amounts (`formatAmountShown`). No browser popups.
- **Redesign notes**: coin control and fee-lookup opt-out buried in Advanced; Sweep is a mode toggled from Advanced inside Send; cross-check cost appears/disappears depending on amount. Pending records are per-browser only (sends from another device aren't known/locked here).

## Sweep (empty the address) — inside Send
- **Routes & params**: `/btc.html#send` → Advanced → "Sweep: empty this address…"; programmatic `sweepTo={ principal }` prop (intended principal-migration wizard — not wired anywhere yet).
- **Purpose**: send every confirmed, unreserved coin (including uneconomic dust) to one destination with no change.
- **Information shown**: warning "Sweep: sends every confirmed coin… Nothing comes back as change."; destination kind radios; principal field with "Its Sneed Wallet address: …" (derived offline) or error; "Amount: everything / X BTC after the network fee (N coins)". Review: "Review the sweep"; "To the Sneed Wallet of principal …" + derived address; "Small coins included" (count, sats); "Stays behind" (reserved by pending sends / small coins left out); always a balance cross-check.
- **Actions**: "Back to a normal send"; radio "to a Bitcoin address" / "to another principal's Sneed Wallet"; principal text input (plain, no naming/autocomplete); Advanced "Leave out coins that cost more to spend than they are worth (N now, X sats)"; type the last 8 characters of the destination to confirm; "Sweep X BTC". Same call chain as Send (kind `sweep`).
- **Entry points & cross-links**: only from Send's Advanced section.
- **Shared building blocks**: `BtcSend`, utils/btc/sweep (`planSweep`, `sweepTarget`, `confirmTailOk`), utils/btc/address (`btcAddressFor`).
- **Gating/state**: same as Send; cross-check mandatory unless skipped in review Advanced.
- **Redesign notes**: hidden capability (Advanced-only); "send to another principal's Sneed Wallet BTC address" exists only here (not for normal sends). `sweepTo` hook is half-built (no caller).

## Move Bitcoin → ckBTC (btc.html `#move`)
- **Routes & params**: `/btc.html#move`; tab only when the network has a ck pair (mainnet: ckBTC minter `mqygn-kiaaa-aaaar-qaadq-cai` / ledger `mxzaz-hqaaa-aaaar-qaada-cai`; staging testnet4: ckTESTBTC minter `ml52i-qqaaa-aaaar-qaaba-cai` / ledger `mc6ru-gyaaa-aaaar-qaaaq-cai`). Direction pills "Bitcoin → ckBTC" / "ckBTC → Bitcoin". Login.
- **Purpose**: send own native BTC to the ck minter's deposit address for the user's Main account and auto-finish the mint.
- **Information shown**: "Asking the ckBTC minter for your deposit address and its terms…"; description (mints to Main account minus check fee, waits N confirmations, ETA); "Deposit address …"; amount hint "at least N sats" (minter min deposit / check-fee guard); "You get about X ckBTC after the minter's N sats check fee"; review "Review the move to ckBTC": "Moves", "Minter's check fee", "You get about … after N Bitcoin confirmations, ~ETA", network fee, signing cost; Advanced exact request (minter, ledger, `get_btc_address { owner, subaccount = none }`, then `update_balance`, min deposit, check fee). Done: "On its way to the ckBTC minter… Keep this page open… or press Check / finish under Activity → Moves."
- **Actions**: amount / "Max" / fee tier (same form as Send, destination fixed); "Review"; real-BTC tick ("…sends real bitcoin to the ckBTC minter and can't be undone; it comes back as ckBTC."); "Move X BTC to ckBTC"; "Try again" when minter unreachable; "New move", "Activity".
  - Calls: `get_minter_info` + `get_deposit_fee` (queries), `get_btc_address` (update, free), then the full Send chain (`btc_caller_send`, kind `toCkbtc`); afterwards automatic `update_balance` as the user (via `useMoves`, see Activity).
- **Entry points & cross-links**: SPA Bitcoin row "Move to ckBTC"; btc.html Move tab; result → Activity → Moves.
- **Shared building blocks**: `BtcSend` (move mode), utils/btc/moves (`minterTerms`, `depositGuard`, `depositEta`, `newDepositInfo`), utils/ckMinters (`ckbtcInfo`, `ckbtcDepositAddress`).
- **Gating/state**: `ckMoveFor(status)`; deposit records carried in the pending send store (`sneed_btc_pending_v1:…`, `ckbtc` sub-object).
- **Redesign notes**: duplicates /convert "Get ckBTC" (same minter deposit address) but source is the in-wallet address and finishing is automatic; on staging btc.html uses ckTESTBTC while /convert uses real mainnet ckBTC.

## Move ckBTC → Bitcoin (to own address) (btc.html `#move-tobtc`)
- **Routes & params**: `/btc.html#move-tobtc`. Login + ck pair.
- **Purpose**: burn ck tokens via the minter and receive BTC at the user's own derived Bitcoin address (only destination allowed).
- **Information shown**: description ("No signing credit needed (except a one-time address check on this device)"); "Available: X ckBTC" + refresh; amount hint (sats / "at least N sats"); estimate box "Bitcoin network + minter fee (estimate)", "You get about"; plan errors; storage-blocked notice; Advanced: Minimum, ckBTC ledger fee (approval), Bitcoin fee / minter fee, minter + ledger ids, payout address (derived here, checked with signer). Review: "To your own <network> address", "Burns" (+ ledger fee), "Bitcoin network + minter fee", "You get about … usually within an hour", "One-time address check 0.079B cycles"; Advanced exact calls (approve amount + one fee headroom, expires 1 h; `retrieve_btc_with_approval { address, amount, from_subaccount = none }`); Steps (check address once → approve → ask minter). Done: "Requested. The minter burned X (block N)… Track it under Activity → Moves."
- **Actions**: amount / "Max" (balance − ledger fee); "Review"; "Move X ckBTC to my Bitcoin address"; "Top up signing credit" on shortfall; "New move", "Activity".
  - Calls: `icrc1_balance_of`/`icrc1_fee` on ck ledger; `estimate_withdrawal_fee` (debounced query); first time per device: approve cycles + `btc_caller_address`; `icrc2_approve` minter on ck ledger (recorded via approvals module) → `retrieve_btc_with_approval` on minter; later `retrieve_btc_status_v2` polling.
- **Entry points & cross-links**: ckBTC Main row "Move to my Bitcoin address" in the SPA; btc.html Move tab direction pill.
- **Shared building blocks**: utils/ckMinters (`ckbtcWithdraw`, `ledgerInfo`, `ckbtcMinter`), utils/btc/moves (`planMoveToBtc`, `executeWithdraw`, `newWithdrawRecord`), approvals `approveAndRecord`.
- **Gating/state**: records in localStorage `sneed_btc_moves_v1:<network>:<principal>`; paused if storage blocked.
- **Redesign notes**: overlaps /convert "Withdraw BTC" (which allows any address and shows minter-side history). Same ck→BTC flow, two UIs, two record systems.

## Signing credit (btc.html `#credit`)
- **Routes & params**: `/btc.html#credit`; also opened from shortfall errors with a precomputed "need". Login.
- **Purpose**: manage the user's cycles-ledger balance that pays the Chain Fusion Signer, and the signer's allowance.
- **Information shown**: "Signing credit" balance in cycles (privacy-masked) + USD value; "enough for about N sends"; read error; explanation (~134.1B cycles per typical send incl. ledger fee, paid to DFINITY's signer; network fee separate). Pending mint notice ("A top-up of X ICP is not finished (ICP block N). Finishing never sends ICP again."). Top-up: ICP balance hint; "You pay X ICP + 0.0001 ICP ledger fee"; "You get about N cycles · $ at the CMC's current ICP/XDR rate (1T cycles = 1 XDR); the cycles ledger keeps 0.1B"; insufficient-ICP warning; confirmation text naming principal and "public on chain"; result message. Advanced: "Balance (exact)" (cycles ledger id + account), "Allowance to the signer" (amount, expires/expired/no expiry), policy text (each send approves exactly what it costs for one hour), cost table (send base 95B, per coin +37B, per output +1B, typical 134B, address check 0.079B, cycles-ledger fee 0.1B).
- **Actions**: refresh; "Finish top-up" (resume interrupted mint); presets "What this send needs" (when coming from a send), "~3 sends", "~10 sends", "~30 sends", "Other amount" (ICP input); "Top up…" → "Yes, top up" / "Cancel"; Advanced "Revoke allowance" → "Revoke for 0.1B cycles?" "Yes, revoke" / "Cancel".
  - Calls: ICP ledger `icrc1_transfer` to the CMC (`rkp4c-…`) subaccount for the principal with memo MINT → CMC `notify_mint_cycles` (`mintCyclesWithIcp`, dedup-safe, pending record first); `icrc2_approve` amount 0 on cycles ledger (revoke); reads `icrc1_balance_of`, `icrc2_allowance`, CMC `get_icp_xdr_conversion_rate`.
- **Entry points & cross-links**: btc.html tab; "Top up signing credit" buttons in Send/Move/Activity errors. Interrupted mints are also recoverable at `/tools/recover-topup` (same pending store).
- **Shared building blocks**: utils/cyclesTopUp (same flow used for canister top-ups elsewhere), utils/btc/credit, `maskIfPrivate`, `DenominationContext.formatValue`.
- **Gating/state**: localStorage `sneed_pending_cmc_topups_v1` (shared with all CMC top-ups; key `cycles-ledger:<principal>`). Users always pay; no sponsorship.
- **Redesign notes**: an "ICP → cycles for my own cycles-ledger account" capability that lives only here; candidate for a general cycles/credits area. Allowance inspection/revoke is also conceptually part of the wallet's approvals manager but separate here.

## Bitcoin activity (btc.html `#activity`)
- **Routes & params**: `/btc.html#activity`. Login.
- **Purpose**: track pending sends and moves, list sent/received BTC, recover stuck/unknown sends, support log.
- **Information shown**:
  - **Pending**: per send — kind prefix ("Move to ckBTC:", "Sweep:", "Speed-up (CPFP) for <txid>… back to you"), amount, recipient, state pill (sending / waiting for a block / outcome unknown; "faster version · …"), "Replaces <txid> (fee N sats…)", "finding out which version was mined", fee sats, created time, coins reserved, attempts, txid link, unknown-outcome explanation (30-min re-send offer), error, "Not mined yet: N blocks without a confirmation / fee below current Slow", why speed-up isn't possible.
  - **Moves** (newest first): Bitcoin → ckBTC items (amount, phase pill: sending / send outcome unknown / failed / replaced / dropped / waiting for confirmations / minting / minted / too small / quarantined; created time; minted amount; "N of M confirmations · ETA"; minter message; Details: transaction link, deposit address, minter/ledger, check fee, network fee, mint block, checks count, next automatic check). ckBTC → Bitcoin items (amount, state pill: approving / requesting / on its way / outcome unknown / failed / arrived / being refunded / refunded as ck / too small / dismissed; minter status text queued/signing/sending/submitted; reimbursed amount; payout txid link; error; Details: to, burn block, approve block, allowance, fee estimate, minter status, minter/ledger). Empty: "No moves between Bitcoin and ckBTC from this device yet."
  - **Sent**: finished sends (not moves) — prefix (Speed-up to yourself / Sweep), amount, recipient (short), date, state pill (confirmed/failed/replaced/dropped), explorer link, failure error, "Another version was mined instead…".
  - **Received**: amount, block, confirmations, "spent", txid:vout link — derived from UTXOs not created by own sends and remembered per browser (max 500); appear only after 1st confirmation.
  - Advanced "full history from mempool.space": opt-in explanation; list of net ± amounts, block/date or unconfirmed, txid links.
  - Advanced "send log for support": every record with kind/state/id/txid, links (replaces/superseded/child/cpfp/replaced by) and timestamped event log.
- **Actions**:
  - Pending: "Speed up…" (offered after 2 blocks or when rate < current Slow; earlier via "Advanced: speed up now"); "Ask mempool.space" (opt-in click; checks every RBF version; can mark winner/broadcast); "Send again…" → "Send again for N cycles" / "Cancel" (unknown & unseen 30 min; identical tx/txid; re-approves cycles; `btc_caller_send`); "Mark as dropped…" → "Yes, mark dropped" / "Cancel" (broadcast > 14 days; frees coins locally).
  - Moves: "Check / finish" (Bitcoin → ck; immediate `update_balance`, bypassing backoff; disabled until sent; also auto every minute while page open); "Refresh" (ck → BTC; `retrieve_btc_status_v2`); "Show the minter's list" (unknown state; `retrieve_btc_status_v2_by_account`, top 10 burns); "Dismiss" (unknown state, local only).
  - Explorer: "Yes, look up my address on mempool.space" (remembered), "Stop using it", "Older" (pagination).
  - "Copy log (JSON)" (network, principal, address, send records, move records).
- **Entry points & cross-links**: btc.html tab; "Activity" buttons after send/move; mempool.space tx links everywhere.
- **Shared building blocks**: `BtcSpeedUp`; utils/btc `pending` (reconcile states/actions), `history` (receives, mempool.space lookups), `moves`, `speedup`; `ckbtcWithdrawals`/`btcStatusText` (also used by /convert).
- **Gating/state**: localStorage `sneed_btc_pending_v1:…`, `sneed_btc_moves_v1:…`, `sneed_btc_receives_v1:<network>:<principal>`, `sneed_btc_explorer_history_ok_v1`. `useMoves` tick every 60 s while visible: free bitcoin-canister query of deposit confirmations, `update_balance` when the minter's confirmations reached (with backoff), `retrieve_btc_status_v2` for active withdrawals — only while btc.html is open.
- **Redesign notes**: history is device-local (spec target is backend layout store, needs a canister change); a full on-chain history exists only as opt-in Advanced. Moves split across two stores but merged in one list. Finished BTC→ck moves are under Moves, not Sent. Support log is effectively a dev/support tool exposed to users.

## Speed up a stuck send (RBF / CPFP) — inside Activity
- **Routes & params**: `/btc.html#activity` → pending item → "Speed up…". Login.
- **Purpose**: raise the fee of a broadcast-but-unmined send by replacement (RBF) or child-pays-for-parent (CPFP).
- **Information shown**: "Now X sat/vB, fee N sats"; tier buttons (only rates above current), Custom; fee-source note; way cards "Replace (RBF)" ("signs the same coins again with a higher fee") / "Child pays (CPFP)" ("a second transaction pays extra for both") with "· suggested"; RBF rows: "Recipient gets" (same / X less), "New network fee", "Extra fee" (from change / change used up), "Change back to you"; CPFP rows: "Speed-up transaction" (back to own address, extra coins added), "Extra fee" (package rate/fee/vsize); "Signing cost"; explanatory notes (full-RBF, CPFP only while parent in mempools); Advanced: BIP-125 rule numbers, parent/child sizes, exact request JSON.
- **Actions**: pick tier/custom; pick way; consent tick to reduce recipient amount when no change; unusual-fee tick; real-BTC tick; "Speed up: +X BTC fee" / "Cancel"; result "Sped up." → "Close". Calls: UTXO re-read, `ensureCredit` (cycles approve), optional `btc_caller_address`, `btc_caller_send` (replacement or CPFP child; linked records).
- **Entry points & cross-links**: only from Activity pending items.
- **Shared building blocks**: utils/btc/speedup (`planRbf`, `planCpfp`, `speedUpStatus`, `identifyWinner`), utils/btc/send (`executeReplacement`, `executeCpfp`), fees.
- **Gating/state**: offered after `SPEEDUP_AFTER_BLOCKS` = 2 (else Advanced); not for CPFP children, superseded sends, or sends with a pending CPFP child. Fee lookups follow `sneed_btc_no_fee_lookup_v1`.
- **Redesign notes**: implemented but beta explainer says it "comes later". Inline panel inside a list item; deep in the hierarchy.

## Convert page shell — `/convert`
- **Routes & params**: `/convert` (App.jsx route; main SPA `Header`). Query: `?asset=<ck ledger id | key e.g. ckBTC, ckETH, ckUSDC>` (default ckBTC) and `&tab=deposit|withdraw` (**default withdraw** when absent). Page public; every action requires login (in-page "Log in" button → `login()`).
- **Purpose**: BTC ↔ ckBTC, ETH ↔ ckETH, ERC-20 ↔ ckERC20 through DFINITY chain-fusion minters, directly from the user's identity (no Sneed canister in between).
- **Information shown**: intro text; family buttons "Bitcoin" / "Ether" / "ERC-20"; for ERC-20 a "Token" select (`ckSYMBOL ⇄ SYMBOL`), list from ckETH minter `get_minter_info.supported_ckerc20_tokens` (anonymous query; decimals read from each ledger if unknown; fallback static list of 12 tokens: ckUSDC, ckUSDT, ckWBTC, ckEURC, ckXAUT, ckLINK, ckPEPE, ckOCT, ckSHIB, ckWSTETH, ckUNI, ckBAT); tabs "Get {ckSymbol}" / "Withdraw {native}"; local-replica warning; footer (main wallet account only; public on chain; approvals expire after an hour and show in your wallet; minter ids ckBTC `mqygn-…`, ckETH `sv3dd-oaaaa-aaaar-qacoa-cai`).
- **Actions**: switch family / token / tab (URL `replace`); "Log in"; link "wallet" → `/wallet`.
- **Entry points & cross-links**: **only** the token-card "Convert" button (ckBTC/ckETH/known ckERC20 cards) or direct URL — not in the Header nav.
- **Shared building blocks**: `ConvertUi`, `ConfirmDialog`, utils/ckMinters, utils/ethWallet, utils/ckAddress, `QrScanButton`, `revenueData.loadPrice/priceOf` (USD), `maskIfPrivate`.
- **Gating/state**: `isMainnet` = DFX_NETWORK ic **or staging** → staging talks to real mainnet minters (real BTC/ETH), unlike btc.html's testnet4 on staging. No feature flag/premium.
- **Redesign notes**: hidden page (no nav entry). ConvertButton opens it without `tab`, so it lands on "Withdraw" even when the user wanted to get ck tokens.

## Get ckBTC from any Bitcoin wallet (Convert → Bitcoin → "Get ckBTC")
- **Routes & params**: `/convert?asset=mxzaz-hqaaa-aaaar-qaada-cai&tab=deposit`. Login.
- **Purpose**: show the user's personal ckBTC-minter BTC deposit address and mint ckBTC for confirmed deposits.
- **Information shown**: "Bitcoin → ckBTC" + instructions; minter/address errors; deposit address (QR `bitcoin:` + copy, "Bitcoin mainnet only", "yours for good"); warnings: only BTC on Bitcoin network (not Lightning/BSC/Base), minimum deposit; "minter waits for N confirmations (~N×10 min)… nothing happens automatically", check fee. After check: result message (Minted X ckBTC / too small / tainted-quarantined / checked / "Nothing new to mint." / NoNewUtxos) and pending deposits list (amount, x / N confirmations). Advanced "deposits the minter knows" (amount, block, txid:vout → mempool.space), minimum confirmations, minter note.
- **Actions**: copy address; "Check for deposit" → `update_balance { owner = me }`. Other calls: `get_minter_info`, `get_deposit_fee`, `get_btc_address` (update; cached), `get_known_utxos`.
- **Entry points & cross-links**: Convert page tab; mempool.space links.
- **Shared building blocks**: utils/ckMinters (`ckbtcInfo`, `ckbtcDepositAddress`, `ckbtcKnownUtxos`, `ckbtcCheckDeposit`).
- **Gating/state**: localStorage `sneed_ckbtc_deposit_address_v1` (JSON map principal → address). Manual check only (no polling).
- **Redesign notes**: same minter address that btc.html "Move to ckBTC" pays into; btc.html auto-finishes, /convert needs the button. Merge candidate.

## Withdraw ckBTC to any Bitcoin address (Convert → Bitcoin → "Withdraw BTC")
- **Routes & params**: `/convert` (default tab) / `?tab=withdraw`. Login.
- **Purpose**: burn ckBTC and receive BTC at any Bitcoin address; follow all minter withdrawals of the account.
- **Information shown**: "ckBTC → Bitcoin" + description (usually < 1 h); amount with balance hint (masked) and ≈ USD; address with type hint/validation/scan note; summary "You burn", "Bitcoin network + minter fee (estimate)", "You receive about" (+ USD); plan errors (min, insufficient incl. approval fee, doesn't cover fees); irreversibility warning + minimum; Steps; success "Withdrawal requested (burn #N)"; Advanced (ckBTC ledger fee, Bitcoin fee / minter fee, minimum withdrawal, two-call explanation). "Your BTC withdrawals" history (top 20: Burn #N, status pill Queued/Signing/Sending/Submitted/Confirmed/Too small/Will reimburse/Reimbursed/Unknown, detail, txid → mempool.space).
- **Actions**: amount, "Max", address paste or QR scan; ack checkbox "I checked the address, and I understand this cannot be undone."; "Withdraw to Bitcoin" → ConfirmDialog "Withdraw to Bitcoin?" ("Withdraw"); history "Refresh". Calls: `icrc2_approve` minter on ckBTC ledger for exactly the amount, 1 h (recorded via `approveAndRecord`) → `retrieve_btc_with_approval`; `estimate_withdrawal_fee` (debounced); `retrieve_btc_status_v2_by_account` (auto every 30 s while any pending).
- **Entry points & cross-links**: ckBTC token card "Convert".
- **Shared building blocks**: utils/ckMinters (`ckbtcWithdraw`, `planBtcWithdrawal`, `btcStatusText`), `ConfirmDialog`, `QrScanButton`.
- **Gating/state**: localStorage `sneed_approval_spenders_v1` (approvals module records the minter as a known spender).
- **Redesign notes**: superset of btc.html "ckBTC → Bitcoin" (any address; minter-side history across devices) with a different confirm pattern (modal vs in-page review).

## Get ckETH / ckERC20 from Ethereum (Convert → Ether/ERC-20 → "Get …")
- **Routes & params**: `/convert?asset=<ckETH or ckERC20 ledger>&tab=deposit`. Login (manual tab needs login to show encoded principal).
- **Purpose**: deposit ETH or an ERC-20 on Ethereum mainnet and get the ck token credited to the principal.
- **Information shown**: "{native} on Ethereum → {ck}" + ~20 min finality note; minter error; method pills "Browser wallet" / "Deposit address"; Advanced "deposit by hand through the helper contract".
  - Browser wallet: discovery state; "No Ethereum wallet found…" guidance; connected "From (<wallet>)" account (etherscan link) + wallet balance (masked); not-mainnet warning; amount; "More than the wallet holds"; notice of 1 tx (ETH) or up to 3 tx (ERC-20: reset allowance, approve, deposit) to the minter's helper contract (address shown); Steps; sent tx hashes (etherscan links).
  - Deposit address: explanation; minimum; "cannot detect deposits at an address right now"; QR + copy; "Send at least", "Watched until <time>", "Deposit of X detected; the minter is sweeping it."; warning (mainnet only, minimum is cumulative, send before deadline, registration fee).
  - Manual: warning (+ minter dashboard link `https://sv3dd-….raw.icp0.io/dashboard`); copyable Helper contract, Function signature (`depositEth(bytes32,bytes32)` / `depositErc20(address,uint256,bytes32,bytes32)`), Token contract, principal (bytes32), subaccount (bytes32), call data (ETH).
- **Actions**: "Connect <wallet>" (EIP-6963 / window.ethereum); amount; "Deposit {native}" → ConfirmDialog "Deposit X?" ("Continue in wallet") → wallet signs `depositEth`/`approve`+`depositErc20` (switch-to-mainnet request); "Get my deposit address" / "Check status" → minter `deposit_eth` / `deposit_erc20` (update, Unsponsored mode; arms scanning); copy buttons.
- **Entry points & cross-links**: Convert page; etherscan links; minter dashboard.
- **Shared building blocks**: utils/ethWallet (wallet discovery, balances, deposits, receipts), utils/ckAddress (`principalToBytes32`, `subaccountToBytes32`, `encodeDepositEth`), `CopyValue`, `QrCode`.
- **Gating/state**: needs an injected Ethereum wallet for method 1; no localStorage.
- **Redesign notes**: three deposit methods for one goal; the manual one is expert-only and buried under Advanced. Only place in the app that talks to an external (Ethereum) wallet.

## Withdraw ckETH / ckERC20 to Ethereum (Convert → Ether/ERC-20 → "Withdraw …")
- **Routes & params**: `/convert?asset=<ckETH or ckERC20 ledger>` (default tab) / `&tab=withdraw`. Login.
- **Purpose**: burn ckETH/ckERC20 and receive ETH/ERC-20 at an Ethereum address; follow withdrawals.
- **Information shown**: "{ck} → {native} on Ethereum" + fee model (ETH: fee from amount; ERC-20: gas paid in ckETH) + ~20 min; amount with balance (masked), ≈ USD; "ckETH for gas: X" (ERC-20); address with "Checksum OK", validation, "The minter blocks this address"; summary (ETH: "Ethereum fee (at most)", "You receive at least"; ERC-20: "Gas, paid in ckETH (at most)", "You receive"); plan errors (min, insufficient, not enough ckETH for gas); warning (mainnet, 21,000 gas limit for ETH → avoid contracts, no exchange memo addresses); Steps; success "Withdrawal requested (id #N)"; Advanced: gas limit / max fee per gas (gwei), max transaction fee, ledger fees, explanation (20% gas margin). "Your Ethereum withdrawals" (top 20: token #id → recipient, pill Queued/Created/Sent/Done/Reimbursed/Reimbursing/Not found, detail, etherscan tx link).
- **Actions**: amount, "Max", address paste or QR scan (target eth); ack checkbox; "Withdraw to Ethereum" → ConfirmDialog "Withdraw to Ethereum?" ("Withdraw"); history "Refresh". Calls: `eip_1559_transaction_price` (every 60 s), `is_address_blocked`, ETH: `icrc2_approve` ckETH ledger → `withdraw_eth`; ERC-20: approve ckETH (gas + 20%) → approve ckERC20 → `withdraw_erc20`; `withdrawal_status { BySenderAccount }` (auto every 30 s while pending).
- **Entry points & cross-links**: ckETH / ckERC20 token card "Convert".
- **Shared building blocks**: utils/ckMinters (`ckethWithdraw`, `ckerc20Withdraw`, `planEthWithdrawal`, `planErc20Withdrawal`, `ethStatusText`), approvals `approveAndRecord`, `ConfirmDialog`.
- **Gating/state**: `sneed_approval_spenders_v1` (shared).
- **Redesign notes**: two ICRC-2 approvals for ERC-20 withdrawals are a UX wart; gas paid in a different token than the one withdrawn.

## "Convert" button on token cards
- **Routes & params**: rendered in `TokenCard` action row (between "Mint" and "Send") for ledgers matching `ckAssetForLedger` (ckBTC, ckETH, static ckERC20 fallback list). Navigates to `/convert?asset=<ledger>` (no `tab`).
- **Purpose**: entry point to /convert for a ck token.
- **Information shown**: label "Convert", tooltip "Convert between {ck} and {native}".
- **Actions**: click → router navigate (stopPropagation so card doesn't toggle).
- **Entry points & cross-links**: Wallet token cards (and anywhere `TokenCard` is rendered).
- **Shared building blocks**: utils/ckMinters `ckAssetForLedger`.
- **Gating/state**: none (no login check — card only shown in wallet context).
- **Redesign notes**: uses the static ERC-20 list, so ck tokens added by the minter after this release get no button even though /convert lists them. On the ckBTC card it sits next to "Move to my Bitcoin address" (btc.html) — two BTC exits.

## Slice observations

### (a) Entities, their views and their actions

- **Token (an ICRC-1 ledger in the user's server-side list)**
  - Views:
    - wallet TokenCard (grid, TokenCardModal, header quick wallet);
    - the same card in lock mode on `/lock/:id`, `/tokenlock` and `/tokenlocksoverview`;
    - Statistics charts and table; pocket rows;
    - `/transactions?ledger=` and `/transaction`;
    - `/scan_wallet` for any principal.
  - Actions:
    - list management: add by id, scan whitelist, remove (only at zero balance), reorder, hide dust, refresh;
    - moving tokens: send, receive (principal or account ID), Buy/Sell/Swap (SwapModal), Convert (ck tokens), wrap/unwrap (GLDT⇄sGLDT), Sneed Lock deposit/withdraw;
    - other: Mint (minting account only), lock, claim RLL rewards.
- **Balance location / pocket**
  - Kinds: Main, numbered or exact subaccounts, the Sneed Lock wallet (deposited free + locked), Sneedex escrows, ICPSwap unused pool balances, service balances (Sneed Monitor, Sneed Notifications), native BTC address.
  - Views: token card breakdown ("in N pockets"), the pocket switcher's single-pocket list, `/wallet/pockets`, PocketPicker inside the send dialog.
  - Actions: create (next #N or exact bytes), rename, hide/unhide, remove (forget), receive (ICRC-1 text, QR, ICP account ID), send, move between pockets, withdraw, top up, "Move all to Main", check ICP pocket #N, per-pocket approvals.
- **Recipient / account**: principal, principal + subaccount (hex, bytes, principal-shaped), ICRC-1 text, ICP account ID, name/nickname lookup, QR. Views: decoded parts, recipient balance, ICP account ID.
- **SNS neuron (wallet card)**: view stake, dissolve state, VP, maturity, disbursing; create, increase stake, set/increase dissolve delay, start/stop dissolving, disburse, disburse maturity, split, auto-stake toggle, send (permission swap), and a link to `/neuron`.
- **NNS neuron (ICP card)**: view; start/stop dissolving, disburse, spawn maturity, add/remove hotkey (controller only).
- **ICP neuron in a staking bot (Apps tab)**: view details; Manage → `/icp_neuron_manager`.
- **Token lock**: card Locks tab (id, amount, expiry, duration, gas, age, owner); create (LockModal), Link (`/tokenlock`), transfer ownership (as an NFT), Sell on Sneedex, add gas.
- **LP position**: Liquidity tab and PositionCard/PositionCardModal.
  - Actions: scan pools, add pool, remove an empty pool, "Move a position here", claim fees (3 meanings), withdraw the swap balance, withdraw to wallet, send/transfer, lock/extend, add liquidity (locked), Sell on Sneedex, manage on `/position`.
- **Collectible (LP fee, RLL reward, SNS maturity)**: the wallet's Collect card and the header notification, both opening ConsolidateModal; claim selected items in bulk.
- **ICRC-2 approval**: Approvals tab (Main), token badge and row, per-pocket panel. Actions: revoke, change amount/expiry, refresh, see "how these were found". There is no "create approval" anywhere.
- **App / canister** (Sneedapp bot, tracked canister): Apps tab cards and DappCardModal.
  - Actions: add/register, remove from list, refresh, top up with ICP, upgrade, transfer control, chores (start/pause/resume/stop/run now), move between Sneedapp and Other Apps.
  - Links: manage pages, `/canister`.
- **NFT collection / item / ICRC-37 approval / lock NFT**: `/nft`, `/nfts`, `/nft_item`, embeds in Sneedex and Feed.
  - Actions: add/hide collection, send (ICRC-7, EXT), approve/revoke spender, sell on Sneedex, use as avatar, mint (admins of custom collections).
- **Transaction (ledger block)**:
  - `/transactions` (raw log of one ledger), `/transaction` (one block, Previous/Next), and TransactionList inside the Principal/Me/Neuron pages (account history, SNS index only); CSV export.
  - The wallet itself shows no history.
- **Native BTC objects** (address, UTXO, pending send, move, signing credit): btc.html only. The wallet's Bitcoin card has receive and links.
- **ck conversions** (minter deposit address, withdrawal status): `/convert`.
- **Price / denomination / privacy mode**: app-wide formatters. Denomination is chosen on `/me`; privacy is a toggle on the wallet and in the header.

### (b) Overlaps with other areas of the app

- **Wallet vs the header quick wallet (PrincipalBox)**: a second implementation of tokens, sends (`WalletContext.sendToken`), TokenCardModal and PositionCardModal, with feature gaps and bugs. Wallet.jsx also re-fetches most of what WalletContext already holds.
- **Apps tab vs other bot/canister pages**:
  - Pages it duplicates: `/apps` (App Manager: groups, health, batch top-up), `/canister` (top up, controllers), `/icp_neuron_manager/:id` and the trading bot pages (chores, neurons).
  - The low-cycles and outdated-bot banners are shared through WalletContext with the header.
- **Neuron tabs on token cards vs neuron pages**: they overlap `/neuron`, `/me` "My Neurons", `/sns_neuron_wizard` (create) and the neuron slice's managers.
- **Liquidity tab vs other position pages**: it overlaps `/positions`, `/position` and the Lock Wizard. PositionTransferPanel/Guide appears in 4 places.
- **Locks tab vs Sneed Lock pages**: it overlaps `/lock/:id`, `/sneedlock_info`, `/tokenlock` and `/tokenlocksoverview`.
- **Buy/Sell inside the card vs `/swap`**: same engine, separate entry points.
- **Collecting**: the wallet's Collect card duplicates the header collectibles notification, and the claim code is written twice.
- **Pockets vs host pages**:
  - Sneedex escrows overlap `/sneedex_my` (bids and payments).
  - The Sneed Monitor and Sneed Notifications balances overlap their own top-up pages.
  - ICPSwap unused balances overlap PositionCard "Swap Canister Balance" and `/position`.
  - The Sneed Lock wallet overlaps the Deposit/Withdraw modals and the Lock flows.
- **Account and subaccount tools**:
  - SubaccountEditor is shared with `/tools/account`; four different subaccount input styles exist in total.
  - The account-ID toggle in the wallet duplicates the Account Tool.
- **Balances of any principal**: `/scan_wallet` duplicates the Principal page "Wallet" tab.
- **Ledger browsing**: `/transactions` overlaps `/tools/icrc3` and `/tools/money`.
- **BTC ↔ ckBTC**: available on both `/convert` (any wallet or address) and the btc.html Move tabs (own address). The ckBTC card shows "Convert" and "Move to my Bitcoin address" side by side.
- **NFTs spread across pages**:
  - Collection and item pages: `/nft` (wallet), `/nfts` (browser), `/nft_item`.
  - Sales and locks: Sneedex create/offers, and lock NFTs on `/lock/:id` and in the token card Locks tab ("Transfer Ownership").
  - Avatar picker on `/me`.
  - `/wallet` itself has no NFT section.
- **Settings that change the wallet live elsewhere**: `/me` holds the denomination, `collectiblesThreshold` and `alwaysShowRemoveToken`. The neuron/canister cycle thresholds come from NeuronManagerSettings.

### (c) Dead, hidden, half-built or confusing

- **Wallet.jsx is about 11.2k lines** and mixes tokens, DeFi, a canister fleet and allowances. The Apps tab alone is roughly 3k lines that re-implement App Manager features.
- **Hidden entry points**:
  - `/wallet/pockets` is linked only from the pocket switcher, which renders only once a pocket beyond Main and Sneed Lock exists. A new user can only create a first pocket from a token card's "New pocket" button.
  - `/convert`, `/btc.html`, `/scan_wallet`, `/nft_item` and `/transaction` have no menu entries.
  - Per-pocket approvals, "Check pocket #N", hosts status and the exact-subaccount creation exist only on `/wallet/pockets`.
- **Naming of the Sneed Lock subaccount**:
  - The same balance is called "Sneed Lock" (card row), "Deposited" (help docs), "backend wallet"/"frontend" (modals and errors) and a pocket (`h:sneed_lock:wallet`).
  - Deposit appears only on legacy rows and in the Lock flows; Withdraw appears in several places.
- **What "Scan" finds**:
  - Token Scan checks only whitelisted tokens on Main.
  - Position Scan finds only ICPSwap positions the principal owns directly.
  - The token list is a server-side registration list, not discovered holdings. Tokens also join it silently (tips received, LP fee claims, `ensureTokenRegistered` in other flows).
- **Dead code**:
  - In Wallet.jsx: the `SectionHeader` component, the `isSneedLockExpanded`/`sneedLockDisclaimerExpanded` and `consolidationExpanded` state (persisted to localStorage, never rendered), `showDebug = false` (token-card debug output unreachable), and unused `useLocation` and `PrincipalBox` imports.
  - In TokenCard: neuron-cache reads commented out "DEBUG: Disabled reading cached totals".
- **Removing tokens**: "Remove token" shows only at zero balance (unless the `/me` setting is on). There is no way to hide a token that still has dust. The confirmation reads "You are about to unregister ledger canister …?".
- **Claim Fees**: on an unlocked deposited position it also withdraws the position from Sneed Lock, and nothing says so.
- **Bugs in other parts (likely, read from code)**:
  - The quick wallet's position send is a silent no-op, and its token dialog lacks the deposit/withdraw/wrap handlers.
  - AddLedgerCanisterModal never shows registration errors.
  - TransactionList's error state never renders, and its account-mode sorts by Type/Time do nothing.
  - On the NFT item page there is no Send; the ICRC-7 send has no review step.
- **Prices**:
  - Prices come only from ICPSwap pools against ICP (`PriceService`), so tokens without such a pool have no value and are left out of totals.
  - The denomination can be any token, but it is set far away on `/me`.
- **Device-local state**:
  - The NFT collection list and all BTC send/move history live in this browser's localStorage, so they do not follow the user across devices.
  - By contrast, pockets, names and the wallet card order sync through canisters.
- **btc.html is a separate HTML app**: no SPA header or PWA, full page reloads between it and the app, and its own security headers. It is the only place that signs native BTC.
- **SKILL.md is out of date**: it lists the wallet tabs as "Tokens | Liquidity | Apps | Statistics". The code also has an **Approvals** tab, plus pockets and the Bitcoin card, none of which the guide mentions.

### Detail notes from the sub-areas

The sub-area notes below are kept verbatim for reference.

#### Token modals and approvals

**Entities and their views/actions**
- **Token (ICRC-1 ledger) balance, split into sources**:
  - Main = principal default account; Sneed Lock "backend wallet" = sneed_lock subaccount, free + locked; other pockets.
  - Actions: Send, Move, Deposit, Withdraw, Wrap/Unwrap (GLDT/sGLDT only), Lock, register, unregister.
- **Recipient account**: principal, ICRC-1 textual account (extended address), principal + subaccount (three formats), ICP account ID (ICP only); resolved from names or nicknames.
  - Views: validity, decoded parts, ICP account ID, recipient balance.
- **Pocket**: wallet subaccount, hosted, or service balance. Send-from, move-to, top-up, withdraw, create (from the picker), approvals per pocket.
- **Approval (ICRC-2 allowance)**: token, spender (named or unknown), amount, expiry, from-account, risk flags, verified/estimated. Actions: list, refresh, revoke, change (amount + expiry).
- **Collectible item**: position fee, RLL reward, or SNS neuron maturity. Actions: select, claim in bulk, per-item status.
- **Token lock / position**: transfer ownership (ICRC-7 NFT), send position, add liquidity to a locked position.
- **Registered ledgers and swap pools**: per-user server-side tracking lists (register through modals, unregister through cards).

**Overlaps with other app areas**
- Send execution exists three times: Wallet.handleSendToken, WalletContext.sendToken (quick wallet), and pockets `executeTransfer`. Claim logic exists twice: Wallet.jsx and useCollectiblesNotifications.
- Sneed Lock deposit/withdraw has two UIs: Deposit/WithdrawTokenModal, and pockets Move/Withdraw in SendTokenModal. Lock and position flows also deposit and top up on their own.
- Subaccount input exists four ways: SubaccountEditor, SendTokenModal's own, the 64-hex fields in the lock/position transfer modals, and PrincipalInput's dialog.
- Approvals have three views of one list: Wallet tab, token card row/badge, Pockets page. Approvals are created only inside other flows (wrap, SneedPad, Studio, DEX, CyclePay, ckMinters, Sneed Monitor, Notifications).
- PrincipalInput (name and nickname search) and QrScanButton are shared with other send/recipient flows. SwapPoolPicker and TokenSelector are shared with the Sneedex, LockWizard and Positions pages.
- Every simple modal (deposit, withdraw, wrap, lock transfer, position send) confirms through ConfirmationModal ("Confirm Action" / "Confirm" / "Cancel"). With `doAwait=false` the confirm closes immediately and errors appear in the parent modal.

**Dead, hidden, half-built or confusing**
- The quick wallet (PrincipalBox):
  - Position "Send" is a silent no-op that looks like success: WalletContext has no `sendLiquidityPosition`.
  - Its TokenCardModal gets no `handleDepositToBackend`, `handleWithdrawFromBackend` or `openWrapModal`. TokenCard calls these without guards, so the "Deposit"/"Withdraw" buttons in the legacy breakdown and "Wrap to sGLDT"/"Unwrap to GLDT" would throw a TypeError there. Plausible; follows from the props passed.
- AddLedgerCanisterModal closes in `finally`, so registration errors are never shown.
- SendTokenModal with the "Main" pocket explicitly selected refuses ICP account-ID recipients, with a message telling the user to pick Main.
- ConsolidateModal "Selected Totals" drops any amount formatted with thousands separators or masked by privacy mode.
- Claiming fees on an "Unlocked" Sneed Lock position withdraws the position from Sneed Lock, and the dialog does not say so.
- WrapUnwrapModal: "GLDT token not found. Please add GLDT…" contradicts the unwrap auto-register. The modal shows no balance.
- The move dialog keeps "Confirm Send" / "Confirm & Send" wording.
- Deposit and withdraw never name Sneed Lock and use "frontend/backend" jargon in errors.
- `handleDepositToBackend` / `handleWithdrawFromBackend` silently do nothing when the balance is no more than one fee.
- Hidden but useful: the recipient balance lookup (any account), ICP account-ID computation, extended-address conversion both ways, and "How these were found" approval coverage.

#### NFTs

- **Entities**
  - **Collection** (ICRC-7 or EXT canister)
    - Views: wallet card (`/nft`), browser header (`/nfts?collection=`), add-modal preview, NftAssetDetails header, picker chips (browser, Sneedex create, avatar picker).
    - Fields: id, name, symbol, logo, description, total supply, standards (ICRC-7/37/10; EXT), admin flag, user's token count, collection approvals.
    - Actions: add, remove/hide, refresh, browse, mint (admin), approve/revoke the whole collection, dashboard link.
  - **NFT item / token**
    - Views: NftCard (wallet), browser tile, `/nft_item` page, NftAssetIcon/Title/Details (Sneedex/Feed), avatar-picker thumbnail, CharacterPanel tile.
    - Fields: id or index, EXT tid, name, description, image, media (audio/video/3D), attributes, links, raw metadata, ICRC-97 document, owner (principal + subaccount, or EXT account id), "yours", soulbound binding, "carried by" character.
    - Actions: view, send (ICRC-7 / EXT), approve/revoke spender, sell on Sneedex, use as avatar, set 3D model from a game look, add its collection to the wallet, jump to it by id.
  - **Approval (ICRC-37)**: per token or per collection. Fields: spender principal and expiry. Actions: approve (spender plus optional days), revoke. Managed only in the wallet; read-only on the item page.
  - **Sneed Lock NFT**: lock contents and value, status, gas tank. Actions: send (3 places), add gas, sell on Sneedex (`?lock=`), open `/lock/:id`.
  - **Character (wu)**: game facts, inventory, look model, carried/kept items.
  - **NFT avatar**: an on-chain reference in sneed_pid_names, checked again on every display.
- **localStorage keys**: `nft_collections`, `nft_collections_hidden`, `ext_nft_collections`, `nft_browsed_collections`, `nft_avatars_v2`. In-memory session caches: NftAssetView's collection/token/standard caches, ExtUtils' `registryCache` and `metaCache`.
- **Overlaps with other app areas**
  - Wallet (`/wallet`) has **no NFT section**. Help says NFTs live on `/nft`; the wallet only sends locks as NFTs.
  - Sneedex: create pre-fill, offer views and escrow (ICRC-37 approve / EXT transfer).
  - Sneed Lock: `/lock/:id` LockNftPanel, LockGas, lock value model.
  - Profile/naming: `/me` avatar picker; avatars resolved inside PrincipalDisplay everywhere.
  - Feed (offer cards), DiamondClub swap, and help pages (HelpWallet `#nfts`, HelpTools `#nfts`, HelpNames, HelpSneedlock).
  - `/principal` pages show **no** NFTs held by a principal; the owner link only goes to `/principal?id=`.
- **Dead, hidden, half-built or confusing**
  - Two menu entries both named "NFTs": Hub → `/nfts` (browser, public) and Me → `/nft` (wallet, login).
  - Route names `/nft`, `/nfts` and `/nft_item` are easy to mix up. No route takes a collection or token in the path; the item page uses query params only.
  - No send on the item page. Approval management is buried in expanded cards.
  - The ICRC-7 send dialog has no review step and no name lookup; the EXT dialog has both.
  - Owner handling for subaccounts differs: the wallet lists only the default subaccount, while the browser and item page count any subaccount of my principal as "YOURS".
  - The "only mine" filter only applies to pages already loaded.
  - The ICRC-3 standard label in the browser and item page can never show (no `icrc3` key).
  - `Nft.jsx` has an unused `app_sneeddao_backend` import and an `error` banner that is never set.
  - Minting depends on the custom `get_admins`/`mint` interface ("demo / coming launchpad"); it is invisible to normal users.
  - The Sneed Lock collection is not built into the wallet, so lock NFTs appear there only if the user adds the collection by hand. Sending a lock from the wallet card skips gas handling.
  - The collection list is per-browser localStorage, not tied to the account, so it doesn't follow the user across devices.
  - No feature flags, ComingSoon or LaunchGate on any NFT route. The only gate is `namingServiceEnabled`, for avatar features.

#### Transactions and Scan Wallet

- **Entities and their views/actions**:
  - **Transaction / ledger block** (id = block index on a ledger):
    - List rows (TransactionList, both modes).
    - Detail (`/transaction`: Search, Previous/Next, Copy ID).
    - CSV row.
    - Raw block elsewhere: `/tools/icrc3` (ICRC-3 browser; not covered here).
    - Created by sends (SendReceipt "View transaction"), RLL distributions and claims, Rewards claims.
  - **Ledger / token**:
    - Selected on `/transactions` (TokenSelector, `?ledger=`) or implied by SNS (`?sns=`).
    - Metadata shown on `/transaction` (name/symbol/logo).
    - "Follow the money for this token →" (`/tools/money`).
  - **Account (principal + subaccount)**:
    - History in account mode (Principal/User "Transactions" tab with subaccount combo box and `?subaccount=`, Me tab with default subaccount only, Neuron tab with governance + neuron subaccount).
    - From/To filters accept principals, names or ICRC-1 accounts (PrincipalInput subaccount dialog).
    - Copy subaccount, Copy ICRC-1 Account (context menu), "Sub:" links to profile + subaccount.
    - Balances via `/scan_wallet` (main account only).
  - **Principal identity layer** (shared): names, nicknames and verified badges in every row and card. Context menu: "Send Message", "Set/Edit Nickname", "View Profile", "Copy Principal ID".
  - **ICP account identifier**: only on `/transaction` for the ICP ledger ("Account ID: <hex>"). Not linkable to a principal.
- **Overlaps with other app areas**:
  - `/scan_wallet` duplicates the Principal page Wallet tab ("Scan Tokens").
  - `/transactions` (raw log, page-local filters) overlaps `/tools/icrc3` (block log browser) and `/tools/money` (index-based flow graph). There are three different "browse a ledger" tools.
  - Account history in Principal and Me is also partially reimplemented by `utils/approvals.js`, `utils/rllLedger.js` (`/rll_ledger` rows), `tokensafety/sources.js` and `health/checks.js`, which all call `get_account_transactions` separately.
  - TransactionList and Transaction.jsx each have their own type-colour map, amount formatter and timestamp formatter.
- **Dead / hidden / half-built / confusing**:
  - **Error state never renders** in TransactionList: the error box sits inside `{!isCollapsed && !error && …}` (TransactionList.jsx:1076 vs :1553). Any fetch failure (for example ICP on `/transactions`, or a non-SNS ledger without `get_transactions`) shows an empty box.
  - **Type and Time sort do nothing in account mode**: the sort reads `a.kind` / `a.timestamp`, but index rows nest these under `.transaction` (TransactionList.jsx:686–707).
  - **Account mode reloads the whole history on every page or page-size change**: the effect depends on `page` and `pageSize` (TransactionList.jsx:774–777). The Neuron embedding also passes a new `Uint8Array` subaccount on every Neuron render (Neuron.jsx:2533–2536), which likely retriggers the full fetch.
  - **Ledger mode hides Prev/Next when the filtered page is empty** (pagination needs `displayedTransactions.length > 0`). With a filter active you can get stuck; only "Jump to" remains.
  - Ledger mode labels local transactions `startIndex + idx` instead of using `first_index`. A page that straddles the archive boundary likely mislabels ids.
  - **The subaccount combo box does not update `?subaccount=`**: only the in-row "Sub:" click calls `onSubaccountFilterChange`. Picking "Main account", a discovered subaccount, Enter, or "Clear filter" leaves the URL (and the Principal header "Sub:" chip) stale.
  - The subaccount placeholder says "All (type to filter or enter hex)", but empty means **main/default subaccount only**, not all subaccounts. The dropdown item correctly says "Main account (default subaccount)". There is no "all subaccounts" view.
  - **Canister principals' "Sub:" links go to `/canister?id=…&subaccount=…`, but `/canister` ignores `subaccount`**. Examples are the governance-canister subaccounts seen on neuron transfers. The link leads to a page with no transactions.
  - Name/nickname substring filtering uses display info computed only for the currently displayed rows, and the filter effect does not depend on it. Name matching is therefore unreliable; picking a principal from the PrincipalInput dropdown gives an exact match and works.
  - Mobile cards show no subaccounts, so the subaccount filter-by-click and copy are desktop-only.
  - `copyIcrc1Account` helper, `getFullDate` import, the `document.querySelector('input[placeholder="Jump to index..."]')` guard (the real placeholder is "index"), and the collapsible header props are unused.
  - TransactionList calls `list_sns_canisters` for an index even in ledger mode with a provided ledger; the result is unused there.
  - URL param `start` means "ledger index offset" on `/transactions` but "start principal" on `/tools/money`.
  - `/transactions` claims "any ICRC-1 token", but the UI only lists whitelisted tokens (TokenSelector without `allowCustom`). Arbitrary ledgers are reachable only via `?ledger=`.
  - The Hub card mentions "swaps", which this area never shows.

#### Native Bitcoin (btc.html) and Convert

- **Entities**:
  - *Native BTC address* (P2WPKH; derived offline in-browser from principal + signer pubkey/chaincode, path `[0x00, principal]`; prefix bc/tb; verified once per device/network via paid `btc_caller_address`) — views: SPA Bitcoin row Receive (copy/QR/derivation/pubkey), btc.html header; actions: receive, send, sweep, move.
  - *UTXO / coin* (txid:vout, value, confirmations; from IC Bitcoin canister free queries `ghsi2-…`/`g4xu7-…`, mined only) — views: SPA Advanced UTXO list, Send coin control, Received list; actions: coin selection, sweep.
  - *Pending send record* (localStorage; kinds send/sweep/toCkbtc/cpfp; states submitting/broadcast/unknown → confirmed/failed/replaced/dropped; RBF groups `group`/`replaces`/`supersededBy`; CPFP `parent`/`cpfpBy`; event log) — views: Activity Pending/Sent/send log; actions: re-send, mark dropped, ask mempool.space, speed up.
  - *Move* — BTC→ck (deposit info inside a send record: phase, confirmations, checks, mint block) and ck→BTC (own store: approve/burn blocks, minter status) — views: Activity → Moves; actions: Check / finish, Refresh, Show the minter's list, Dismiss.
  - *Signing credit* (user's cycles-ledger `um5iw-…` balance) and *signer allowance* (ICRC-2, 1 h, exact) — views: Signing credit tab; actions: top up from ICP (CMC mint), finish pending top-up, revoke.
  - *Pending CMC mint* (shared `sneed_pending_cmc_topups_v1`) — also `/tools/recover-topup`.
  - *Fee rates* (mempool.space tiers slow/normal/fast) — Send / Speed-up only.
  - *Balance cross-check* (`btc_caller_balance`, replicated) — header "Verify balance", automatic before large sends/sweeps (reused 10 min on same coins).
  - *Received entry* (device-local, max 500) and *explorer history tx* (opt-in mempool.space).
  - *ckBTC minter objects*: deposit address (`get_btc_address`), known UTXOs, update_balance statuses (Minted/ValueTooSmall/Tainted/Checked/NoNewUtxos), withdrawals (`retrieve_btc_status_v2(_by_account)`).
  - *ckETH minter objects*: deposit address with scanning status/deadline, helper contract, gas price, blocked address check, withdrawals (`withdrawal_status`); ck asset list (`supported_ckerc20_tokens`).
  - *Browser Ethereum wallet* (EIP-6963) — Convert deposit only.
  - *Network config* (mainnet / testnet4 / dev-mainnet / beta) with ck pair (ckBTC or ckTESTBTC).
- **Canisters touched**: Chain Fusion Signer `grghe-syaaa-aaaar-qabyq-cai` (`btc_caller_send`, `btc_caller_address`, `btc_caller_balance`; `btc_caller_sign` in IDL but unused), IC Bitcoin canister (`bitcoin_get_utxos_query`, `bitcoin_get_balance_query`; `get_blockchain_info` in IDL unused), cycles ledger, ICP ledger, CMC, ckBTC/ckTESTBTC minter + ledger, ckETH minter + ledgers. External HTTP: mempool.space (fees, tx lookups, address history), etherscan links.
- **Overlaps with other app areas**:
  - BTC↔ckBTC exists twice: /convert (any wallet/any address, manual deposit check, minter-side history) vs btc.html Move (own native address only, auto-finish, device-local records). Different networks on staging (convert = mainnet; btc.html = testnet4/ckTESTBTC).
  - ckBTC card shows both "Convert" and "Move to my Bitcoin address"; native BTC balance is folded into the ckBTC card total (mainnet only).
  - `ConvertUi` is a de-facto design kit used by DiamondClub, Drive, PocketRows, btc components — misnamed.
  - ICP→cycles top-up util shared with canister top-ups and `/tools/recover-topup`; ICRC-2 approvals made here feed the wallet approvals manager (`sneed_approval_spenders_v1`) but the signer's cycles-ledger allowance is managed only on btc.html.
  - `QrScanButton` shared with other SPA send flows; `parseBtcAddress` shared between btc.html and /convert.
  - Pockets: `btc_native` adapter is registered only when the flag is on; listed on `/wallet/pockets` as read-only chain address.
  - Confirmation patterns differ: btc.html in-page review + tick boxes (no popups by design); /convert ConfirmDialog modal + ack checkbox.
- **Dead / hidden / half-built / confusing**:
  - `sweepTo` prop of BtcSend (principal-migration wizard) has no caller; `assertSigningPrincipal` exported but unused; `btc_caller_sign` and `get_blockchain_info` declared, never called.
  - Stale copy: beta explainer says speed-up and sweeping "come later" (both implemented); `btc_native.js` header says "A1 is receive-only… sending (A2) and moves (A3) come later" (they exist on btc.html; adapter caps intentionally stay receive-only).
  - `main.jsx` comment says page "refuses to run… when the native-Bitcoin flag is off" — actually it renders the beta offer / "not available" notice.
  - Hidden: `#credit`/`#activity` only reachable from within btc.html; Sweep, coin control, fee-lookup opt-out, cross-check skip, explorer history, send log all under "Advanced"; Speed-up only after 2 blocks unless Advanced; developer mainnet via `btcnet` URL param (allowlisted principal); `/convert` has no nav link; ConvertButton lands on Withdraw tab by default.
  - All BTC send/move history is per browser (localStorage); clearing storage loses it, other devices don't see it, blocked storage blocks sends/moves.
  - Static ERC-20 list for ConvertButton vs dynamic list on /convert.
  - btc.html is outside the PWA (no manifest/service worker) and outside the SPA chrome; navigation between them is a full reload.
