# Apps, canisters, cycles, and token & developer tools

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Scope: `/sneedapp*`, `/apps`, `/canister`, `/canisters`, cycles top-up (dialogs, CyclePay, pending/recovery), `/tools/*` (main, account, icrc3, money, recover-topup, pad), Token Studio (`/studio*`, `/custody`), `/guardian`, plus the shared modals these pages open. Bot consoles (`/icp_neuron_manager/:id`, `/trading_bot/:id`) and `/create_icp_neuron` are another slice; only the links into them are noted here.

Header placement today (components/Header.jsx): "Sneedapp" menu = Browse Apps (`/sneedapp`), My Apps (`/apps`). "Sneed Me" menu also has Apps (`/apps`); `/apps` and `/canister` highlight "Sneed Me". "Sneed Hub" menu has Canisters (`/canisters`). "Sneed Tools" menu = Tools, Token Studio, Custody (`/custody`), Follow the Money, ICRC-3 Browser, Account Tool, Recover Top-up, + SNS Jailbreak / My Scripts (logged in only). "Sneed DAO" menu has Custody (`/guardian`). Admin menu: Sneedapp (`/admin/sneedapp`), Sneed Pad (`/admin/pad`), Sneed Monitor (`/admin/mon`); `/admin/canisters` (group limits) is only on the admin dashboard.

---

## Sneedapp store catalogue

- **Routes & params**: `/sneedapp`. Public browse; "Mint" and "My Canisters" need login.
- **Purpose**: browse app canisters (bots and other apps) that users can mint as their own canister.
- **Information shown**
  - Enabled apps from `sneedapp.getApps()` (disabled apps hidden), publishers from `getPublishers()`.
  - Per app card: icon (iconUrl / bot icon / first letter), name, publisher name + verified tick, "N minted" (`getMintLogCountForApp`, fetched one app at a time), family tags, description, mint price "X ICP", "Premium: Y ICP" when cheaper.
  - Family filter chips: "All" + each family tag across apps (only `sneed-bots` has an icon).
  - Two synthetic Sneed DAO cards when the pad canister exists (or any app has a `/studio…` mintUrl): "Token launchpad" ("N launched", fee "X ICP (Y Premium) + cycles" from `pad.get_config`) and "Index canister" (index fee). Both carry a "coming soon" ribbon (they lead into admin-only Token Studio tabs). Apps whose mintUrl starts with `/studio` are folded into the launchpad card. Hidden when a family filter is active.
  - "My Canisters (N)" (collapsible, logged in): `sneedapp.getMyWallet()` entries, re-typed by module hash: builds hash→appId from every version of every app (incl. disabled), then `getCanisterInfo` (backend `get_canister_info`) per canister; grouped by resolved app ("Unknown" if no match). Row: icon, canister id, "View" / "Manage" links from the app's `viewUrl` / `manageUrl` templates (`CANISTER_ID` substituted).
  - Logged out: "Connect your wallet to see your canisters and mint new ones" + "Connect Wallet".
- **Actions**
  - Click card → `/sneedapp/app/:appId`.
  - "Mint Bot" (family `sneed-bots`) / "Mint App": logged out → login; else navigate to app's `mintUrl` override or `/sneedapp/mint/:appId`.
  - "Mint a token" → `/studio/launch`; "Mint an index" → `/studio/index` (both gated).
  - "View" / "Manage" per owned canister (e.g. → `/icp_neuron_manager/:id`, `/trading_bot/:id`).
- **Entry points**: Header "Sneedapp → Browse Apps"; Hub; Products page; Help pages (`/help/sneedapp`, liquid staking, trading bot); breadcrumbs on detail/mint/publisher pages.
- **Shared building blocks**: `BotIcon`, `ComingSoonRibbon` (LaunchGate), `padIdl`/`launchSummaries`, `getCanisterInfo` (BackendUtils).
- **Gating/state**: none beyond login; ribbon only (cards still clickable).
- **Redesign notes**
  - "My Canisters" is a third copy of the user's Sneedapp canister list (also `/apps` → Wallet → Sneedapp, and Wallet → Apps tab), each with its own type-resolution code.
  - No search, no sort, no price-tier badge here (only on the mint page). Mint counts are N sequential calls.
  - Token launchpad / index "apps" are not real Sneedapp apps; they mix Token Studio into the store.

## App detail page

- **Routes & params**: `/sneedapp/app/:appId`. Public.
- **Purpose**: marketing + facts page for one app before minting.
- **Information shown**
  - Breadcrumb "Sneedapp / <name>"; icon; name; tagline; "by <publisher>" (+ verified tick) linking to `/sneedapp/publisher/:id`; "N minted"; latest version `vX.Y.Z` (latest with WASM).
  - Price "X ICP" and "Premium: Y ICP" when cheaper.
  - "About" (hero description or app description); "Read the full guide" (helpUrl).
  - Hard-coded rich content only for `sneed-icp-staking-bot` and `sneed-trading-bot` (APP_CONTENT in the page): "World First" splash (trading bot), "Key Features" grid, "Why choose this bot?" highlights. Other apps show just the description.
  - "Version History (N releases)": latest card collapsed; expand to see all versions: version, "Latest", "No WASM" tag, release notes. (Release date, wasm hash, wasm/source URLs exist in data but are not shown.)
  - Bottom CTA "Ready to get started?".
- **Actions**: "Mint Now" / "Mint for X ICP" (login, then mintUrl or `/sneedapp/mint/:appId`); "Read the full guide" / "Learn More" → `/help/icp-neuron-manager` or `/help/trading_bot`; publisher link; "Back to Sneedapp".
- **Entry points**: store cards; Hub, Products, Header, `utils/revenueData.js` links to `/sneedapp/app/...`; SKILL guide.
- **Shared building blocks**: `BotIcon`, `Header`.
- **Redesign notes**: marketing copy for the two bots lives in this page file, not in the app record. Publisher link leads to an owner-only console (dead end for everyone else, see below). No wasm hash / source link shown, so users can't verify what they mint here (Canister page and Token Studio have verification UIs).

## Mint an app canister (generic mint wizard)

- **Routes & params**: `/sneedapp/mint/:appId`. Login required ("Connect your wallet to continue" + "Connect Wallet"). Premium-affected price.
- **Purpose**: pay and create your own canister running a chosen app version, optionally with extra cycles.
- **Information shown**
  - Breadcrumb, publisher (+ verified), step bar Version → Review → Gas → Confirm.
  - Step "Select Version": versions with `vX.Y.Z`, "No WASM" (not selectable), release notes (first 200 chars); latest with WASM preselected.
  - Step "Review Payment": "Minting Price" = `getAppMintPrice(appId, me).applicable` + `PriceTierBadge` (Admin/Partner price via `mintPriceTier` → `get_special_prices_for`); premium member: "Premium Member Discount", struck regular price, "Save X ICP (N% off)"; non-premium: "Premium members pay less", "Premium price", "Become a Premium member" → `/premium`; "+ 0.0001 ICP transfer fee"; "Your Wallet Balance" (polled every 5 s) / "Need X ICP"; existing deposit in the payment subaccount ("You have X ICP already deposited…"); "Revenue split: N% Sneed DAO / M% <publisher>" (third-party publishers only).
  - Step "Extra Gas (Optional)": "Extra Gas Amount" ICP input, "≈ +N cycles", "Current rate: 1 ICP ≈ N cycles" (CMC `get_icp_xdr_conversion_rate`), insufficient-funds warning.
  - Step "Confirm & Mint": App, Version, Price, "Transfer from wallet" (shortfall + fee), "Extra Gas", Publisher, "Total from wallet"; progress messages ("Sending payment...", "Creating canister...", "Topping up canister with gas...").
  - Step "Canister Created!": new canister id (copy), gas top-up block "#N", or warning "Gas top-up not completed (the canister was still created): …" + `PendingTopUpsNotice`.
- **Actions**
  - "Next" / "Back" / "Skip"; "Mint Canister":
    1. if deposit < price: `icrc1_transfer` ICP → sneedapp canister, subaccount = `getPaymentSubaccount(me)` (shortfall; fee 0.0001).
    2. `sneedapp.mintCanister(appId, [major], [minor], [patch])`.
    3. if extra gas: `topUpCanisterWithIcp` (ICP → CMC subaccount + `notify_top_up`).
  - Friendly errors: PublisherNotFound, PublisherNotVerified, AppNotFound, AppNotEnabled, InsufficientPayment, TransferFailed ("Your deposit is safe…").
  - "View Canister" (app `viewUrl`), "Back to Sneedapp", copy id.
- **Entry points**: store card "Mint Bot/App", detail page "Mint Now", SKILL guide (trading bot path). Staking bot uses its own `mintUrl` (→ `/create_icp_neuron`, other slice).
- **Shared building blocks**: `PriceTierBadge`, `utils/sneedappPricing.mintPriceTier` + `utils/specialPrices`, `utils/cyclesTopUp`, `PendingTopUpsNotice`, `icrc1Transfer`.
- **Gating/state**: Premium price auto-applied; admin/partner special price badge.
- **Redesign notes**: two different mint flows for the two bots (generic wizard vs `/create_icp_neuron`). Payment uses a deposit subaccount (leftover deposits are silently reused, never refundable from the UI). No "My canisters" link at the end.

## Publisher console (manage apps you publish)

- **Routes & params**: `/sneedapp/publisher/:publisherId`. Login + must be one of the publisher's owners (else "You are not an owner of this publisher" + "Back to App Store"). Tabs `info | apps | versions | revenue` (state only, not in URL).
- **Purpose**: self-service management of a publisher, its apps' versions/WASMs, and revenue.
- **Information shown**
  - Header: name, "Verified", "Publisher #N"; success/error banners.
  - Info: description, website, "Payment:" default ICRC-1 account, "DAO Cut: N%"; Owners (truncated principals); Family Tags.
  - Apps: "Your Apps (N)": name, appId, numeric id, families, description, "Price: X ICP | Premium: Y ICP", enabled state.
  - Versions (per selected app): `vX.Y.Z`, "WASM (N bytes)" / "No WASM", release notes.
  - Revenue: "Publisher Revenue Balance", "Total Revenue", "Total Withdrawn", "Total Mints"; "Per-App Balances".
- **Actions** (all `sneedapp` factory calls)
  - Info: "Edit" → Name, Website, Description, "Default Payment Account (ICRC-1)" (`PrincipalInput` with subaccount) → "Save" (`updatePublisher`) / "Cancel".
  - Owners: "Principal to add" + "Add" (`addPublisherOwner`); × remove (`removePublisherOwner`; `window.confirm`, extra warning when removing yourself).
  - Families: "Family tag" + "Add" (`addPublisherFamily`); × remove.
  - Apps: "Enabled"/"Disabled" toggle (`setAppEnabled`). Creating apps is admin-only ("Ask an admin to add apps for your publisher").
  - Versions: "Select App"; per version trash (`removeAppVersion`, confirm) and file picker + "Upload WASM" (`uploadAppVersionWasm`, checks `\0asm` magic); "Add Version" form: Major, Minor, Patch, WASM Hash, Release Date, WASM URL, Source URL, Release Notes (`addAppVersion`).
  - Revenue: "Withdraw" publisher balance (`withdrawPublisherFunds`) and per-app "Withdraw" (`withdrawAppFunds`) → to the publisher payment account.
- **Entry points**: only the "by <publisher>" link on the app detail page. Not in any menu.
- **Redesign notes**: hidden page; public visitors following "by Sneed DAO" hit a "not an owner" wall (no public publisher profile exists). Uses `window.confirm` (the app has its own dialog elsewhere). Duplicates much of `/admin/sneedapp` (publisher/app/version admin).

## App Manager (/apps): overview, adding and tracking apps

- **Routes & params**: `/apps`. Route public; content requires login ("Please log in to track apps"). Premium-affected limits.
- **Purpose**: one board of every canister the user tracks, with cycle health, versions and quick links.
- **Information shown**
  - Hero "App Manager" with counts: "N custom" (groups), "N wallet" (tracked canisters), "N Sneedapp" (bots registered with the factory); "How it works" → `/help/dapp-manager`.
  - "Cycle Status" legend: Healthy / Low / Critical / Unknown, "Customize →" → `/me?tab=settings` (per-user red/orange thresholds, separate for bots and other canisters; utils/NeuronManagerSettings).
  - Overall status summary: "N Apps Total", "N critical / warning / healthy / unknown / outdated".
  - Per canister (all sections): type icon (from naming context canister types), crown "You are a controller", SNS pill, health lamp, name/nickname (`PrincipalDisplay` with copy), "⚡ cycles", "💾 memory", app id or "Known app version ✓" / "🔒 WASM unknown" (not controller) / "⚠️ WASM not recognized"; bots: version (⚠️ when outdated, tooltip "Newer version available"), "🧠 N neurons" (staking bot).
  - Data sources: management `canister_status` as the user (works when controller) → fallback backend `get_canister_info` (module hash only); SNS canisters' cycles from SNS root `get_sns_canisters_summary` (progressive, 3 s delayed, one root at a time); bots: `getVersion` / `getNeuronIds` on the bot; known hashes from all Sneedapp app versions (`getApps` + `getAppVersions`) and `getOfficialVersions`.
- **Actions**
  - "Add an app to track" (`PrincipalInput`, canisters) + "Add" → adds to Ungrouped (backend canister groups via `setCanisterGroups`); errors "Invalid app canister id format", "This app is already being tracked".
  - Per canister: edit icon → `/canister?id=`; bot icon → `/icp_neuron_manager/:id` / `/trading_bot/:id`; trash → inline "Remove?" "Yes"/"No".
  - Banners: "N bots can be upgraded to newer versions — Upgrade" (opens UpgradeBotsDialog); "N canisters low on cycles — Top Up" (opens TopUpCyclesDialog).
- **Entry points**: Header (Sneedapp → My Apps; Sneed Me → Apps); Hub; Me; Wallet ("/apps" link from Apps tab); Canister page "Back to App Manager"; Premium page; Create ICP neuron, Sneedex create, help pages.
- **Shared building blocks**: `PrincipalInput`, `PrincipalDisplay`/`getCanisterTypeIcon`/`SnsPill`, `usePremiumStatus`, `UpgradeBotsDialog`, `TopUpCyclesDialog`, `WalletLayoutContext` (shares ordering with Wallet), react-dnd.
- **Gating/state**: section collapse state in localStorage (`canisters_customExpanded`, `canisters_neuronManagersExpanded`, `canisters_walletExpanded`, `canisters_walletCanistersExpanded`).
- **Redesign notes**: three separate "lists of my canisters" stored in three places (backend canister groups, backend tracked canisters = "wallet", Sneedapp factory wallet), presented as sections of one page; users must understand the difference to know where an app lives. No per-row top-up or "Pay by card" (only via banner or `/canister`). Not reachable from the store's own "My Canisters".

## App Manager: groups (folders), subgroups and limits

- **Routes & params**: `/apps` → "Groups" section. Login. Limits differ for Premium.
- **Purpose**: organise tracked canisters into nested folders with health roll-ups.
- **Information shown**: "Groups" header with lamp + count; summary "N Apps", per-status counts (labels hidden on mobile), "used/limit folders", "used/limit apps", "PREMIUM" tag; each group: name, health lamp ("Group health"), contents, subgroups, empty message; "Ungrouped (N)" drop zone (counts SNS sub-canisters too). Limits from backend `get_canister_groups_limits_config` + `get_my_canister_groups_usage`.
- **Actions**
  - "New Group" (name + ✓/×); per group: "Add app to this group" (canister id input), "Add subgroup", "Rename group", "Delete group" (`window.confirm` "Apps will be moved to ungrouped"); "Expand" / "Collapse" all; remove app from group.
  - Drag & drop: reorder apps in a group / in Ungrouped (debounced save), move app between groups/Ungrouped, drag a group into another group (not into itself/children). Each change saves the whole tree (`setCanisterGroups`).
  - Over-limit save → "Upgrade to Premium" modal (message from backend, "Premium members get: Up to N folders (vs M) / apps per folder / total apps") with "Maybe Later" / "View Premium" → `/premium`.
- **Gating/state**: limits enforced by backend; admin edits limits at `/admin/canisters` ("Regular User Limits", "Premium User Limits").
- **Redesign notes**: moving and reordering are drag-and-drop only (react-dnd HTML5 backend, no touch backend, no "move to" menu) — not usable on phones. Code comments call groups a "Premium Feature" though everyone has them with lower limits.

## App Manager: SNS root folders

- **Routes & params**: `/apps` (any tracked canister that is an SNS root). Login.
- **Purpose**: tracking an SNS root shows the whole SNS as a virtual folder.
- **Information shown**: SNS logo + name, "SNS health" lamp, "View root canister"; sub-folders "System" (governance, ledger, swap, index, root), "Dapps", "Archives", each with health lamp and per-canister cycles/memory (from SNS root summary; works without being controller).
- **Actions**: expand/collapse root and sub-folders; drag the folder like an app; links to `/canister?id=`.
- **Redesign notes**: hidden capability (no hint that adding a root unfolds an SNS); a good "monitor a DAO's canisters" feature buried in a personal app list. SNS sub-canisters also feed the low-cycles banner, so users can top up a whole SNS.

## App Manager: Wallet section (Sneedapp bots and "Other Apps")

- **Routes & params**: `/apps` → "Wallet" section ("Open Wallet" → `/wallet`). Login.
- **Purpose**: the same canisters the Wallet's Apps tab shows, with health.
- **Information shown**
  - "Sneedapp" subsection: canisters in the Sneedapp factory wallet (`getMyWallet`); staking bots with a known hash render as bot cards (version, neurons, cycles, memory); others as canister cards with WASM-recognition text; inline "Health" summary incl. "outdated". Empty: "No Sneedapp canisters" + "Create your first ICP staking bot →".
  - "Other Apps" subsection: backend tracked canisters; detected bots render as bot cards. Empty: "No apps in wallet".
- **Actions**
  - Sneedapp: "Create" → `/create_icp_neuron`; "Add bot" (canister id) + "Add" → `sneedapp.registerCanister(id, '')`; remove → "Remove?" "Yes" → `deregisterCanister`.
  - Other Apps: "Add app" + "Add" → `registerTrackedCanister`; remove → `unregisterTrackedCanister`.
  - Drag between Groups / Sneedapp / Other Apps: add to destination first, then remove from source ("App moved to Sneedapp" / "App moved to groups").
  - Per row: manage link (bot console), details (`/canister?id=`).
- **Shared building blocks**: `WalletLayoutContext.sortByLayout` (shares the Wallet tab's ordering).
- **Redesign notes**: duplicates the Wallet → Apps tab (same lists, same add/remove, different UI: Wallet has DappCardModal, chores, send/transfer). "Create" here goes to the staking-bot page, not the store. "Add bot" registers any canister in the Sneedapp factory wallet (labelled "Manager added to list").

## Bulk bot upgrade dialog (UpgradeBotsDialog)

- **Where**: `/apps` "Upgrade" banner; Header outdated-bots pill; Wallet; PrincipalBox (header wallet dropdown).
- **Purpose**: upgrade several controlled bots (and pad-launched ledger suites) to the latest version in one go.
- **Information shown**: list taken when opened (stable while open), per bot current → target version, unknown-WASM warning and count, per-row result.
- **Actions**: "Select all" / "Deselect all" (unknown-hash bots unselected by default); upgrade: steps through every published version in order; wasm from the app version's `wasmUrl` (or SNS-W by hash for ledger suites) → management `install_code` mode upgrade; retry skips finished ones.
- **Shared building blocks**: used by 4 surfaces; `utils/ledgerSuite` for NNS-approved ledger/index hashes.
- **Redesign notes**: preconditions (controller, known hash) are only visible as banner counts.

## Bulk cycles top-up dialog (TopUpCyclesDialog)

- **Where**: `/apps` "Top Up" low-cycles banner; Header low-cycles pill; Wallet; PrincipalBox. Login (pays from the user's wallet).
- **Purpose**: top up many low-cycles canisters at once with ICP or ck-USDC.
- **Information shown**
  - "Top Up Cycles", "N canisters below critical level"; "Your ICP/ckUSDC Balance"; conversion rate (CMC, or CyclePay terms for ck-USDC).
  - ck-USDC note: "Through CyclePay … $min to $max per canister; each payment costs two ck-USDC ledger fees"; stale-rate warning.
  - Per canister row: checkbox, name, type label ("ICP Staking Bot"…), current cycles, "critical: N", planned "+X ICP", "Pay by card" link, status (spinner / ✓ "+N cycles · block B" / error tooltip).
  - Totals "Total ICP needed (incl. fees)", "All selected canisters are already at or above this level", "Insufficient balance".
- **Actions**
  - "Pay with ICP" / "Pay with ck-USDC" (mainnet/staging only).
  - "Top-Up Mode": "Top up each to a target cycle level" (input "e.g. 5T, 1T, 500B", presets "Critical" / "Healthy", smart prefill to the best affordable level; 10% buffer; min 0.001 ICP or CyclePay min), "Split ICP equally" (total + "MAX"), "Same amount for each" (per canister + "MAX").
  - "Select all" / "Deselect"; "Top Up N Canisters" → sequentially per canister: ICP `topUpCanisterWithIcp` (ICP → CMC subaccount memo TPUP, `notify_top_up`; fee 0.0001 each) or ck-USDC `topUpWithCkUsdc` (approve CyclePay + `topup_ckusdc`, waits ≤ 2 min; Delivered / Deferred "daily limit" / Refunded / Rejected). "All Done!" / "Done (x/y succeeded)".
- **Gating/state**: the caller passes the list. From `/apps`: canisters below the *critical* (red) threshold, non-controllers and SNS sub-canisters included (top-up is permissionless). From Header/Wallet/PrincipalBox (WalletContext): canisters below the *healthy* (orange) threshold, with severity critical/warning, controlled bots only.
- **Redesign notes**: the same dialog gets different "low cycles" lists depending on where it is opened (Apps: red only incl. non-controlled; WalletContext: orange+red, controlled), so counts in the Header pill and the Apps banner disagree. There is no "top up any canisters I pick" multi-select; a healthy canister can only be topped up from `/canister`.

## Pay by card / CyclePay links

- **Where**: `CardTopUpLink` in TopUpCyclesDialog rows and on `/canister` ("No ICP? Top up with a card, Apple Pay or Google Pay (CyclePay, $5 to $100)").
- **Purpose**: let anyone (no login) buy cycles for a canister by card.
- **Actions**: external link `https://www.cyclepay.org/c/<canister>/<usd=5>` (Stripe checkout; tooltip "card fee 2.9% + $0.30").
- **Redesign notes**: two CyclePay paths with different limits/fees (card link $5–$100 no login; in-app ck-USDC $1–$200 per terms, logged in).

## Pending top-ups and top-up recovery

- **Routes & params**: `PendingTopUpsNotice` on Wallet, `/canister` top-up section, Sneedapp mint result, `/icp_neuron_manager`, BotManagementPanel, `/tools/recover-topup`. `/tools/recover-topup` is public (notify works anonymously).
- **Purpose**: never lose ICP when a top-up's notify step fails.
- **Information shown**
  - Notice (only when records exist): "Pending cycles top-ups (N)": canister id, ICP amount, "block N" or "transfer not confirmed", created time, "last answer: …"; explanation "Finishing a top-up never sends ICP again."; link "Recover a top-up".
  - Recover page: explainer of the two-step CMC flow; result text ("Cycles minted: N cycles went to <canister> (block B)…" or normalized error); hint to find the CMC transfer (`rkp4c-…`) in wallet history or the ICRC-3 browser.
- **Actions**: notice "Finish top-up" (resend deduped transfer or re-notify stored block), "Dismiss" → "Yes, forget it" / "Keep" (warns to keep canister + block for recovery). Recover page: "Canister being topped up" (`PrincipalInput`), "ICP ledger block index of the transfer to the CMC", "Notify the CMC" (`notify_top_up`, sends no ICP).
- **Shared building blocks**: `utils/cyclesTopUp` (pending records in localStorage `sneed_pending_cmc_topups_v1`, event `sneed:pending-topups-changed`; created_at_time dedupe; final-outcome removal; same mechanism for `mintCyclesWithIcp` to the user's cycles-ledger account), `makeTopUpActors`.
- **Gating/state**: per-browser (localStorage); a pending record made by another identity is only re-notified.
- **Redesign notes**: recovery state lives only in this browser; the tool is buried in the Tools menu. Good shared "transaction safety" pattern that other flows could reuse.

## Canister detail page (/canister)

- **Routes & params**: `/canister?id=<canisterId>`. Public (public info); logged-in controllers get full status and management. Header SNS dropdown hidden.
- **Purpose**: inspect any canister and, if you control it, manage it (cycles, name, controllers, code).
- **Information shown**
  - "Canister Info Lookup" input + "Lookup"; hint "If you are a controller… full status will be shown" / "Login to see full canister status".
  - Access badge: "Controller Access" (management `canister_status` as the user) vs "Public Info" (backend `get_canister_info`: controllers + module hash only).
  - "Canister Details": type icon + SNS pill, canister id, user pill + "View User Profile" (if the id is a user) or "View Balances, Neurons & More" → `/user?id=`.
  - "View on External Sites": "ICP Dashboard", "Candid UI".
  - "Public Name" (verified tick) and "Your Nickname (private to you)".
  - Controller only: "Status" (running/stopping/stopped), "Cycles" (T + exact, colour by user thresholds), "Memory Size" (MB + bytes).
  - "Module Hash" or "No module installed".
  - "Controllers (N)" with names; self marked.
- **Actions**
  - Name: "Edit" → "Enter public name" → "Save" (`setCanisterName`; controllers only; "As a controller, you can set a public name visible to all users.").
  - Nickname: "Edit" → "Save" (`setPrincipalNickname`; any logged-in user).
  - "Top Up with Cycles" (any logged-in user): "Add Cycles"/"Cancel" toggle → "Current Rate", "Your ICP Balance", "Amount (ICP)" + MAX, "Estimated Cycles to Add", "Top Up Canister" → `topUpCanisterWithIcp` (success shows cycles + ICP block); `PendingTopUpsNotice`; card link.
  - Controller only: "Upgrade Canister" section: "Upgrade Mode" Upgrade (preserves state) / Reinstall (clears all; extra warning + confirm checkbox); "WASM Source" "Upload File" (`.wasm`/`.wasm.gz`, drag & drop) or URL + "Fetch WASM" (hint: raw.githubusercontent.com); "Init Arguments": None (empty) / Principal / Optional Principal / DIDC Blob / Raw Hex; button "Upgrade Canister" / "Reinstall Canister" → management `install_code`.
  - Controller only: "Add Controller" (principal) → `update_settings`; per controller "Remove" → "Confirm?" / "⚠️ Remove yourself?" "Yes"; last controller cannot be removed.
  - "Back to App Manager" → `/apps`.
- **Entry points**: Apps rows, DappCardModal "View Details", Wallet, Principal page, SNS page, Sneedex offer, bot health dialogs, token safety report leaves, Canister Explorer (via `getPrincipalProfileUrl`), SKILL guide.
- **Shared building blocks**: `CardTopUpLink`, `PendingTopUpsNotice`, `cyclesTopUp`, `PrincipalInput`, naming (`setCanisterName`, `setPrincipalNickname`), `getCyclesColor`.
- **Gating/state**: everything depends on whether `canister_status` succeeds as the caller.
- **Redesign notes**: no "add to App Manager / track" button (the main place users land after looking up a canister). Fetched but hidden: idle cycles burned per day; never fetched/shown: freezing threshold, reserved cycles, query stats, log visibility. No start/stop/delete, no settings editing beyond controllers. Upgrade has no hash check against Sneedapp/NNS versions (unlike Token Studio's verifier). TransferCanisterModal (wallet) is a separate controller-change UI.

## Canister Explorer (/canisters)

- **Routes & params**: `/canisters`. Public.
- **Purpose**: directory of canisters that have a public name or one of my nicknames.
- **Information shown**: "Canister Explorer"; "N named canisters", "N matching filter"; rows with `PrincipalDisplay` (name, nickname, verified) + "View Profile"; pagination "Page X of Y".
- **Actions**: top `PrincipalInput` "Search or enter canister ID to view..." → `/canister?id=`; "Filter:" by name/id; "Show:" Both / Public names only / Nicknames only; "Per page" 10/20/50/100; "Previous"/"Next".
- **Data**: purely the naming context (`principalNames`, `principalNicknames`) filtered to canister principals; no on-chain reads.
- **Entry points**: Header Sneed Hub → Canisters; Hub; help pages.
- **Redesign notes**: an address-book view of canisters, not an explorer (no cycles/status/type filter); overlaps `/users` and `/names`. Sort config exists (name asc) but no UI to change it.

## Dapp card modal (wallet app detail popup)

- **Where**: Wallet → Apps tab and PrincipalBox (header wallet dropdown). Not used by `/apps`.
- **Purpose**: quick detail and controls for one app canister.
- **Information shown**: icon, name, type ("Trading Bot" / "ICP Staking Bot" / "App"), crown; "Canister ID"; staking-bot "Version", "Neurons" count; "Cycles", "Memory"; staking bot "Staked ICP", "Maturity", "Neurons (N)" list (state Locked/Dissolving/Dissolved, stake; expand: dissolve delay, maturity, age, staked maturity); "You are a controller" / "Not a controller". Tabs "Info" / "Chores" (staking bots with chore data; lamps Scheduler / Conductor / Task, "Next: in …").
- **Actions**: refresh (auto once if cycles/memory missing); chores "▶ Start" / "▶ Resume" / "⏸ Pause" / "⏹ Stop" / "⚡ Run Now"; "Full Chore Management" → `/icp_neuron_manager/:id?tab=chores`; "Send" (controllers: opens transfer); "Manage Neurons" → `/icp_neuron_manager/:id`; "Manage Trading Bot" → `/trading_bot/:id`; "View Details" → `/canister?id=`; "Remove from Wallet".
- **Redesign notes**: chores control and neuron summary exist here but not on `/apps`; the same canister has different capability sets depending on which list you open it from.

## Transfer canister control

- **Where**: `TransferCanisterModal` from PrincipalBox; Wallet has its own inline implementation (same behaviour). Controllers only.
- **Purpose**: hand a canister/bot to another principal.
- **Information shown**: "Transfer Staking Bot/App", warning, canister name/id.
- **Actions**: "New Owner (Principal ID)" → "Transfer Control": `canister_status` → `update_settings` controllers = current minus me plus recipient (other controllers kept).
- **Redesign notes**: duplicated code (modal vs Wallet inline); `/canister` has add/remove controller but no one-step "transfer"; Sneedex sells bots/apps as another way to transfer.

## Sneed Monitor (proposal monitoring service) section

- **Where**: `SneedMonitorSection` on `/notifications` (other slice owns the page). Login. Renders nothing where `sneed_mon` has no canister id (prod today: staging-only).
- **Purpose**: paid off-app alerts for proposals the user's neurons can vote on (via Sneed Notifications).
- **Information shown**: "Sneed Monitor balance" (prepaid ICP, ≈ USD), watched targets and "Estimated cost ≈ X a month" + price-tier badge, paused state; Advanced: "Prices per check" (per frequency tier, per month), "Watches (N of max)" (frequency, topics, new proposals/reminders, reminder window, neurons registered, pending, next check), "NNS neurons", balance breakdown (deposited/charged/checks/withdrawn, per day, per round), "Charges" (paged, with tier, outcome, auto top-ups), auto top-up status (amount, threshold, allowance left/expiry, last failure).
- **Actions**: switch "Tell me about proposals I can vote on" (auto-allows the app in Sneed Notifications, watches SNSes with the user's neurons + NNS); "Watch them too"; "Top up" → "Top up from my wallet" (two ledger fees) or deposit address/account id + "I've sent it: check deposit" (`credit_deposit`); auto top-up (approve sneed_mon on ICP ledger as cap; "Allow" / "Change" / "Turn off"; optional subaccount); per watch: frequency select, "Topics: …" editor ("All", "Save topics"), checkboxes, unwatch, refresh neurons; add watch ("Choose an SNS…", NNS) + "Watch"; NNS neuron ids "Save" / "Use my NNS neurons (N)"; "Withdraw" (+ "All"); charges "Newer"/"Older"/"Refresh".
- **Redesign notes**: assigned to this slice but conceptually belongs with notifications/governance; it's another prepaid ICP balance (next to the notifications message balance). Admin at `/admin/mon`.

## Special prices card (admin) and price-tier badges

- **Where**: `SpecialPricesCard` on admin pages (`/admin/sneedapp`, `/admin/pad`, `/admin/sneedlock`, `/admin/sneedex`, `/admin/mon`, `/admin/premium`, SNS jailbreak admin) and `/guardian` (deployment admins). `PriceTierBadge` on Sneedapp mint, pad/index quotes, Guardian op fee, Sneed Monitor, and other slices' fee screens.
- **Purpose**: per-canister admin price and partner prices (principal, public label, per-fee overrides, expiry); the canister charges the lowest applicable of regular/premium/admin/partner.
- **Actions**: "Save admin prices"; partners "Edit", remove, add (principal, "Label (public…)", fee fields, "Expires (empty: no expiry)") "Save partner"/"Cancel".
- **Information shown**: badge "Admin price" / "Partner price" (tooltip with partner label) only when that tier is below the user's regular/premium price; Sneedapp uses `mintPriceTier`, the pad uses `padQuoteTiers` (pad config base fees vs `get_special_prices_for(me)`), Guardian uses `effectivePrice`.
- Card fallback: "This canister does not have special prices yet"; partner removal uses `window.confirm`.
- **Redesign notes**: all special-price config is public (`get_special_prices`) but never shown to users except as a badge on their own price; the same card is reused across 8 admin pages.

## Tools hub (/tools/main)

- **Routes & params**: `/tools/main`. Public.
- **Purpose**: landing page listing the tools as cards.
- **Information shown/links**: "Sneed Tools"; cards with "Free"/"Premium" badges:
  - Token Studio ("Free"): "Open Token Studio" → `/studio`, "Launch a token" → `/studio/launch` (gated).
  - Follow the Money ("Free"): "Open the graph" → `/tools/money`, "SNEED" preset.
  - SNS Jailbreak ("Premium"): "Launch Wizard" → `/tools/sns_jailbreak`, "My Saved Scripts" (other slice).
  - Account Tool ("Free"): "Open Account Tool".
  - ICRC-3 Browser ("Free"): "Open ICRC-3 Browser", "Sneed Lock log" preset.
  - "More Tools Coming Soon" placeholder.
- **Redesign notes**: no cards for Recover Top-up, Custody, `/tools/escrow` (coming soon page) though they are in the menu/routes. "SNS Jailbreak: Premium" badge vs the tool actually charging an ICP fee with premium discount (other slice). Feature bullets for Follow the Money overstate the standalone page.

## Account Tool (/tools/account)

- **Routes & params**: `/tools/account`. Public. No URL params.
- **Purpose**: convert/inspect ICP account formats and check a balance.
- **Information shown**: input modes "Principal" / "Principal + Subaccount" / "ICRC-1 Account"; "Account Summary": Principal, Subaccount (Hex, Bytes, "As Principal" with index, or "(default — all zeros)" / "Not interpretable as a principal-derived subaccount"), "ICRC-1 Account" (textual with checksum), "Legacy Account ID"; "Token Balance" for the account on a chosen token; collapsible help: Overview, ICRC-1 Accounts, Legacy Account Identifiers, Subaccounts (hex/number/text/bytes/principal formats), Principal-Derived Subaccounts, Quick Comparison table, Practical Advice (CEX → legacy id).
- **Actions**: type inputs; `SubaccountEditor` (multiple formats); `TokenSelector` → `icrc1_balance_of` + metadata; copy buttons per field; help toggle.
- **Shared building blocks**: `SubaccountEditor` + `utils/pockets/subaccount` (shared with Wallet Pockets), `utils/AccountParser`, `TokenSelector`.
- **Redesign notes**: only link-ins are Tools menu/hub/help. `SubaccountEditor` is shared with Wallet Pockets (`/wallet/pockets`) and `AccountParser` with the send-token modal; balance lookup overlaps `/scan_wallet`.

## ICRC-3 Browser (/tools/icrc3)

- **Routes & params**: `/tools/icrc3?canister=<id>&page=<n>&size=10|25|50|100&order=asc&view=specific|compact|verbose&block=<n>` (URL kept in sync; `block` opens its page with it expanded). Public; anonymous queries.
- **Purpose**: browse and verify the block log of any ICRC-3 canister.
- **Information shown**
  - Quick picks: Sneed Lock (+ staging), RLL, Sneedex, Sneed Premium, Sneedapp, Sneed forum, Sneed Hub backend, Sneed SMS, Sneed Notifications, SNEED, ckBTC, ckETH, ckUSDC ledgers.
  - Overview: Canister (named chip), Blocks (log length), Archives count + ranges, Token symbol/decimals (if ledger), Standards, Block types (with URLs), "Certified tip" result (block #, hash).
  - Blocks: card per block: #id, type pill (projection label e.g. "Token lock created", "Transfer", "Approve"), relative time, chain pills "linked" / "broken link" / "certified tip", one-line summary; expanded: Time, Block type, Parent hash, Block hash (computed), typed field table (principals with names → profile links, amounts with symbol/decimals, lock ids → `/lock/:id`, times), "JSON" raw view.
  - Views: "Specific" (projections for Sneed Lock's ~20 block types and ICRC-1/2 xfer/mint/burn/approve), "Compact" (guessed), "Verbose" (raw); falls back to compact with a notice.
- **Actions**: canister `PrincipalInput` + "Browse"; "Follow the money" (→ `/tools/money?ledger=`); "Verify tip and this page" / "Verify again" (certificate vs IC root key + phash chain on page); view switch; "Newest first"/"Oldest first"; page size; type filter "All types on this page" / "clear filter"; "Go to block #" + "Go"; "Expand all"/"Collapse all"; first/"Prev"/"Next"/last; copy buttons.
- **Entry points**: Tools menu/hub; Wallet token "Block log"; RLL activity card; Mr Sneed "Log"; Recover top-up hint; safety report code leaves; `/api/sneed_lock` docs link back.
- **Shared building blocks**: `utils/icrc3/client` (fetchOverview, fetchBlocks following archives, `verifyTip`, `hashValue`; `parsePrincipal` reused by Token Studio/Follow the Money), `utils/icrc3/projections`, `utils/icrc3/value`, `TokenIcon`, naming context.
- **Redesign notes**: overlaps `/transactions` + `/transaction` (ledger history views) and Sneed Lock/RLL activity views; the generic, verifiable view of every Sneed canister's audit log is buried under Tools.

## Frontend update detection (supporting utility)

- `utils/frontendCanisterUtils.js`: frontend canister ids (prod `pxtkg-…`, staging `2icdp-…`); polls `/version.json` buildId (asset canisters don't change module hash) for `FrontendUpdateContext` → Header "update available" indicator (setting `notifyUpdates`). No page of its own.

## Management-canister helper (supporting utility)

- `utils/management.js`: browser-side `canister_status` / `update_settings` routed to the target subnet (`readStatus`, `setControllers`) and `ledgerIndexUpgradeArg` (ICRC-106 announce). Used only by Token Studio HandoverPanel / IndexPanel; `/canister`, `/apps`, TransferCanisterModal, UpgradeBotsDialog each define their own management IDL inline.

## Install-as-app card (misfiled in this slice)

- `components/InstallAppCard.jsx`: Hub-only, phones only, not in standalone mode: "Sneed Hub as an app… How it works →" (`/help/app`); dismiss stored in localStorage `install_app_card_dismissed`. Unrelated to the app store despite the name.

## Admin pointers (not cataloged in depth)

- `/admin/sneedapp` (SneedAppAdmin): publishers (create, verify/unverify, DAO cut, owners, families), apps (add/edit), versions, immutable mint log, revenue/publisher stats, factory balances & aggregates, payment config, CMC rate, fee settings, canister creation cycles, premium discount, admins, Sneed governance, "Register Canisters for Users" + bulk JSON, special prices.
- `/admin/canisters` (CanisterGroupsAdmin): regular and premium App Manager limits (`update_canister_groups_limits`).
- `/admin/pad`: see Token Studio admin pointer below.

---

_Token developer tools (Token Studio, safety report, Harden, launch/index/custody, Follow the Money, gating). Routes: `/studio`, `/studio/:ledgerId[/:tab]`, `/custody`, `/tools/token_safety[/:id]` (redirect), `/tools/money`, `/tools/pad`, `/guardian`._

## Token Studio shell (token picker, tab bar, token header)

- **Routes & params**
  - `/studio` (no token, opens on Check placeholder), `/studio/:ledgerId` (Check), `/studio/:ledgerId/:tab` with tab ∈ `harden | handover | index` (any other tab value falls back to Check).
  - Pseudo-ledger segments: `/studio/launch` (Launch tab), `/studio/index` (Index tab without a ledger).
  - `/custody` = `<TokenStudio intent="handover">`: with no ledger it opens the Hand over tab; submitting a ledger goes to `/studio/<id>/handover`.
  - Query `?index=<principal>`: overrides index discovery for the Check report (set by IndexPanel's "Re-run the safety check with this index").
  - Hash `#locks|#liquidity|#control|#index|#minting|#cycles` on `/studio/:id/harden` scrolls to and highlights that Harden card.
  - Auth: public. Check and Harden need no login; Hand over / Index / Launch are gated (see "Coming-soon gating") and need login to act.
- **Purpose**: one place per ICRC ledger to rate it, see how to harden it, hand it to custody, add an index, or launch a new token.
- **Information shown**
  - Token header (when a valid ledger id is in the URL): logo (`icrc1:logo`) or 3-letter symbol placeholder, `name (symbol)` or "Reading the ledger…", ledger id linking to dashboard.internetcomputer.org.
  - On the Harden tab only: the live rating label/colour from the Check report, with "(still checking)" while running.
  - Placeholder text when no ledger: "Pick a token above, or paste any ledger canister id, to get its safety report."
- **Actions**
  - Token picker (`TokenSelector`, `allowCustom`): choosing a token auto-submits; "Check" submit button (spinner while the report runs). Invalid input → "That is not a principal".
  - Tab buttons: "Check", "Harden", "Hand over" (disabled without a ledger), "Index" (→ `/studio/<id>/index` or `/studio/index`), "Launch" (→ `/studio/launch`).
- **Entry points**: Header Tools menu "Token Studio" (`/studio`) and "Custody" (`/custody`); ToolsMain card "Open Token Studio" / "Launch a token" (`/studio/launch`); Hub featured products "Token Studio" (`/studio`) and "Sneed Custody" ("Hand over a token" → `/custody`, badge "Flagship"); SneedApp catalogue cards "Token launchpad" (`/studio/launch`, "Mint a token") and "Index canister" (`/studio/index`), both with ComingSoonRibbon; Wallet app-canister detail for icrc-ledger/icrc-index apps ("Safety report" and ledger id → `/studio/<ledger>`); Guardian children "safety check"; pad launch result "Check it", "Your launches" "check"; `/admin/pad` launches list; legacy `/tools/token_safety[/:id]` redirect; Help `/help/token-studio`, `/help/tools`.
- **Shared building blocks**: `TokenSelector`, `Header`, `health/TreeView` styles (`customStyles`, `primary`), `fetchTokenHeader` (checks.js), `parsePrincipal` (utils/icrc3/client), `ComingSoonGate`.
- **Gating/state**: Header highlights "Tools" section for `/studio*` and `/custody`. The Check panel stays mounted (hidden) on every tab once a ledger is set, so opening `/studio/<id>/handover|index|harden` silently runs the full safety scan in the background.
- **Redesign notes**: `/custody` and Header "Custody" (Tools menu) land on the studio hand-over tab, while Header "Custody" under the Sneed DAO menu goes to `/guardian` (console): same label, two pages. "launch"/"index" occupy the `:ledgerId` slot. TokenStudio fetches the token header itself and CheckPanel fetches it again (the `onHeader` prop exists but is unused).

## Token safety report (Check tab)

- **Routes & params**: `/studio/:ledgerId` (+ `?index=`). Public. `CheckPanel` is exported from `pages/TokenSafety.jsx` and only used by TokenStudio.
- **Purpose**: rate any ICRC ledger from public, verifiable chain data; tree of checks like `/health`.
- **Information shown**
  - Headline rating label + one-line note (see "Safety rating"), progress line: "Listing checks…", "Running X of Y checks, listing more…", "Scanning the block log: X of Y blocks.", or "N checks, finished <time> in S s" + "(kept from your last visit; run again for fresh data)".
  - Overall status badge (worst leaf), count badges: "N passed / warnings / failed / skipped / pending".
  - The check tree (root = "<name> (<symbol>), ledger <id>"): groups Gates, The DAO behind it, Minting, Supply and metadata, Distribution, Locked supply, Liquidity (ICPSwap), Cycles and memory. Each group row shows per-status counts and "listing checks…" while dynamic children load. Each leaf row: status icon, name, one-line summary, fix button (warn/fail only), elapsed ms, "open" link (internal hub page or external). Expanding a leaf shows its `details` object as pretty JSON (lists of holders, positions, locks, controller tree lines, etc.) plus, for non-passing code leaves, the "Verify a recompressed build" box.
  - Footnote explaining data sources (read_state for controllers/hashes, NNS-approved + DFINITY releases, browser-side log scan, Sneed Lock, ICPSwap, blackhole/SNS root/NNS root for cycles).
- **Actions**: "Run again"; "Problems only" (toggle: hides pass/skip rows); "Expand all"; "Expand problems" (opens failing/warning groups and leaves with details); "Collapse" (back to token+gates); "Thresholds" (opens config panel); "Follow the money" (appears once the block log was scanned; toggles the inline MoneyGraph on the scan data). Clicking a row toggles it. All reads are anonymous queries; no writes.
- **Shared building blocks**: `health/runner.js` (`runTree`: concurrent leaves, bounded to 6; dynamic groups listed in parallel; per-leaf timeout = `callTimeoutSec`+5 s, 20 min for scan-dependent leaves; memoised loaders per run), `health/TreeView.jsx` (`Node`, `Badge`, `STATUS`, `FixLink`, `ExternalOrInternal`) shared with `/health`; `MoneyGraph`; `VerifyRecompressed`; `scanLedgerOffThread` (Web Worker).
- **Gating/state**: in-memory report cache per (ledger, thresholds) for 30 min, survives in-app navigation, keeps expansion state; thresholds in localStorage. Mobile: TreeView CSS moves the summary cell under the name below 640 px.
- **Redesign notes**: this is the old `/tools/token_safety` page, now only reachable inside the studio. Hub links ("open") from leaves point out to ~12 other pages (canister, principal, ICRC-3 browser, transactions, transaction, money, token locks, position locks, SNS, neurons, neuron, proposals), so the report is a de-facto token hub.

## Safety checks: what each check inspects and how it is rated

Statuses: `pass`, `warn`, `fail`, `skip`, plus `pending`/`running` (`error` ranks as fail). A group's status = worst child (fail > warn > running > pass > skip). A leaf that throws becomes `fail` "Error: …". Thresholds in parentheses are config defaults (see "Thresholds").

- **Gates → Controllers** (one leaf per canister; reads controllers + module hash + `git_commit_id` metadata via IC `read_state`; link `/canister?id=` or `/guardian?deployment=&child=`)
  - "Ledger", "Archive <id>" per archive (archives from `icrc3_get_archives`, fallback legacy `archives()`; with none: "Archives" pass "No archive canisters yet"), "Index" (skip if none found).
  - Index discovery: `?index=` override (verified via the index's `ledger_id()`, error if it indexes another ledger) → ICRC-106 → SNS-W registry → launchpad record (`pad.get_launch_by_ledger`).
  - pass: no controllers; only Sneed custody canister (pass with "Rules: <policy>" if registered there via `get_child`, warn if not registered); only blackhole(s) (v0 `e3mmv…`, v2 `5vdms…`); only NNS root; only one SNS root (labelled "<DAO name> (SNS root …)").
  - Unclassified canister controllers: controller tree walked up to 3 levels / 20 canisters via read_state; all paths end in trusted roots → pass only if every controlling canister runs known-safe code (list is empty) otherwise warn "partial"; several trusted controllers → warn "any one can act alone"; tree reaches users/unreadable/unexplored → fail; trusted + other mixed → fail "Mixed controllers"; user principals → fail. Details include ASCII `controller_tree`, `policy`, `guardians`.
- **Gates → Code** (link `/canister?id=`; ICRC-3 ones link `/tools/icrc3?canister=`)
  - "Ledger wasm", "Archive <id> wasm", "Index wasm": fail no code; pass hash in SNS-W `list_upgrade_steps` ("NNS-approved version n of N"); pass official DFINITY standalone ledger-suite release (dfinityReleases.js: 84 hashes, 14 releases 2024-09-13 … 2026-03-09, incl. u256); warn a DFINITY release of another kind; pass a verified recompressed copy (repackaged.js: 1 entry, OhShii Launcher ledger); pass if NNS-root-controlled ("Installed by NNS proposal"); fail "Built from DFINITY IC commit … but not a DFINITY build known here" (links DFINITY SHA256SUMS); fail unknown wasm. Non-pass with a hash → Verify-recompressed box.
  - "Standards announced": fail no ICRC-1; warn no ICRC-3; pass list.
  - "Block log certified": skip without ICRC-3; pass "Certified tip at block N" (`verifyTip`); fail no tip certificate.
- **The DAO behind it (SNS governance)** (only if the token is an SNS ledger or its sole controller is an SNS root; else one skip leaf). Reads nervous-system params, all neurons (100/page up to `maxNeuronPages`), last 20 proposals, metadata.
  - "DAO": always pass; neurons, voting principals, age, total VP/stake, min dissolve delay to vote.
  - "Largest voting principal": fail no votable VP; fail ≥ 50%; warn ≥ 25% (top 10 listed). Link to principal profile.
  - "Influence after following": own VP + catch-all-topic followers (one level); fail ≥ 50%; warn ≥ 25%. Link `/neuron?neuronid=&sns=`.
  - "Stake that can vote": warn if < 50% of stake has enough dissolve delay (hard-coded).
  - "Genesis (developer) neurons": neurons created within a day of the first, with vesting; warn if share of stake ≥ 20% (reuses largest-holder warn threshold).
  - "Participation in recent proposals": warn no proposals; warn avg cast < 10%.
- **Minting**
  - "Minting account cannot mint at will": pass no minting account / ledger itself / anonymous / management canister / NNS system canisters / blackholes / SNS governance; fail user principal; canister → its controllers classified: pass or warn; warn uninspectable canister.
  - "Supply created outside the log" (scan): supply not explained by mint blocks; real balances of top-15 "deficit" accounts queried to estimate genesis seeding; pass ≤ 0.1% hidden; else fail if still-unseen share ≥ 50% else warn. Link `/tools/money?ledger=`.
  - "Mints after genesis" (scan): pass all mints in day 1; mints later → warn if > 10% of supply (hard-coded) else pass. Link `/transactions?ledger=`.
- **Supply and metadata**
  - "Token": always pass; name, symbol, decimals, fee, supply; details = full ICRC-1 metadata.
  - "Age" (scan): warn < 30 days since first block (hard-coded); first/last block timestamps.
- **Distribution** (needs the block-log scan; if skipped: one "Holders" skip leaf telling the user to raise "Max blocks to scan"). Tagged (excluded) accounts: minting account, Sneed Lock, the ledger, SNS governance/swap/root, ICPSwap pools.
  - "Holders rebuilt from the log": warn unknown block types; fail rebuilt balances exceed supply by > 0.1%; warn > 0.1% supply in never-mentioned accounts; pass. Details: blocks, accounts, owners, reconstructed vs reported supply, fee collector, tagged accounts, top 20 holders (+ locked amounts), records lost in archive.
  - "Largest holder": max(share, unseen share) ≥ 50% fail / ≥ 20% warn. Link to holder profile.
  - "Ten largest holders": ≥ 90% fail / ≥ 60% warn.
  - "Number of holders": warn < 100.
  - "Largest holder after folding satellites": satellites (single-sender, never-sent accounts) folded into funder; same thresholds as largest holder. Link `/tools/money?ledger=&start=<holder>`.
  - "Wallet-to-wallet fan-out": warn if a user wallet sent to ≥ 25 other wallets. Link `/tools/money?…&start=`.
  - "Where balances came from": % bought from a pool / minted directly / from genesis accounts / from other holders; warn if single-source dormant share ≥ 40%.
- **Locked supply** (Sneed Lock token locks with time left; SNS neurons' dissolve delay/vesting; SNS treasury = governance default account + derived treasury subaccount)
  - "Locked for at least the minimum time": Sneed Lock ≥ 90 d + neurons ≥ 90 d; warn nothing locked; pass ≥ 20% else warn; treasury shown but not counted. Link `/tokenlocksoverview?ledger=`.
  - "In Sneed Lock": skip none; else pass (count, %, year-weighted, 10 largest locks).
  - "In SNS neurons": skip non-SNS / no stake; else pass. Link `/neurons?sns=`.
  - "In the DAO treasury": skip no SNS / zero; else pass. Link to governance principal profile.
  - "Unlock cliffs ahead": Sneed Lock expiries + dissolving neurons within 30 d; warn ≥ 10% of supply. Link `/tokenlock?ledger=&locks=`.
- **Liquidity (ICPSwap)** (factory `getPools`, pool `metadata`/`getTokenAmountState`, ledger balances; ICP/USD from the deepest ICP/ckUSDC pool)
  - "Pool exists": fail none (link app.icpswap.com/liquidity); else pass with main pool (deepest ICP pair).
  - "Pool depth": skip no ICP pool / no ICP price; warn TVL < $1,000; pass TVL ≥ 5% of market cap or ≥ $50,000; else warn. Details: every pool, token price ICP/USD, market cap.
  - "Liquidity under safe control": pages positions (100/page, shrinks on instruction limit, ≤ 2000), inspects top 60; owner classes: Sneed Lock with an active lock ≥ 90 d, DAO (SNS governance/root, Sneed DeFi canister `ok64y…`), burned (anonymous, the pool), immutable (blackholes), other. pass ≥ 50% of in-range liquidity safe; warn some safe; otherwise warn if TVL ≥ min else fail; fail if positions can't be listed / none with liquidity; warn none in range. Details: per-position range, share, holder.
- **Cycles and memory** (one leaf per ledger/archive/index): observable via blackhole `canister_status`, SNS root `get_sns_canisters_summary`, custody `get_child` status, or NNS root; skip if not observable; fail < 0.5 T or not running; warn < 2 T; pass. Shows cycles, memory, state, "via".

## Safety rating (headline)

- **Information shown**: label + colour + note, computed by `rateTree` from group statuses:
  - "Rating…" while anything runs.
  - "Unverifiable" (red) if Controllers or Code group fails (note says which).
  - "Captured DAO" (red) if the DAO group fails.
  - Over the five dimensions minting, distribution, locks, liquidity, cycles: Controllers warn and ≤ 1 fail → "Fair"; 0 fails and ≤ 1 warn → "Strong"; ≤ 1 fail → "Fair"; else "Weak".
- Shown at top of Check and in the studio header on the Harden tab.
- **Redesign notes**: "Supply and metadata" and DAO warnings never affect the rating; the rating isn't persisted or exposed anywhere else (no rating badge on token lists/wallet).

## Report thresholds (config)

- **Route**: "Thresholds" button in the Check report.
- **Information/fields** (25 numbers, label: default): Max blocks to scan 500,000; Largest holder warning 20% / failure 50%; Top 10 warning 60% / failure 90%; Minimum holders 100; Locked supply good 20%; Minimum lock 90 days; Cliff window 30 days; Cliff warning 10%; Locked liquidity good 50%; Minimum liquidity $1,000; Good depth 5% of mcap; Deep liquidity $50,000; DAO one principal warning 25% / failure 50%; DAO following influence failure 50%; DAO low participation 10%; Max neuron pages 300; Dormant airdrop supply 40%; Wallet fan-out warning 25; Cycles warning 2 T / failure 0.5 T; Call timeout 45 s.
- **Actions**: "Save and re-run" (saves + re-runs), "Defaults" (reset draft), "Cancel". "Saved in this browser only."
- **State**: localStorage `sneed_token_safety_config_v1`; cache key includes thresholds.

## Fix suggestions on report lines

- **Information/actions** (button on a leaf only when it is warn/fail; `fixes.js`):
  - "Hand control over" → `/studio/<id>/handover` (leaf "Ledger").
  - "Add or hand over the index" → `/studio/<id>/index` (leaf "Index").
  - "Move to vetted code" → `/studio/<id>/harden#control` (leaf "Standards announced").
  - "What can be done" → `/studio/<id>/harden#minting` (minting account, mints after genesis).
  - "Lock tokens" → `/lock_wizard?type=token&token=<id>` (largest holder, top 10, folded largest holder, locked total, cliffs).
  - "Lock the LP position" → `/lock_wizard?type=position&token=<id>` (liquidity under safe control).
  - "Add liquidity" / "Deepen liquidity" → `/studio/<id>/harden#liquidity` (pool exists/none, depth).
- **Redesign notes**: dead mappings: `controllers`, `code`, `cycles` are group ids (fix buttons render only on leaves), `ctl:noarchive`/`lp:exists`/`locks:sneedlock` never warn/fail; archive controller, all wasm-hash and all cycles leaves have no fix. Handover/Index fixes lead to gated "coming soon" panels for non-admins.

## Verify a recompressed build

- **Where**: inside an expanded, non-passing code leaf (Ledger/Archive/Index wasm) of the report.
- **Purpose**: prove a module whose hash is unknown is an official DFINITY .wasm.gz recompressed.
- **Actions**: choose a local `.wasm.gz` file, or paste a URL + "Check URL" (only GitHub raw/objects/github.com and `*.icp0.io`/`*.ic0.app` allowed by CSP). Browser computes sha256(file) vs module hash, gunzips, sha256(wasm) vs every official release's raw hash.
- **Information shown**: mismatch / not gzip / not official / "Match: … Ask the Sneed team to add it to the verified list" / official but installed as another kind.
- **State**: nothing stored or sent; acceptance into `repackaged.js` is a manual code change by the team.

## Harden tab

- **Route**: `/studio/:ledgerId/harden[#section]`. Public.
- **Purpose**: per report dimension, what the owner can do, with buttons where the hub has an answer.
- **Information shown** (6 cards; id = hash anchor): Lock supply (`#locks`); Liquidity (`#liquidity`); Who controls the code (`#control`) with copyable `dfx … update-settings --set-controller <blackhole e3mmv…>` and `--set-controller <sns root>` commands, note on vetted code; Index canister (`#index`, badge "from the studio: soon"); Minting (`#minting`, text only: minting account is immutable); Cycles (`#cycles`, badge "ic-unicycle: integration coming").
- **Actions**: "Lock <SYM> in Sneed Lock" → `/lock_wizard?type=token&token=`; "Lock an LP position" → `/lock_wizard?type=position&token=`; "Create or grow a position" → `/position`; "Hand it to Sneed custody" → `/studio/<id>/handover`; "Create an index" → `/studio/<id>/index`; "Dashboard" (external IC dashboard).
- **Redesign notes**: the Index card still says "soon" though the Index tab exists (gated); several promises ("The studio will file these from here", "file that proposal with your own hotkeyed neuron next", ic-unicycle hand-off) are not built. Pure help/links page; Lock wizard and Position pages belong to other areas.

## Hand over a ledger suite to Sneed custody

- **Routes**: `/studio/:ledgerId/handover`; `/custody` (no ledger: prompt "Pick the ledger you control above…"). Gated by `ComingSoonGate` ("Hand over to Sneed custody"). Requires login; acting requires the logged-in principal to be a controller.
- **Purpose**: make a custody canister (sneed_guardian deployment) the controller of a ledger, its index(es) and archives, and register them with a veto-council policy.
- **Information shown**
  - Deployment picker (buttons, one per deployed custody canister: "Sneed Guardian", "Sneed Guardian Blackhole", "Sneed Guardian Multi-DAO") + blurb.
  - "The canisters": ledger, indexes (ICRC-106 + indexes the pad created + optional typed one), archives (legacy `archives()`); per row via management `canister_status` as the user: "already under <deployment>", "not readable: are you a controller?", "you are not a controller (controllers: …)", "code <hash>… is not an NNS-blessed <kind> version; upgrade it first" (checked against the guardian's `blessed` hash lists), "ready (N controllers)".
  - "Rules" summary line (`policySummary`), progress log, final "Under custody." message.
- **Actions**
  - Optional extra index (`PrincipalInput`); link "No index yet? Create one" → `/studio/<id>/index`.
  - Veto council: "Add a veto council member" (≤ 20; first defaults to me), trash per line; "Vetoes that block", "Go-aheads required", "Veto window (days)" (0–90); checkbox "Follow the NNS: upgrade to each newly blessed version without a vote" (default on); checkbox "Afterwards, leave <deployment> as the only controller" (default on).
  - "Hand over N canister(s)": validation (principals, counts ≤ council, window ≥ 1 day when vetoes > 0); ledger first; per canister: management `update_settings` (add custody canister as controller) → `guardian.register(canister, kind, policy, suite?, "<SYM> <kind>")`; then, if step-down, `update_settings` controllers = [custody]. Free (no fee). No confirmation dialog before stepping down.
  - After: "See the rules and status" → `/guardian?deployment=<key>&child=<ledger>`; "re-run the safety check" → `/studio/<id>`.
- **Shared building blocks**: `PrincipalInput`, `utils/management.js` (`readStatus`, `setControllers`), `guardianIdl` (`GUARDIANS`, `policySummary`), `padIdl.get_indexes_of_ledger`.
- **Gating/state**: "No custody canister is deployed in this build yet." when no `CANISTER_ID_SNEED_GUARDIAN*` env ids; all three deployments are `live: false`.
- **Redesign notes**: policy editor (council, vetoes, go-aheads, window, auto-upgrade) is duplicated in LaunchPanel and Guardian "Change rules". Archives read via legacy method only (unlike the report).

## Index launcher (create an index for an existing ledger)

- **Routes**: `/studio/:ledgerId/index`, `/studio/index` (with its own TokenSelector; choosing navigates to `/studio/<id>/index`). Gated ("Index launcher"). "Get the price" needs login.
- **Purpose**: pad creates an ICRC index canister on NNS-blessed index code pointed at any ledger; then optionally announce it via ICRC-106.
- **Information shown**: "Announced by the ledger (ICRC-106)", "Created here before" (`pad.get_indexes_of_ledger`), custody note if the ledger is a custody child (`get_child` on each deployment). Quote table: "Index fee" (+ ", Sneed Premium member rate"), "Handover fee", "Cycles for the index (N T)", "Ledger fees", "Total" (ICP), each fee with `PriceTierBadge` (admin/partner). Result: index id, ledger, controllers, "It is listed in your wallet."
- **Actions**
  - "Who controls the index" radio: "Me"; "The blackhole"; "<deployment>, like the ledger" (if ledger is under custody; reuses the child's policy) or "Sneed custody" (+ deployment select if > 1; default policy: no council, auto-upgrade); "Sneed DAO"; "Another SNS DAO" (+ SNS root PrincipalInput).
  - "Get the price" → `pad.get_index_quote` (signed) + `padQuoteTiers(index)`.
  - "Approve and create the index" → `icrc2_approve` on ICP ledger (spender pad, total + 0.0001 ICP fee; spender recorded in localStorage via `approveAndRecord`) → `pad.create_index`.
  - If under custody: "Stage the announcement op" → `guardian.stage_op(ledger, install{current module hash, ICRC-106 upgrade arg})` → link "Pay its fee and propose approval" → `/guardian?deployment=&child=`. Else: copyable `dfx canister install --mode upgrade … index_principal …` command + link "hand the ledger over".
  - "Re-run the safety check with this index" → `/studio/<ledger>?index=<new index>`.
- **Gating/state**: "The launchpad canister is not deployed in this build yet." without `CANISTER_ID_SNEED_PAD`.
- **Redesign notes**: no "your index launches" list though the pad has `get_index_launches_by_creator`/`list_index_launches`; "Sneed DAO" control says "register it with your neuron afterwards" but (unlike LaunchPanel) offers no proposal filing; custody choice can't set a council here. Wallet listing relies on pad-side registration only (LaunchPanel also calls SneedApp itself).

## Token launcher (launch a new ICRC token)

- **Routes**: `/studio/launch` (gated "Token launcher"); legacy `/tools/pad` (page-gated, then redirects to `/studio/launch`). Launch needs login; quote works anonymously (signed when logged in).
- **Purpose**: configure and mint a DFINITY ledger-suite token (+ optional index) via `sneed_pad`.
- **Information shown**
  - Pad config banner: "Fee X ICP (Y for Sneed Premium members)", "Handing control to Sneed DAO X more", ledger/index starting cycles, ledger code source ("the latest NNS-approved version, checked daily" / "NNS-approved, pinned by the DAO" / "not loaded yet", "last check failed"), "paused".
  - Running total supply of allocations; validation problem list; quote table: "Launch fee" (+ premium note; "to the DAO's reward loop") + badge, "Handover fee" + badge, "Cycles for the ledger and index (N T)", "Ledger fees", "Total (plus one approval fee)".
  - DAO-control neuron check: "You hold N neuron(s) here that may propose…" / "You hold no neuron in this DAO that may propose…" / "Log in to check your neurons".
  - Result: "<name> (<SYM>) is live", ledger/index ids, controllers, custody registration link, registration proposal link `/proposal?proposalid=&sns=` or error.
  - "Your launches": newest 50 by creator (lazy logo via `get_launch_logo`), name/symbol, ledger id, failed reason or date, "check" → `/studio/<ledger>`.
- **Actions / form**
  - 1. The token: Name (≤ 64), Symbol (≤ 12, upper-cased), Decimals (0–18), Transfer fee (in tokens), Logo file (≤ 150 kB; data URL ≤ 200 kB).
  - 2. Who gets the supply: allocation lines (principal/name + amount; first defaults to me), "Add an allocation", trash (≥ 1 required; minted as mint blocks).
  - 3. The rules: Minting after launch "Nobody" (minting account = ledger) / "Sneed DAO" / "A principal" (+ principal). Controller "Me" (+ "Add a controller next to me", ≤ 10 → `keep_with`) / "The blackhole" / "Sneed DAO" (handover fee) / "Another SNS DAO" (known-SNS select from `getAllSnses` + root input) / each custody deployment (veto council ≤ 20 with "Add a veto council member", "Vetoes that block", "Go-aheads required", "Veto window (days)", "Follow the NNS…" checkbox, Rules summary). If DAO control and no proposal-capable neuron: launch disabled + "Do that" (switch to Me + DAO root as extra controller). Extras: "Index canister" checkbox (default on), optional "Fee collector".
  - 4. "Get the price" → `pad.get_quote`; "Approve and launch" → no-logo warning ("Add a logo first" / "Launch without a logo") → `icrc2_approve` ICP (total + fee) → `pad.create_token` → if DAO control: `manage_neuron MakeProposal RegisterDappCanisters` from the first proposal-capable neuron → `backend.register_ledger_canister_id(ledger)` + `sneedapp.registerCanister(ledger,'icrc-ledger')` (+ index `'icrc-index'`).
  - "Name them in Sneed names": public names (only when control = Me) "Set public names" (`setCanisterName`), private nicknames "Set private nicknames" (`setPrincipalNickname`), max 32 chars; refreshes naming context.
  - "Check it" → `/studio/<ledger>`; "Dashboard" (external).
- **Shared building blocks**: `PrincipalInput`, `PriceTierBadge`, `padQuoteTiers`/`specialPrices`, `approveAndRecord`, `LaunchLogo`, `getAllSnses`, `BackendUtils` naming, `useNaming`.
- **Redesign notes**: `onLaunched` prop is never passed, so the studio doesn't switch to the new token's report. Post-launch naming duplicates the generic naming feature. Locking allocations is deferred to Harden/Lock wizard.

## Follow the money graph (shared component)

- **Where**: inline in the Check report ("Follow the money", scan-backed) and on `/tools/money` (index-backed).
- **Purpose**: explorable directed graph of who sent the token to whom (principals; subaccounts folded).
- **Information shown**: nodes sized by balance share, coloured by kind (holder, DEX pool [square], Sneed Lock, DAO/treasury, minting, burned [diamonds]; scan-only: hub = funded ≥ 5 dormant accounts, spray = wallet sent to many wallets, satellite, genesis account); dashed ring = not expanded; labels from Sneed names/nicknames/verified names. Node tooltip: name, id, tag, balance + % supply, received/sent totals and counterparty counts, "(long history, most recent only)", satellites funded, wallets sprayed, "satellite of". Edge tooltip: from → to, amount in N transfers, date range, reverse flow. Counts "X of N principals, M edges" (scan) or "X shown, Y expanded" (index). Legend.
- **Actions**: "Start at a principal or name" (`PrincipalInput`) + "Go"; "Start from…" select (Minting; Funders of dormant accounts; Largest holders — scan only except Minting); "Fold satellites" (scan only); "Prune" (remove unexpanded nodes); "until" date slider (scan only); click = expand (12 neighbours, max 400 nodes), shift-click = fold, alt/ctrl/cmd-click = remove, double-click = open principal profile, drag, wheel zoom, pan.
- **Shared building blocks**: `useNaming`, `getPrincipalDisplayInfoFromContext`, `getPrincipalProfileUrl`, `PrincipalInput`, `sources.js` (`scanSource`, `indexSource`).
- **Redesign notes**: mouse/wheel handlers only (no touch pinch/pan) and a fixed 320 px start input; canvas 560 px high.

## Follow the Money page

- **Route**: `/tools/money?ledger=<id>&start=<principal>`. Public.
- **Purpose**: same graph for any ledger with an ICRC index, read lazily per node (no scan).
- **Information shown**: token logo/name/symbol, status ("Looking for the index canister…", "Index <id>. Labelling pools…"), errors (no index → suggests the token safety page). Tags: Sneed Lock, SNS governance/swap/root, ICPSwap pools, minting account.
- **Actions**: TokenSelector (writes `?ledger=`); graph actions above. Expanding a node: index `list_subaccounts` (≤ 25) + `get_account_transactions` (100/page, ≤ 1000 tx per account).
- **Entry points**: Header Tools "Follow the Money"; ToolsMain card ("Open the graph", "SNEED" preset); ICRC-3 browser "Follow the money"; Transactions page "Follow the money for this token →" (with `&start=<me>`); report leaves' "open" links (`/tools/money?ledger=[&start=]`); HelpTools.
- **Redesign notes**: ToolsMain advertises "Satellites and fan-out folded" for this page, but folding/hub/spray/genesis overlays and the time slider exist only in the scan-backed report graph. Changing the start node doesn't update the URL (`start` read once).

## Custody console (/guardian)

- **Route**: `/guardian?deployment=<guardian|blackhole|multidao>&child=<canister>`. Whole page behind `ComingSoonPage` ("Sneed custody"). Read public (for admins of the site, given the gate); actions need login; some need roles.
- **Purpose**: per custody deployment: config, children, ops (staging, approving, veto/go-ahead, execute, pay, refund).
- **Information shown**
  - Deployment switcher + reload. Config card: blurb, Canister, Approvers "T of N", Children, Op fee (premium), Own cycles, Blessed versions (ledger/index/archive counts, error), Status run time, approver list, "paused".
  - Children: title, kind, id, "safety check" → `/studio/<suite ledger or id>`, dashboard link; cycles (red if under low threshold), running, module hash, read time or status error; Rules + council members; last upgrade / last upgrade error; "N can move to a newer blessed version". `?child=` highlights that card.
  - Ops (open by default; "show closed ops"; "Load older ops"): #id + description, state (pending/approved/vetoed/executed/failed: msg/expired/cancelled), staged time/by, approvals x/T, go-aheads, vetoes, window end, memo, fee paid/due/refunded + payer. Role notes ("You are an approver/admin… your approval alone counts as the whole threshold"), admins list ("trial period"), approve_op function id.
- **Actions**
  - Per child (logged in): "Change rules" (stage `set_policy`), "Change controllers" (stage `set_controllers`; warning that omitting the deployment releases custody), "Upgrade to <hash>" (only if upgradable and auto-upgrade off; stage `install` next blessed hash), "Read status" (`refresh_status`, button shown to anyone).
  - Stage form: council lines, vetoes/go-aheads/window, auto-upgrade, controllers list, Memo (≤ 500); shows op fee via `effectivePrice` + `PriceTierBadge` + premium note; "Stage" / "Cancel"; fee > 0 opens payment panel.
  - Per op: "Approve (you are an approver)" (`approve_op`); "Propose approval" / "Pay and propose approval" (needs registered function id + a Sneed neuron with SubmitProposal: pays due fee by `icrc1_transfer` to the op's payment account, polls payment, `validate_approve_op`, then Sneed governance `manage_neuron MakeProposal ExecuteGenericNervousSystemFunction(function_id, candid(op id))`); "Go-ahead" / "Veto" (council members: `ok_op`/`veto_op`); "Pay the fee" → panel ("Send X ICP … to the op's payment account", "Pay from my wallet", "Close"); "Execute" (approved; anyone; `execute_op`); "Cancel" (stager or approver; `cancel_op`); closed op with fee: "Refund what is left to the payer" (`refund_op`).
  - Admin of the deployment: Special prices card (Op fee), "remove me" (`admin_remove_admin`), approve_op function id "set" (app prompt), "Run status now", "Run auto-upgrade now", "Re-read blessed versions".
- **Entry points**: Header Sneed DAO menu "Custody"; report controller leaves for custody-controlled canisters; HandoverPanel/LaunchPanel/IndexPanel result links; `/health` custody checks (only `live` deployments: none today); Guardian header links "Launch a token under custody" → `/studio/launch`.
- **Redesign notes**: mixes a public status board, a member action console and admin ops. Guardian IDL capabilities without UI: staging `set_config`/`set_approvers` ops (displayed only), `get_text_log`, rate limits, `get_cycles_report` (used by /health), `list_ops_for`, `get_op`.

## Mint from the wallet (MintTokenDialog)

- **Where**: wallet `TokenCard` — coin icon in the collapsed row and a "Mint" button in the expanded card, only when the logged-in principal's default account is the token's minting account (`useIsMinter`, minting account cached per ledger per session).
- **Purpose**: mint more of a token you are the minting account of (ICRC-1 transfer from the minting account).
- **Information shown**: explanation (supply grows, no fee), success "Minted X SYM to <principal>, Ledger block N".
- **Actions**: Recipient (`PrincipalInput` with subaccount option), Amount; "Mint X SYM" (`icrc1_transfer`), "Cancel", "Mint more", "Done".
- **Redesign notes**: the only post-launch token-admin action outside the studio; not linked from the studio or the "Minting" check (which fails exactly such tokens).

## Ledger-suite upgrades for pad-launched canisters (wallet)

- **Where**: wallet app canisters whose app id is `icrc-ledger`/`icrc-index` (SneedApp registrations from the pad). `utils/ledgerSuite.js` lists NNS-approved ledger/index hashes (SNS-W, cached 1 h), treats them as known versions, and `UpgradeBotsDialog` fetches the wasm bytes from SNS-W (hash-verified) to upgrade.
- **Information shown** (Wallet app-canister detail): Token name/symbol, decimals, fee, supply, minting account, ledger (for index), archives; links "Safety report" (`/studio/<ledger>`), "Transactions", "Block log".
- **Redesign notes**: token-developer upgrade/inspection lives in the wallet, not the studio.

## Coming-soon gating (LaunchGate / ComingSoon)

- **How it decides**: `useSiteAdmin()` → backend `caller_is_admin()` for the logged-in principal (cached per principal per session); anonymous → not admin. No feature flags, no staging check, no premium check.
- **Gated**: tabs `/studio/:id/handover` and `/custody` ("Hand over to Sneed custody"), `/studio[/:id]/index` and `/studio/index` ("Index launcher"), `/studio/launch` ("Token launcher") via `ComingSoonGate` (admins see the panel under "Admin preview: … shows "coming soon" to everyone else."; others see "<title>: coming soon … built and being tested"); whole pages `/guardian` ("Sneed custody") and `/tools/pad` ("Token launcher") via `ComingSoonPage`. `ComingSoonRibbon` on SneedApp's launchpad and index cards (cards still clickable).
- **Not gated**: Check, Harden, `/tools/money`, `/admin/pad` (own admin check).
- Separately, availability depends on build env: `CANISTER_ID_SNEED_PAD`, `CANISTER_ID_SNEED_GUARDIAN[_BLACKHOLE|_MULTIDAO]` (absent → "not deployed in this build" messages).
- `components/ComingSoon.jsx` (plain "Coming Soon" page) is unrelated: used by `Tokenomics`, `MeInfo`, `ToolsEscrow`, `TokenomicsInfo` pages only.

## Legacy and redirect routes

- `/tools/token_safety`, `/tools/token_safety/:ledgerId` → `<Navigate replace>` to `/studio[/:id]` (TokenSafety.jsx default export is only the redirect).
- `/tools/pad` → ComingSoonPage → (admins) `<Navigate>` to `/studio/launch`; non-admins see the gate panel and are never redirected.
- Help still links `/tools/token_safety`.

## Admin: /admin/pad (pointer only)

- `pages/admin/SneedPadAdmin.jsx`: launchpad admin (backend admin check): ledger code source ("Check the NNS now" = `refresh_wasms`), settings (`admin_set_config`: fees, cycles, fee destination, paused, follow NNS, guardians), "Rescue a canister" (`admin_set_canister_controllers`, per failed launch "Rescue ledger/index"), launches list (links to `/studio/<ledger>`), registration neuron, "Withdraw ICP", pad log, Special prices card. Relates to Launch and Index tabs. Entry: Header Admin menu "Sneed Pad", Admin dashboard.

## Slice observations

**(a) Entities, with the views and actions that exist for each**

- **App (Sneedapp catalogue entry)**: views: store card (`/sneedapp`), detail page (`/sneedapp/app/:id`), publisher "Your Apps" list, admin list. Actions: mint (generic wizard or custom `mintUrl`), enable/disable (publisher), create/edit (admin only). Fields: name, description, icon, families, mint/premium price, publisher, view/manage/mint URL templates, mint count.
- **App version**: views: detail "Version History", mint wizard version picker, publisher Versions tab, admin. Actions: add (metadata), upload WASM, remove (publisher/admin); upgrade bots to it (UpgradeBotsDialog). Fields: semver, wasm hash, wasm size, wasm/source URL, release notes/date (date/URLs/hash never shown to users).
- **Publisher**: views: name/verified tick on cards/detail/mint; owner-only console. Actions: edit info and payment account, owners, family tags, withdraw revenue (publisher + per app). Admin: create, verify, set DAO cut. No public publisher profile.
- **Minted app canister ("my Sneedapp canisters", factory wallet)**: views: store "My Canisters", `/apps` Wallet → Sneedapp, Wallet → Apps tab, PrincipalBox. Actions: register/deregister (`/apps` "Add bot"), view/manage via app URL templates, upgrade, top up, transfer control, sell on Sneedex (other slice).
- **Tracked canister ("wallet apps")**: views: `/apps` → Other Apps, Wallet → Apps tab, DappCardModal. Actions: add/remove, move to groups/Sneedapp (drag).
- **Canister group (folder/subgroup) and Ungrouped**: views: `/apps` Groups. Actions: create, rename, delete, nest, add/remove apps, reorder/move (drag only), expand/collapse; limits (premium) and admin limits editor.
- **Canister (any)**: views: `/canister?id=` (public info or controller status), `/canisters` (named only), App Manager rows, DappCardModal, safety-report leaves. Actions: public name (controller), nickname (anyone), top up (ICP, card, ck-USDC in bulk), add/remove controllers, transfer control (wallet), upgrade/reinstall WASM (controller), track (only from `/apps` input, not from `/canister`).
- **SNS (as a folder of canisters)**: views: `/apps` SNS root folder (System/Dapps/Archives with cycles). Actions: top up via banner.
- **Cycle health / thresholds**: views: lamps and counts on `/apps`, Header pill, Wallet; DappCardModal colour; `/canister` colour. Actions: set thresholds at `/me?tab=settings`.
- **Cycles top-up (payment)**: views: single (`/canister`), bulk dialog, mint extra gas, card link. Actions: pay ICP via CMC, ck-USDC via CyclePay, card via CyclePay site.
- **Pending top-up record (browser-local)**: views: PendingTopUpsNotice (6 places), `/tools/recover-topup`. Actions: finish, dismiss, recover by block index.
- **Bot (staking/trading) as a canister**: views: `/apps` bot cards (version, neurons, outdated), DappCardModal (neurons, chores). Actions: upgrade (bulk), chores start/pause/stop/run (DappCardModal only), open console (other slice).
- **Token-tool entities** (ledger/token, safety check, rating, thresholds, fix, pad launch, index launch, custody, money-flow graph, special prices):
  - Ledger/token: header (studio, money page), safety report, Harden, wallet app-canister detail; check/re-run, lock tokens/LP (external wizard), mint (wallet dialog only), upgrade (wallet, pad-launched only), hand over, add index, name (post-launch only).
  - Safety check (leaf): status, summary, details JSON, "open" hub link, fix button, recompressed-build verifier; ~39 leaf types in 8 groups; expand, "Problems only", re-run.
  - Safety rating: Check header + Harden header only; not shown on token lists or wallet.
  - Check thresholds: edit/save/defaults in the report; localStorage only.
  - Pad launch: create (Launch tab), "Your launches" (newest 50), admin list/rescue; no per-launch page. Index launch: create only; no "my index launches" list.
  - Custody deployment / child / op / policy: `/guardian` console (status, rules, ops lifecycle), HandoverPanel (register), IndexPanel (stage ICRC-106 op); policy editor in three places.
  - Money-flow graph: inline in report (scan-backed, full overlays) and `/tools/money` (index-backed, lazy).
  - Special prices: badge on own price; admin card on 8 admin pages + `/guardian`.
- **ICRC-3 block / block log**: views: `/tools/icrc3` (overview, blocks in 3 views, certified tip, chain links), report "Block log certified" leaf, wallet "Block log" link. Actions: browse, page, filter by type, jump, verify, JSON.
- **Account (ICRC-1 / legacy id / subaccount)**: views: `/tools/account`. Actions: parse/convert/copy, balance lookup.
- **Sneed Monitor watch / balance** (staging-only): views/actions in `/notifications` section (watch, topics, frequency, top-up, auto top-up allowance, withdraw, charges).

**(b) Overlaps with other areas of the app**

- **My canisters, three lists, four UIs**: store "My Canisters" (`/sneedapp`), `/apps` (Groups + Wallet→Sneedapp + Other Apps), Wallet → Apps tab, PrincipalBox header dropdown. Wallet/PrincipalBox use DappCardModal (chores, neurons, send, remove); `/apps` has groups, SNS folders, health summaries, drag moves; the store has app-typed View/Manage links. Type resolution by module hash is reimplemented in SneedApp.jsx, Apps.jsx and Wallet/WalletContext.
- **Top-up**: ICP top-up on `/canister`, bulk dialog (4 openers: Apps, Header, Wallet, PrincipalBox), mint extra gas, bot consoles (other slice, via PendingTopUpsNotice); card and ck-USDC via CyclePay; recovery tool. One shared safe engine (`cyclesTopUp`), many UIs.
- **Upgrade**: UpgradeBotsDialog (Apps, Header, Wallet, PrincipalBox) vs `/canister` manual WASM upgrade vs bot consoles' own upgrade (other slice) vs custody "Upgrade to <hash>".
- **Controllers / ownership transfer**: `/canister` add/remove, TransferCanisterModal (PrincipalBox) and a Wallet inline copy, Token Studio hand-over (`update_settings` via utils/management), Guardian "Change controllers", Sneedex selling apps/bots (other slice).
- **Names**: `/canister` public name + nickname, `/canisters` directory, LaunchPanel post-launch naming, `/names` address book and `/users` (other slice) — same naming backend.
- **Premium**: App Manager limits (upgrade modal → `/premium`), Sneedapp premium mint price, pad/index premium fees, Guardian op premium fee, Tools "Premium" badge on SNS Jailbreak.
- **Ledger history**: ICRC-3 Browser vs `/transactions` / `/transaction` vs Follow the Money vs Sneed Lock/RLL activity cards.
- **Health / monitoring**: `/apps` cycle lamps, token report "Cycles and memory" leaves, `/health` (custody, same TreeView/runner), Sneed Monitor (proposal alerts), Header notification pills.
- **Token tools**: `/health` shares runner/TreeView; Harden/fixes delegate to Sneed Lock pages and `/position`; report leaves deep-link to SNS, neuron, proposal, canister, principal, transactions, ICRC-3 pages; wallet hosts the mint dialog and pad-launched ledger upgrades; LaunchPanel/Guardian file SNS proposals from the user's neurons (same capability as governance pages); pad payments go through `approveAndRecord` (appear in wallet approvals).

**(c) Dead, hidden, half-built or confusing**

- **Hidden**: publisher console (`/sneedapp/publisher/:id`) only linked from "by <publisher>", and it is owner-only (everyone else hits "You are not an owner"); SNS root folders in `/apps`; `/tools/recover-topup` (menu only, not on Tools hub); chores control only in DappCardModal; Canister page's idle-cycles-per-day fetched but not shown; ICRC-3 browser is the only generic audit-log viewer for every Sneed canister but sits under Tools.
- **Confusing**:
  - `/apps` mixes three storage concepts (groups, "wallet" tracked canisters, Sneedapp factory registrations) with drag-only moves between them; "Wallet" section inside the App Manager duplicates the Wallet's Apps tab.
  - `/apps` "Create" → `/create_icp_neuron` (staking bot only), not the store; staking bot and trading bot have different mint flows.
  - Low-cycles lists differ by entry point (Apps: red incl. non-controlled; WalletContext: orange+red, controlled).
  - `/canister` has no "track in App Manager"; `/canisters` is named-only (an address book, not an explorer).
  - "Custody" label → `/custody` (Tools) vs `/guardian` (Sneed DAO menu).
  - Header highlights `/apps` and `/canister` under "Sneed Me" while the Sneedapp menu also lists "My Apps".
  - Two CyclePay paths with different limits ($5–$100 card link, no login; $1–$200 ck-USDC in-app).
  - Store mixes real Sneedapp apps with Token Studio "launchpad"/"index" pseudo-apps (gated, ribbon only).
- **Half-built / gated**: Token Studio Launch, Index, Hand over; `/custody`, `/guardian`, `/tools/pad` — admin preview only (`caller_is_admin`), yet promoted from Hub ("Sneed Custody", "Flagship"), store cards, Tools hub ("Launch a token"), Harden buttons and fix buttons. All custody deployments `live: false`. Sneed Monitor renders nothing on prod. Tools hub "More Tools Coming Soon"; `/tools/escrow` is a ComingSoon page (not on the hub).
- **Dead / stale**: dead fix mappings, unused `onHeader`/`onLaunched` props, `KNOWN_SAFE_CONTROLLER_CODE` empty, duplicate `link` key bug on "Liquidity under safe control", stale Harden copy, unused pad/guardian IDL methods (see token tools section); `TokenSafety.jsx` is a redirect + exported panel; Help still links `/tools/token_safety`; `/help/canister-manager` kept as backwards-compat alias of `/help/dapp-manager`; `utils/management.js` is used only by the studio while four other files define their own management IDL; `components/ComingSoon.jsx` (generic) is unrelated to LaunchGate.
- **Mobile**: `/apps` drag-and-drop (HTML5 backend) has no touch support, so moving apps between folders/sections is impossible on phones; MoneyGraph has no touch gestures; `/apps` hides status words on mobile (numbers only).
- **Confirmations**: publisher console and group delete use `window.confirm`; HandoverPanel steps down controllers with no confirmation; `/canister` controller removal uses inline "Confirm?"; reinstall needs a checkbox.
