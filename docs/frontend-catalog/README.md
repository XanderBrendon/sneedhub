# Sneed Hub frontend: functionality catalog

What the Sneed Hub frontend (app.sneeddao.com) can do, and what information it shows, organised by **capability** rather than by today's pages and menus. It was written to plan a front-end-only redesign that separates functionality more clearly.

- **Snapshot:** commit `ecc0eadf` (6 Nov 2026). This is the build that proposal #487 put live, so the catalog describes production as it stands.
- **Sources:** the frontend code in `src/app_sneeddao_frontend/src` (routes in `App.jsx`, menus in `components/Header.jsx`), checked against the live site.
- **Scope:** every route, page, modal, panel and global UI element, including admin and features that are hidden, gated or half-built. Backend capabilities are listed only where the frontend exposes them.
- **Redesign proposal:** a sitemap for a restructured site, built from this catalog, is in [../frontend-redesign/sitemap.md](../frontend-redesign/sitemap.md).

## How to read it

This page is the map. It has four parts:

1. **Capability domains:** what users can do and see, grouped by job, with where each piece lives today.
2. **Entity index:** each kind of object (token, neuron, lock, offer, bot…), with every place it is shown and every place it can be acted on.
3. **Findings for the restructure:** duplication, naming collisions, buried and gated features, broken pages, mobile and safety gaps.
4. **Route inventory:** all 159 routes, with today's menu section and the domain each belongs to.

The detail files hold one record per capability: routes and parameters, auth, information shown, actions (with button labels as the code has them), entry points, shared components, gating, and redesign notes.

| Detail file | Covers |
|---|---|
| [01-wallet-assets.md](01-wallet-assets.md) | Wallet tabs, token cards, pockets/subaccounts, send/receive and wallet modals, approvals, NFTs, transactions, scan wallet, BTC/ETH convert, native Bitcoin page |
| [02-lock-liquidity.md](02-lock-liquidity.md) | Sneed Lock (wizard, lock page, explorers, lock NFTs, gas), ICPSwap positions and pools |
| [03-sneedex-swap.md](03-sneedex-swap.md) | Sneedex marketplace (browse, offer page, create wizard, My Sneedex), swap, escrow tools |
| [04-neurons-governance.md](04-neurons-governance.md) | SNS selector, SNS directory, neuron explorer and detail, proposals (SNS and NNS), voting, liquid staking, SNS Jailbreak, voting rewards, proposal filing |
| [05-bots.md](05-bots.md) | ICP Staking Bot and Trading Bot consoles, the shared bot console, SneedScript, bot health and upgrades |
| [06-apps-canisters-tools.md](06-apps-canisters-tools.md) | Sneedapp store and publisher console, App Manager, canister page, cycles top-ups, Tools, Token Studio and safety checks, custody, Follow the Money |
| [07-social.md](07-social.md) | Forum, feed, polls, reactions, tips, mentions, Sneed SMS, notifications, header alert chips |
| [08-identity-shell.md](08-identity-shell.md) | Navigation model, header and global UI, login/pairing/agent delegation, /me, public profiles, address book, naming, Premium, Hub landing |
| [09-dao-help.md](09-dao-help.md) | DAO pages (dashboard, health, tokenomics/RLL, revenue, rewards, products…), donations, feedback, Mr Sneed, help, API docs, architecture, the game |
| [10-admin.md](10-admin.md) | Every admin page, who can use it, admin controls on ordinary pages, Sneed Drive |

## At a glance

- **159 routes** in `App.jsx`: 26 admin, 26 help and docs (`/help/*`, `/api`, `/architecture`), 107 user-facing. Roughly 125 page components and 300k lines of JS/JSX. A second HTML entry (`btc.html`) serves the native Bitcoin wallet page.
- **10 menu sections:** Sneed Hub, Sneed Me, Sneed DAO, Sneedapp, Sneedex, Sneed Lock, Liquid Staking, Help, Tools, and Admin (admins only).
- **14 routes belong to no section** and show the Sneed Hub submenu by fallback. They include `/wallet/pockets`, `/convert`, `/scan_wallet`, `/trading_bot/:id`, `/pair`, `/connect-agent` and `/outrun`.
- **Multi-DAO by design:** most governance, neuron and forum pages work for any SNS through a global SNS selection (`?sns=` or localStorage). Many personal features (wallet, Sneedex, locks, bots) do not depend on the SNS.
- **Login is Internet Identity only** (1.0 or 2.0). Public pages work logged out. Personal pages show a login prompt, but some show forms that do nothing until you log in.

---

## 1. Capability domains

Thirteen domains plus the global shell. Each one lists the jobs users come to do, the capabilities and information the app offers for them, where they live today, and signals for the redesign.

The domains are a way to see the whole app, not a proposed menu. Three different axes are mixed in today's menus:
- **Product brand:** Sneedex, Sneed Lock, Sneedapp.
- **Personal vs public:** Sneed Me vs Sneed Hub.
- **Job:** Liquid Staking, Tools.

The entity index (section 2) and route inventory (appendix) tag each item so any of these axes can be chosen.

### 1.1 Account & identity

**Jobs:** sign in and stay signed in · show myself to others (name, avatar) · remember who other people are · membership · preferences.

**Capabilities**
- **Sign in** with Internet Identity 1.0 (identity.ic0.app) or 2.0 (id.ai), from the global login modal or `/login`.
  - Sessions last 7 days. An expiry watcher shows a session-expired banner.
  - Logout lives inside the header quick-wallet popup.
- **Pair the installed app with a browser** (`/pair?key=`, PairAppPanel). This is for iPhone PWAs, where the login popup doesn't return to the app. The user matches a short code and copies an approval back.
- **Delegate to an AI agent** (`/connect-agent`). The user picks the canisters and a duration from 1 hour to 7 days, then delivers or copies the delegation. Delegations can't be listed or revoked.
- **Public identity**
  - Display name: 32 characters, moderated, with a ban warning; admins can mark it verified.
  - Avatar: upload, URL, an ICRC-7 or EXT NFT you own ("Use as my avatar" also appears on NFT pages), or remove.
  - Public names for neurons (needs hotkey access) and for canisters (controllers only).
- **Multi-principal account** (AccountPanel on `/me`): merge principals into one account, with owner and member roles. Also hotkey accounts, dapp labels, and leave.
- **Private address book:** nicknames for principals, neurons (per SNS), canisters and subaccounts/pockets. They can be set on `/names`, `/principal`, `/neuron`, `/canister`, in the universal person menu and in AccountPanel. How many you can store depends on your Premium tier.
- **Universal person menu** on every principal display (right-click or long-press only): send message, set nickname, view profile, copy.
- **Sneed Premium** (`/premium`):
  - Status and expiry.
  - A live benefits comparison, read from the other canisters.
  - Ways in: buy with ICP (tiers), claim with Sneed voting power, redeem a promo code. A stranded deposit can be withdrawn.
  - Shows up as a crown on the header logo and badges on users.
  - Separate from Premium: a "Sneed DAO members only" gate that checks staked voting power.
- **Preferences:**
  - Display: theme, privacy mode (hide amounts; Shift+P), display currency.
  - Header: which notification chips show, quick-links layout, colour coding, particle effects.
  - Behaviour: auto-update, cycle thresholds for bots and apps, swap slippage, Sneedex DM alerts, "SMS when mentioned", cache tools.
  - Some are synced to the backend, but only pulled into the browser when `/me` is opened.

**Information exposed:**
- Your principal and account ID (only in the quick-wallet popup; `/me` never shows them).
- Profile hero and stats tiles.
- Membership state and benefits.
- Merged principals.
- Every nickname you have set.

**Today:**
- Pages: `/me`, `/names`, `/premium`, `/login`, `/pair`, `/connect-agent`.
- Global UI: the header "Quick Links" dropdown (currency, theme, privacy) and the quick-wallet popup (PrincipalBox).

**Redesign signals**
- `/me` mixes five jobs: profile editing, a launcher grid, a read-only neuron browser, one SNS's transactions, and the whole settings panel. The settings panel also holds product settings for bots, swap, Sneedex and SMS.
- Global settings are split over four places: the Quick Links dropdown, `/me` Settings, the quick-wallet popup, and a keyboard shortcut.
- "Premium" (paid membership) and the "Sneed DAO members only" gate are different concepts with similar names.
- Agent delegation and multi-principal accounts are real, security-relevant features with no obvious place to find them.

Detail: [08-identity-shell.md](08-identity-shell.md).

### 1.2 Wallet & assets

**Jobs:** see what I hold and what it's worth · receive and send · move between my own accounts · collect what I've earned · control who can spend my tokens · hold and move NFTs · bridge BTC and ETH · look up history.

**Capabilities**
- **Portfolio** (`/wallet`, five tabs: Tokens, Liquidity, Apps, Approvals, Statistics):
  - Header: identity bar (principal, account ID toggle), total value in the display currency, privacy toggle.
  - Statistics tab: portfolio analytics (charts and a table).
- **Token list:**
  - Your server-side list of registered ledgers. "+ Add Token" adds any ICRC-1 ledger by canister id; "Scan" checks only whitelisted tokens on Main.
  - Other actions: reorder, hide dust, refresh, "Remove" (only at zero balance).
  - Tokens also get added silently by other flows: tips received, fee claims, locks.
- **Token card** (each card is a small app):
  - Header actions: Send, Receive (principal, ICRC-1 text, QR, ICP account ID), Buy/Sell/Swap (an in-card SwapModal), Convert (ck tokens → `/convert`), Wrap/Unwrap (GLDT ⇄ sGLDT), Sneed Lock Deposit/Withdraw, Mint (minting account only), Claim rewards.
  - Balance tab: where the token sits, across all pockets.
  - Locks tab: create, link, transfer ownership, sell on Sneedex, add gas.
  - Neurons tab: a near-complete SNS neuron manager, plus basic ICP neuron actions (see 1.5).
  - Info tab: metadata.
- **Send** (SendTokenModal):
  - To a principal, principal + subaccount (hex, bytes or principal-shaped), ICRC-1 text, ICP account ID, a name or nickname lookup, or a QR scan.
  - Shows the decoded recipient and its balance, then a receipt.
  - Pockets mode: send from one pocket, or move between your own pockets.
- **Pockets** (`/wallet/pockets` and the wallet's pocket switcher):
  - One model for every place your tokens sit: Main, numbered or exact subaccounts, the Sneed Lock balance (free + locked), Sneedex escrows, ICPSwap unused pool balances, service balances (Sneed Monitor, Notifications), and the native BTC address.
  - Create (next #N or exact bytes), rename, hide, forget, receive, send, move, withdraw, top up, "Move all to Main", check ICP pocket #N.
  - Approvals per pocket.
- **Collect** (Collect card, header collectibles chip → ConsolidateModal): bulk-claim LP fees, RLL rewards and SNS maturity above a threshold.
- **Approvals** (ICRC-2):
  - Lists every allowance found on Main, with known spenders labelled; also shown as a badge and row on token cards, and per pocket.
  - Revoke, change amount or expiry, see "how these were found".
  - There is no way to create an approval.
- **Liquidity tab:**
  - ICPSwap positions held directly, deposited in Sneed Lock, or locked.
  - Scan pools, "+ Add Swap Pair", "Move a position here".
  - Claim fees, withdraw swap balance, withdraw to wallet, send/transfer, lock or extend, add liquidity (locked positions only), Sell on Sneedex, manage on `/position`.
- **Apps tab:** Sneedapp bots and tracked canisters (DappCardModal). Register, remove, top up, upgrade, transfer control, run chores, move between sections, open the console. It largely duplicates `/apps` (see 1.7).
- **NFTs** (`/nft`):
  - Your ICRC-7 and EXT collections (collection list kept in this browser), "Add Collection".
  - Per NFT: details, send (ICRC-7 or EXT), ICRC-37 approvals (per token or the whole collection), "Sell on Sneedex", "Use as my avatar".
  - Mint (collection admins only).
  - Media: images, 3D, audio, video.
  - Game-character NFTs and soulbound items.
  - Diamond Club migration banners.
- **Bridge:**
  - `/convert`: get ckBTC from any Bitcoin wallet, withdraw ckBTC to any BTC address; get or withdraw ckETH and ckERC-20.
  - `btc.html`, a separate page, the only place that signs native BTC: send, sweep, move BTC ⇄ ckBTC, signing credit, activity, speed up a stuck send (RBF/CPFP). It is a beta opt-in.
- **History:**
  - `/transactions?ledger=`: one ledger's raw log, with filters and CSV.
  - `/transaction`: one block, with Previous/Next.
  - Account history appears only inside `/principal`, `/me` and `/neuron`, for SNS tokens. The wallet itself shows no history.
- **Any principal's balances:** `/scan_wallet` (legacy; duplicates the `/principal` Wallet tab).

**Information exposed:**
- Per token: balance by pocket, value, price, locked and deposited amounts, neurons, rewards, metadata, approvals.
- Portfolio totals and allocation.
- Positions with value and unclaimed fees.
- NFT metadata and media.
- BTC UTXOs, pending sends and moves.
- ck minter addresses and withdrawal status.
- Ledger blocks.
- Prices come only from ICPSwap pools against ICP, so tokens without a pool have no value and drop out of totals.

**Today:**
- Wallet, NFTs and Apps under Sneed Me.
- NFTs (browse) and Transactions under Sneed Hub.
- `/wallet/pockets`, `/convert`, `btc.html`, `/scan_wallet`, `/nft_item` and `/transaction` have no menu entry.
- A second wallet lives in the header quick-wallet popup (PrincipalBox).

**Redesign signals**
- `/wallet` (about 11,200 lines) mixes four jobs: tokens, DeFi positions, a canister fleet, and allowances.
- The token card is a hub for swap, lock, neurons, rewards and bridging, each of which has its own dedicated page elsewhere.
- Sends run through three executors (wallet, quick-wallet popup, pockets).
- The quick-wallet popup has gaps: its position Send does nothing, and its token dialog lacks deposit, withdraw and wrap.
- Pockets unify scattered balances but are hard to find: the switcher appears only once you already have an extra pocket.
- The Sneed Lock balance has four or five names, and three ways to move tokens in and out of it.
- Settings that change the wallet (display currency, collect threshold, show-remove) live on `/me`.
- NFT collections and BTC history are kept in this browser only, while pockets, names and card order sync across devices.
- SKILL.md lists four wallet tabs. It misses Approvals, pockets and the Bitcoin card.

Detail: [01-wallet-assets.md](01-wallet-assets.md).

### 1.3 Trade: swap and Sneedex

**Jobs:** swap one token for another · sell or auction assets through escrow · buy, bid on, or buy out what others sell · settle and claim.

**Capabilities**
- **Swap** (`/swap`, plus the "Buy"/"Sell" SwapModal on wallet token cards and offer pages):
  - Quotes one ICPSwap pool per pair, merged with matching Sneedex buyouts ("Buy from Auction", "Split Trade").
  - Slippage tolerance setting (saved to the backend).
  - Dormant code: KongSwap is gone, and split swaps and multi-hop routing are not live.
- **Marketplace** (`/sneedex_offers`): browse, search, filter (asset type, state, token), sort. USD estimates per offer.
- **Offer page** (`/sneedex_offer/:id`):
  - Due diligence per asset, behind "Info" toggles:
    - Canisters: controllers, cycles, memory, module hash.
    - Staking bots: official-WASM verification, neurons, botkeys.
    - SNS neurons: full stats and permissions.
    - Tokens: metadata.
    - NFTs: asset details.
    - Locks: value.
  - Bid history.
  - Buyer: "Place Bid" (reserve, pay, confirm), "Buyout", then claim the assets once won.
  - Seller: escrow each asset, activate, "Accept Highest Bid", "Cancel Offer", "Claim Payment".
  - Anyone: "Process Expiration" / "Finalize Auction".
- **Create an offer** (`/sneedex_create`, a 4-step wizard):
  - Asset types: App canister, ICP Staking Bot, Trading Bot, SNS Neuron, ICRC-1 token, ICRC-7 NFT (Sneed Locks are sold as their lock NFT), EXT NFT.
  - Pricing: price token, minimum bid, buyout, bid increment, expiry.
  - Private/OTC offers with approved bidders; public note and note to buyer.
  - A creation-fee deposit, with Premium and partner prices. Assets are escrowed automatically.
  - Deep links from elsewhere: `?nft=`, `?lock=`.
- **My Sneedex** (`/sneedex_my`): my offers and my bids, with the same settle actions as the offer page, plus "Refund Bid" (only here).
- **Escrow recovery:** bid escrows and the creation-fee deposit can also be withdrawn from wallet Pockets.
- **Related tools:**
  - `/tools/escrow` is a ComingSoon placeholder.
  - `/tools/escrow/swap` is an OpenChat escrow lookup, unrelated to Sneedex.

**Information exposed:**
- Offer fields: assets, price token, min bid, buyout, expiry, private flag, notes, fee rate.
- Offer states (Draft, PendingEscrow, Active, Completed, Expired, Cancelled, Claimed, Reclaimed) and bid states.
- Market statistics and fee configuration.
- Swap quotes, spot prices and price impact.

**Today:**
- The Sneedex section: Swap, Marketplace, Create Offer, My Sneedex, About.
- Wallet "Buy"/"Sell" buttons and "Sell on Sneedex" buttons on NFTs, token locks and locked positions.
- Offers also appear in the Hub widget, `/feed` and `/principal` (Trades tab).

**Redesign signals**
- Settle actions exist on the offer page and on My Sneedex, and the buyout flow is written twice (offer page and swap widget). Offer cards are drawn three ways.
- "Sell on Sneedex" exists only for NFTs, locks and locked positions. Tokens, neurons, bots and apps must be picked inside the wizard, which rebuilds inventories the wallet, App Manager and neuron pages already have.
- The offer page's due-diligence views overlap `/canister`, the bot consoles and `/neuron`.
- `/swap` does nothing for logged-out visitors: no quotes, and its "Connect Wallet" button is disabled.
- Browsing the marketplace silently runs `processExpiration` writes as the viewer.

Detail: [03-sneedex-swap.md](03-sneedex-swap.md).

### 1.4 Lock & liquidity

**Jobs:** lock tokens or an LP position (vesting, commitment proof) · manage a lock's time · transfer or sell a lock · manage ICPSwap liquidity · prove what's locked for a token.

**Capabilities**
- **Create a lock** (`/lock_wizard`, 3 steps):
  - Token lock or ICPSwap position lock.
  - Shows the available balance and deposit instructions, with "Check again".
  - Guides moving a position in from ICPSwap and checks the pool.
  - Pays the ICP lock fee: Premium, admin and partner tiers apply, and the fee can come from the Sneed Lock balance.
  - Sets amount, expiry and an optional dissolve delay, with a "cannot be undone" warning.
  - A quicker lock modal in the wallet lacks the delay, the warning and the pool check.
- **Manage one lock** (`/lock/:id`): start/stop dissolving, increase the delay, extend, add gas (anyone), and send the lock as an ICRC-7 NFT. Value and summary.
- **Lock as an asset:**
  - Each lock is an ICRC-7 NFT in the Sneed Lock collection, shown on NFT pages and sellable on Sneedex.
  - Expired token locks become free "Deposited" balance.
  - Expired positions can be withdrawn.
- **Wallet-only lock actions:**
  - Claim fees on a locked position.
  - Add liquidity to a locked position.
  - "Sell on Sneedex".
  - Extend a position lock (via the legacy call).
- **Explore locks:**
  - `/sneedlock_info`: all locks, Active/Expired, filter by owner or ledger, TVL.
  - `/tokenlocksoverview?ledger=`: one token's locks.
  - Legacy share views: `/tokenlock`, `/positionlock`.
- **ICPSwap positions** (`/position`, `/positions`):
  - View a position: amounts, range, fees owed, owner, value.
  - Owner actions: "Collect fees", "Move range", "Remove", "Transfer", "Lock in Sneed Lock", "Withdraw to wallet".
  - Open a new position in a pool.
  - Create a new pool (paid; only via `/position` with no params).
  - List mine or a pool's positions, with pool stats.
  - Withdraw unused pool balance.
- **Bulk fee collection** across all positions, from the header "collectibles" chip (ConsolidateModal).

**Information exposed:**
- Lock fields: id, owner, ledger or pool and position, amount, expiry, dissolve delay and state, created date, NFT id, gas balance.
- TVL, total and per token.
- Position fields: liquidity, ticks, token amounts, fees owed.
- Pool data: price/tick, fee tier, verified flag.
- The deposited (Sneed Lock subaccount) balance.

**Today:**
- Sneed Lock section: Create Lock, All Locks, Overview.
- "Positions" sits under Sneed Hub.
- Most actions live on wallet token and position cards.

**Redesign signals**
- The wallet is where locks and positions are actually managed; the Sneed Lock pages mostly render the same cards read-only. Creating, extending and transferring a lock each have 2 or 3 UIs that behave differently. The ICPSwap import guide appears in 4 places.
- `/position` "Transfer" lacks the recipient guard that wallet "Send" has.
- The Sneed Lock subaccount has five names: "Deposited", "backend wallet", "Sneed wallet ICP", "vault" and "free balance".
- TVL and position value are computed differently on different pages.

Detail: [02-lock-liquidity.md](02-lock-liquidity.md).

### 1.5 Stake: neurons and liquid staking

**Jobs:** stake tokens in a neuron · manage dissolve, maturity and permissions · keep stakes liquid (transferable or sellable) · import a neuron created elsewhere · recover a stake that didn't finish.

**Capabilities**
- **SNS neurons you control** (hotkey or owner):
  - On the wallet token card's Neurons tab: create, increase stake, dissolve controls, disburse, disburse maturity, auto-stake maturity, split, send.
  - On `/neuron`: dissolve controls; principals and permissions ("Full Owner"/"Hotkey" presets); followees (many neurons, many topics).
  - On `/me` My Neurons: set the public neuron name (only here).
- **NNS neurons** where your principal is a hotkey: dissolve, disburse, maturity and hotkey actions on the wallet ICP token card. Voting is covered in 1.6. There is no NNS neuron detail page.
- **Liquid staking:**
  - `/liquid_staking`: a static hub.
  - `/sns_neuron_wizard`: stake SNS tokens into a neuron your wallet controls. A second copy of the token card's "Create New Neuron".
  - `/create_icp_neuron`: mint an ICP Staking Bot, a canister that owns NNS neurons and is managed in its console (see 1.7).
- **SNS Jailbreak** (`/tools/sns_jailbreak`, `/tools/sns_jailbreak_list`): a paid, Premium-discounted wizard. It generates a browser-console script that adds your Sneed Hub principal as a full controller of an SNS neuron you hold in the NNS dapp, so the neuron becomes manageable and sellable here. It can re-show, verify and delete scripts, and withdraw deposited ICP.
- **Pending-stake recovery:** a local journal of unfinished stakes, with "Finish staking", shown on `/wallet`, the SNS wizard and the staking bot.

**Information exposed:**
- Per neuron: id, stake, maturity and staked maturity, dissolve state and delay, age, voting power, permissions per principal, followees by topic, stake account, and voting history (Sneed only).
- Neuron explorer (`/neurons`): every neuron of an SNS, with CSV export. A hidden holder index (`?tab=stakers`) shows stake owned vs hotkeyed per principal.
- Header "Your neurons" voting-power bar.

**Today:**
- Neurons under Sneed Hub.
- Neuron management inside the wallet (Sneed Me).
- Staking flows under Liquid Staking.
- Jailbreak under Tools, shown only when logged in.
- Help pages under Help.

**Redesign signals**
- SNS neuron management is split between `/neuron` and the wallet token card, two separate implementations that overlap on dissolve. `/neuron` lacks disburse, split and send, and has no stake refresh, although it shows a "send tokens here" stake account.
- Neuron creation is implemented twice: the wizard and the token card.
- The staking bot's NNS neuron tabs repeat the same verbs with different UI and wording (see 1.7).
- Jailbreak is really "import an existing neuron", but it sits under Tools.
- Three neuron caches can disagree; the header has a "Refresh neurons" button to work around it.

Detail: [04-neurons-governance.md](04-neurons-governance.md), with the staking bot in [05-bots.md](05-bots.md).

### 1.6 Govern

**Jobs:** see what's being decided across DAOs · vote with all my neurons · discuss proposals · claim voting rewards · propose changes to Sneed DAO · understand an SNS's rules.

**Capabilities**
- **SNS directory** (`/sns`): every SNS, with logo, name, token and price. Per SNS: governance parameters, hotkey-permission policy, reward parameters, canister health. Choosing one also changes the global SNS selection.
- **Proposals:**
  - Lists: `/proposals` (SNS) and `/icp_proposals` (NNS), with filters, "load all", CSV export, and quick vote per row.
  - Detail pages: `/proposal` and `/icp_proposal`. They show summary, payload (Sneed proxy payloads decoded), tally vs threshold and minimum participation, voting history (Sneed via the RLL archive), the embedded forum discussion, and per-neuron "Adopt"/"Reject" or "Vote with all eligible neurons".
- **Cross-DAO voting inbox** (`/active_proposals`, menu label "Voting"): every votable SNS and NNS proposal for your neurons, "Show only votable", and bulk "Adopt All"/"Reject All" per DAO.
- **Votable-proposals alert:** a header chip linking to the inbox.
- **Voting rewards (RLL):**
  - Claim per token on `/rewards` (Sneed Me) or `/rll` (Sneed DAO).
  - Also on wallet token cards and in the wallet's collect-all.
- **Filing Sneed DAO proposals:**
  - Live only on `/mr-sneed`, for Mr Sneed's personality, writers and limits.
  - A general in-app proposal pipeline (governance registry, `useProposeAction`, `ProposeDialog`, proxy payloads) is built but mounted nowhere.
  - `/proposals` "Create proposal →" goes to ic-toolkit.app.

**Information exposed:**
- Proposal fields: id, title, topic or action type, proposer (with name), status, created and deadline, summary, payload, ballots, tallies.
- SNS parameters and policies.
- Your eligible neurons and their voting power.
- Reward stats, distributions, credits and claims.

**Today:**
- SNS, SNS Proposals, ICP Proposals and Neurons under Sneed Hub.
- Voting and Rewards under Sneed Me.
- A second Rewards (`/rll`) under Sneed DAO.
- Proposal filing in Mr Sneed (Sneed DAO).

**Redesign signals**
- Voting UI exists in 5 places with 3 implementations.
- The SNS and NNS list and detail pages are near-copies of each other.
- Two "Rewards" pages make the same canister calls.
- The "add your principal as a hotkey" onboarding exists only on help pages; the in-page version is dead code.
- Picking an SNS on `/sns` or in a wizard silently changes the SNS that every other page shows.

Detail: [04-neurons-governance.md](04-neurons-governance.md); rewards also in [09-dao-help.md](09-dao-help.md).

### 1.7 Apps, canisters & bots

**Jobs:** get a bot or app canister of my own · keep my canisters alive (cycles) and up to date · organise them · run my staking bot or trading bot · manage any canister I control · publish apps.

**Capabilities**
- **App store** (`/sneedapp`, `/sneedapp/app/:id`, `/sneedapp/mint/:id`):
  - Browse apps: price, Premium price, publisher, families, mint count. App detail shows the version history.
  - Mint with the generic wizard: pick a version, add extra gas, review, pay. The staking bot uses its own mint page instead.
  - "My Canisters" lists your minted apps.
- **Publisher console** (`/sneedapp/publisher/:id`, owners only): edit publisher info, payment account, owners and family tags; manage apps and versions (add, upload WASM, remove); enable or disable apps; withdraw revenue.
- **App Manager** (`/apps`):
  - Track any canister.
  - Groups and subgroups (Premium raises the limits); SNS root folders (system, dapps, archives).
  - Health lamps and cycle rollups; outdated-bot flags.
  - Bulk "Upgrade" (UpgradeBotsDialog) and bulk "Top-Up" (ICP via the Cycles Minting Canister or ck-USDC via CyclePay, to a target level, split equally or the same amount each).
  - Moving things between groups is drag-and-drop only.
- **Any canister** (`/canister?id=`):
  - Anyone: public status, top up with ICP, or "Pay by card" (CyclePay, no login).
  - Controllers: status, add or remove controllers, public name, WASM upgrade or reinstall with init-argument encoders.
  - Anyone can set a nickname.
  - Control can be transferred from wallet app cards.
- **Cycles safety:** a local journal of pending top-ups with resume, shown in 6 places, and `/tools/recover-topup` to recover a top-up by block index.
- **ICP Staking Bot console** (`/icp_neuron_manager/:id`):
  - Bot tabs:
    - Info: overview, WASM verification, upgrade/reinstall/install official version, controllers, top-up, withdraw tokens.
    - Botkeys: about 30 fine-grained permissions.
    - Chores: confirm following, refresh stake, collect maturity, distribute funds; with schedules and status lamps.
    - Log, Events.
  - Neurons section: create (stake from wallet), Stake, Maturity, Following, Dissolve, Disburse, Hot Keys, Advanced (split, merge, visibility).
  - No voting UI, although `Vote` and `MakeProposal` permissions can be granted.
- **Trading Bot console** (`/trading_bot/:id`):
  - Setup wizard: DCA, Range Trade, Self-Balancing Portfolio, AI Script.
  - Shared console: Info, Chores, Botkeys, Events.
  - Lower tab strip:
    - Chores: trade, rebalance, move funds, distribute, snapshot instances; each with Control, Actions, Purse, Fallback, Logs and Config.
    - Wallet: token registry, balances, valuation, pause/freeze, fund, withdraw, send, purses.
    - Performance: P&L, equity curve, capital flows, price history.
    - Logs: bot log, trade log, portfolio snapshots, settings.
    - Circuit Breaker: rules with nested conditions and 12 action types, plus an event log.
    - Quick Trade.
    - Recovery: stuck ICPSwap funds.
    - Script: SneedScript, a text format for the whole bot with an AI guide; changes run as reviewed operations.
- **Bot health:** header indicators for low cycles, outdated bots and chore health, with dialogs, plus lamps on each page and in the App Manager.

**Information exposed:**
- Per canister: status, cycles, memory, module hash, controllers, version and official-version match, idle burn (fetched but not shown).
- App and version catalogue.
- Per bot: chores (schedules, last and next runs, health), botkeys, logs and events, neurons, purses and balances, trade history, P&L.

**Today:**
- Sneedapp section: Browse Apps, My Apps.
- `/apps` and `/canister` highlight Sneed Me.
- `/canisters` under Sneed Hub.
- Staking bot pages under Liquid Staking.
- Trading bot in no section.
- Bots also reachable from the wallet Apps tab and the quick-wallet popup (DappCardModal).

**Redesign signals**
- "My canisters" is spread across three stores and four screens: the store, App Manager, the wallet Apps tab and the header popup. Each offers different actions; chores and Send appear only in DappCardModal.
- The two bot consoles share almost nothing. The staking bot rebuilds the shared console inline.
- Upgrade exists 3 times and cycles top-up 5 times. Bots are reachable from 5 places, but there is no "My bots" hub and no trading-bot list.
- There are four automation systems with different vocabularies: chores, circuit breakers, event reactions and SneedScript.
- `?tab=` deep links into the bot pages are ignored.

Detail: [06-apps-canisters-tools.md](06-apps-canisters-tools.md) (store, App Manager, canisters, cycles) and [05-bots.md](05-bots.md) (bot consoles).

### 1.8 Community & messaging

**Jobs:** discuss in a DAO's forum · react, vote on and tip posts · run polls · follow activity · message people directly · get notified.

**Capabilities**
- **Forum** (one per SNS: `/forum`, `/topic/:id`, `/thread`, `/post`):
  - Create a forum if an SNS has none; create special topics (General, Governance, Preproposals).
  - Start a thread, optionally with a poll.
  - Comment, reply, edit, delete posts.
  - Neuron-weighted up/down votes on posts and threads, with the neurons chosen per thread.
  - Emoji reactions, up to 10 per user per target.
  - Polls with voting-power weighting, an end time and vote-change rules.
  - Tips on posts and threads: a ledger transfer from your wallet, recorded by the forum.
  - @-mentions, optionally paired with an SMS.
  - "Message" the author.
  - A per-post vote breakdown on `/post`.
  - Proposal pages embed their thread.
- **Activity:**
  - `/feed`: across SNSes, with its own multi-SNS filter; Sneedex auctions are mixed in.
  - `/posts`: Replies to Me, Mentions, My Posts, My Threads.
  - `/tips`: post tips sent and received.
- **Sneed SMS** (`/sms`, `/msg/:id`):
  - Inbox, Sent and All lists.
  - Conversations: chat bubbles, group conversations, "Notes to self"; on phones, the list or the open conversation.
  - Compose, reply, reply all, remove yourself.
  - A global "Send Message" dialog from any principal.
  - Limits are higher with Premium. Messages are not encrypted.
- **Notifications** (`/notifications`, tabs Inbox and Settings; `/notifications/:id`):
  - Allow or block apps that want to notify you, with levels and caps per app and per topic.
  - Channels: Sneed SMS, phone push, Discord. Telegram and email are "coming soon".
  - Level routing, always-copy, quiet hours, pause.
  - A prepaid ICP message balance: top up, withdraw, history.
  - Sneed Monitor proposal watching (staging only).
- **Header alert chips:** client-polled counts of replies, mentions, messages, tips, votable proposals, collectibles, bot health, low cycles, and update available. Each is switched on or off in `/me` settings. Several open real actions (collect, top up, upgrade).

**Information exposed:**
- Posts and threads: author, body (Markdown), score, reactions, tips, polls and results, read/unread state.
- Messages: subject, body, participants, reply chains.
- Notifications: app, topic, level, title, body, link, deliveries.
- Channels and balances.

**Today:**
- Feed and Forum under Sneed Hub; Messages, Notifications, Posts and Tips under Sneed Me.
- The SMS-when-mentioned opt-in and the header-chip settings are on `/me`.

**Redesign signals**
- Two discussion engines: ThreadViewer, and the older Discussion used only before a proposal has a thread.
- Two notification systems: header chips and the Notifications service, which has no badge and keeps read state per device.
- Three message composers.
- Personal activity is spread across `/posts`, `/tips`, `/sms`, `/notifications`, `/me` tiles and the `/principal` Posts tab.
- Topic lists, the feed and `/posts` show almost none of the thread's richness (scores, reactions, tips).
- `/tips` ignores thread tips.

Detail: [07-social.md](07-social.md).

### 1.9 Explore: public data about anyone and anything

This domain cuts across the others: the public, read-only views that work for any principal, canister, ledger or SNS, mostly without login. Today most of them sit under "Sneed Hub".

| What | Where | Shows |
|---|---|---|
| Landing | `/`, `/hub` | SNEED stats (FDV, market cap, staked, supply, circulating, members); forum and Sneedex widgets; feature and product cards. The same for every visitor. |
| A user | `/principal?id=`, alias `/user` | Name, avatar, posts and threads, Sneedex offers and bids, neurons across SNSes, transactions (one SNS), balance scan, total value. Actions: send message, nickname; on your own page, "Set Name". It does not show locks, LP positions, NFTs, tips or apps. |
| Users | `/users` | Directory of named principals |
| Any wallet | `/scan_wallet?principal=` | Balances across whitelisted tokens |
| Transactions | `/transactions`, `/transaction` | Ledger history per ledger or SNS; one transaction |
| Ledger audit log | `/tools/icrc3` | ICRC-3 blocks in three views, certified tip, chain verification |
| Money flows | `/tools/money` | Follow the Money graph from a principal |
| Token safety | `/studio/:ledgerId` | Safety rating and about 39 checks in 8 groups |
| NFTs | `/nfts?collection=`, `/nft_item` | Any ICRC-7 collection; one item, including Sneed Lock NFTs |
| Canisters | `/canisters`, `/canister?id=` | Only canisters that have a name or your nickname; one canister's public status |
| SNSes | `/sns` | Directory and per-SNS parameters |
| Neurons | `/neurons`, `/neuron` | All neurons of an SNS (+ hidden holder index); one neuron |
| Proposals | `/proposals`, `/icp_proposals`, `/proposal`, `/icp_proposal` | Lists and details |
| Locks | `/sneedlock_info`, `/tokenlocksoverview`, `/lock/:id` | All locks and TVL; one token's locks; one lock |
| Positions | `/positions`, `/position` | A pool's positions and stats; one position |
| Marketplace | `/sneedex_offers`, `/sneedex_offer/:id` | Offers and asset due diligence |
| Forum & feed | `/forum`, `/topic`, `/thread`, `/post`, `/feed` | Discussions in every SNS |
| DAO data | `/dao_info`, `/rll_info`, `/rll_ledger`, `/revenue`, `/health`, `/architecture` | See 1.11 |

**Redesign signals**
- There is no global search: each explorer has its own input.
- The public profile is heavy and overlaps `/scan_wallet`, `/neurons`, `/sneedex_my` and `/posts`.
- Despite its name, `/canisters` is a list of named canisters, not an explorer.
- Explorers for one kind of entity (a token, say) are scattered: safety report, locks, transactions, ICRC-3 and money flows are all separate pages, and only the safety report links across to the others.

### 1.10 Token & developer tools

**Jobs:** judge whether a token is safe · fix my token's weaknesses · launch a token or add an index (gated) · hand a token over to DAO custody (gated) · trace funds · decode accounts · read raw logs.

**Capabilities**
- **Token Studio** (`/studio`, `/studio/:ledgerId`, `/studio/:ledgerId/:tab`):
  - Check: a safety rating from public data, about 39 checks in 8 groups (controllers, WASM, archives, index, minting, liquidity, locks, cycles…). Each check has a status, summary and details. There is a "Problems only" filter, editable thresholds (kept in this browser), fix suggestions that link to the lock wizard, `/position` and the Harden tab, and a recompressed-build verifier.
  - Harden: owner guidance per weakness.
  - Launch, Index and Hand over: admin preview only.
  - Related token-owner actions live elsewhere, in the wallet: a mint dialog and ledger-suite upgrades for pad-launched tokens.
- **Custody console** (`/guardian`) and `/custody`: admin preview only. All deployments are `live: false`.
- **Token launcher** (`/tools/pad`): admin preview only.
- **Follow the Money** (`/tools/money`): a money-flow graph from a principal on a ledger.
- **ICRC-3 Browser** (`/tools/icrc3`): browse and verify the block log of any Sneed canister or ledger.
- **Account Tool** (`/tools/account`): convert between principals, subaccounts and account IDs; look up balances.
- **Recover Top-up** (`/tools/recover-topup`): see 1.7.
- **Tools hub** (`/tools/main`).

**Information exposed:**
- Ledger metadata and controllers tree.
- Archive and index status.
- Minting account.
- Liquidity and lock coverage.
- Cycles.
- Block logs.
- Money-flow graphs.

**Today:**
- Tools section.
- Token Studio also promoted from the Hub ("Flagship: Sneed Custody").
- Custody also in the Sneed DAO menu (`/guardian`).

**Redesign signals**
- The safety report is already a hub for one token: it links out to about 12 pages. Its rating is never shown on token lists or in the wallet.
- Gated features (launch, index, custody, pad) are promoted from the Hub, the store, the Tools hub and the fix buttons.
- "Custody" points to two different pages.
- Generic developer tools (ICRC-3, Account Tool) are buried, and Recover Top-up isn't on the Tools hub.

Detail: [06-apps-canisters-tools.md](06-apps-canisters-tools.md).

### 1.11 Sneed DAO: transparency and community

**Jobs:** understand Sneed DAO · check its health and treasury · see where revenue goes · support or give feedback · follow Mr Sneed.

**Capabilities**
- **About and lore:** `/dao`, `/lore`, `/disclaimer`.
- **Dashboard** (`/dao_info`):
  - Shows token, governance and neuron statistics and partners.
  - Computes treasury value and reconciliation but never displays them.
- **Health** (`/health`): a tree of automated checks covering SNS core, revenue, vectors and pylons, rewards and custody. Thresholds are kept in this browser.
- **Tokenomics** (`/rll_info`, the Recursive Liquidity Loop):
  - Treasury and NAV breakdown (the only treasury view, behind expanders), loop diagram, component list, ratios.
- **Loop ledger** (`/rll_ledger`): flows in and out of each loop component, configured vs actual shares.
- **Revenue** (`/revenue`): revenue by product and source (neuron maturity, LP fees, product fees).
- **Rewards** (`/rll`): distribution stats and your credits and claims. Duplicates `/rewards`.
- **Products, Partners, Projects** (`/products`, `/partners`, `/projects`): three different lists. `/products` USD values always show 0.
- **Donations:** the header heart and `/donate` (a thank-you wall).
  - Gifts go to the people who build Sneed Hub, not the treasury.
  - On success the donor's principal sends the builders an automatic SMS. The confirm step doesn't mention it.
- **Feedback** (`/feedback?kind=bug|idea`): bugs and ideas become public forum threads; security reports go privately by SMS.
- **Mr Sneed** (`/mr-sneed`):
  - The mascot's on-chain memory, personality, writers and limits.
  - "Forget about me".
  - Filing SNS proposals to change him.
- **Diamond Club** (`/diamond-club`): migrate EXT cards to ICRC-7.

**Information exposed:**
- SNEED supply, circulating and staked amounts, price, FDV and market cap.
- Neuron and dissolve statistics.
- Treasury holdings and NAV.
- Loop configuration and flows.
- Revenue per product.
- Reward distributions.
- Partners, projects and donors.
- Health status.

**Today:** the Sneed DAO section has 19 items, including two external links in the middle.

**Redesign signals**
- Revenue and treasury data is spread over six places (`/rll_info`, `/rll_ledger`, `/revenue`, `/health`, `/dao_info`, Hub). Supply and staked figures are fetched separately on three of them.
- There are five different product lists.
- The DAO menu holds non-DAO tools: an NFT migration (Diamond Club), support (Feedback), custody (`/guardian`), and the only proposal-filing flow (Mr Sneed).
- Several pages wait on canister upgrades: RLL stats banners, admin-only revenue totals, "not open yet" states.

Detail: [09-dao-help.md](09-dao-help.md).

### 1.12 Help & docs

**Capabilities**
- `/help`: an index of 20 topic pages, plus one alias. They use two different layouts, and there is no search.
- Topics: neurons, wallet, install app, Sneed Lock, ICP neuron manager, Sneedex, dapp manager, SNS Jailbreak, trading bot, forum, SMS, notifications, monitor, names, DAO, proposals, Token Studio, tools, Sneedapp, liquid staking.
- No help page for: Swap, Diamond Club, Custody, Convert, Premium. Premium, rewards, donations, feedback, Mr Sneed and lore are folded into `/help/dao`.
- `/api` and `/api/:slug`: per-canister API references (Markdown, Candid, JS IDL). Some docs can only be reached by URL.
- `/architecture`: a live map of about 50 canisters and 29 services. Every service has a route, so it already works as a sitemap.
- Published for agents: `llms.txt`, `llms-full.txt`, `/.well-known/agent-skills/*` (sneed-hub plus six product skills), and the MCP server (in the repo).

**Redesign signals**
- Help is reached through its own menu section, not from the pages it documents. Many feature pages don't link to their help page.
- Route naming mixes `_` and `-`.

Detail: [09-dao-help.md](09-dao-help.md).

### 1.13 Admin & moderation

**Capabilities:**
- **Moderation:** user bans, word blacklist, public names and verification, forum content (also inline on `/thread`, `/post` and `/proposal`), the donation wall.
- **Directories:** partners, projects, token whitelist, the RLL token whitelist, Sneedapp publishers and apps, Sneedex asset types.
- **Fees and prices:**
  - Per product: lock, Sneedex, Sneedapp/factory, pad, monitor, jailbreak, Premium tiers.
  - Admin and partner prices on 8 pages.
  - Fee destination accounts on 8 pages.
- **Limits:** names, canister folders, SMS, forum, Sneedex.
- **Wiring between canisters:** the Premium canister id can be set in 9 places; integration ids; system sender; governance ids.
- **Jobs:** RLL loop, cycle and imports; monitor rounds; the Sneedex expiration timer; token-list maintenance; pad WASM refresh.
- **Code:** factory and app WASMs and official versions.
- **Logs** per product.
- **Treasury and rescue:** withdraw, top up, rescue a canister, return a misplaced NFT.
- **Other:** a profits dashboard; Premium members and promo codes.

**Who can use it:**
- `useAdminCheck` asks only the backend's global `caller_is_admin`, and that only opens the page. Every product canister enforces its own admin list.
- Pages are inconsistent: some have no check in the browser, some flash the page before redirecting, and some check only their own canister.
- 10 admin pages are not in the Admin menu; they are reachable only from the `/admin` dashboard or by URL.
- Two pairs of pages manage the same canister: `/admin/neuron-manager-factory` and `/admin/sneedapp`; `/admin/rewards` and `/admin/rll`.

**Sneed Drive** (`/drive`) is a user product, a notes-first drive with folders, versions, trash, tags, search and sharing. It is soft-launched only for backend admins, behind a browser flag, on a canister that exists only on staging.

Detail: [10-admin.md](10-admin.md).

### 1.14 Global shell and cross-cutting capabilities

These span every domain, and the redesign has to give each one a single home.

| Capability | What it does today | Where |
|---|---|---|
| Section menu + submenu | 10 sections; the active section is guessed from two hard-coded URL lists that disagree, with "Sneed Hub" as the fallback. Clicking a section carries the current query string to the new page. | `Header.jsx` |
| Per-page header | Each of about 135 page files renders its own `<Header>` with props, so header state resets on every navigation. | all pages |
| SNS context | Global current SNS (`?sns=` → localStorage → Sneed). The picker shows only on about 16 SNS pages. Three different pickers exist (header, `/sns`, wizards). | SnsContext |
| Your-neurons VP bar | Voting power reachable through your hotkey neurons for the current SNS, plus "Refresh neurons". | SNS pages only |
| Alert chips | Replies, mentions, SMS, tips, votable proposals, collectibles, bot health, low cycles, update. Toggled in `/me`. | header |
| Quick links | Icon row or hamburger dropdown: Feed, Forum, Sneedex, Sneedapp, Swap, Trading Bot, SNS Jailbreak, Wallet, Rewards, Messages, Premium. Also holds the currency, theme and privacy controls. | header |
| Quick-wallet popup | Principal and account-ID copy, logout, plus a second full wallet: send, lock, transfer canister, bot chores, upgrade, top-up. It has gaps and bugs (see 1.2). | header account button (PrincipalBox) |
| Donate | Heart pill on every page. | header |
| Naming everywhere | Principal, neuron and canister displays resolve public names and your private nicknames, with avatars and Premium badges; the person menu opens on right-click or long-press. | PrincipalDisplay, NeuronDisplay |
| Prices & denomination | One ICPSwap-pool-based price service feeds every USD or other-currency figure; a global display currency. | PriceService, DenominationContext |
| Privacy mode | Hides amounts app-wide (Shift+P). | PrivacyToggle |
| Theme | Light and dark. | ThemeContext |
| Premium tiers | Discounts (Sneedex, lock fees, bots, jailbreak, pad) and higher limits (SMS, forum, nicknames, folders); price-tier badges at checkout. | PremiumContext + each product |
| Fees | ICP service fees shown before confirming (lock, offer, mint, jailbreak, pool creation, notifications). | each flow |
| Pending-operation journals | Local journals for unfinished stakes and top-ups, with "Finish" or "Resume" notices. | 6+ places |
| In-app dialogs | AppDialogHost replaces `window.confirm`/`alert`, though several pages still use the browser dialogs. | global |
| Update detection | Detects a new frontend build; a refresh dialog and an update chip. | FrontendUpdateContext |
| PWA | Installable. The installed app gets a bottom Back/Forward/Refresh bar, not a tab bar. Pairing for iOS login; push notifications through a service worker. | Layout, PwaNavBar, sw.js |
| Staging banner | Shown on the staging copy. | global |
| Coming-soon gate | `LaunchGate` shows admins the real page and everyone else a placeholder. | 6 routes and tabs |

---

## 2. Entity index

Each object the app deals with, where it can be seen, and where it can be acted on. "Wallet" means the `/wallet` token, position or app cards and their modals. "Popup" means the header quick-wallet popup.

| Entity | Seen at | Acted on at |
|---|---|---|
| **Principal / user** | `/principal` (`/user`), `/users`, popup (self), `/me` (self), name and avatar displays everywhere | Send message, nickname, view, copy (person menu everywhere); set own name and avatar (`/me`; name also on own `/principal`) |
| **Name / nickname / avatar** | Everywhere through the naming displays | Public name and avatar: `/me`. Nicknames: `/names`, person menu, `/principal`, `/neuron`, `/canister`, AccountPanel. Verify names: `/admin/names` |
| **Multi-principal account** | `/me` AccountPanel | Merge, roles, hotkey accounts, leave (`/me` only) |
| **Session / pairing / agent delegation** | Login modal, banner, `/pair`, `/connect-agent` | Log in and out (popup), pair, delegate. Delegations can't be listed or revoked |
| **Premium membership** | `/premium`, crown and badges, price badges at checkout | Buy, claim with VP, promo code (`/premium`); grant and revoke (`/admin/premium`) |
| **Token (ICRC-1 ledger)** | Wallet token cards (and Statistics), popup, `/scan_wallet`, `/principal` Wallet tab, `/studio/:id`, `/transactions`, `/tools/icrc3`, `/tools/money`, `/tokenlocksoverview`, lock pages (card in lock mode) | Wallet card: send, receive, buy/sell/swap, convert (ck), wrap, Sneed Lock deposit/withdraw, lock, claim rewards, mint (minting account); add, scan, reorder, remove (zero balance only). Check and harden in Studio. Whitelist (`/admin/tokens`) |
| **Pocket (where a balance sits)** | Token card "in N pockets", pocket switcher, `/wallet/pockets`, send dialog picker | Create, rename, hide, forget, receive, send, move, withdraw, top up, "Move all to Main", per-pocket approvals. Several of these exist only on `/wallet/pockets` |
| **Sneed Lock balance ("Deposited")** | Token card row, wizard "Available to lock", pockets | Deposit and withdraw dialogs, pocket Move, lock flows (three paths) |
| **Recipient / account** | Send dialog (decoded parts, recipient balance), `/tools/account` | Enter as principal, subaccount, ICRC-1 text, account ID, name or QR (four input styles across the app) |
| **Approval (ICRC-2)** | Wallet Approvals tab, token card badge and row, pocket panel | Revoke, change amount or expiry. No "create approval" |
| **Collectible (LP fee, reward, maturity)** | Wallet Collect card, header chip | Bulk claim (ConsolidateModal); per item on wallet cards |
| **Transaction** | `/transactions`, `/transaction`, account history inside `/principal`, `/me`, `/neuron` (SNS tokens only), `/rll_ledger` rows. The wallet itself shows no history | View, CSV export |
| **Native BTC (address, UTXO, send, move, credit)** | `btc.html`, the wallet's Bitcoin card | Send, sweep, move BTC ⇄ ckBTC, signing credit, speed up (RBF/CPFP) |
| **ck conversion** | `/convert` | Get or withdraw ckBTC, ckETH, ckERC-20 |
| **ICRC-3 block** | `/tools/icrc3`, Studio "Block log certified" check | Browse, verify |
| **NFT (ICRC-7 / EXT)** | `/nft` (mine), `/nfts`, `/nft_item`, Sneedex | Send, approve a spender, sell on Sneedex, use as avatar, mint (collection admins), Diamond Club migration |
| **Token lock** | `/lock/:id`, `/sneedlock_info`, `/tokenlocksoverview`, `/tokenlock`, wallet Locks tab, Sneedex and NFT summaries | Create (wizard, wallet modal); dissolve, delay, extend (`/lock/:id`); transfer (`/lock/:id`, wallet); add gas (anywhere it is shown); sell (wallet) |
| **Position lock** | As for a token lock, plus `/positionlock`, `/positions` (🔒), the wallet position card | Create (wizard, wallet modal); extend (wallet legacy call or `/lock/:id`); claim fees and add liquidity (wallet only); transfer; sell (wallet); withdraw after expiry |
| **Lock NFT** | `/lock/:id` panel, `/nft`, `/nft_item`, Sneedex | Send (3 UIs, 3 labels), sell |
| **ICPSwap position** | `/position`, `/positions`, wallet Liquidity tab, `/positionlock` | `/position`: collect, move range, remove, transfer, lock, withdraw. Wallet: send, claim, withdraw. Header: bulk collect. Import guide in 4 places |
| **Pool** | `/position` header, `/positions?swap=`, pool pickers | Create pool (`/position`, no params); register (wallet "+ Add Swap Pair", wizard); withdraw unused balance |
| **Sneedex offer** | `/sneedex_offers`, `/sneedex_offer/:id`, `/sneedex_my`, Hub widget, `/feed`, `/principal` Trades tab, swap auction cards | Create (wizard); escrow, activate, accept, cancel, claim payment (offer page, My Sneedex); process expiration (anyone) |
| **Bid** | Offer page history, My Sneedex, wallet pockets | Place, buyout, confirm pending, claim assets, refund (My Sneedex only), withdraw escrow (pockets) |
| **Swap quote / route** | `/swap`, SwapModal | Swap, buy from auction, split trade; slippage (`/swap`, `/me`) |
| **SNS (DAO)** | `/sns`, header picker, `/active_proposals` groups, `/apps` SNS folders | Select as current SNS; create its forum |
| **SNS neuron** | `/neurons`, `/neuron`, wallet Neurons tab, `/me` My Neurons, `/principal`, VP bar, proposal voting panels, Sneedex | Create (wizard, wallet); stake, disburse, maturity, split, send (wallet only); dissolve (`/neuron`, wallet); permissions and followees (`/neuron` only); public name (`/me` only); nickname (3 places); sell (Sneedex); import (Jailbreak) |
| **NNS neuron (hotkeyed)** | Wallet ICP card, `/icp_proposal` panel, `/active_proposals` | Vote (3 places); dissolve, disburse, maturity, hotkeys (wallet). No detail page |
| **NNS neuron (bot-owned)** | `/icp_neuron_manager/:id`, wallet and DappCardModal summaries, Sneedex | Full management in the bot console. No voting |
| **Proposal (SNS / NNS)** | `/proposals`, `/icp_proposals`, `/proposal`, `/icp_proposal`, `/active_proposals`, `/neuron` history, forum Proposals topic | Vote (5 places, 3 implementations); discuss; CSV export |
| **Sneed DAO proposal (filing)** | — | `/mr-sneed` only (registry and proxy built, not mounted); otherwise off-site |
| **Voting reward (RLL)** | `/rewards`, `/rll`, wallet token card, `/rll_info`, `/health` | Claim (`/rewards`, `/rll`, wallet, collect-all) |
| **Jailbreak script** | `/tools/sns_jailbreak`, `/tools/sns_jailbreak_list` | Pay and generate, re-show, verify, delete, withdraw ICP |
| **App (store entry) / version / publisher** | `/sneedapp`, `/sneedapp/app/:id`, mint wizard, publisher console | Mint; publisher: info, owners, apps, versions, WASMs, revenue; admin: create, verify, set cut |
| **Canister** | `/canister`, `/canisters`, `/apps`, wallet Apps tab, popup, Studio checks, Sneedex | Top up (ICP, card, ck-USDC bulk); controllers, upgrade and reinstall (`/canister`); transfer control (wallet, popup); name and nickname; track and group (`/apps`); sell (Sneedex) |
| **Canister group** | `/apps` | Create, rename, nest, delete, move apps (drag only) |
| **Pending top-up / pending stake** | Notices in 6+ places, `/tools/recover-topup` | Resume, finish, dismiss, recover |
| **Bot (staking / trading)** | Bot console, `/apps`, wallet Apps tab, DappCardModal, header health dialogs, `/create_icp_neuron` (staking list) | Upgrade (3 UIs), top up (5), controllers, botkeys, chores (bot page, DappCardModal, SneedScript), withdraw |
| **Chore** | Bot console, wallet / DappCardModal, header health | Start, pause, stop, Run Once, schedule, configure |
| **Botkey** | Bot console Botkeys tab | Add, edit, remove |
| **Trading purse / strategy / trade** | Trading Bot Wallet, Chores, Performance and Logs tabs; Script | Fund, reclaim, send; action, rebalance and distribution editors; Quick Trade; circuit-breaker rules; SneedScript |
| **Forum / topic / thread / post** | `/forum`, `/topic`, `/thread`, `/post`, `/feed`, `/posts`, `/principal` Posts tab, proposal pages, Hub widget | Create forum and special topics; thread (+poll); post, reply, edit, delete; vote; react; tip; mention; message author. No thread delete for users |
| **Poll** | Thread view; read-only in feed | Create (topic form, thread view: two forms); vote |
| **Tip** | Tip pills, `/tips` (post tips only), header chip | Send (TipModal from posts and threads) |
| **Message / conversation** | `/sms` (lists, Conversations), `/msg/:id` | Compose (3 composers), reply, reply all, remove self |
| **Notification / channel / message balance** | `/notifications` Inbox, `/notifications/:id` | Settings tab: apps, levels, channels, quiet hours, pause; top up and withdraw balance |
| **Header alert** | Header chips | Opens the target page or an action dialog (collect, top up, upgrade) |
| **Donation / feedback** | `/donate` wall, `/feedback` recent | Donate (header, `/donate`); file a bug, idea or security report |
| **DAO metrics / treasury / loop / revenue** | `/dao_info`, `/rll_info`, `/rll_ledger`, `/revenue`, `/health`, Hub tiles | View only (admin jobs in `/admin/rewards`, `/admin/rll`) |
| **Product / partner / project** | `/products`, `/partners`, `/projects`, `/revenue`, `/dao_info`, Hub | Admin CRUD for partners and projects only |
| **Mr Sneed memory** | `/mr-sneed` | Forget-about-me; propose personality, writers or limits changes |
| **Custody deployment / pad launch / index launch** | `/guardian`, Studio tabs, `/tools/pad` (admin preview) | Admin preview only |
| **Drive note / folder** | `/drive` (admins, staging) | Create, version, trash, tag, search, share |

---

## 3. Findings for the restructure

### 3.1 How navigation works today

- **Sections:** Sneed Hub (public explorer and landing), Sneed Me (personal), Sneed DAO, Sneedapp, Sneedex, Sneed Lock, Liquid Staking, Help, Tools, Admin. Only the active section's submenu shows, as a row of links under the top bar.
- **The section switcher sits behind the page title**, and the hamburger holds quick links and global settings. On the live site it takes a click on "Sneed Hub" to discover that other sections exist.
- **The active section is guessed from the URL:**
  - Two hard-coded path lists in `Header.jsx` disagree. Anything unlisted falls back to Sneed Hub; `/wallet/pockets`, for example, shows the Hub submenu.
  - Several pages map to a section whose submenu doesn't list them (`/canister` → Sneed Me; `/icp_neuron_manager/*` → Liquid Staking).
- **Launchers that duplicate the menu, each grouped differently:** the header quick links, the `/me` Quick Access grid, the Hub feature cards, the quick-wallet popup's links, and the alert chips.
- **There is no mobile-specific navigation.** The same header collapses on scroll. The installed app adds only Back/Forward/Refresh.

### 3.2 One capability, many UIs

| Capability | Separate UIs today |
|---|---|
| Manage an SNS neuron | `/neuron` (dissolve, permissions, followees) · wallet token card (stake, disburse, maturity, split, send, dissolve) · `/me` (public name) · staking bot console (same verbs for NNS) |
| Create an SNS neuron | `/sns_neuron_wizard` · wallet "Create New Neuron" |
| Vote | `/proposals` and `/icp_proposals` quick vote · `/active_proposals` (+ bulk) · `/proposal` panel · `/icp_proposal` panel |
| Claim voting rewards | `/rewards` · `/rll` · wallet token card · wallet collect-all |
| Create a lock | Lock wizard · wallet LockModal / LockPositionModal (fewer features, fewer safeguards) |
| Extend a lock | `/lock/:id` (`extend_lock`) · wallet "Lock" on a position (`update_position_lock`) |
| Transfer a lock | `/lock/:id` "Send the NFT" · wallet "Transfer Ownership" · wallet "Transfer" · NFT pages |
| Collect LP fees | Wallet card · header Consolidate · `/position` "Collect fees" (unlocked only) |
| Import a position from ICPSwap | Guide in 4 places, with 3 polling implementations |
| Settle a Sneedex offer | Offer page · My Sneedex · wallet pockets (escrow withdrawals) |
| Buy out an offer | Offer page · swap widget |
| "My canisters" | Store "My Canisters" · `/apps` · wallet Apps tab · popup DappCardModal |
| Top up cycles | `/canister` · bulk dialog (4 openers) · both bot consoles · mint extra gas · card link · recover tool |
| Upgrade a bot | Staking console · shared console (trading) · UpgradeBotsDialog (4 openers) · `/canister` manual WASM |
| Bot console | IcpNeuronManager inline (about 4,300 lines) · BotManagementPanel (trading) |
| Discuss a proposal | ThreadViewer · legacy Discussion (before a thread exists) |
| Send a direct message | `/sms` compose modal · global MessageDialog · Conversations composer |
| Create a poll | Topic form · `Poll.jsx` (different validation) |
| Alerts | Header chips · `/notifications` service · Sneedex DM alerts on `/me` · "SMS when mentioned" on `/me` |
| Wallet | `/wallet` · quick-wallet popup (PrincipalBox), with its own send code and feature gaps |
| Send tokens | Wallet.jsx executor · popup (WalletContext) executor · pockets executor |
| Move tokens in or out of the Sneed Lock balance | Deposit/Withdraw dialogs · pocket Move · lock flows |
| Collect fees, rewards, maturity | Wallet Collect card · header collectibles chip (claim code written twice) · per-card claims |
| Enter a subaccount | Four input styles (SubaccountEditor, send dialog, Account Tool, pockets) |
| BTC ⇄ ckBTC | `/convert` (any wallet or address) · `btc.html` Move tabs (own address) |
| Swap | `/swap` · SwapModal in token cards and offer pages · Quick Trade in the trading bot (no quote) |
| Settings | Header Quick Links dropdown · `/me` Settings · popup · Shift+P |
| Personal activity | `/posts` · `/tips` · `/sms` · `/notifications` · `/me` tiles · own `/principal` |
| Product lists | `/products` · `/revenue` · `/dao_info` · `/projects` · Hub featured · `/architecture` |
| Revenue and treasury | `/rll_info` · `/rll_ledger` · `/revenue` · `/health` · `/dao_info` · Hub |

### 3.3 Same label, different place

- **"Rewards":** `/rewards` (Sneed Me) and `/rll` (Sneed DAO).
- **"NFTs":** `/nfts` (browse any collection, Hub) and `/nft` (mine, Me).
- **"Custody":** `/guardian` (Sneed DAO) and `/custody` (Tools).
- **"Apps" / "My Apps":** both `/apps`, in Sneed Me and in Sneedapp.
- **"Hot Keys"** (NNS neuron hotkeys) vs **"Botkeys"** (bot permissions, called "hotkey" in the API) vs **SNS neuron hotkeys/permissions**.
- **"Sneed wallet":** means both the logged-in principal and the Sneed Lock subaccount.
- **"Premium":** paid membership vs the VP-based "Sneed DAO members only" gate.
- **Page titles that don't match their menu labels:**
  - "Voting" vs "Active Proposals".
  - "Tokenomics" (`/rll_info`).
  - "Loop ledger" vs "Revenue loop ledger".
  - "SNS Liquid Staking" vs "SNS Neuron Wizard".
  - "Thank-you wall" vs "Support Sneed Hub".

### 3.4 Buried or hidden capabilities

- **Reachable only by URL or a deep link:** `/connect-agent`, `/outrun` (game), `/tools/escrow/swap`, the `/neurons?tab=stakers` holder index, hidden `/api/:slug` docs, `/sns_neuron_wizard` with params, `/sms?recipient=`.
- **Unlinked pages:** `/wallet/pockets`, which appears only once an extra pocket exists, so new users create their first pocket from a token card's "New pocket". Also `/convert` (only from ck token cards), `btc.html`, `/scan_wallet`, `/nft_item`, `/transaction`.
- **Strong features in the wrong place:**
  - The ICRC-2 approvals manager is the fourth wallet tab.
  - NFT approvals live inside expanded `/nft` cards.
- **Only inside a modal or one specific page:**
  - Bot chores outside the bot pages (DappCardModal).
  - "Refund Bid" (My Sneedex).
  - Pool creation (`/position` with no params).
  - Neuron public names (`/me`).
  - Multi-principal accounts (`/me`).
  - Lock gas tanks.
  - Withdrawing a deposited-but-unlocked position (behind "Claim Fees").
  - Trading-bot engine settings (Logs → Logging Settings).
  - Capital-deployed adjustment (a hidden edit icon).
- **The person menu** (message, nickname, profile) opens only on right-click or long-press.
- **The only treasury breakdown** sits behind expanders on `/rll_info`.
- **Owner-only consoles linked from public places:** the publisher console is linked from "by <publisher>" on app pages, and everyone who isn't an owner sees "You are not an owner".

### 3.5 Promoted but not available

- **Admin preview only** (everyone else sees a placeholder): Token Studio Launch, Index and Hand over; `/custody`; `/guardian`; `/tools/pad`.
  - They are promoted from the Hub ("Flagship: Sneed Custody"), store pseudo-apps, the Tools hub ("Launch a token") and Studio fix buttons.
- **Staging only:** Sneed Monitor (inside notification settings), Sneed Drive, Sneed Pad, Guardian canisters.
- **Placeholders:** `/tools/escrow`, `/doc`, `/tokenomics`, `/tokenomics_info`, `/me_info`.
- **Waiting on canister upgrades or config:** RLL stats banners (proposal #444), admin-only revenue totals, Diamond Club swap "not open yet", donations and feedback "not open yet" states, the Mr Sneed proposal types, Telegram and email notification channels.
- **Swap** is advertised as comparing DEXes with split and multi-hop routes. In fact it quotes one ICPSwap pool plus Sneedex buyouts.

### 3.6 Broken, dead or legacy

- **Broken:**
  - `/dashboard` misreads lock data, and help pages still link to it.
  - `/tokenlocksoverview`, `/tokenlock` and `/positionlock` spin forever without their parameters; `/positionlock` shows only the first position.
  - `/products` USD values are always 0.
  - `/me` links `/posts?tab=my-threads`, which opens "Replies to Me".
  - Bot `?tab=` deep links are ignored.
  - The staking-bot stats can show the wrong neuron after a refresh.
  - The trading-bot action editor wipes frequency limits and preferred DEX on save.
  - The safety report's "Liquidity under safe control" link always goes to `/sneedlock`.
  - The quick-wallet popup's position Send does nothing.
  - Token registration errors are never shown.
  - The transaction list's error state never renders, and its Type/Time sorts do nothing in account mode.
  - The Liquidity tab's "Claim Fees" on an unlocked deposited position also withdraws it from Sneed Lock, and nothing says so.
- **Legacy aliases:**
  - `/managelp` → `/position`; `/user` → `/principal`.
  - `/help/canister-manager` → `/help/dapp-manager`.
  - `/tools/token_safety*` → `/studio*`.
  - `/notifications/inbox` → `?tab=inbox`.
  - `/tokenlock` and `/positionlock` (older than `/lock/:id`, still link targets).
- **Dead code worth deleting before restructuring:**
  - Unmounted `ProposeDialog` / `useProposeAction`.
  - The non-inline modes of `HotkeyNeurons` and `IcpHotkeyNeurons`.
  - TradingBot `AccountsPanel` and `PursesPanel` (about 1,300 lines).
  - About 600 lines of Feed renderer, plus old Posts and Tips renderers.
  - `Banner.jsx`.
  - Header's `showTotalValue` prop.
  - `BotChoreHealthDialog` and `BotLogAlertDialog`.

### 3.7 Mobile gaps

- Drag-and-drop only, with no touch support:
  - `/apps`: moving apps between groups, and reordering.
  - Trading bot: rebalance targets and token chips.
- No bottom tab bar; the submenu row slides away on scroll.
- `/sms` Conversations is the one area with a deliberate phone layout (list or conversation, not both).
- The money-flow graph has no touch gestures.

### 3.8 Safety gaps the redesign could fix in passing

- **Irreversible or money-moving actions with no confirmation:**
  - `/position`: remove, move range, transfer.
  - `/lock/:id`: extend, increase delay.
  - Bots: disburse, split, merge, start dissolving, withdraw and send, token removal, recovery, executing a script.
- **`/position` "Transfer"** has no stuck-recipient guard; it allows the Sneed Lock canister, the anonymous principal, or a pool.
- **Trading-bot panels** never check botkey permissions in the UI; users see every tab and get raw canister errors.
- **The disclaimer** isn't linked from login or any flow that moves money.
- **Browser dialogs** (`window.confirm`/`alert`) remain in PositionCard, `Poll.jsx`, `/names`, `/me` thresholds and the publisher console.

### 3.9 "My stuff" is everywhere

| Personal data | Places it appears |
|---|---|
| Balances | `/wallet`, popup, own `/principal` Wallet tab, `/scan_wallet` |
| Neurons | Wallet Neurons tab, `/me` Neurons tab, `/neurons`, own `/principal`, VP bar, voting panels |
| Transactions | Wallet history, `/me` tab, `/principal` tab, `/transactions` |
| Trades | `/sneedex_my`, own `/principal` Trades tab, wallet pockets |
| Posts and tips | `/posts`, `/tips`, `/me` tiles, own `/principal` Posts tab |
| Canisters and bots | Store "My Canisters", `/apps`, wallet Apps tab, popup, `/create_icp_neuron` |
| Locks | Wallet Locks tab, `/sneedlock_info?owner=me`, `/me` "My Locks" link |
| Escrowed or parked funds | `/wallet/pockets` (Sneedex escrows, ICPSwap leftovers, service balances), `/sneedex_my`, `/position`, `/notifications` balance |
| Settings | `/me` Settings (including wallet, bot, swap, Sneedex and SMS settings), header Quick Links, popup |

A redesign that gives each personal object one canonical home (and links to it from everywhere else) would remove most of the duplication in 3.2.

---

## Appendix A. Route inventory

"Section" is the Header section the route highlights today ("(none)" means it is unmapped and falls back to the Sneed Hub submenu). "Domain" is the catalog domain from section 1. "Kind":
- **Explore:** public read.
- **Mine:** personal; needs login.
- **Flow:** a create, act or convert wizard.
- **Console:** manage one thing.
- **Info:** static or DAO information.
- **Landing**, **Help**, **Admin**, **Alias**, **Legacy**, **Placeholder**, **Hidden**.

| Route | Page component | Section today | Domain | Kind | Notes |
|---|---|---|---|---|---|
| `/` | Hub | Sneed Hub | Explore | Landing | Hub landing page (same for everyone) |
| `/canisters` | Canisters | Sneed Hub | Explore | Explore | Only named/nicknamed canisters |
| `/hub` | Hub | Sneed Hub | Explore | Landing | Alias of / |
| `/nft_item` | NftItem | Sneed Hub | Explore | Explore | One NFT |
| `/nfts` | NftBrowser | Sneed Hub | Explore | Explore | Browse any ICRC-7 collection |
| `/principal` | Principal | Sneed Hub | Explore | Explore | Public user profile (5 tabs) |
| `/scan_wallet` | ScanWallet | (none) | Explore | Explore | Any principal's balances; no nav section |
| `/transaction` | Transaction | Sneed Hub | Explore | Explore | One ledger transaction |
| `/transactions` | Transactions | Sneed Hub | Explore | Explore | Ledger transactions (per SNS / ledger) |
| `/user` | Principal | Sneed Hub | Explore | Alias | Same component as /principal |
| `/users` | Users | Sneed Hub | Explore | Explore | Named principals directory |
| `/connect-agent` | ConnectAgent | (none) | Account & identity | Flow | AI-agent delegation; no nav entry, no revoke/list |
| `/login` | Login | (none) | Account & identity | Flow | Login page; login is normally the global modal |
| `/me` | Me | Sneed Me | Account & identity | Mine | Profile + launcher + neurons + transactions + settings |
| `/me_info` | MeInfo | (none) | Account & identity | Placeholder | ComingSoon |
| `/names` | Names | Sneed Me | Account & identity | Mine | "Address Book" (private nicknames) |
| `/pair` | PairApp | (none) | Account & identity | Flow | Approve app<->browser pairing; no nav section |
| `/premium` | Premium | Sneed Me | Account & identity | Flow | Sneed Premium: buy / claim with VP / promo |
| `/btc.html` | BtcApp (separate HTML entry) | (none: no SPA header) | Wallet & assets | Flow | Native Bitcoin wallet: #send, #move, #move-tobtc, #credit, #activity |
| `/convert` | Convert | (none) | Wallet & assets | Flow | BTC/ETH <-> ck tokens; no nav section |
| `/nft` | Nft | Sneed Me | Wallet & assets | Mine | My NFTs (vs /nfts browse) |
| `/wallet` | Wallet | Sneed Me | Wallet & assets | Mine | Tabs: Tokens, Liquidity, Apps, Approvals, Statistics |
| `/wallet/pockets` | Pockets | (none) | Wallet & assets | Mine | Shows the Sneed Hub submenu (unmapped) |
| `/sneedex` | Sneedex | Sneedex | Trade | Info | About Sneedex (lists 4 of 7 asset types) |
| `/sneedex_create` | SneedexCreate | Sneedex | Trade | Flow | Create offer wizard |
| `/sneedex_my` | SneedexMy | Sneedex | Trade | Mine | My offers & bids |
| `/sneedex_offer/:id` | SneedexOffer | Sneedex | Trade | Explore/Console | Offer page: inspect, bid, buyout, settle |
| `/sneedex_offers` | SneedexOffers | Sneedex | Trade | Explore | Marketplace browse/filter |
| `/swap` | Swap | Sneedex | Trade | Flow | Swap (ICPSwap + Sneedex buyouts); inert when logged out |
| `/tools/escrow` | ToolsEscrow | Tools | Trade | Placeholder | ComingSoon |
| `/tools/escrow/swap` | EscrowSwap | Tools | Trade | Hidden | OpenChat escrow lookup; URL-only |
| `/dashboard` | Dashboard | (none) | Lock & liquidity | Legacy | Broken legacy lock list; still linked from help |
| `/lock/:id` | Lock | Sneed Lock | Lock & liquidity | Console | One lock: dissolve, extend, NFT send, gas |
| `/lock_wizard` | LockWizard | Sneed Lock | Lock & liquidity | Flow | Create lock (token or position) |
| `/managelp` | Position | Sneed Hub | Lock & liquidity | Alias | Alias of /position (not a redirect) |
| `/position` | Position | Sneed Hub | Lock & liquidity | Console | ICPSwap position page; no params = create pool |
| `/positionlock` | PositionLock | Sneed Lock | Lock & liquidity | Legacy | Shareable position-lock view; shows only first position |
| `/positions` | Positions | Sneed Hub | Lock & liquidity | Explore | Positions (mine / by pool); listed under Sneed Hub |
| `/sneedlock` | Sneedlock | Sneed Lock | Lock & liquidity | Info | "Overview" marketing page |
| `/sneedlock_info` | SneedlockInfo | Sneed Lock | Lock & liquidity | Explore | "All Locks" explorer + TVL |
| `/tokenlock` | TokenLock | Sneed Lock | Lock & liquidity | Legacy | Shareable token-lock view; still the target of "Link" buttons |
| `/tokenlocksoverview` | TokenLocksOverview | (none) | Lock & liquidity | Explore | Per-token locks; spins forever without ?ledger |
| `/create_icp_neuron` | CreateIcpNeuron | Liquid Staking | Stake | Flow | Mint ICP Staking Bot; list my staking bots |
| `/liquid_staking` | LiquidStaking | Liquid Staking | Stake | Info | Static hub linking staking flows |
| `/neuron` | Neuron | Sneed Hub | Stake | Explore/Console | SNS neuron detail + dissolve/permissions/followees |
| `/neurons` | Neurons | Sneed Hub | Stake | Explore | SNS neuron explorer + hidden ?tab=stakers holder index |
| `/sns_neuron_wizard` | SnsNeuronWizard | Liquid Staking | Stake | Flow | Stake SNS tokens into a neuron |
| `/tools/sns_jailbreak` | SnsJailbreak | Tools | Stake | Flow | Import SNS neuron (add full controller); paid; logged-in nav only |
| `/tools/sns_jailbreak_list` | SnsJailbreakList | Tools | Stake | Mine | "My Scripts" |
| `/active_proposals` | ActiveProposals | Sneed Me | Govern | Mine | "Voting": cross-DAO votable inbox, bulk vote |
| `/icp_proposal` | IcpProposal | Sneed Hub | Govern | Explore | NNS proposal detail + vote |
| `/icp_proposals` | IcpProposals | Sneed Hub | Govern | Explore | NNS proposals list, quick vote, CSV |
| `/proposal` | Proposal | Sneed Hub | Govern | Explore | SNS proposal detail, vote, payload, discussion |
| `/proposals` | Proposals | Sneed Hub | Govern | Explore | SNS proposals list, quick vote, CSV |
| `/rewards` | Rewards | Sneed Me | Govern | Mine | "Rewards" (Me menu): claim RLL voting rewards; duplicates /rll |
| `/sns` | Sns | Sneed Hub | Govern | Explore | SNS directory + per-SNS parameters |
| `/apps` | Apps | Sneed Me | Apps, canisters & bots | Mine | "App Manager": groups, health, bulk upgrade/top-up |
| `/canister` | Canister | Sneed Me | Apps, canisters & bots | Explore/Console | Any canister; controllers get mgmt + upgrade |
| `/icp_neuron_manager/:canisterId` | IcpNeuronManager | Liquid Staking | Apps, canisters & bots | Console | ICP Staking Bot console (NNS neurons) |
| `/sneedapp` | SneedApp | Sneedapp | Apps, canisters & bots | Explore | App store catalogue + My Canisters |
| `/sneedapp/app/:appId` | SneedAppDetail | Sneedapp | Apps, canisters & bots | Explore | App detail + versions |
| `/sneedapp/mint/:appId` | SneedAppMint | Sneedapp | Apps, canisters & bots | Flow | Generic mint wizard |
| `/sneedapp/publisher/:publisherId` | SneedAppPublisher | Sneedapp | Apps, canisters & bots | Console | Publisher console (owner-only) |
| `/tools/recover-topup` | ToolsRecoverTopUp | Tools | Apps, canisters & bots | Flow | Recover a cycles top-up by block index; not on Tools hub |
| `/trading_bot/:canisterId` | TradingBot | (none) | Apps, canisters & bots | Console | Trading Bot console; no nav section |
| `/feed` | Feed | Sneed Hub | Community & messaging | Explore | Cross-SNS activity feed (+ Sneedex auctions) |
| `/forum` | Forum | Sneed Hub | Community & messaging | Explore | Per-SNS forum index |
| `/msg/:id` | Message | Sneed Me | Community & messaging | Mine | One message thread |
| `/notifications` | Notifications | Sneed Me | Community & messaging | Mine | Tabs: Inbox, Settings |
| `/notifications/:id` | NotificationView | Sneed Me | Community & messaging | Mine | One notification (also push landing) |
| `/notifications/inbox` | NotificationInbox | Sneed Me | Community & messaging | Alias | Redirects to ?tab=inbox |
| `/post` | Post | Sneed Hub | Community & messaging | Explore | One post in context + vote breakdown |
| `/posts` | Posts | Sneed Me | Community & messaging | Mine | Replies to me, mentions, my posts, my threads |
| `/sms` | SMS | Sneed Me | Community & messaging | Mine | Inbox/Sent/All, Conversations, compose |
| `/thread` | Thread | Sneed Hub | Community & messaging | Explore | Forum thread |
| `/tips` | Tips | Sneed Me | Community & messaging | Mine | Tips sent/received (post tips only) |
| `/topic/:topicId` | Topic | Sneed Hub | Community & messaging | Explore | Topic: subtopics + threads |
| `/custody` | TokenStudio intent="handover" | Tools | Token & developer tools | Placeholder | Token Studio hand-over intent; gated |
| `/guardian` | [ComingSoon:Sneed custody] Guardian ComingSoonPage | Sneed DAO | Token & developer tools | Placeholder | Custody console; ComingSoon for non-admins; label "Custody" also used by /custody |
| `/studio` | TokenStudio | Tools | Token & developer tools | Flow | Token Studio (Check / Harden; Launch, Index, Hand over gated) |
| `/studio/:ledgerId` | TokenStudio | Tools | Token & developer tools | Explore | Safety report for a ledger |
| `/studio/:ledgerId/:tab` | TokenStudio | Tools | Token & developer tools | Flow | harden \| handover \| index |
| `/tools/account` | ToolsAccount | Tools | Token & developer tools | Flow | Account Tool (principal/subaccount/account-id) |
| `/tools/icrc3` | ToolsIcrc3 | Tools | Token & developer tools | Explore | ICRC-3 block log browser/verifier |
| `/tools/main` | ToolsMain | Tools | Token & developer tools | Info | Tools hub |
| `/tools/money` | FollowMoney | Tools | Token & developer tools | Explore | Follow the Money graph |
| `/tools/pad` | [ComingSoon:Token launcher] SneedPad ComingSoonPage | Tools | Token & developer tools | Placeholder | Token launcher; gated |
| `/tools/token_safety` | TokenSafety | Tools | Token & developer tools | Alias | Redirect to /studio |
| `/tools/token_safety/:ledgerId` | TokenSafety | Tools | Token & developer tools | Alias | Redirect to /studio/:ledgerId |
| `/dao` | Dao | Sneed DAO | Sneed DAO | Info | About |
| `/dao_info` | DaoInfo | Sneed DAO | Sneed DAO | Info | "Dashboard" |
| `/diamond-club` | DiamondClub | Sneed DAO | Sneed DAO | Flow | EXT -> ICRC-7 NFT migration tool |
| `/disclaimer` | Disclaimer | Sneed DAO | Sneed DAO | Info | Not linked from login or money flows |
| `/donate` | Donate | Sneed DAO | Sneed DAO | Info | "Thank-you wall" + donate dialog (donations go to builders) |
| `/feedback` | Feedback | Sneed DAO | Sneed DAO | Flow | Bug/idea -> public forum thread; private report -> SMS |
| `/health` | Health | Sneed DAO | Sneed DAO | Info | Automated health checks (tree) |
| `/lore` | Lore | Sneed DAO | Sneed DAO | Info |  |
| `/mr-sneed` | MrSneed | Sneed DAO | Sneed DAO | Info | Memory browser + only live in-app SNS proposal filing |
| `/partners` | Partners | Sneed DAO | Sneed DAO | Info |  |
| `/products` | Products | Sneed DAO | Sneed DAO | Info | Hard-coded product list; USD values always 0 |
| `/projects` | Projects | Sneed DAO | Sneed DAO | Info | Admin-maintained list |
| `/revenue` | Revenue | Sneed DAO | Sneed DAO | Info | Revenue by product |
| `/rll` | RLL | Sneed DAO | Sneed DAO | Mine | "Rewards" (DAO menu): RLL stats + claim; duplicates /rewards |
| `/rll_info` | RLLInfo | Sneed DAO | Sneed DAO | Info | "Tokenomics": treasury/NAV, loop diagram, components |
| `/rll_ledger` | RllLedger | Sneed DAO | Sneed DAO | Info | "Loop ledger" |
| `/tokenomics` | Tokenomics | (none) | Sneed DAO | Placeholder | ComingSoon |
| `/tokenomics_info` | TokenomicsInfo | (none) | Sneed DAO | Placeholder | ComingSoon |
| `/api` | Api | Help | Help & docs | Help | Developer API index |
| `/api/:slug` | ApiDoc | Help | Help & docs | Help | Some slugs only reachable by URL |
| `/architecture` | Architecture | Help | Help & docs | Help | Live map of canisters & services; usable as sitemap |
| `/doc` | Doc | (none) | Help & docs | Placeholder | "Documentation coming soon!" |
| `/help` | Help | Help | Help & docs | Help |  |
| `/help/app` | HelpApp | Help | Help & docs | Help | Install as app |
| `/help/canister-manager` | HelpCanisterManager | Help | Help & docs | Alias | Back-compat alias of /help/dapp-manager |
| `/help/dao` | HelpDao | Help | Help & docs | Help | Also covers Premium, rewards, donations, feedback, Mr Sneed, lore |
| `/help/dapp-manager` | HelpCanisterManager | Help | Help & docs | Help |  |
| `/help/forum` | HelpForum | Help | Help & docs | Help |  |
| `/help/icp-neuron-manager` | HelpIcpNeuronManager | Help | Help & docs | Help |  |
| `/help/liquid-staking` | HelpLiquidStaking | Help | Help & docs | Help |  |
| `/help/monitor` | HelpMonitor | Help | Help & docs | Help | Presents Monitor as live; prod has none |
| `/help/names` | HelpNames | Help | Help & docs | Help |  |
| `/help/neurons` | HelpNeurons | Help | Help & docs | Help |  |
| `/help/notifications` | HelpNotifications | Help | Help & docs | Help |  |
| `/help/proposals` | HelpProposals | Help | Help & docs | Help |  |
| `/help/sms` | HelpSms | Help | Help & docs | Help |  |
| `/help/sneedapp` | HelpSneedapp | Help | Help & docs | Help |  |
| `/help/sneedex` | HelpSneedex | Help | Help & docs | Help |  |
| `/help/sneedlock` | HelpSneedlock | Help | Help & docs | Help | Links to broken /dashboard |
| `/help/sns_jailbreak` | HelpSnsJailbreak | Help | Help & docs | Help |  |
| `/help/token-studio` | HelpTokenStudio | Help | Help & docs | Help |  |
| `/help/tools` | HelpTools | Help | Help & docs | Help |  |
| `/help/trading_bot` | HelpTradingBot | Help | Help & docs | Help |  |
| `/help/wallet` | HelpWallet | Help | Help & docs | Help |  |
| `/admin` | Admin | Admin | Admin | Admin |  |
| `/admin/admins` | AdminsAdmin | Admin | Admin | Admin |  |
| `/admin/canisters` | CanisterGroupsAdmin | Admin | Admin | Admin |  |
| `/admin/donations` | AdminDonations | Admin | Admin | Admin |  |
| `/admin/feedback` | AdminFeedback | Admin | Admin | Admin |  |
| `/admin/forum` | AdminForum | Admin | Admin | Admin |  |
| `/admin/mon` | SneedMonAdmin | Admin | Admin | Admin |  |
| `/admin/names` | AdminNames | Admin | Admin | Admin |  |
| `/admin/network` | NetworkAdmin | Admin | Admin | Admin |  |
| `/admin/neuron-manager-factory` | IcpNeuronManagerFactoryAdmin | Admin | Admin | Admin |  |
| `/admin/pad` | SneedPadAdmin | Admin | Admin | Admin |  |
| `/admin/partners` | AdminPartners | Admin | Admin | Admin |  |
| `/admin/premium` | SneedPremiumAdmin | Admin | Admin | Admin |  |
| `/admin/profits` | ProfitsAdmin | Admin | Admin | Admin |  |
| `/admin/projects` | AdminProjects | Admin | Admin | Admin |  |
| `/admin/rewards` | RewardsAdmin | Admin | Admin | Admin |  |
| `/admin/rll` | RllAdmin | Admin | Admin | Admin |  |
| `/admin/sms` | SmsAdmin | Admin | Admin | Admin |  |
| `/admin/sneedapp` | SneedAppAdmin | Admin | Admin | Admin |  |
| `/admin/sneedex` | SneedexAdmin | Admin | Admin | Admin |  |
| `/admin/sneedlock` | SneedLockAdmin | Admin | Admin | Admin |  |
| `/admin/sns_jailbreak` | AdminSnsJailbreak | Admin | Admin | Admin |  |
| `/admin/tokens` | TokensAdmin | Admin | Admin | Admin |  |
| `/admin/users/bans` | UserBans | Admin | Admin | Admin |  |
| `/admin/words` | WordBlacklist | Admin | Admin | Admin |  |
| `/drive` | Drive | Admin | Admin | Hidden | Sneed Drive beta (admin + browser flag + staging canister) |
| `/outrun` | OutRun | (none) | Hidden | Hidden | "SNEED RUN" arcade game; no header, no links |

The 26 `/admin/*` routes are listed with their access checks in [10-admin.md](10-admin.md).
