# Account & identity, the app shell, and the Hub landing page

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Source root: `src/app_sneeddao_frontend/src`. Code read: `App.jsx`, `components/Header.jsx`, `components/Layout.jsx`, `components/PwaNavBar.jsx`, `components/SnsDropdown.jsx`, `components/DonateButton.jsx`, `PrincipalBox.jsx`, banners/dialog hosts, contexts (Auth, Theme, Denomination, FrontendUpdate, Naming, Premium, Sns), `pages/Me.jsx`, `pages/Principal.jsx`, `pages/Users.jsx`, `pages/Names.jsx`, `pages/Premium.jsx`, `pages/Hub.jsx`, `pages/PairApp.jsx`, `pages/ConnectAgent.jsx`, `Login.jsx`, the naming/avatar/premium utils and hooks, `public/manifest.webmanifest`, `public/sw.js`, `main.jsx`.

---

## Current navigation model

- **No shared layout header.** `Layout.jsx` only wraps the routed page, adds the PWA bar and the global `LoginModal`. *Every page renders its own `<Header>`* (≈130 call sites), passing optional props:
  - `showSnsDropdown` (true on SNS-scoped pages only: `/me`, `/names`, `/principal` + `/user`, `/neurons`, `/neuron`, `/proposals`, `/proposal`, `/transactions`, `/transaction`, `/forum`, `/feed`, `/topic/:id`, `/thread`, `/post`, `/sns`, `/admin/names`). The SNS dropdown and the VP bar only exist on those pages.
  - `onSnsChange` callback, `customLogo` (Sneed Lock pages use `/sneedlock-logo4.png`), `showTotalValue` (accepted but **never passed by any page**: dead prop).
  - Consequence: header state (menu open, collapse) resets on every navigation; header data (nervous-system params) is cached at module level to survive this.
- **Sections** (hamburger "section menu", object order): `Sneed Hub`, `Sneed Me`, `Sneed DAO`, `Sneedapp`, `Sneedex`, `Sneed Lock`, `Liquid Staking`, `Help`, `Tools` (shown as "Sneed Tools" in the title, "Tools" in the menu), `Admin` (only when `useAdminCheck` → backend `caller_is_admin` is true).
- **Submenus** (rendered as a row of links under the top bar for the *active* section only):
  - Sneed Hub (default `/hub`): Hub `/hub`, SNS `/sns`, Feed `/feed`, Forum `/forum`, SNS Proposals `/proposals`, ICP Proposals `/icp_proposals`, Neurons `/neurons`, Transactions `/transactions`, Positions `/positions`, NFTs `/nfts`, Users `/users`, Canisters `/canisters`.
  - Sneed Me (default `/me`): Me `/me`, Wallet `/wallet`, NFTs `/nft`, Apps `/apps`, Voting `/active_proposals`, Messages `/sms`, Notifications `/notifications`, Posts `/posts`, Tips `/tips`, Rewards `/rewards`, Address Book `/names`, Premium `/premium`.
  - Sneed DAO (default `/dao`): DAO, Dashboard `/dao_info`, Health, Custody `/guardian`, Tokenomics `/rll_info`, Loop ledger `/rll_ledger`, Revenue, Rewards `/rll`, Products, Partners, Thank-you wall `/donate`, Feedback, Mr Sneed, Diamond Club, Lore, Discord ↗ (external), X ↗ (external), Projects, Disclaimer.
  - Sneedapp (default `/sneedapp`): Browse Apps `/sneedapp`, My Apps `/apps`.
  - Sneedex (default `/sneedex_offers`): Swap `/swap`, Marketplace `/sneedex_offers`, Create Offer `/sneedex_create`, My Sneedex `/sneedex_my`, About `/sneedex`.
  - Sneed Lock (default `/lock_wizard`): Create Lock `/lock_wizard`, All Locks `/sneedlock_info`, Overview `/sneedlock`.
  - Liquid Staking (default `/liquid_staking`): Overview, ICP Liquid Staking `/create_icp_neuron`, SNS Liquid Staking `/sns_neuron_wizard`.
  - Help (default `/help`): Help, API `/api`, Architecture `/architecture`.
  - Tools (default `/tools/main`): Tools, Token Studio `/studio`, Custody `/custody`, Follow the Money `/tools/money`, ICRC-3 Browser `/tools/icrc3`, Account Tool `/tools/account`, Recover Top-up `/tools/recover-topup`; **logged-in only**: SNS Jailbreak `/tools/sns_jailbreak`, My Scripts `/tools/sns_jailbreak_list`.
  - Admin (default `/admin`): Dashboard, User Bans, Word Blacklist, Partners, Names, Donations, Feedback, Projects, SneedLock, Sneedapp, Sneed Pad, Sneedex, RLL Rewards, Sneed Monitor, SNS Jailbreak, + Sneed Drive `/drive` only when the browser-local beta flag `localStorage.sneedDriveBeta === '1'`.
- **Active section is inferred from the URL** (`Header.jsx`): hard-coded path lists, evaluated twice (a `useState` initializer and a `useEffect` on pathname) and the two lists **diverge** (initializer maps `/swap`→Sneedex, `/tools/*`,`/studio*`,`/custody`→Tools, `/drive`→Admin; the effect omits `/swap` and `/drive`, and Tools is re-asserted by a third effect only when authenticated). Mapping:
  - `/msg/*` → Sneed Me (checked first).
  - Sneed Hub: `/`, `/hub`, `/sns`, `/proposals`, `/icp_proposals`, `/icp_proposal`, `/neurons`, `/transactions`, `/positions`, `/position`, `/managelp`, `/nfts`, `/nft_item`, `/neuron`, `/proposal`, `/transaction`, `/principal`, `/user`, `/users`, `/canisters`, `/forum`, `/feed`, `/thread`, `/post`, `/topic/*`.
  - Liquid Staking: `/liquid_staking`, `/sns_neuron_wizard`, `/create_icp_neuron`, `/icp_neuron_manager/*`.
  - Sneed Me: `/me`, `/names`, `/premium`, `/rewards`, `/tips`, `/posts`, `/sms`, `/notifications(/*)`, `/wallet`, `/nft`, `/canister`, `/apps`, `/active_proposals`.
  - Sneed DAO: `/dao`, `/dao_info`, `/health`, `/rll_info`, `/rll_ledger`, `/revenue`, `/guardian`, `/rll`, `/products`, `/partners`, `/donate`, `/feedback`, `/mr-sneed`, `/diamond-club`, `/lore`, `/projects`, `/disclaimer`.
  - Sneed Lock: `/sneedlock`, `/sneedlock_info`, `/tokenlock`, `/positionlock`, `/lock_wizard`, `/lock/*`.
  - Sneedex: `/sneedex*`, `/sneedex_offer/*` (+ `/swap` on first mount only).
  - Sneedapp: `/sneedapp(/*)`. Admin: `/admin(/*)`, `/drive`. Help: `/help(/*)`, `/api(/*)`, `/architecture`.
  - **Fallback = Sneed Hub** for anything unlisted, e.g. `/wallet/pockets`, `/scan_wallet`, `/convert`, `/pair`, `/connect-agent`, `/login`, `/dashboard`, `/doc`, `/tokenomics*`, `/me_info`, `/outrun`. So `/wallet/pockets` shows the *Sneed Hub* submenu although it is a wallet page.
  - Several pages map to a section whose submenu doesn't list them (e.g. `/canister` → Sneed Me; `/principal`/`/user` → Sneed Hub but submenu has only "Users"; `/icp_neuron_manager/*` → Liquid Staking).
- **Submenu highlight** (`isSubmenuItemActive`): exact path, `/`→Hub, plus "legacy mappings" (`/proposal`→SNS Proposals, `/neuron`→Neurons, `/transaction`→Transactions, `/position`,`/managelp`→Positions, `/nft_item`→NFTs, `/thread`,`/post`,`/topic/*`→Forum, `/msg/*`→Messages; `/tokenlock`,`/positionlock`,`/lock/*` → an item `/sneedlock` (labelled "Overview")).
- **Section click** navigates to the section's `defaultPath` **carrying over the current query string** (e.g. `?sns=…`, but also unrelated `?id=…`).
- **Desktop vs mobile:** no separate mobile nav. Same header; `isCompact` (width ≤ 768 px or standalone PWA) makes the collapse pill bigger and makes the submenu row and VP bar slide away on scroll-down (they share the `isVpBarVisible` scroll state). CSS `hide-on-narrow`/`show-on-narrow` at 768 px. The installed app adds a bottom Back/Forward/Refresh bar (`PwaNavBar`), not a tab bar.
- **Secondary navigation surfaces** that duplicate the menu: header Quick-links (icon row or hamburger dropdown), `/me` "Quick Access" grid, `/hub` feature cards, quick-wallet popup "View Full Wallet" links, notification chips.

---

## Global page frame and hosts (App.jsx / Layout)

- **Routes & auth:** wraps all routes. Provider order: Theme → Auth → Sns → Neurons → Forum → Router → Premium → WhitelistTokens → Wallet → Denomination → WalletLayout → Naming → Pockets → Sms → FrontendUpdate.
- **Purpose:** global, page-independent UI and services.
- **Mounted once, outside pages:**
  - `GlobalNamingSetup`: exposes `window.getNeuronDisplayName`.
  - `CacheRefreshDialog`: full-screen overlay "Updating to latest version — Clearing cache and refreshing…" while `FrontendUpdateContext.isRefreshing`.
  - `PushOpenHandler`: listens to service-worker `sneed-open` messages (tapped push notification) → navigates, or fires app refresh if already on that URL.
  - `AppDialogHost` + `utils/appDialog`: promise-based in-app `appAlert` / `appConfirm` / `appPrompt` (queue, one at a time, rendered with `ConfirmDialog`; defaults "Something went wrong"/"Done"/"Notice"/"Please confirm"/"Enter a value", buttons "OK"/"Confirm"/"Cancel"). If no host, confirms resolve to "no".
  - `StagingBanner` (only when `DFX_NETWORK === 'staging'`): "STAGING: a preview of Sneed Hub… names, posts and settings are separate from the real site" + link "Sneed Hub is at app.sneeddao.com" + "Run by the team, not the DAO. It uses real tokens at real prices…".
  - `SessionExpiredBanner`: after delegation expiry, "Your session expired. Log in again to see your data and to send, vote or post." + "Log in" (opens login modal). Stays until next login.
  - `PrivacyHost`: global Shift+P shortcut (ignored while typing) and floating pill "Amounts hidden" (click to turn off).
  - `Layout`: page remount key (for PWA Refresh), PWA spacer + `PwaNavBar` (standalone only), global `LoginModal`.
- **Gating/state:** staging build flag; standalone detection.
- **Redesign notes:** the app already has a single place to host global chrome (`Layout`) but the header lives inside pages; moving `Header` into `Layout` would remove remount churn and the per-page prop plumbing.

---

## Header — top bar (logo, section title, collapse, update badge, SNS picker, quick links, donate, account)

- **Routes & auth:** every page; public; parts need login.
- **Information shown:**
  - Logo (or `customLogo`) with 👑 overlay when the logged-in user is Premium (`usePremiumStatus`).
  - Active section name (clickable; shows "Admin" while admin check loads on admin pages).
  - Collapse pill (`−` / `...`, title "Collapse header"/"Expand header"): hides submenu row, VP bar and notifications row (not persisted).
  - Update badge (when a new frontend build is detected): "Update" or a countdown "Ns" if auto-update is on; click = refresh now.
  - SNS dropdown (SNS pages only), quick links, donate heart, account button or login icon.
- **Actions:**
  - Click logo or title → open/close the section menu (left drawer listing sections; click → navigate to section default path with current query string).
  - **SNS dropdown** (`SnsDropdown`): shows selected SNS logo + ▼; list of all SNSes (Sneed first, then A–Z, logos lazy-loaded, "Loading more SNSes…" while the SNS cache fills). Selecting sets global `SnsContext` (persisted `localStorage.selectedSnsRoot`) and the `?sns=` URL param (removed for Sneed). `?sns=` in URL overrides stored selection; when absent and selection ≠ Sneed, the dropdown writes it into the URL.
  - **Quick links**, two modes controlled by setting `expandQuickLinksOnDesktop` (default off):
    - Expanded (wide screens only): Theme toggle + icon buttons Feed `/feed`, Forum `/forum`, Sneedex Marketplace `/sneedex_offers`, Sneedapp `/sneedapp`, Trading Bot `/sneedapp/app/sneed-trading-bot`, SNS Jailbreak `/tools/sns_jailbreak`, Wallet `/wallet`, Rewards `/rewards`, Swap `/swap`, Messages `/sms`, Premium `/premium`.
    - Hamburger "Quick Links" dropdown (always on narrow; everywhere when not expanded): Feed, Forum, Sneedex, Sneedapp, Swap, Trading Bot, SNS Jailbreak, Wallet, Rewards, Messages, Premium, then settings: **Currency** (TokenSelector "Select currency...", any token incl. custom ledger; current sign shown; "Reset to USD"; saved to localStorage + backend `set_my_settings.denomination_token` if logged in), **Theme** toggle, **Privacy mode** toggle ("Hide amounts"/"Amounts hidden").
  - **Donate heart** ("Donate"/"Support Sneed Hub") → Donate dialog (see below).
  - Logged in: wallet icon → **quick-wallet popup** (`PrincipalBox`, see below). Logged out: sign-in icon (title "Login") → login modal.
- **Entry points & cross-links:** everywhere.
- **Shared building blocks:** `SnsDropdown`, `ThemeToggle`, `PrivacyToggle`, `TokenSelector`, `DonateButton`, `PrincipalBox`, `useAdminCheck`, `usePremiumStatus`.
- **Gating/state:** `expandQuickLinksOnDesktop` (localStorage + backend settings); SNS selection (localStorage + URL).
- **Redesign notes:**
  - Quick links are a hand-picked, hard-coded "favourites" list that partially duplicates sections (Swap, Trading Bot, SNS Jailbreak, Rewards, Premium) — two icons both use the exchange arrow (Sneedex and Swap).
  - With quick links expanded on desktop, the **Currency selector and Privacy toggle disappear from the header** (they only live in the hamburger dropdown); currency then only on `/me` → Settings, privacy only in the quick-wallet popup / Shift+P.
  - Global settings (currency, theme, privacy) are hidden inside a "Quick Links" menu.
  - SNS context is global but the picker is only visible on ~16 pages; other pages silently use the stored SNS.

---

## Header — "Your neurons" voting-power bar

- **Routes & auth:** only on pages with `showSnsDropdown`, logged in, setting `showVpBar` on (default), user has neurons with VP > 0 in the selected SNS.
- **Purpose:** show the user's governance weight in the selected SNS.
- **Information shown:** robot+brain icon with SNS logo (desktop); `{VP} VP hotkeys ({n})` = neurons where the user has hotkey/vote permission (tooltip "Hotkeyed and owned neurons - can vote on SNS proposals"); `{VP} VP reachable (+{n})` = additional neurons of the same owners, not hotkeyed (tooltip "…for forum voting"). VP computed client-side from `get_nervous_system_parameters` + neuron data from WalletContext's global neuron cache.
- **Actions:** Refresh neurons (⟳, "Refresh neurons (e.g. after transfer, Sneedex claim, or permission change)" → `refreshAllNeurons`); ✕ "Hide VP bar (re-enable in Settings)" → sets `showVpBar=false`.
- **Gating/state:** `localStorage.showVpBar` (+ backend), scroll-hide on all sizes.
- **Redesign notes:** a key personal stat (your voting power) that disappears on non-SNS pages; "hotkeys" vs "reachable" concept reappears on `/me` and `/principal` ("Include reachable") with different wording.

---

## Header — notifications row (badges) and the dialogs it opens

- **Routes & auth:** every page; most chips need login; master toggle `showHeaderNotifications`; each chip has its own toggle (all default on; set on `/me` → Settings → Notification Bar Settings).
- **Purpose:** in-app "needs attention" counters (not the Sneed Notifications service).
- **Chips (icon + count) → target:**
  - Update available (independent of the master toggle; `notifyUpdates`) → refresh now (shows countdown when auto-update on).
  - Replies (`notifyReplies`; new replies to your posts, forum canister count since last seen) → `/posts`.
  - Mentions (same setting as replies; forum @-mentions, polled every 2 min) → `/posts?tab=mentions`.
  - Messages (`notifyMessages`; unread Sneed SMS from shared SmsContext inbox) → `/sms`.
  - Votable proposals (`notifyVotableProposals`; active proposals you can vote on and haven't; every 5 min) → `/active_proposals`.
  - Tips (`notifyTips`; new tips received) → `/tips`.
  - Collectibles (`notifyCollectibles`; LP fees, rewards, maturity worth ≥ `collectiblesThreshold` USD; tooltip shows total USD, masked in privacy mode) → **ConsolidateModal "all"** ("Collect All", per-item Collect Fees / Claim Rewards / Collect Maturity / Disburse Maturity).
  - Low cycles (`notifyLowCycles`; red if critical) → **TopUpCyclesDialog** (select canisters, "Top up each to a target cycle level" / "Same amount for each", "Pay with ICP" or CyclePay).
  - Outdated bots/apps (`notifyOutdatedBots`) → **UpgradeBotsDialog** (select all/deselect, upgrade wasm).
  - Bot health (`notifyBotChores` or `notifyBotLogErrors`; chore lamp warn/error + unseen log errors/warnings, `notifyBotLogWarnings`) → **BotHealthDialog** ("View chores ›" per bot).
- **Gating/state:** counts use `useStickyCount` per principal; all toggles localStorage (+ backend).
- **Redesign notes:** this is a second, unrelated notification system next to `/notifications` (Sneed Notifications canister, push, inbox); the header row also embeds real actions (collect, top up, upgrade) that otherwise live in `/wallet` and `/apps`.

---

## Quick-wallet popup (PrincipalBox, header account button)

- **Routes & auth:** header, logged in only (rendered `compact`; Wallet.jsx and RLLInfo.jsx import `PrincipalBox` but never render it).
- **Purpose:** a mini wallet + account menu.
- **Information shown:**
  - Banner: your public name (click → `/me`) or "Set up profile" (→ `/me`).
  - Identity card: Principal ID or ICP Account ID (toggle "Show account ID"/"Show principal"); click copies ("Principal copied!"/"Account ID copied!").
  - "Total Balance" = tokens + LP positions + ICP-staking-bot neurons, in the chosen denomination.
  - Tabs **Tokens** ("Wallet", "Hide dust" < $0.01, total, refresh), **Liquidity** (pairs, value + fees, locked marker, total, refresh), **Apps** (staking bots, trading bots, tracked canisters: name, type, neuron count, cycles, memory, controller crown, chore status lamp, USD value; banners "N bots outdated → Upgrade", "N canisters low on cycles → Top Up").
  - Admin only: "Quick Wallet Debug Report" (copyable totals breakdown).
- **Actions:**
  - Token row → TokenCardModal (balance details, locks list, "Send", "Lock" → LockModal → deposit + `sneed_lock.create_lock`, refresh). Row "Send" → SendTokenModal.
  - Position row → PositionCardModal (+ send position → SendLiquidityPositionModal → `sendLiquidityPosition`). Row "Send".
  - App row → DappCardModal (refresh `canister_status`, chores start/stop/pause/resume/trigger on the bot canister, send). Row "Send" (controller only, "Transfer control") → TransferCanisterModal.
  - "View Full Wallet" / "View All Liquidity" / "View All Apps" → `/wallet`.
  - Privacy mode toggle (Shift+P hint). "Log Out".
- **Shared building blocks:** WalletContext data (same as `/wallet`), TokenCardModal, PositionCardModal, DappCardModal, SendTokenModal, LockModal, TransferCanisterModal, UpgradeBotsDialog, TopUpCyclesDialog, ChoreStatusLamp, PrivacyToggle.
- **Gating/state:** `hideDust_Wallet` (shared with /wallet), canister status fetched lazily only while open.
- **Redesign notes:** a full second wallet UI (send, lock, transfer canister, chores) inside the account menu; logout is buried at the bottom of a wallet popup; the account menu has no links to profile settings, address book, premium or messages beyond the name link.

---

## Donate dialog (header heart; also used by /donate)

- **Routes & auth:** header everywhere; public view logged out, donation logged in.
- **Information shown:** "Support Sneed Hub", recipient (name/avatar), "Built and run for free…", public-ness notice ("Donations are public… Gifts can't be refunded"). Logged out: donation account to copy — "Principal (any ICRC-1 token)", "Subaccount", "ICP account ID (exchanges)" + "Log in to donate" ("Only donations made here while logged in go on the wall.").
- **Actions (logged in):** choose "Token" (TokenSelector, default ICP), "Amount" (USD presets 5/20/50/100 converted to a round token amount, "Other amount", "Max", balance and "Network fee" shown), "Message (optional, shown on the wall)" → "Continue" → confirm → "Send donation" (ledger transfer to donation account, then `registerDonation` on the wall + notify recipient). Success "Thank you! … you are on the thank-you wall" + "See the thank-you wall". Pending/unregistered donations: "A donation you sent is not on the wall yet." → "Add it" (retry).
- **Cross-links:** `/donate` thank-you wall.
- **Gating/state:** "Donations are not open yet" when no account configured.
- **Redesign notes:** global CTA in the header next to the account button; overlaps DAO section's "Thank-you wall".

---

## Global settings: theme, privacy mode, denomination, and settings persistence

- **Theme** (`ThemeContext`, `ThemeToggle`): dark / bright; `localStorage.sneedDao_theme`; **not** synced to the backend. Toggles in: header quick links (expanded row), Quick-Links dropdown, `/me` Settings → General.
- **Privacy mode** (`utils/privacyMode`, `usePrivacyMode`, `PrivacyToggle`, `PrivacyHost`): masks all balances/amounts as `••••` in shared formatters (inputs and the amount on confirm steps stay visible); `localStorage.sneed_privacy_mode_v1`, cross-tab; Shift+P; floating "Amounts hidden" pill; toggles in Quick-Links dropdown and quick-wallet popup (and the wallet header per comment). Not synced to backend.
- **Denomination** (`DenominationContext`): display currency for all USD values = any token (default ckUSDC = "USD"); price refreshed every 60 s; `localStorage.denominationToken`; set from header Quick-Links or `/me` Settings; written to backend settings but **read back from the backend only when `/me` is opened**.
- **Backend-synced settings** (`app_sneeddao_backend.get_my_settings/set_my_settings`): principal/neuron colour coding, VP bar, header notifications master + 11 per-chip toggles, collectibles threshold, expand quick links, particle effects, ICP-staking-bot and app cycle thresholds, auto-update/clear-cache/check-interval/countdown, swap slippage, always-show-remove-token, denomination. Sync logic lives **inside `/me`**: on load, if backend looks default and local has custom values, local is pushed up; otherwise backend values overwrite localStorage. `FrontendUpdateContext` separately reads its four values from the backend on login.
- **Browser-only state** (localStorage): `selectedSnsRoot`, `quickAccessExpanded`, `hideEmptyNeurons_Me`, `includeReachable_Me`, `hideDust_Wallet`, `install_app_card_dismissed`, `sneed_pwa_nav` (force PWA bar), `sneedDriveBeta`, caches (`sneed_*_names_cache`, `avatars_cache_v1`, `nft_avatars_v2`, `sneed_premium_status_<principal>`).
- **Redesign notes:** settings are split three ways (header dropdown, `/me` Settings tab, per-page toggles), and cross-device sync silently depends on visiting `/me`. Theme/privacy are device-local while near-identical display prefs are synced.

---

## Frontend update detection (FrontendUpdateContext)

- **Purpose:** notice a new deploy of the asset canister and refresh.
- **Behaviour:** when running on a canister, polls `version.json` build id every `checkIntervalSec` (30–3600, default 600); on change sets `hasUpdateAvailable` → header badge + notifications chip. If auto-update is on, counts down `countdownSec` (10–300, default 300) then reloads; optionally clears all caches first (`clearAllCaches`) with the CacheRefreshDialog overlay.
- **Settings:** `/me` → Settings → General ("Auto-update on new version", "Clear cache on update", "Update check interval", "Update countdown timer"); localStorage + backend.
- **Redesign notes:** defaults: auto-update **off** (code default false), so users get a manual "Update" chip.

---

## PWA / installed app

- **Manifest:** name "Sneed Hub", standalone, start `/`, black theme, icons 192/512/maskable. `index.html` links it. `public/sw.js` (registered on https only) caches nothing; handles push (shows notification; stores 6-digit verification codes for the Notifications page) and notification clicks (focus/navigate/open window, else `sneed-open` message).
- **Install prompts:** no `beforeinstallprompt` handling; install is manual (browser menu). `InstallAppCard` on `/hub` only, phones only (≤768 px), not inside the app: "Sneed Hub as an app. Add it to your home screen for a full-screen app. How it works →" `/help/app`; dismiss persisted (`install_app_card_dismissed`).
- **Standalone behaviour:**
  - `PwaNavBar` (bottom): Back / Forward (in-app history via `history.state.idx`, max kept in sessionStorage), Refresh (tap = `requestAppRefresh` or remount page keeping login/caches; long-press 550 ms = full reload), then "Still stuck? Reload app" for 5 s. Hides while scrolling down or typing. Forceable in a normal tab with `localStorage.sneed_pwa_nav='1'`.
  - Header compact mode (bigger collapse pill, scroll-hide).
  - Login modal adds the pairing option (see Pairing).
- **Redesign notes:** mobile has no bottom tab navigation at all; the PWA bar replaces browser chrome only.

---

## QR scan button (shared input helper)

- **Where:** next to recipient fields in `SendTokenModal`, BTC send, BTC/ETH convert.
- **Actions:** camera button → dialog "Scan a QR code" (BarcodeDetector or jsQR fallback), "Read a picture" (from file), "Cancel"; parses by target (`icrc`/`icp`/`btc`/`eth`) and **only fills the field** ("The code only fills in the address. Check it before you send."). Clear error texts for denied/no/busy camera.

---

## Login (LoginModal, /login, session lifecycle)

- **Routes & auth:** global modal (opened by any `login()` call: header icon, banner, page prompts); page `/login` (public).
- **Purpose:** sign in with Internet Identity.
- **Information shown:** "Sign In to Sneed DAO — Choose your preferred sign-in method"; two options **"Internet Identity 1.0"** (identity.ic0.app • Classic passkeys) and **"Internet Identity 2.0"** (id.ai • New interface) with explanations; "Don't have an Internet Identity?" → "Create on II 1.0 →" / "Create on II 2.0 →" (external); auth errors with dismiss ✕.
- **Actions:** pick provider → II popup (`AuthClient.login`, Ed25519 session key, max TTL **7 days**, idle logout disabled) → identity set, modal closes. In the installed app: extra button "On iPhone, login doesn't come back to the app? Pair it with Safari" (other OS: "…Pair it with a browser") → PairAppPanel; "Back to Internet Identity".
- **`/login` page:** auto-opens modal; "Please sign in to continue" + "Open Sign In"; after login redirects to `?from=<path>` (keeps other params) or `/hub`. Nothing in the app links to `/login?from=`.
- **Session lifecycle (`AuthContext`, `utils/authExpiry`):** watches delegation expiry (timer + focus/visibility re-checks); on expiry logs out and shows SessionExpiredBanner; also detects replica "delegation expired" errors (pages call `reportIfAuthExpired`, plus unhandled rejections). `ensureSession()` for write actions. Logout only from the quick-wallet popup.
- **Redesign notes:** no account/session screen (no "logged in as / session ends at"); the logout control is inside the wallet popup; several pages' logged-out states say "Connect Your Wallet" though there is no wallet connect, and often offer no login button (`/me`, `/names`, `/premium`).

---

## App ↔ browser pairing (/pair, PairAppPanel)

- **Routes & auth:** in-app panel (standalone login modal); `/pair?key=<app session public key hex>` (needs login in the browser).
- **Purpose:** log the installed PWA in through a browser where the user is logged in (iPhone II popup can't return to the app).
- **App side (PairAppPanel):** shows link `…/pair?key=…` + "Copy link"; a check code (e.g. `K7Q-2MX`, SHA-256 of the key); "Paste approval" (reads clipboard, else textarea + "Log in"); warning "Only approve a code you see here, in your own app…". Completion stores the delegation chain in IndexedDB and reloads auth.
- **Browser side (/pair):** "Approve the Sneed Hub app" — shows the code; "Allowing gives that app your Sneed Hub login until this browser's login expires. If someone sent you this link, don't allow it." → "Log in to continue" or "Allow" → "Copy approval" + textarea; error "This pairing link is incomplete…".
- **Under the hood:** signs one more delegation (browser session → app key) expiring with the browser session; no canister call.
- **Redesign notes:** purely an auth utility; reachable only from the standalone login modal (no menu entry), which is fine; route falls into the Sneed Hub section by fallback.

---

## AI-agent delegation (/connect-agent)

- **Routes & auth:** `/connect-agent?key=…&name=…&callback=http://127.0.0.1:PORT/…`; login required to approve. Linked only from `/api` text and the MCP server's `connect_as_me`.
- **Purpose:** let a local agent (Sneed Hub MCP server) act as the user with a scoped, time-limited delegation.
- **Information shown:** "Let an AI agent act as you"; agent name; check code; "It will act as <principal>. Everything it does on-chain is public…"; always-included Sneed canisters (Sneedex, Sneed Lock, Forum, Sneed SMS, Notifications, Names, Nicknames, Rewards (RLL), Sneedapp, Premium, Sneed Monitor).
- **Actions:** scope checkboxes "Your ICP", "Your SNEED", "Sneed DAO governance", "NFT collections and other token ledgers I choose" (+ canister id list, validated), "Everything" (no target restriction); duration "1 hour" / "24 hours" / "7 days" (capped at own session end); "Allow" → signs delegation with `targets` → POST to loopback callback; on failure "Send it by opening the agent's page" (navigates with `?d=`) or "Copy approval". Done: "…can now act as you until <time>… This cannot be revoked on-chain".
- **Gating:** callback must be http on 127.0.0.1/localhost (else refused); key hex validated.
- **Redesign notes:** no list of active agent delegations (can't be revoked anyway); belongs with security/session settings rather than nowhere.

---

## My profile — /me (hero, identity editing, account panel, stats, quick access)

- **Routes & auth:** `/me` (self only; logged out shows "Connect Your Wallet — Connect your wallet to view your profile, manage neurons, and access settings." with no button). Params: `?sns=` (syncs SNS), `?tab=settings` (opens Settings and expands App Manager settings). Header with SNS dropdown.
- **Purpose:** the user's own profile + personal hub + settings.
- **Information shown:** public name (or "My Profile") with verified ✓; Premium badge; avatar (or initial); activity stats from the forum canister: Posts (→ `/posts`), Threads (→ `/posts?tab=my-threads`), Tips ↓ received (→ `/tips?tab=received`), Tips ↑ given (→ `/tips?tab=given`). **Does not show the principal ID** (that's only in the quick-wallet popup and `/wallet`).
- **Actions:**
  - **Display name:** "Set Display Name"/"Change Display Name" → input "Display Name" ("This is shown publicly…", max 32, letters/digits/space/`-_.'`) → "Save" → confirmation ("…visible to everyone… Inappropriate names can result in a user ban…") → `setPrincipalName` (naming service `sneed_pid_names.set_my_name`, or legacy backend). "Cancel".
  - **Avatar** (pencil/camera on avatar: "Set an avatar"/"Change avatar"): "Upload picture…" (client-side shrink to ≤128 px data URL, stored on-chain in `sneed_avatars`), or "https://… link to an image" (≤512 chars), preview with size in kB, "Save avatar", "Remove avatar", "Cancel". When the naming service is live: **NFT avatar picker** "Use an NFT I own…" (lists ICRC-7 collections saved in the NFT wallet + EXT collections such as Diamond Club; click an NFT → `set_my_nft_avatar` / `set_my_ext_nft_avatar`; ownership re-checked whenever shown), "Stop showing the NFT", hint to add collections in `/nft` or use "Use as my avatar" on an NFT page.
  - **Your account panel** (AccountPanel, naming service only) — see next section.
  - "View My Public Page" → `/user?id=<me>` ("Your shareable profile page…").
  - **Quick Access** grid (collapsible, persisted): My Public Page, My Wallet `/wallet`, My Apps `/apps`, Voting `/active_proposals`, My Trades `/sneedex_my`, My Posts `/posts`, My Tips `/tips`, My Messages `/sms`, My Rewards `/rewards`, My Locks `/sneedlock_info?owner=<me>`, My Contacts `/names`.
  - Tabs: **Neurons** (default), **Transactions**, **Settings** (below).
- **Entry points:** Sneed Me section default; quick-wallet popup name/"Set up profile"; many Help pages link `/me`; `/apps` and HelpCanisterManager link `/me?tab=settings`.
- **Shared building blocks:** ConfirmationModal, NftAvatarPicker, AccountPanel, NeuronDisplay, TransactionList, TokenSelector, ThemeToggle, InfoTooltip, PremiumBadge.
- **Gating/state:** `namingServiceEnabled` (real canister ids for all naming canisters) gates NFT avatar and account panel; `notificationsEnabled` gates links to `/notifications`.
- **Redesign notes:** `/me` mixes (1) identity/profile editing, (2) a launcher grid duplicating the Sneed Me submenu, (3) a neuron browser, (4) one SNS's transactions, (5) the entire app settings panel. Labels drift ("Address Book" vs "My Contacts", "Display Name" vs "Public Name").

---

## Multi-principal account (AccountPanel on /me)

- **Routes & auth:** `/me`, self only, naming service only.
- **Purpose:** one name/avatar for several principals (II gives a different principal per dapp) — `sneed_pid_names` accounts.
- **Information shown:** "Your account": member principals (owner/member, "· you", optional "on which dapp?" label from the `sneed_profiles` dapp registry), verified badge; "Hotkeys" list; "Accounts you hold a hotkey on" (paged, name, member count, members). Explanations: principals in an account are public; a hotkey can rename, change avatar and merge.
- **Actions:** set member dapp label (`setMember`); "Make owner"/"Make member"; remove member ✕ / "Leave" (confirm); "Add hotkey" (PrincipalInput "Principal to give a hotkey"); remove hotkey ✕; for accounts you hold a hotkey on: "Rename" (new name ≤32), "Merge into my account" (confirm explains name effects), give up hotkey ✕, "Remove me from all" (confirm), "Show more".
- **Redesign notes:** a powerful, privacy-relevant identity feature buried mid-profile; no view of it on the public page; "no account yet: set a name above, or add a hotkey below".

---

## /me — Neurons tab

- **Information shown:** "My Neurons" + total count; SNS sub-tabs for every SNS where the user has neurons (from WalletContext global neuron cache, sorted by stake, with logo and count); filters "Hide empty neurons" and "Include reachable" (tooltip: neurons reachable through ownership chains, useful for forum voting) — both persisted; neuron group tabs (when reachable: "My Neurons" vs groups "Hotkey access to <owner>"); group header with total stake; neuron cards: stake + USD value, public name ✓ / private nickname, access-level icon (owner/hotkey/manager…), NeuronDisplay (copy, context menu: View Neuron, Copy Neuron ID, Set/Edit Nickname); expanded: Created, Dissolve State, Maturity, Voting Power, Permissions (owners + other principals with permission icons).
- **Actions:** edit pencil → "Set Nickname" (private, `setNeuronNickname`) or "Set Name" (public; only with hotkey access; confirmation about visibility/ban) — validated ≤32; "Manage" → `/neuron?neuronid=…&sns=…`; link to `/help/neurons`.
- **Redesign notes:** read-only neuron list; management happens on `/neuron` or `/wallet` token card Neurons tab — three places list "my neurons". **Possible bug/confusion:** names are read/written against the header SNS (`selectedSnsRoot`) while the list shows the sub-tab SNS (`activeNeuronSns`), so naming a neuron of a non-selected SNS targets the wrong SNS key.

---

## /me — Transactions tab

- **Information shown:** "My {SNS name} Transactions" — `TransactionList` for the header-selected SNS ledger, filtered to the user's principal; "Select a DAO to view transactions" when none.
- **Redesign notes:** only SNS tokens of the selected SNS (no ICP or other tokens); `/transactions` and the wallet token history cover the same ground.

---

## /me — Settings tab

Collapsible sections (each control writes localStorage, fires a window event so open headers update, and saves to backend settings):

- **General Settings:** "Theme"; "Denomination Currency" (TokenSelector, "Reset to USD"); "Principal Color Coding"; "Neuron Color Coding"; "Show Voting Power Bar"; "Expand Quick Links on Desktop"; "Particle Effects" (tip sparkles / fireworks on Tips page); "Always show remove button on tokens" (wallet); "Auto-update on new version"; "Clear cache on update"; "Update check interval" (30–3600 s); "Update countdown timer" (10–300 s).
- Links (when notifications service enabled): "Notifications from apps" → `/notifications` ("Which apps may notify you, where, quiet hours, channels"); "Notification inbox" → `/notifications?tab=inbox`.
- **Notification Bar Settings:** "Show Header Notifications" (master); "SMS When Mentioned" (forum canister `set_mention_sms`; a real delivery setting, not a header setting); then (if master on) Reply, Tip, Message, Collectibles (+ "Collectibles Threshold" USD), Votable Proposals, Outdated Bots, Low Cycles, Bot Chore Health, Bot Log Errors, Bot Log Warnings, App Updates.
- **ICP Staking Bot Settings:** cycle thresholds red/orange (e.g. "1T, 500B"), "Save Settings", "Reset to Defaults" (1T / 5T); validation via browser `alert()`.
- **App Manager Settings:** "App Cycle Thresholds" for apps in Wallet and App Manager (same controls).
- **Sneedex:** "Swap Slippage Tolerance" (0.5/1/2/5 % or custom); Sneedex event notifications sent by **direct message** (Sneedex canister `get/setMyNotificationSettings`): New Bids, Outbid, Offer Sold, Auction Won, Offer Expired, Offer Cancelled, Private Auction Invite → "Save Settings".
- **Cache Management:** "Clear All Caches" (IndexedDB, localStorage, sessionStorage, Cache API; keeps login); "Completely Clear All Site Data" (also service workers, cookies; logs out; reloads); manual DevTools instructions; "Clear Gray List (n)" (ledger skip list of failing ledgers).
- **Redesign notes:** app-wide preferences, product-specific settings (bots, App Manager, Sneedex, swap), a messaging delivery setting and troubleshooting tools all live in a tab of the profile page; settings for a product are far from the product (bot thresholds vs `/apps`, slippage vs `/swap`, Sneedex DM alerts vs `/notifications`).

---

## Public user page — /principal and /user (same component)

- **Routes & params:** `/principal?id=<principal>` and `/user?id=<principal>`; optional `&subaccount=<hex>` (opens Transactions, filtered), `&sns=` (SNS for transactions). Public; extra actions when logged in; self gets "Set Name". `getPrincipalProfileUrl` sends users to `/user`, canisters to `/canister`, unknown to `/principal`. Header with SNS dropdown; active section Sneed Hub.
- **Purpose:** what anyone can see about any principal ("User Explorer").
- **Information shown (what a public page reveals):**
  - Hero: avatar (NFT/uploaded) or icon (crown for premium, cube for canister), verified check, public name, **your private nickname** ("…" in banner), Premium badge (viewed user), "Canister" badge, principal (short, copy), subaccount chip (copy, clear filter), **grand total value** = scanned token balances + all neuron stake in USD/denomination.
  - Stats: Posts, Threads, Offers, Neurons counts (click → tab).
  - Tabs:
    - **Posts & Threads:** sub-tabs Posts (title, body with "Show More", net vote score, date, → `/post?postid=`) and Threads (Thread #id, title, created date, post count, → `/thread?threadid=`). Forum canister `getPostsByUser`/`getThreadsByUser`.
    - **Trades:** Offers (by creator) and Bids (by bidder) from Sneedex; "Active only" toggle; offer cards (Offer #id, asset icons, state, Min Bid, Buyout, time left; SOLD/EXPIRED/CANCELLED banners) and bid cards (amount, Won/Lost/Pending) → `/sneedex_offer/:id`.
    - **Neurons:** scans **every SNS** for neurons where the principal has direct permissions ("Scanning all DAOs for neurons..."); "Include reachable" (on demand, all neurons via owners); SNS sub-tabs with count + USD; active SNS summary (total stake, USD); group tabs "Direct Access" / "Reachable via <owner>"; cards with stake, USD, names, access level, expandable Created / Dissolve State / Maturity / Voting Power multiplier / Permissions.
    - **Transactions:** TransactionList for the selected SNS ledger, with subaccount filter (synced to URL).
    - **Wallet:** balances of **all whitelisted tokens** for the default account (auto-scan on open; "Scan Tokens" button with progress "Scanning x/y (n found)"; skip-list for broken ledgers), USD per token and total.
  - Not shown: locks, LP positions, NFTs, tips, apps/canisters owned, donations, linked account principals.
- **Actions:** "Search Users" (PrincipalInput by name/nickname/principal; "My Principal" shortcut; "Search Another User"); copy principal/subaccount; "Set Nickname"/"Edit Nickname" (private, ≤32, `setPrincipalNickname`); self: "Set Name"/"Change Name" (public, confirmation); others (logged in): "Send Message" → MessageDialog (Sneed SMS compose prefilled); canister: "View Canister Details" → `/canister?id=`.
- **Entry points:** every `PrincipalDisplay` link and context-menu "View Profile"; `/me` "View My Public Page"; `/users`; mentions in Markdown; feed cards (`/principal?id=`).
- **Shared building blocks:** PrincipalInput, PrincipalDisplay, NeuronDisplay, TransactionList, MessageDialog, ConfirmationModal, TokenIcon, usePremiumStatus, useAvatar, WhitelistTokensContext, LedgerSkipList.
- **Redesign notes:** heavy (scans all SNS governances and all whitelisted ledgers on open). Overlaps `/scan_wallet?principal=` (balances), `/neurons` (neuron search), `/sneedex_my` (own trades), `/posts` (own posts), `/me` (own neurons/transactions). Viewing your own page gives "Set Name" but no avatar editing and no link back to `/me`. Two route names for one page.

---

## User explorer — /users

- **Routes & auth:** `/users`, public (your nicknames included when logged in).
- **Information shown:** "User Explorer — Browse all principals with public names or nicknames"; count of named users / matching; list rows (avatar, name/nickname, "View Profile").
- **Actions:** PrincipalInput "Search or enter principal to view..." (select → profile); filter text "By name or principal ID..."; "Show:" Both / Public names only / Nicknames only; "Per page:"; Previous/Next pagination. Rows link to `getPrincipalProfileUrl`.
- **Data:** entirely from NamingContext's full download of public names + your nicknames (canisters excluded); avatars via batched `useAvatars`.
- **Redesign notes:** duplicates the search on `/principal`; filed under "Governance" on the Hub page.

---

## Address book — /names

- **Routes & auth:** `/names` (login; logged out "Connect to Access…" without button). Header with SNS dropdown (scopes neuron section). Menu label "Address Book"; `/me` calls it "My Contacts".
- **Information shown:** counts (principals, neurons); **Principal Nicknames** list (PrincipalDisplay, "Public: <name>", nickname); **Neuron Nicknames** list for the selected SNS ("Scoped to selected SNS"; short neuron id → `/neuron`, SNS id, public name, nickname).
- **Actions:** add principal nickname (PrincipalInput "Enter principal ID or search...", "e.g. Alice", "Add"); search "Search by principal, public name, or nickname…"; edit ✓/✕; remove (browser `window.confirm`, errors via `window.alert`). Same for neurons (NeuronInput "Enter neuron ID or search...", "e.g. Team treasury"; requires a selected SNS). Writes `setPrincipalNickname` / `setNeuronNickname` (empty = remove).
- **Limits:** nickname counts limited per tier (Premium higher, see Premium page).
- **Redesign notes:** doesn't cover **account (subaccount) nicknames / pockets** that the naming service also stores (`setAccountNickname`, `/wallet/pockets`); nicknames are also editable from every PrincipalDisplay context menu, `/principal`, `/neuron`, `/me` neuron cards. Uses native browser dialogs despite the app's appDialog policy.

---

## Naming system and principal display building blocks

- **NamingContext:** loads all public neuron names, all public principal names (+ verified flags), and (logged in) the caller's neuron and principal nicknames; caches in localStorage, background refresh; merges SNS canister names and whitelisted ledger names as verified; `principalCanisterTypes` (sns_root/governance/ledger/swap/index/dapp/archive/whitelist_ledger) for icons; batched avatar queue (`requestAvatar`, 50 per call, ~60 ms debounce, 30 min cache) incl. NFT-avatar ownership re-check; `useAvatar`, `getNeuronDisplayName`, `getPrincipalDisplayName`.
- **NamingService (`utils/NamingService`):** routes naming calls to the new canisters (`sneed_pid_names`, `sneed_neuron_names`, `sneed_pid_nicknames`, `sneed_neuron_nicknames`, `sneed_avatars`, `sneed_profiles`) when all have real ids (`namingServiceEnabled`), else legacy backend via BackendUtils. Covers: names, nicknames, canister names (`set_canister_name`), avatars (URL or uploaded blob), NFT avatars (ICRC-7/EXT), accounts/hotkeys/merge, name search, dapp registry, **account nicknames and pockets** (ICRC-1 accounts with subaccounts, mirrored locally), admin verify/unverify/clear avatar.
- **PrincipalDisplay (`utils/PrincipalUtils`):** everywhere a principal appears: avatar/premium crown/canister-type icon, name/nickname/verified, colour coding, copy button, link (plain `<a href>`) to profile; right-click/long-press **PrincipalContextMenu**: "Send Message" (MessageDialog), "Set Nickname"/"Edit Nickname" (NicknameDialog: "Private Nickname… Only visible to you. Leave empty to remove.", ≤32, no `<>'"&`, "Save Nickname"), "View Profile", "Copy Principal ID", optional "Copy ICRC-1 Account". Props allow hiding send/view.
- **NeuronDisplay / NeuronContextMenu:** "View Neuron", "Copy Neuron ID", "Set/Edit Nickname".
- **PrincipalInput** (used in ~55 files): text field with dropdown search over named principals (tabs Private / Public / All; users/canisters toggles), ICRC-1 encoded account parsing, "Add subaccount" dialog ("Account with Subaccount": principal, subaccount hex, encoded account, parsed values, copy buttons). Ranking shared with @-mention textarea.
- **Avatars:** `useAvatars`/`useAvatarOf` (batched), `BigAvatar` (any size; fallback initial/user/cube; verified check; premium gold ring), `LogoAvatar`, `AvatarPair` (logo + person, e.g. proposer). Used in forum, SMS, tips, Sneedex, proposals, donate, users, principal pages. `utils/nftAvatars`: resolves NFT avatar pictures and ownership (10 min cache).
- **Redesign notes:** identity is rendered consistently through these components — the redesign can rely on PrincipalDisplay's context menu as the universal "person actions" affordance (message, nickname, profile, copy), but it is discoverable only by right-click/long-press.

---

## NFT character & soulbound helpers (used by /nft_item, NFT cards)

- **CharacterPanel:** for character NFTs with `wu:stats` / `wu:items` metadata: per game "In {game}" (site link, root id), facts grouped into families (quests, reputation…), inventory document (bags/worn/keychain), a 3D "look" (owner can "Use this look" → collection `set_model`), and "Its things (NFTs)" (carried by the character / kept by the owner per game). `CarriedBy` ("Carried by the character X"), `SoulboundNote` ("Soulbound — Bound to X…"), `SoulboundBadge`.
- **Redesign notes:** identity-adjacent (characters as owners) but lives in the NFT area; only `NftItem` uses CharacterPanel.

---

## Premium — /premium

- **Routes & auth:** `/premium`, public tiers; actions need login. Menu: Sneed Me → Premium; header quick link 👑; Hub banner; many Help pages and fee screens link here.
- **Purpose:** buy/claim Sneed Premium membership and see what it unlocks.
- **Information shown:** status chip "Premium Active • time left" / "Membership Expired" / "Not a member yet"; detail card (expires, time remaining, or expired date, or "Become a Premium Member"); generic perks (Exclusive Access, Discounts, Premium Badge); **"Premium Benefits & Savings"** read live from other canisters, only where premium is better: Sneedex (offer creation fee, auction cut %), ICP Staking Bot (creation fee, from Sneedapp app `sneed-icp-staking-bot`), SNS Jailbreak (script price), SNS Forum (post/thread length), Direct Messages (length, rate limit, max recipients), Address Book (neuron/principal nickname limits), App Manager (folders, apps per folder, total apps); each with "Learn more" link (`/sneedex_offers`, `/create_icp_neuron`, `/tools/sns_jailbreak`, `/forum`, `/sms`, `/names`, `/apps`).
- **Actions:**
  - **Purchase with ICP:** tier cards (name, ICP price, regular price struck through and PriceTierBadge "Admin price"/"Partner price" for special prices from `get_icp_tier_prices_for`, denomination value, duration); "Pay Directly from Wallet" panel (wallet ICP balance, selected cost) → "Pay Now - X ICP" → confirm "Purchase Premium Membership" ("This will extend your existing membership by …" or "You'll become a Sneed Premium member!", "Pay X") → transfer only the missing amount to the user's deposit subaccount (`getDepositAccount`), then `purchaseWithIcp`. Preconditions: tier selected, balance ≥ price + fee. If the deposit already holds more than the price: "Withdraw your Premium deposit first" → "Withdraw deposit" (`withdrawDeposit`). Detailed error texts (InsufficientFunds, BadFee, InsufficientPayment…).
  - **Claim with Sneed Voting Power:** VP tiers (min VP → duration, "Grants membership"); shows "Your Voting Power" and "You qualify for <tier> ✓" (hotkeyed Sneed neurons via `useSneedMembership`); "Claim with Voting Power" → confirm "Claim Membership" → `claimWithVotingPower` (errors NoEligibleNeurons, InsufficientVotingPower, AlreadyClaimedRecently with interval/wait time).
  - **Redeem Promo Code:** input "Enter promo code" → "Redeem Code" → confirm → `claimPromoCode` (errors InvalidCode, CodeExpired, CodeFullyClaimed, CodeInactive, AlreadyClaimed).
- **Shared building blocks:** InfoModal, PriceTierBadge, `planPremiumPayment`, `icrc1Transfer`, `useSneedMembership`, DenominationContext, PriceService.
- **Redesign notes:** the comparison omits Premium effects that exist elsewhere (Sneed Lock fee discounts in LockModal/LockWizard, Sneedapp mint prices, trading bot); the Hub banner promises "priority support" that nothing else mentions; Premium is filed under "Sneed Me" though it's also a product.

---

## Premium status plumbing and gates (used app-wide)

- **PremiumContext:** batched public `checkMemberships` (≤500 per query, 50 ms debounce, 10 min stale, 1 min retry) for any principal shown → `isPremiumMember(p)` drives crowns/rings in PrincipalDisplay, BigAvatar etc.; `refreshPremiumMembers`.
- **usePremiumStatus(identity, principal?)** + `PremiumBadge`: single-principal check (`checkMembership`), cached in localStorage until expiry; used for the header crown, `/me`, `/principal` (viewed user), and fee logic in Lock, Sneedex create, Sneedapp mint, Jailbreak, CreateIcpNeuron, SMS, forum, Apps, Wallet.
- **PriceTierBadge:** "Admin price"/"Partner price" next to prices when special prices apply (Lock, Sneedex create, Sneedapp mint, ICP staking bot, Jailbreak, Pad, Guardian, Studio index, Premium, Sneed Monitor).
- **useSneedMembership + SneedMemberGate:** "Sneed DAO member" = hotkeyed Sneed neurons with VP > 0. Gate components ("🧪 Closed Beta – Sneed DAO Members Only" / "🌱 Premium Feature – Sneed DAO Members Only", how-to steps with links to `/wallet`, `/neurons`; "Beta Feature" warning banner; member badge with neuron count + VP). Currently used only by `/create_icp_neuron` (beta gate with countdown) and `/premium` (VP claim).
- **Redesign notes:** two different "premium" concepts coexist: paid **Sneed Premium** membership vs the **"Premium Feature – Sneed DAO Members Only"** gate (staking VP). The wording collides.

---

## Hub landing — / and /hub

- **Routes & auth:** `/` and `/hub` (same page); public; **not personalised** when logged in (identity only used for fetch agents and the neuron cache).
- **Purpose:** marketing front page + DAO stats + recent activity + launcher.
- **Information shown:**
  - Price corners: SNEED price in ICP and ≈ denomination; ICP price in USD.
  - Hero "Sneed Hub" + tagline; CTAs "Explore Marketplace" `/sneedex_offers`, "Sneedapp" `/sneedapp`, "Trading Bot" `/sneedapp/app/sneed-trading-bot`.
  - SNEED stats grid (tooltips): FDV, Market Cap (circulating), Total Staked (→ `/neurons?sns=<Sneed>`), Total Supply, Circulating Supply (total − treasury − 3 hard-coded ICPSwap LP positions), DAO Members (unique owners of Sneed neurons, from loading all Sneed neurons) (→ `/dao_info`); links "Tokenomics" `/rll_info`, "Staking & Neurons" `/dao_info`.
  - "Sneed Forum — Latest discussions": 5 items of the forum `get_feed` (all SNSes, not just Sneed) as FeedItemCards; "View All" → `/feed`.
  - "Sneedex — Active marketplace offers": 5 active public offers with bid info, estimated value (tokens, SNS neuron stake, ICP staking bot stake, Sneed Lock NFT values) as OfferCards; "Browse All" → `/sneedex_offers`; empty: "Create First Offer" → `/sneedex_create`.
  - Value props: Secure Escrow, Liquid Staking, Multi-DAO Hub, Token Locks.
  - **Featured Products:** Token Studio `/studio` ("…launch a new token…, add an index canister…" — those parts are "coming soon"), Sneedex Swap `/swap`, Sneedex Marketplace, Sneedapp, Trading Bot, Liquid Staking `/liquid_staking`, Sneed Lock `/sneedlock_info` (CTA says "Create Lock" but opens All Locks), SNS Jailbreak `/tools/sns_jailbreak`.
  - **Community & Social:** SNS Forum `/forum`, Activity Feed `/feed`, Direct Messages `/sms`.
  - **Governance:** Proposals `/proposals`, Neurons `/neurons`, Users `/users`, SNS Directory `/sns`.
  - **Utilities:** Wallet `/wallet`, Apps `/apps`, Canisters `/canisters`, Transactions `/transactions`.
  - "Sneed Premium ✨ … priority support…" banner → `/premium`.
  - Phones only: InstallAppCard.
- **Actions:** navigation only.
- **Data calls:** PriceService, SNEED ledger (`icrc1_total_supply`, treasury balance), ICPSwap pool positions, `useNeuronsCache` (all Sneed neurons), forum `get_feed`, Sneedex `getOfferFeed` + `getOfferView` per offer, token metadata/prices, Sneed Lock NFT values.
- **Redesign notes:** the landing page is a fourth launcher (after section menu, quick links, `/me` grid) with its own grouping (e.g. Users under Governance, Canisters/Transactions under Utilities, which the header puts under Sneed Hub); logged-in users get nothing personal (no balances, votes due, messages) although all of that exists in the header chips; DAO stats here duplicate `/dao_info` and `/rll_info`.

---

## Legacy / placeholder routes in this slice

- `/me_info` → `ComingSoon "Profile Info"`; nothing links to it; falls into Sneed Hub section by default. Dead.
- `/hub` and `/` render the same page (Hub submenu item points to `/hub`).
- `/principal` and `/user` render the same page; `/principal` without `id` shows "No Principal Selected" + search.
- `/login` exists mostly for direct links (supports `?from=` which nothing uses); the modal is the real login.

---

## Slice observations

**(a) Entities and their views/actions**

- **Principal / user:** views — `/user` & `/principal` (public profile, 5 tabs), `/users` (directory), PrincipalDisplay everywhere (+ context menu), quick-wallet popup identity card (principal / account ID copy), `/me` (self). Actions — search, copy, view profile, send message, set nickname; self: set public name.
- **Public name:** set on `/me` ("Display Name") or on your own `/user` page ("Public Name"); 32 chars, moderated/ban warning, verified flag (admin, `/admin/names`); account name shared by merged principals; neuron public names on `/me` neuron cards (hotkey required). Canister names on `/canister` (controllers).
- **Nickname (private):** principal nicknames via `/names`, `/principal`, every PrincipalDisplay context menu, AccountPanel hotkey input; neuron nicknames via `/names` (SNS-scoped), `/me` cards, `/neuron`, neuron context menu; account/subaccount nicknames & pockets via naming service (`/wallet/pockets`, not on `/names`). Limits per Premium tier.
- **Avatar:** set on `/me` only (upload, URL, NFT ICRC-7/EXT, remove, stop NFT); also "Use as my avatar" on NFT pages; shown via PrincipalDisplay/BigAvatar everywhere; public page shows it.
- **Profile:** own profile `/me` (hero, stats, account panel, quick access, neurons, transactions, settings); public profile `/user` (posts, trades, neurons, SNS transactions, wallet scan, total value). No bio/links fields.
- **Multi-principal account (naming service):** AccountPanel on `/me` (members, owner/member roles, dapp labels, hotkeys, merge, leave, rename hotkey accounts). No public view.
- **Session:** LoginModal (II 1.0 / II 2.0), `/login`, 7-day delegation, expiry watcher + SessionExpiredBanner, logout in quick-wallet popup. No session info screen.
- **Pairing (PWA ↔ browser):** PairAppPanel in standalone login modal; `/pair?key=` approve in browser.
- **Agent delegation:** `/connect-agent` (scope + duration, deliver or copy). No list/revoke.
- **Premium membership:** `/premium` (status, benefits comparison, buy with ICP, claim with VP, promo code, deposit withdrawal); crown on header logo; badges/rings on users; special-price badges at checkout across products; separate "Sneed DAO member" VP gate.
- **Global settings:** theme, privacy mode, denomination, SNS selection, header/notification toggles, quick links mode, colour coding, particle effects, auto-update, thresholds, slippage, cache tools — split between header Quick-Links dropdown, `/me` Settings, quick-wallet popup, and Shift+P.

**(b) Overlaps with other areas**

- "My stuff" lists in many places: neurons (`/me` Neurons tab, `/wallet` token Neurons tab, `/neurons`, `/principal` of self, header VP bar), transactions (`/me` tab, `/principal` tab, `/transactions`, wallet history), balances (`/wallet`, quick-wallet popup, `/principal` Wallet tab, `/scan_wallet`), trades (`/sneedex_my`, `/principal` Trades tab), posts/tips (`/posts`, `/tips`, `/me` stats, `/principal` Posts tab).
- Launchers: section menu + submenu, header quick links (icon row/hamburger), `/me` Quick Access grid, `/hub` feature cards, Help pages — each with its own grouping and labels.
- Same names, different targets: "NFTs" (`/nfts` browse vs `/nft` mine), "Rewards" (`/rewards` claim vs `/rll` DAO rewards), "Custody" (`/guardian` in DAO vs `/custody` in Tools), "Apps"/"My Apps" (`/apps` in both Sneed Me and Sneedapp), "Address Book" vs "My Contacts".
- Notifications: header chips (forum/SMS/tips/proposals/bots/cycles/collectibles) vs `/notifications` service vs Sneedex DM alert settings on `/me` vs "SMS When Mentioned" on `/me`.
- Wallet actions in the account popup duplicate `/wallet` (send, lock, transfer canister, chores, upgrade, top-up).
- Product settings on `/me` (bot/app cycle thresholds, swap slippage, Sneedex alerts, remove-token button) duplicate or belong to `/apps`, `/swap`, `/sneedex_*`, `/wallet`.
- DAO stats on `/hub` duplicate `/dao_info`, `/rll_info`.

**(c) Dead, hidden, half-built or confusing**

- Dead: `/me_info` (ComingSoon); Header `showTotalValue` prop; PrincipalBox non-compact mode and its imports in `Wallet.jsx`/`RLLInfo.jsx`; Login `?from=` unused; stale comment in Hub about neurons "shared with /users page".
- Hidden: multi-principal accounts (AccountPanel) deep in `/me`; agent delegation `/connect-agent` (only via MCP/`/api` text); PrincipalDisplay context menu (right-click/long-press only); Sneed Drive admin item behind a browser flag; PWA bar force flag; global settings inside the "Quick Links" hamburger; logout inside the wallet popup; currency/privacy vanish from the header when quick links are expanded.
- Half-built / gated: naming-service features (NFT avatar, account panel) only when all naming canisters have ids; notification links only when the notifications service is live; Hub's Token Studio card advertises launch/index features that are "coming soon".
- Confusing: active-section inference is two divergent hard-coded lists with a "Sneed Hub" fallback (e.g. `/wallet/pockets` shows the Sneed Hub submenu); section click carries the old query string; SNS context is global but visible only on SNS pages; `/me` neuron naming uses the header SNS while the list uses its own SNS sub-tab; "Premium" (paid membership) vs "Premium Feature – Sneed DAO Members Only" (VP gate); "Connect Your Wallet" wording for an II login and logged-out pages without a login button; Hub "Sneed Forum" panel shows all SNS forums; Hub Sneed Lock "Create Lock" goes to All Locks; native `alert`/`confirm` in `/names` and `/me` threshold settings despite the in-app dialog system; backend settings are pulled into the browser only when `/me` is visited; `/me` doesn't show your principal ID; viewing your own public page gives no way back to `/me` or to avatar editing.
