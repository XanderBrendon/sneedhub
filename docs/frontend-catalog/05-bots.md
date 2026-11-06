# Bots: ICP Staking Bot and Trading Bot consoles

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Scope: `/icp_neuron_manager/:canisterId` (pages/IcpNeuronManager.jsx), `/create_icp_neuron` (pages/CreateIcpNeuron.jsx + components/CreateIcpNeuronWizard.jsx), `/trading_bot/:canisterId` (pages/TradingBot.jsx + TradingBotWizard, TradingBotDSLPanel, dsl/), the shared bot console (BotManagementPanel, BotEventPanel), and the app-wide bot surfaces (header notifications, health/upgrade dialogs, lamps, utils). Bot canisters are minted from the Sneedapp factory (`declarations/sneedapp`) and are controlled by the user; each page talks directly to the bot canister, to the IC management canister (`aaaaa-aa`, controller-only calls) and to ledgers/CMC/NNS.

Structure of this section:
1. Minting and listing (brief).
2. ICP Staking Bot: create page and wizard; manager page (page shell, Bot console tabs, Neurons section, handler reference).
3. Trading Bot: page shell, Setup Wizard, Script (SneedScript DSL) tab.
4. Shared bot console: BotManagementPanel and BotEventPanel (used by Trading Bot; IcpNeuronManager uses only BotEventPanel).
5. Trading Bot specific consoles: lower tab bar (Chores, Wallet, Performance, Logs, Circuit Breaker, Quick Trade, Recovery), per-chore config tabs, DEX settings, permission set, dead panels.
6. App-wide bot surfaces (header indicators, BotHealthDialog, UpgradeBotsDialog, lamps, BotIcon, settings, utils).
7. Slice observations.

Two pages, two architectures: the Trading Bot page is a thin shell around the shared `BotManagementPanel` plus a second tab strip of trading panels; the ICP Staking Bot page re-implements the same bot console inline (~4,300 lines) and shares only `BotEventPanel`.

## Minting and listing bots (brief — the Sneedapp/App Manager pages are covered by another slice)
- **Routes & params**: `/sneedapp` (store, public), `/sneedapp/app/:appId` (detail; "Mint Now"), `/sneedapp/mint/:appId` (login; steps 'Version', 'Review', 'Gas', 'Confirm' → "Select Version", "Review Payment", extra gas, "Confirm & Mint"; calls `factory.mintCanister(appId, [major], [minor], [patch])`; success links via the app's `viewUrl` with `CANISTER_ID` substituted). App ids: `sneed-icp-staking-bot` (legacy `icp-staking-bot`), `sneed-trading-bot`. `/create_icp_neuron` is a second, staking-bot-only mint path (below). `/apps` (App Manager, login) lists all owned canisters with groups, health lamps, top-up.
- **Purpose**: create a bot canister owned by the user; find it again.
- **Information shown**: mint price (regular / premium / applicable, `PriceTierBadge`, premium discount), included cycles, versions.
- **Actions**: pay mint price (ICP transfer to the factory payment subaccount), mint.
- **Entry points & cross-links** to the bot pages: Wallet "Apps" tab bot cards (Wallet.jsx; neuron list per staking bot, chore controls, "Manage" links incl. `?tab=chores`), `/apps` rows (Apps.jsx → `/icp_neuron_manager/<id>` or `/trading_bot/<id>`, else `/canister?id=`), the header principal dropdown quick wallet (PrincipalBox.jsx → `DappCardModal`, which shows staking-bot neurons and offers chore start/stop/pause/resume/trigger through `createManagerActor` for any bot), header notification dialogs (BotHealthDialog, UpgradeBotsDialog, TopUpCyclesDialog), `/create_icp_neuron` ("Manage" / "Open Manager"), `/help/icp-neuron-manager`, `/help/trading_bot`, `/help/sneedapp`. The trading bot page has no list page of its own ("Navigate to a specific trading bot from your wallet or the Sneedapp page." when no id).
- **Shared building blocks**: WalletContext (`allBotEntries`, `appInfoMap`, `managerChoreStatuses`, `botLogAlerts`, `outdatedManagers`, `lowCyclesCanisters`, `latestVersionByApp`, `versionsByApp`, `isKnownWasmHash`) — the app-wide bot registry that feeds every bot surface outside the two pages.
- **Gating/state**: bot type is resolved from the WASM module hash (`resolvedAppId`); manage URLs come from the factory's `manageUrl` with hard-coded fallbacks.
- **Redesign notes**: bots are reachable from at least five places (Wallet Apps tab, /apps, PrincipalBox/DappCardModal, header dialogs, Sneedapp), each with its own subset of bot info and controls; there is no single "My bots" hub, and the Trading Bot has no index page.

## ICP Staking Bot — create page (`/create_icp_neuron`)
- **Routes & params**: `/create_icp_neuron`; login required for everything except the hero and info cards. Beta gate: access if Sneed DAO member (`useSneedMembership`) OR after `BETA_END_DATE` 2026-01-24 (already passed, so the gate is effectively dead code: countdown "⏰ Opens to everyone in …", "📅 January 24th, 2026", `SneedMemberGateMessage`, `SneedMemberBadge`, `BetaWarningBanner` only render before that date). Premium-affected: mint price uses `factory.getAppMintPrice('sneed-icp-staking-bot', caller)` (regular/premium/applicable + partner tier via `mintPriceTier`).
- **Purpose**: list the user's ICP staking bots and create a new one (with optional first neuron).
- **Information shown**:
  - Hero "ICP Staking Bot" / "Create dedicated canisters to manage your ICP NNS neurons", link "Learn how it works" → `/help/icp-neuron-manager`.
  - "Create New Staking Bot" card: "Latest version: vX.Y.Z", "Creation fee: N ICP" + `PriceTierBadge` + "👑 N% OFF" (premium).
  - "✨ Staking Bot Created" card after a mint: canister id, "Copy", "Open Manager" → `/icp_neuron_manager/<id>`, "Dismiss".
  - "Your Staking Bots (N)" collapsible list (from `factory.getMyWallet()` filtered to staking-bot app ids): per bot `PrincipalDisplay` (name/nickname), "Neurons: N / None yet" (`bot.getNeuronCount`), cycles "⚡ 1.2T" coloured by thresholds (management `canister_status`, controllers only), version "vX.Y.Z" with "⚠️" + tooltip "Newer version available" (vs `factory.getAppVersions('sneed-icp-staking-bot')` / fallback `getOfficialVersions`).
  - Static cards: "How It Works" (5 steps), "Your Neurons Are Always Safe" (cycles-depletion explainer, link `/help/icp-neuron-manager#cycles-depletion`), "📍 Where to Find Your Managers" (links `/wallet`, `/apps`).
- **Actions**: "Login with Internet Identity"; "Create Staking Bot" (opens wizard inline); "Manage" per bot; "Refresh List"; copy created id; "Open Manager".
- **Entry points & cross-links**: SKILL guide, `/help/icp-neuron-manager`; links out to `/wallet`, `/apps`, manager pages.
- **Shared building blocks**: `CreateIcpNeuronWizard`, `BotIcon`, `PrincipalDisplay`, `PriceTierBadge`, `utils/NeuronManagerSettings` (cycle colours), `SneedMemberGate` components.
- **Gating/state**: `caller_is_admin` is fetched "for showing total managers" but never used (dead); `paymentBalance`, `creating`, `copiedDeposit` etc. are unused leftovers. No version picker (always mints the factory default/latest), unlike `/sneedapp/mint`.
- **Redesign notes**: duplicates the Sneedapp mint flow (different steps, no version choice, staking-only) and the bot lists in /apps and the Wallet Apps tab (this list is the only one that shows neuron count next to version and cycles). Candidate to fold into a generic "Create bot" + "My bots" hub.

## ICP Staking Bot — create wizard (CreateIcpNeuronWizard)
- **Routes & params**: inline on `/create_icp_neuron` (login + beta access). Steps: 'Fund Wallet', 'Configure Gas', 'Stake ICP', 'Create' (step circles clickable backwards; forward only when valid).
- **Purpose**: pay, mint a staking bot, optionally top up extra cycles and stake a first NNS neuron in one guided run.
- **Information shown**:
  - Step 1 "Fund Your Wallet": "Your ICP Balance" (auto-refresh every 10 s), "Minimum required: …", deposit principal ("Copy"), `<details>` "Sending from a CEX? Show Account ID" (account id, "Copy"), "Creation Fee:" + tier badge + "👑 N% OFF", "Your canister will receive ~X cycles" (`targetCyclesAmount` − ~500B creation overhead).
  - Step 2 "Configure Gas": "Base Gas (included): ~X", "Extra Gas (Optional)" ICP input with "≈ +X cycles", "Current rate: 1 ICP ≈ X cycles" (CMC `get_icp_xdr_conversion_rate`).
  - Step 3 "Stake ICP": options "Stake Now" (inputs "Amount to Stake (min 1 ICP)", default 1; "Dissolve Delay (days)", default 365, "MIN" = 183, "MAX" = 730; hint "Min 183 days to vote. Max 730 days (2 years) for maximum rewards.") or "Stake Later"; insufficient-balance warning with "Go back to fund your wallet".
  - Step 4 "Confirm & Create": summary rows "Creation Fee", "Extra Gas", "Initial Stake", "Dissolve Delay", "Total"; balance check line.
  - Progress overlay "⏳ Creating Your Staking Bot..." → "🎉 Success!": step list with statuses, "Canister Created! Your new canister ID:", "Neuron ID:".
- **Actions**: "🔄 Refresh Balance", "Cancel", "Continue", "Back", "Create Staking Bot" → (1) ICRC-1 transfer of the applicable fee to the factory payment subaccount (`factory.getPaymentSubaccount`), (2) `factory.mintCanister('sneed-icp-staking-bot', [], [], [])` (parent `onComplete` fires immediately so the bot is listed even if later steps fail), (3) optional extra-gas top-up via `topUpCanisterWithIcp` (ICP → CMC + `notify_top_up`, journaled as a pending top-up), (4) optional stake: `bot.generateMemo`, `bot.getStakeAccount(memo)`, journal the stake (`utils/stakeJournal`), ICP transfer to the neuron subaccount, wait 2 s, `bot.claimNeuronFromDeposit(memo, dissolveDelaySeconds)`. After: "Back to List", "Open Manager", "Close" (error case). Fees: mint price + 0.0001 ICP per transfer. No confirmation dialog beyond the review step.
- **Entry points & cross-links**: "Create Staking Bot" on `/create_icp_neuron`; failure text points to "Finish staking" in the wallet (PendingStakesNotice).
- **Shared building blocks**: `icrc1Transfer`, `topUpCanisterWithIcp`/`describeTopUpResult` (utils/cyclesTopUp), `journalStake`/`updateStake`/`removeStake` (utils/stakeJournal), `PriceTierBadge`, `mintPriceTier`.
- **Gating/state**: requires balance ≥ fee + fees + extra gas + stake; stake ≥ 1 ICP. Dissolve delay not validated against MIN/MAX (input accepts any number).
- **Redesign notes**: the staking step duplicates the manager page's "Create neuron" flow and SnsNeuronWizard's journal+claim pattern; gas top-up duplicates every other top-up UI. Generic "mint any app" (`/sneedapp/mint`) has version + gas steps but no "first neuron" step.

## ICP Staking Bot — manager page shell and access states (`/icp_neuron_manager/:canisterId`)
- **Routes & params**: `/icp_neuron_manager/:canisterId` (route in App.jsx). Login required for everything on this page; the page never renders bot data anonymously.
- **Purpose**: Console for one user-owned ICP Staking Bot canister (the "Bot" section, this part) plus its NNS neurons (the "Neurons" section, part inm-a).
- **Information shown**:
  - Logged out: card "ICP Staking Bot" / "Please log in to manage your neuron." and button "Login with Internet Identity".
  - Loading: "Loading manager...".
  - Load failure with no data: error card ("Failed to load manager: …").
  - Invalid canister: warning card "App Canister Not Recognized as ICP Staking Bot" + reason (for example "missing getVersion method" or "missing getNeuronIds method") + a hint to reinstall the official WASM from the Bot section. The Bot section expands itself automatically in this case.
  - No access: card "No Access", "You are not a controller of this Staking Bot and have no botkey permissions." This shows when the user is neither a controller nor holds any botkey permission. The Bot section (Info and Botkeys tabs) still renders above it, but the Neurons section is hidden.
- **Actions**: "Login with Internet Identity" (calls `login`).
- **Loads on entry**: `getVersion`, `getNeuronIds` (a trap here is read as "no ViewNeuron permission", not as an invalid bot), the user's ICP balance (`icrc1_balance_of`), management `canister_status` (succeeds only for controllers), CMC `get_icp_xdr_conversion_rate`, `getChoreStatuses`/`getChoreConfigs` (loaded early so the header lamps can show), and `callerPermissions` (the user's botkey permissions). The factory's official versions (`sneedapp.getAppVersions('sneed-icp-staking-bot')`, falling back to `getOfficialVersions`) load without auth.
- **Entry points & cross-links**: Linked from /apps (Apps.jsx), the Wallet Apps tab (Wallet.jsx), /create_icp_neuron and CreateIcpNeuronWizard ("Open Manager"), DappCardModal, and the header BotHealthDialog (BotChoreHealthDialog / BotLogAlertDialog are unused leftovers).
- **Gating/state**: Two access concepts. **Controller** status comes from `canister_status` controllers and grants all permissions implicitly. A **botkey** grants per-permission access via `callerPermissions`; `FullPermissions` covers everything.
- **Redesign notes**: There are three "can't use this" states (logged out, invalid, no access) and they look different. The No Access card sits below a fully rendered Bot section, which is confusing.

## ICP Staking Bot — Bot identity header (hero)
- **Routes & params**: Same page. Login required.
- **Purpose**: Identify the bot and show its overall health at a glance.
- **Information shown**:
  - Title "ICP Staking Bot".
  - Version badge `v{version}`, with ✓ when the module hash matches an official release.
  - All-chores summary pill: "Chores: Error / Attention / Active / OK / Idle", from `getAllChoresSummaryLamp`.
  - Canister principal (PrincipalDisplay with copy button and the user's nickname or public name; no link).
- **Actions**:
  - "Learn how it works" links to /help/icp-neuron-manager.
  - A toggle labelled "Name this bot" (controllers) or "Set nickname" (others), or "Hide naming options" when open.
- **Shared building blocks**: BotIcon (`type="staking"`), StatusLamp (ChoreStatusLamp.jsx), PrincipalDisplay.
- **Redesign notes**: The TradingBot hero is an almost identical copy, apart from its extra "Setup Wizard" button. Good candidate for a shared bot header.

## ICP Staking Bot — Naming the bot (nickname and public name)
- **Routes & params**: Same page. A nickname needs login; a public name needs the user to be a **controller**.
- **Purpose**: Label the bot canister across the app.
- **Information shown**:
  - "Name Your Staking Bot" panel.
  - Current nickname ("Current: …") and current public name.
  - The warning "This name will be visible to anyone who views this canister across the app."
- **Actions**:
  - "Private Nickname" → "Save": `setPrincipalNickname`, then refetches names (naming backend; only the user sees it).
  - "Public Name" → "Save" (controllers only): `setPrincipalNameFor`; visible to everyone.
- **Shared building blocks**: NamingContext (`fetchAllNames`) and BackendUtils. TradingBot has a byte-identical copy of this panel.
- **Redesign notes**: This is generic principal naming, the same thing done for users and neurons elsewhere. It could become a shared "Name this canister" component or move to the App Manager.

## ICP Staking Bot — "Bot" collapsible section and tab bar
- **Routes & params**: Same page. Rendered for any logged-in user once manager info has loaded.
- **Purpose**: Container for canister-level management.
- **Information shown**:
  - Header row "Bot".
  - Cycles badge `⚡ {cycles}`, colored by the user's local thresholds (`getCyclesColor` and `getNeuronManagerSettings`: red below 1T, orange below 5T by default, stored in localStorage). Controllers only, because it comes from `canister_status`.
  - One StatusLamp per chore instance, plus the label "Chores".
- **Actions**:
  - Expand/collapse (expanded by default).
  - Tabs:
    - "Info": everyone.
    - "Chores" (with a summary lamp): needs `ViewChores` or any chore manage/configure permission.
    - "Botkeys": everyone.
    - "Log": needs `ViewLogs`.
    - "Events": needs `ViewEvents` or `ManageEvents`.
- **Gating/state**: Each tab lazy-loads its data when opened while the section is expanded.
- **Redesign notes**: This duplicates `BotManagementPanel`, which has the same tab set and gating. See the comparison section at the end of this part.

## ICP Staking Bot — Info tab: canister overview and WASM verification
- **Routes & params**: Same page. Status fields need a **controller** (`canister_status`); everyone else sees only the Canister ID and Version.
- **Purpose**: Show what is running in the canister and whether it is genuine.
- **Information shown**:
  - Canister ID (with copy).
  - Cycles (threshold-colored).
  - Version, with ✓ "Version verified against official registry" when the hash matches and the reported version agrees.
  - Status (running/stopping/stopped).
  - Memory in MB.
  - Module Hash, with a badge "✓ Official vX.Y.Z" or "⚠ Unverified", or "No module installed".
- **Actions**: Links "View Source →" and "Download WASM →" (from the matched official version).
- **Shared building blocks**: The factory official-version registry (`sneedapp`). The same verification UI exists in BotManagementPanel and in the App Manager.
- **Redesign notes**: Non-controllers see no cycles at all. BotManagementPanel falls back to the bot's public cycles query (`utils/botCycles.fetchBotCyclesBalance`); this page does not.

## ICP Staking Bot — Info tab: upgrade, reinstall and install official version
- **Routes & params**: Same page. Needs a **controller**.
- **Purpose**: Move the bot to official WASM releases.
- **Information shown**:
  - "🚀 Upgrade Available": "vX.Y.Z is available". This is the next higher official version that has a wasmUrl, and it only shows when the current hash matched an official release. A "View release notes →" link (sourceUrl) appears with it.
  - "⚠️ Version Mismatch": the hash matches an official version but the canister reports a different one.
  - "⚠️ Unknown WASM — Proceed With Care": an unverified hash, with a full explanation and the module hash.
  - "Target: ICP Staking Bot vX.Y.Z — View source code →".
  - Success and error messages.
  - Non-controllers who hit an issue (invalid, unverified, or mismatched) see "🔒 Upgrade/Reinstall Not Available", "You are not a controller…", and the latest official version.
- **Actions**:
  - "⬆️ Upgrade" ("Upgrading..."): fetches the WASM from wasmUrl, checks its sha256 against wasmHash (`verifyDownloadedWasm`), then runs management `install_code` in `upgrade` mode with empty args. State is kept. There is no confirmation, except for an unknown WASM, which asks through `appConfirm`.
  - "🔄 Reinstall": `install_code` in `reinstall` mode, after `appConfirm` "Reinstall will WIPE ALL CANISTER STATE…".
  - Afterwards the page refetches status and manager data and forces the displayed version to the new one (works around query caching).
- **Gating/state**:
  - "Upgrade Available" upgrades one step, to the next version.
  - "Install Official Version" and the Botkeys-tab upgrade go straight to the **latest** version.
  - BotManagementPanel instead walks every intermediate version (`utils/upgradePath.upgradeSteps`). This page has no multi-step path.
- **Redesign notes**: Upgrade logic is duplicated, and the two copies behave differently. The App Manager's UpgradeBotsDialog is a third path for bulk upgrades.

## ICP Staking Bot — Info tab: controllers management
- **Routes & params**: Same page. Visible only to a **controller** (the list comes from `canister_status`).
- **Purpose**: Manage the canister's IC controllers.
- **Information shown**:
  - "Controllers (n)": each principal with its name, copy button and link, plus a "YOU" badge.
  - The warning "Removing all controllers will make the canister permanently uncontrollable."
- **Actions**:
  - "Remove", then an inline confirm ("Confirm?", or "⚠️ Remove yourself?") with "Yes" / "No".
  - "Add Controller" (PrincipalInput with name search) → "Add".
  - Both run management `update_settings` with the new controllers list. Removing the last controller is blocked client-side, and duplicate adds are rejected.
- **Shared building blocks**: PrincipalInput, PrincipalDisplay. BotManagementPanel has the same feature.
- **Redesign notes**: Success shows inline ("✅ Controllers updated successfully"). **Errors go to the page-level error banner, which renders inside the Neurons section** (see Part notes).

## ICP Staking Bot — Info tab: Top Up Cycles
- **Routes & params**: Same page. Any logged-in user can top up; there is no permission check.
- **Purpose**: Turn the user's ICP into cycles for the bot.
- **Information shown**:
  - "Top Up Cycles" / "Convert ICP to cycles for this canister".
  - PendingTopUpsNotice: unfinished earlier top-ups that can be resumed.
  - "Your ICP Balance".
  - "Current Rate: 1 ICP ≈ N cycles" (from the CMC).
  - "Estimated Cycles to Add".
  - Fee note: "Converts ICP to cycles via the Cycles Minting Canister (CMC). A small ICP fee (0.0001) applies."
- **Actions**:
  - "⛽ Add Cycles" / "Cancel" toggles the form.
  - "Amount (ICP)" input with "MAX" (balance minus one fee).
  - "Top Up Canister": `topUpCanisterWithIcp` sends ICP to the CMC subaccount, then calls `notify_top_up`. The transfer is journaled as a pending top-up so a retry finishes it without paying again.
  - Afterwards a success dialog shows "Cycles Added", "ICP Spent" and "ICP ledger block #", with "Done". Balance is checked against amount plus fee first.
- **Shared building blocks**: utils/cyclesTopUp, PendingTopUpsNotice. The same card is in BotManagementPanel, and top-up also exists in /apps and the Wallet Apps tab.
- **Redesign notes**: Cycles top-up appears in at least four places.

## ICP Staking Bot — Info tab: Withdraw Tokens from the bot
- **Routes & params**: Same page. Needs **botkey `WithdrawFunds`** (or controller).
- **Purpose**: Move ICP or any ICRC-1 token held by the bot canister's main account to a principal.
- **Information shown**:
  - "💸 Withdraw Tokens" (collapsible): "Withdraw ICP or any ICRC1 token from this canister".
  - "Available: {bot balance} {symbol}", read directly from the ledger (`icrc1_balance_of`, symbol, decimals, fee).
  - "Fee: …".
- **Actions**:
  - "Select Token": "From list" (TokenSelector) or "Custom ledger" (principal input).
  - "Amount", "Destination Principal" (plain text with no subaccount, unlike other inputs on the page).
  - "💸 Withdraw": bot `withdrawIcp(amount, account)` for ICP, otherwise `withdrawToken(ledger, amount, account)`. Success shows "Withdrew … Block height: N".
- **Redesign notes**: Results and errors show in the Neurons-section banner. The destination field takes neither a subaccount nor name search. This is a wallet-like capability buried in the "Info" tab.

## ICP Staking Bot — Botkeys tab: bot permission management
- **Routes & params**: Same page. **Viewable by everyone** logged in; editing needs **botkey `ManagePermissions`** (or controller).
- **Purpose**: Let other principals operate the bot without being a controller.
- **Information shown**:
  - An explainer, including "Botkeys vs Neuron Hotkeys": botkeys are bot-canister permissions, while NNS hotkeys sit on the neuron itself.
  - "Botkey Principals (n)": each principal with permission badges (hover shows the description).
  - "No botkey principals configured."
  - Unsupported versions: "Botkeys Not Available… require v0.9.1 or newer. Your bot is currently running vX", with an "⬆️ Upgrade to vX.Y.Z" button for controllers.
- **Actions** (ManagePermissions):
  - "Edit" opens checkboxes, then "Save" / "Cancel". This diffs into `addHotkeyPermissions(principal, toAdd)` and `removeHotkeyPermissions(principal, toRemove)`.
  - "Remove", "Remove?", "Yes" / "No": `removeHotkeyPrincipal`.
  - "Add Botkey Principal": PrincipalInput plus permission checkboxes with descriptions, "Select All" / "Clear All", then "Add Botkey Principal" (`addHotkeyPermissions`).
- **Loads**: `listHotkeyPrincipals`, `listPermissionTypes` (the permission list is **dynamic, from the canister**), and `callerPermissions`. Older bots are detected by "has no query method" errors.
- **Permission catalog (ICP Staking Bot; labels as the UI shows them; unknown keys fall back to a split CamelCase label with no description)**:

  | Key | Label | What it allows |
  |---|---|---|
  | FullPermissions | Full Permissions | All permissions, including future ones |
  | ManagePermissions | Manage Permissions | Add or remove botkey principals and their permissions |
  | ViewNeuron | View Neuron | View neuron info, list neurons, check balances; read-only access to all neuron tabs |
  | StakeNeuron | Stake Neuron | Create neurons; increase or refresh stake |
  | AutoStakeMaturity | Auto-Stake Maturity | Set the auto-stake maturity setting |
  | StakeMaturity | Stake Maturity | Stake maturity rewards |
  | MergeMaturity | Merge Maturity | Merge maturity into stake |
  | DisburseMaturity | Disburse Maturity | Disburse maturity rewards |
  | Spawn | Spawn | Spawn maturity into a new neuron |
  | ConfigureDissolveState | Configure Dissolve State | Start or stop dissolving; set dissolve delay |
  | Disburse | Disburse | Disburse neuron stake |
  | Split | Split | Split a neuron |
  | MergeNeurons | Merge Neurons | Merge neurons together |
  | ManageFollowees | Manage Followees | Set followees; confirm following |
  | Vote | Vote | Vote on proposals; refresh voting power |
  | MakeProposal | Make Proposal | Submit NNS proposals using a neuron |
  | ManageNeuronHotkeys | Manage NNS Hotkeys | Add or remove NNS hotkeys on the neuron |
  | ManageVisibility | Manage Visibility | Set neuron visibility |
  | WithdrawFunds | Withdraw Funds | Withdraw ICP or tokens from the canister |
  | ManageConfirmFollowing | Manage Confirm Following | Start, stop, pause, resume and configure that chore |
  | ManageRefreshStake | Manage Refresh Stake | Start, stop, pause, resume and configure that chore |
  | ManageCollectMaturity | Manage Collect Maturity | Start, stop, pause, resume and configure that chore |
  | ManageDistributeFunds | Manage Distribute Funds | Start, stop, pause, resume and configure that chore |
  | ConfigureCollectMaturity | Configure Collect Maturity | Set the collection threshold and destination |
  | ConfigureDistribution | Configure Distribution | Add, edit and remove distribution lists |
  | ViewChores | View Chores | View chore statuses, configs and settings |
  | ViewEvents | View Events | View event types, listeners, subscriptions and logs |
  | ManageEvents | Manage Events | Configure subscriptions, reaction rules and emission |

  - The UI also gates on `ViewLogs` (Log tab) and `ManageLogs` (log settings). Neither is in the known-label map, so they show as "View Logs" / "Manage Logs" with no description.
- **Neuron-tab mapping** (used by part inm-a):
  - stake = StakeNeuron, AutoStakeMaturity
  - maturity = StakeMaturity, MergeMaturity, DisburseMaturity, Spawn
  - following = ManageFollowees
  - dissolve = ConfigureDissolveState
  - disburse = Disburse
  - hotkeys = ManageNeuronHotkeys
  - advanced = Split, MergeNeurons, ManageVisibility
  - Each tab is also visible with ViewNeuron.
- **Shared building blocks**: PrincipalInput (`defaultTab="private"`). BotManagementPanel has its own copy, with labels passed in as props.
- **Redesign notes**:
  - "Botkeys" (bot permissions) and the neuron "hotkeys" tab (NNS-level) sit in different sections. The explainer exists because the two get confused.
  - The SNS neuron pages also have hotkey/permission management, so a single "access / keys" view per asset could reduce confusion.

## ICP Staking Bot — Chores tab: schedule overview and chore navigation
- **Routes & params**: Same page. Needs **`ViewChores`** or any chore manage/configure permission.
- **Purpose**: Show the bot's automation tasks and choose one to inspect or configure.
- **Information shown**:
  - Explainer: "Bot Chores are automated tasks that run on a schedule…".
  - "Schedule" strip: every enabled chore with a next run, running ones first. Each shows a relative time ("in 3m 05s" ticks by the second in the last 5 minutes; also "in 2h 10m", "overdue", "running"). Dot color marks running, paused, overdue or scheduled. Hover shows the absolute time; clicking jumps to that chore.
  - One tab per chore **type**, each with its worst-instance lamp and an instance count. Instance sub-tabs (lamp plus label) show when a type has more than one instance.
  - Chore types (choreTypeId), each with interval help text:
    - `confirm-following`: "NNS requires following confirmation at least every 6 months. We recommend 30 days or less."
    - `refresh-stake`: "…refresh stake on all managed neurons to pick up any deposited ICP."
    - `collect-maturity`: "…check and collect maturity from all managed neurons."
    - `distribute-funds`: "…check distribution lists and distribute funds when thresholds are met."
  - Empty state: "No chores available. This bot version may not support automated chores yet."
- **Actions**:
  - "+" (add instance to a multi-instance type), or "+ Add another {type}" for single-instance `distribute-funds` / `collect-maturity`.
  - "New {type} Instance" form → "Create": `createChoreInstance(typeId, typeId-<base36 time>, label)`; or "Cancel".
  - For non-default instances: "Rename" → "Save" (`renameChoreInstance`); "Delete" (only when stopped, `appConfirm`, `deleteChoreInstance`).
- **Gating/state**: Status auto-refreshes adaptively:
  - every 5s while any chore runs
  - 3s after the next scheduled run, capped at 60s
  - 8s when a chore is overdue
  - 60s when idle
- **Redesign notes**: The schedule strip and type/instance tabs mirror BotManagementPanel. The two bots share the chore framework.

## ICP Staking Bot — Chores tab: per-chore status card
- **Routes & params**: Same page. Needs `ViewChores` (or manage permissions).
- **Purpose**: Detailed health of one chore instance.
- **Information shown**:
  - The chore description.
  - "State": Stopped / Paused / Running.
  - Three lamps, each with a label from ChoreStatusLamp: "Scheduler", "Conductor", "Task".
  - "Interval" (for example "30 days", or a randomized range "1 hr–2 hr").
  - "Next Scheduled Run".
  - "Last Completed".
  - "Runs (Success / Fail)".
  - "Last error" with its timestamp.
  - "Current task" with its id and start time.
- **Actions**: "Set" next to Next Scheduled Run opens a datetime picker → "Save". This calls `setChoreNextRun(choreId, ns)` and updates the display optimistically. After 2.5s it re-fetches to verify and warns "the backend may not have saved the new time… Try upgrading". It shows only when the chore is enabled, and the UI does not check permissions here.
- **Shared building blocks**: StatusLamp and the lamp-state helpers (ChoreStatusLamp.jsx), shared with TradingBot and the App Manager.

## ICP Staking Bot — Chores tab: chore controls and schedule
- **Routes & params**: Same page. Needs **botkey `Manage<Chore>`** for that chore type (ManageConfirmFollowing, ManageRefreshStake, ManageCollectMaturity or ManageDistributeFunds), or controller.
- **Purpose**: Run, stop and schedule a chore.
- **Actions**:
  - "Start" (when stopped): `startChore` runs now and schedules the next run. Its split arrow "▼" opens "Schedule first run at:" (datetime) → "Confirm" (`scheduleStartChore`, must be in the future) or "Cancel".
  - "Pause" (`pauseChore`, keeps the schedule); "Resume" (`resumeChore`); "Stop" (`stopChore`, clears the schedule).
  - "Run Once" (when stopped) or "Run Now" (when enabled and idle): `triggerChore`.
  - "Frequency: Every [n] [minutes|hours|days]" → "Save". Calls `setChoreInterval` and then `setChoreMaxInterval`. The interval must be between 1 minute and 365 days.
  - "Randomize interval..." / "Randomized range active (up to X) — edit": an optional "Max" interval for a randomized schedule (clear it to disable). It must be greater than the base interval.
- **Redesign notes**: Some inputs are uncontrolled DOM fields toggled with `document.getElementById`, a fragile pattern. There is **no "Stop All" chores** button here; BotManagementPanel has one (`stopAllChores`).

## ICP Staking Bot — Chores tab: Collect Maturity settings
- **Routes & params**: Same page. Needs **botkey `ConfigureCollectMaturity`**. Applies per instance, and collect-maturity supports several instances.
- **Purpose**: Choose when and where collected neuron maturity goes.
- **Information shown**:
  - "Collection Settings".
  - The current threshold ("Current: X ICP", or "No minimum set — will collect any available maturity.").
  - The current destination ("Current: principal (with subaccount)", or "Default: Bot's own account").
- **Actions**:
  - "Minimum maturity to collect (ICP)" → "Save": `setCollectMaturityThreshold(instance, opt)`. Empty or 0 removes the threshold.
  - "Send collected maturity to:" (PrincipalInput; a pasted ICRC-1 account string is decoded into principal plus subaccount, and the subaccount chip has a "✕" to clear it) → "Save": `setCollectMaturityDestination(instance, opt account)`. Empty resets to the bot's own account.
- **Loads**: `getCollectMaturitySettings(instanceId)`.

## ICP Staking Bot — Chores tab: Distribute Funds (distribution lists)
- **Routes & params**: Same page. Needs **botkey `ConfigureDistribution`**. Applies per instance, with several instances possible (for example "ETH Distribution").
- **Purpose**: Automatically split balances the bot holds (any token) among recipients once a threshold is reached.
- **Information shown**:
  - "Distribution Lists (n)". Each list shows its name, token chip (icon and symbol), "Threshold" and "Max/round" with USD values (priceService), and the "Targets" count.
  - Mode text: "Each round: X% of the balance, at most the max per round".
  - Per target: principal (with context menu), subaccount hex, long account string, and the effective % next to the configured % (or "auto").
  - "X% undistributed".
  - Warnings when assigned percentages pass 100% ("Will be renormalized").
- **Actions**:
  - "+ Add List"; "Edit"; "Remove" (`appConfirm`, `removeDistributionList`).
  - The form ("New Distribution List" / "Edit Distribution List") has:
    - "Name"
    - "Token" (TokenSelector, custom ledger allowed)
    - "Amount per round": "The whole balance, up to the max per round" or "A percentage of the balance, up to the max per round", with "Percentage of the balance (%)"
    - "Threshold", "Max per round"
    - "Targets": "Recipient" (PrincipalInput; a pasted ICRC-1 account is decoded), "% Share" (blank means auto), "Subaccount (hex, optional)", "Remove", "+ Add Target", plus a live "Effective: X%"
  - "Create List" (`addDistributionList`), "Save Changes" (`updateDistributionList`), "Cancel".
  - Client-side checks: name, token, at least one target, each target principal, and 0 < % ≤ 100.
- **Loads**: `getDistributionLists(instanceId)`.
- **Gating/state**: The source is always the bot's main account (`sourcePurseId: []`). Purse targets (`choreInstanceId`) are a trading-bot feature not exposed here.
- **Redesign notes**:
  - `minDistributionAmount` is stored and sent, but **has no input field**: it is preserved from existing data, so it is half-built.
  - This feature overlaps with the Trading Bot's DistributionConfigPanel (purse-aware). It is a strong candidate for a shared "distribution" component.

## ICP Staking Bot — Log tab: bot activity log
- **Routes & params**: Same page. Needs **botkey `ViewLogs`**; settings need **`ManageLogs`**.
- **Purpose**: Audit trail of API calls, chore actions, permission changes and errors.
- **Information shown**:
  - "Currently storing N of M max entries at {level} level."
  - Entries, newest first: level badge, source, message, timestamp, `#id`, tag chips (key: value), and caller principal.
  - Empty state: "No log entries found (matching the current filters)".
  - Footer: "Showing n of N matching · total stored · Max".
- **Actions**:
  - Level filter "Show:" Error / Warning / Info / Debug / Trace. It is cumulative ("Warning+" means Error and Warning). Click again or use "All" to reset.
  - "Source:" api / permissions / chore / system / log (single select).
  - "Refresh"; "Auto-refresh" checkbox (polls every 5s).
  - "Per page:" 10, 25, 50, 100, 200 or 500.
  - Pagination: "« Newest", "Older »".
  - With ManageLogs:
    - "Write level:" Off / Error / Warning / Info / Debug / Trace (`setLogLevel`).
    - "Max entries:" 1,000 to 50,000 (`setMaxLogEntries`, with `appConfirm` warning how many entries will be trimmed).
    - "Clear Logs" (`appConfirm`, `clearLogs`).
- **Loads**: `getLogs(filter)`, `getLogConfig()`.
- **Redesign notes**:
  - The trading bot's log is richer: token/amount tag rendering, alert summaries, "Mark as seen".
  - The header bot-log notifications (useBotLogNotification / BotLogAlertDialog) point here, but this tab has **no "Mark as seen"**. BotManagementPanel has one.

## ICP Staking Bot — Events tab (mount of the shared BotEventPanel)
- **Routes & params**: Same page. Needs **botkey `ViewEvents`** or **`ManageEvents`**.
- **Purpose**: The bot's event system: event types, listeners, subscriptions and reaction rules (detailed in the BotEventPanel part).
- **Mount**: `<BotEventPanel canisterId identity theme accentColor hasPermission choreStatuses />`. It does not pass `tokenRegistryEntries`, which the trading bot can pass.
- **Shared building blocks**: BotEventPanel is the only bot-console piece this page shares with BotManagementPanel.

## ICP Staking Bot — Comparison: this page's built-in bot console vs. the shared BotManagementPanel (used by TradingBot)
- **Same in both**:
  - Tab names (Info / Chores / Botkeys / Log / Events) and their permission gating.
  - Info card fields, module-hash verification and Upgrade/Reinstall.
  - Controllers editor, Top Up Cycles card (with PendingTopUpsNotice), Withdraw tokens.
  - Botkey editor ("Select All" / "Clear All").
  - Chore type/instance tabs, lamps, Start / Pause / Resume / Stop / Run Once.
  - Log filters and settings.
  - BotEventPanel mount.
- **Only in BotManagementPanel**:
  - "Stop All" chores (with ConfirmDialog, `stopAllChores`).
  - Multi-step upgrade path (`upgradeSteps`).
  - Public cycles fallback for non-controllers (`fetchBotCyclesBalance`).
  - Log alert summary with "Mark as seen".
  - Circuit-breaker-aware lamps (`LAMP_CB`, cbEvents).
  - A `renderChoreConfig` render prop for bot-specific chore settings.
  - `hideLogTab`.
  - An imperative `navigateToChore`.
- **Only here**: Staking-specific chore settings (Collect Maturity threshold/destination, Distribute Funds lists) coded inline. The randomized max interval and next-run picker exist in both but are implemented separately.
- **Redesign notes**: ~4,300 lines here re-implement the shared panel. Moving the ICP Staking Bot onto BotManagementPanel with a `renderChoreConfig` for collect-maturity and distribute-funds would give one "bot console" pattern for both bots, and close the feature gaps listed above.

## ICP Staking Bot — page access & loading states (neuron-relevant)
- **Routes & params**: `/icp_neuron_manager/:canisterId`; login required for everything except the (invisible) official-version fetch. Access tiers: controller (from `canister_status`) = all permissions; botkey principal = permissions from `callerPermissions`; anyone else = "No Access".
- **Purpose**: gate the bot console and the neuron section by validity of the canister and the caller's rights.
- **Information shown**:
  - Logged out: "ICP Staking Bot" / "Please log in to manage your neuron." + "Login with Internet Identity".
  - Loading: "Loading manager..."; hard error banner if nothing loaded.
  - Invalid canister: "App Canister Not Recognized as ICP Staking Bot" + reason (missing `getVersion` / `getNeuronIds`) + hint to reinstall the official WASM from the Canister section (which auto-expands). Neuron section hidden.
  - No rights: "No Access" card ("You are not a controller of this Staking Bot and have no botkey permissions…").
  - Global success/error banner (top of Neurons section) for all neuron actions, incl. step messages ("📤 Sending ICP to NNS Governance...", "🧠 Claiming neuron from deposit...", "🔄 Refreshing neuron stake...").
- **Actions**: "Login with Internet Identity".
- **Entry points & cross-links**: Wallet Apps/bot cards (Wallet.jsx 7754, 7880, 8036, 9021), DappCardModal (971), Apps App Manager, CreateIcpNeuron / CreateIcpNeuronWizard "Open Manager" (post-mint), the header's BotHealthDialog (BotChoreHealthDialog / BotLogAlertDialog exist but are not imported anywhere). Several of these link with `?tab=chores` or `?tab=log` (DappCardModal 512, Wallet 8408/9291, BotChoreHealthDialog 13, BotLogAlertDialog 21) — **the page never reads the query string**, so the deep link lands on the default tabs.
- **Shared building blocks**: `useAuth`, `HttpAgent`, management-canister IDL (duplicated inline here and in TradingBot.jsx), `getNeuronManagerSettings` (cycle thresholds).
- **Gating/state**: Header highlights "Liquid Staking"-group nav for `/icp_neuron_manager*` (Header.jsx 235/375). Neuron section requires `!isInvalidManager && hasAnyPermission`.
- **Redesign notes**: page = bot console (canister/chores/botkeys/log/events) + full NNS neuron manager in one scroll, two independently collapsible sections ("Canister"/bot and "Neurons"). Natural split: Bot console vs. Neuron(s).

## ICP Staking Bot — Neurons section: list & selector
- **Routes & params**: same page; requires any permission; list needs `ViewNeuron` (or controller) — without it `getNeuronIds` traps and the list is silently empty.
- **Purpose**: pick which of the bot's neurons (one bot can hold many) to manage.
- **Information shown**: collapsible header "🧠 Neurons" + "{n} neuron(s)" badge; card "{n} Neuron(s) Managed" / "Select a neuron to manage"; dropdown of "Neuron #id"; scrollable clickable list "Neuron #id" (highlighted when selected). No per-neuron stake/state in the list.
- **Actions**: select neuron (dropdown or list) → `getNeuronInfo` + `getFullNeuron`; collapse/expand section.
- **Gating/state**: first neuron auto-selected; selector shown even with one neuron.
- **Redesign notes**: list carries no data (only IDs) whereas Wallet/DappCardModal show per-neuron stake/maturity/dissolve for the same bot — inconsistent. Selection is not in the URL.

## ICP Staking Bot — Create neuron (stake ICP from wallet into the bot)
- **Routes & params**: same page; botkey `StakeNeuron` (or controller). Section title "Create Your First Neuron" (auto-expanded when the bot has 0 neurons) or "Create Another Neuron" (collapsed).
- **Purpose**: create an additional NNS neuron owned by the bot, paid from the user's wallet.
- **Information shown**: inputs "Amount to Stake (ICP)" (default 1) and "Dissolve Delay (days)" (default 365); hint "💡 Min 183 days to vote, max 2 years. Stakes directly from your wallet ({balance} ICP available)".
- **Actions**: "🚀 Create Neuron" — disabled if wallet < 1 ICP. Validates ≥1 ICP and balance ≥ amount + 0.0001 ICP fee. Flow: `generateMemo` → `getStakeAccount(memo)` → journal → ICP transfer to NNS governance neuron subaccount → `claimNeuronFromDeposit(memo, delay)`. No confirmation dialog. Success "🎉 Neuron created! ID: …". Errors: governance error, "Invalid dissolve delay: min … max … days".
- **Entry points & cross-links**: also possible at mint time via `/create_icp_neuron` wizard (covered elsewhere).
- **Shared building blocks**: `icrc1Transfer`, `utils/stakeJournal` (journalStake/updateStake/removeStake), PendingStakesNotice.
- **Gating/state**: client-side dissolve-delay validation is commented out (canister enforces).
- **Redesign notes**: duplicates the "stake" step of CreateIcpNeuronWizard and the SNS staking flow of SnsNeuronWizard (same journal pattern).

## ICP Staking Bot — Unfinished neuron stakes recovery (PendingStakesNotice)
- **Routes & params**: mounted at the top of the Neurons section (7428); also on Wallet (Wallet.jsx 5850) and SnsNeuronWizard (on staking error). Login (uses identity).
- **Purpose**: recover stakes where tokens were sent to a neuron subaccount but the claim never confirmed (journal in localStorage `sneed_pending_stakes_v1`).
- **Information shown**: "Unfinished neuron stakes ({n})"; per entry: kind ("ICP neuron (staking bot)" / "SNS neuron"), "ledger block N" or "transfer not confirmed", created time, last result; "Details": ledger, governance, staking bot, amount (base units), memo, subaccount, block, last answer.
- **Actions**: "Finish staking" (repeats claim only: bot `claimNeuronFromDeposit(memo, delay)` or SNS `manage_neuron ClaimOrRefresh MemoAndController`); "Details"/"Hide details"; "Dismiss" → two-step "Yes, forget it" / "Keep" with warning.
- **Gating/state**: renders nothing when empty; lists all of the user's pending stakes (SNS and other bots), not only this bot's; browser-local (lost on another device).
- **Redesign notes**: belongs to a cross-cutting "pending operations" area together with PendingTopUpsNotice (used in the Info tab).

## ICP Staking Bot — Neuron summary & Overview tab
- **Routes & params**: same page; selected neuron; Overview tab always available to anyone with any permission.
- **Purpose**: at-a-glance state of the selected neuron.
- **Information shown**:
  - Summary card: "Neuron #id", state badge (Locked / Dissolving / Dissolved / Spawning / Unknown); stat boxes Stake (ICP), Voting Power, Dissolve Delay (y/d), Age, Maturity (ICP).
  - Overview tab "Neuron Overview": dissolving alert "⏳ Dissolving — {time} remaining until dissolved"; Visibility (🌐 Public / 🔒 Private); Controller (truncated principal — the bot); Available Maturity; Staked Maturity; Hotkeys (count); Following Topics (count).
- **Actions**: "🔗 View on IC Dashboard →" (external `dashboard.internetcomputer.org/neuron/{id}`); "🔄 Refresh Data" (`fetchManagerData`).
- **Gating/state**: tab bar shows Overview + tabs the user can access (see permission model).
- **Redesign notes**: fields overlap the ICP neuron cards in DappCardModal/Wallet and NNS neuron display in IcpHotkeyNeurons (/icp_proposal).

## ICP Staking Bot — Stake tab
- **Routes & params**: tab "Stake"; visible with `StakeNeuron`, `AutoStakeMaturity` or `ViewNeuron`.
- **Purpose**: add ICP to the neuron and manage auto-stake.
- **Information shown**: "Current Stake"; "Neuron Deposit Account" (ICRC-1 text encoding of NNS governance + neuron subaccount) with explanation that the "Refresh Stake" chore picks up deposits; wallet balance "Your wallet: {x} ICP available"; Auto-Stake Maturity "Current: ✅ Enabled / ❌ Disabled".
- **Actions**:
  - "Copy" deposit account (toast "Neuron deposit account copied to clipboard!").
  - "Increase Stake" → "➕ Add to Stake" (amount ICP; disabled if wallet < 0.01 ICP; balance ≥ amount+fee) → `getNeuronAccount` → ICP transfer → `refreshStakeFromDeposit`. Not permission-gated in UI ("permissionless", user's own funds).
  - "🔄 Refresh Stake from NNS" → `refreshStake` ("Use this if you sent ICP to the neuron externally…"). Not gated in UI.
  - Auto-Stake Maturity "Enable"/"Disable" → `setAutoStakeMaturity` (PermissionGate `AutoStakeMaturity`).
- **Shared building blocks**: `encodeIcrcAccount`, `icrc1Transfer`.
- **Redesign notes**: auto-stake maturity lives in Stake tab but its state is also shown in Maturity tab; "Refresh Stake" chore (Chores tab) automates the manual button — cross-link missing.

## ICP Staking Bot — Maturity tab
- **Routes & params**: tab "Maturity"; visible with `StakeMaturity`, `MergeMaturity`, `DisburseMaturity`, `Spawn` or `ViewNeuron`.
- **Purpose**: use accumulated maturity.
- **Information shown**: Available Maturity, Staked Maturity, Auto-Stake On/Off; shared "Percentage of Maturity" input (default 100%).
- **Actions** (each in its own PermissionGate; disabled when maturity = 0):
  - "📈 Stake Maturity" (`stakeMaturity`, `StakeMaturity`) — "takes 7 days to become liquid".
  - "🔄 Merge into Stake" (`mergeMaturity`, `MergeMaturity`).
  - "Spawn New Neuron" → "🐣 Spawn Neuron" (`spawnMaturity`, `Spawn`), optional "Controller (optional, defaults to this canister)"; disabled if maturity < 1 ICP ("Need at least 1 ICP equivalent in maturity to spawn.").
  - "💸 Disburse Maturity" (`disburseMaturity`, `DisburseMaturity`), optional "Destination (optional, defaults to this canister)" principal; "Subject to a 7-day modulation period".
- **Gating/state**: no confirmations. Merge maturity is a deprecated NNS command (may fail on current NNS).
- **Redesign notes**: "Collect Maturity" chore (Chores tab) automates disburse-maturity with threshold/destination — same concept in two places. SNS Neuron page has its own Stake/Disburse Maturity UI.

## ICP Staking Bot — Following tab
- **Routes & params**: tab "Following"; visible with `ManageFollowees` or `ViewNeuron`; whole content in PermissionGate `ManageFollowees` (read-only users see it dimmed with "🔒 No permission").
- **Purpose**: set NNS followees per topic and keep the neuron active (periodic confirmation).
- **Information shown**: "Current Following" — per topic (name from hardcoded NNS_TOPICS 0–18) list of followees with known-neuron names "Name (id)"; activity card "Active neuron"/"Inactive neuron" + "{d} days, {h} hours to confirm" (deadline = `voting_power_refreshed_timestamp_seconds` + ~6 months; urgent < 30 days; "Inactive - confirmation needed!"). Topic checklist with descriptions and ⚠️ for critical topics (4 Governance, 14 SNS & Community Fund); warning that critical topics don't inherit from "All Topics"; preview text of replace/clear effect.
- **Actions**:
  - "✅ Confirm Following" (`confirmFollowing`) — only shown when the neuron already has followees.
  - Topic quick-select: "Select All", "⚠️ Critical Only", "Non-Critical Only", "Clear All"; per-topic checkboxes.
  - Followee picker: dropdown "+ Add known neuron..." (from `list_known_neurons`, sorted), "🗑️ Clear all followees"; free text "Selected neurons (or enter custom IDs, comma-separated)"; pills with "×".
  - Submit "✅ Set Following for N Topic(s)" / "🗑️ Clear Following for N Topic(s)" → `setFollowing` once per topic.
- **Gating/state**: default selected topic = 0 (All Topics); resets after submit.
- **Redesign notes**: "Confirm Following" chore automates the confirm button. Followee UI is entirely separate from the SNS Neuron page's followee editor (Neuron.jsx "Add Followees") — two different follow editors. Topic list is hardcoded and may drift from NNS.

## ICP Staking Bot — Dissolve tab
- **Routes & params**: tab "Dissolve"; visible with `ConfigureDissolveState` or `ViewNeuron`; actions in PermissionGate `ConfigureDissolveState`.
- **Purpose**: change lock state.
- **Information shown**: "Current Dissolve Delay" (y/d and days); preview "New total will be: …"; "✅ Neuron is fully dissolved and ready to disburse." when state 3.
- **Actions**: "Increase Dissolve Delay" → "Days to Add" → "+ Add Days" (`setDissolveDelay`, adds seconds; note says max 730 days total); "⏳ Start Dissolving" (enabled only when Locked); "⏹️ Stop Dissolving" (enabled only when Dissolving). No confirmations.
- **Redesign notes**: SNS equivalents in Neuron.jsx / TokenCard.jsx / LockDissolvePanel (Sneed Lock) — three dissolve UIs with different semantics (add-days vs set-timestamp).

## ICP Staking Bot — Disburse tab
- **Routes & params**: tab "Disburse"; visible with `Disburse` or `ViewNeuron`; PermissionGate `Disburse`.
- **Purpose**: withdraw stake of a dissolved neuron.
- **Information shown**: if not dissolved: "⚠️ Neuron must be fully dissolved before disbursing. Current state: …" + guidance (start dissolving / time remaining).
- **Actions**: "Amount (ICP) - leave empty to disburse all"; "Destination Account ID (hex) - leave empty to send to this canister" (64-hex legacy account id); "💸 Disburse" → `disburse` → "Disbursed! Block height: …". No confirmation.
- **Redesign notes**: default destination is the bot canister itself; funds then need the separate "Withdraw" (Info tab, `WithdrawFunds`) step to reach the user — two-step flow split across sections. Destination format (hex account id) differs from every other input on the page (principals).

## ICP Staking Bot — Hot Keys tab (NNS neuron hotkeys)
- **Routes & params**: tab "Hot Keys"; visible with `ManageNeuronHotkeys` or `ViewNeuron`; PermissionGate `ManageNeuronHotkeys`.
- **Purpose**: manage NNS-level hotkeys on the neuron (they can vote/follow, not disburse/dissolve).
- **Information shown**: "Current Hot Keys" list (principal text).
- **Actions**: "Remove" per hotkey (`removeHotKey`, no confirmation); "Add Hot Key" principal input → "➕ Add Hot Key" (`addHotKey`).
- **Redesign notes**: this is the only route to vote manually with a bot neuron — add your own principal as NNS hotkey, then vote from `/icp_proposal` (IcpHotkeyNeurons). Not explained/linked here. Terminology clash with "Botkeys" (bot permissions tab), whose API is literally `listHotkeyPrincipals`/`addHotkeyPermissions`.

## ICP Staking Bot — Advanced tab (split, merge, visibility)
- **Routes & params**: tab "Advanced"; visible with `Split`, `MergeNeurons`, `ManageVisibility` or `ViewNeuron`.
- **Purpose**: structural neuron operations.
- **Information shown**: Current Stake; merge prerequisites list (same controller, no hotkeys, not dissolving, not in Community Fund); visibility status card (🌐 Public / 🔒 Private with explanation).
- **Actions**:
  - "✂️ Split Neuron" → "Amount for New Neuron (ICP)" → "✂️ Split" (`splitNeuron`; disabled if stake < 2 ICP; min 1 ICP) (PermissionGate `Split`).
  - "🔗 Merge Neurons" → "Source Neuron ID (to merge FROM)" → "🔗 Merge" (`mergeNeurons`) (PermissionGate `MergeNeurons`).
  - "🌐 Make Public" / "🔒 Make Private" (`setVisibility` 2/1) (PermissionGate `ManageVisibility`), warning about public voting history.
- **Gating/state**: no confirmations for split/merge.
- **Redesign notes**: merge source is free-text although candidates are exactly the bot's other neurons (already listed in the selector). SNS Neuron page has its own Split UI.

## ICP Staking Bot — Neuron-section permission model (botkeys applied to neuron tabs)
- **Routes & params**: same page; permissions from `callerPermissions`; controllers and `FullPermissions` holders pass everything.
- **Purpose**: fine-grained delegation of neuron operations to other principals.
- **Information shown**: tabs hidden when the user has none of the tab's keys; sections inside a visible tab dimmed with "🔒 No permission" badge when the specific key is missing (`PermissionGate`).
- **Tab → keys** (`TAB_PERMISSIONS`): stake [StakeNeuron, AutoStakeMaturity, ViewNeuron]; maturity [StakeMaturity, MergeMaturity, DisburseMaturity, Spawn, ViewNeuron]; following [ManageFollowees, ViewNeuron]; dissolve [ConfigureDissolveState, ViewNeuron]; disburse [Disburse, ViewNeuron]; hotkeys [ManageNeuronHotkeys, ViewNeuron]; advanced [Split, MergeNeurons, ManageVisibility, ViewNeuron]. Create-neuron section: StakeNeuron.
- **Full permission catalog with labels/descriptions** (KNOWN_PERMISSION_LABELS/DESCRIPTIONS, unknown keys fall back to CamelCase split): Full Permissions, Manage Permissions, Configure Dissolve State, Vote, Make Proposal, Disburse, Split, Merge Maturity, Disburse Maturity, Stake Maturity, Manage Followees, Spawn, Manage NNS Hotkeys, Stake Neuron, Merge Neurons, Auto-Stake Maturity, Manage Visibility, Withdraw Funds, View Neuron, Manage Confirm Following / Refresh Stake / Collect Maturity / Distribute Funds (chores), Configure Collect Maturity, Configure Distribution, View Chores, Manage Events, View Events. Chore-tab visibility: `ViewChores` or any chore manage/configure key (`canManageAnyChore`, 3542); per-chore manage key mapped by chore type prefix (`choreManagePerm`).
- **Gating/state**: tab auto-resets to Overview if access is lost. Version-dependent: old bots without `callerPermissions` give botkey users no rights; old bots without botkey API show unsupported state.
- **Redesign notes**: permissions `Vote` and `MakeProposal` exist (grantable in Botkeys tab) but the page has **no voting or proposal UI** at all. Permission catalog is hardcoded here and separately in TradingBot.jsx (PERMISSION_LABELS) — candidate for a shared bot-permission registry.

## ICP Staking Bot — Handler → canister call reference (IcpNeuronManager.jsx 216-3015, plus inline calls in 3016-7333)

Actors used: `manager` = the user's staking bot (`declarations/sneed_icp_staking_bot`); `mgmt` = IC management canister `aaaaa-aa` (effectiveCanisterId = bot); `ledger` = ICP ledger `ryjl3-…` (or any ICRC-1 ledger); `cmc` = Cycles Minting Canister; `factory` = Sneedapp (`declarations/sneedapp`); `gov` = NNS governance `rrkah-…` (mainnet, anonymous); naming = `utils/BackendUtils`.

| Handler / loader (line) | Calls | Triggered by (render line) | Notes |
|---|---|---|---|
| `fetchManagerData` (607) | `manager.getVersion`, `manager.getNeuronIds`, then `fetchNeuronInfo` + `fetchUserBalance` | on load; after every neuron action; "🔄 Refresh Data" (7835) | Sets page-wide `loading` → whole page shows "Loading manager..." spinner on every refresh. Detects invalid manager ("has no query method") and ViewNeuron denial (trap/assertion → empty neuron list). Always re-fetches info for `neurons[0]`, not the selected neuron (bug, see notes). |
| `fetchUserBalance` (719) | `ledger.icrc1_balance_of(user)` (ICP) | inside fetchManagerData, after stake/top-up | User's own wallet ICP balance (for stake/increase/top-up). |
| `fetchNeuronInfo` (1183) | `manager.getNeuronInfo(id)`, `manager.getFullNeuron(id)` | neuron select, fetchManagerData | Opt results. |
| `handleNeuronSelect` (1209) | → `fetchNeuronInfo` | neuron dropdown / list click (7444, 7477) | |
| `fetchKnownNeurons` (538) | `gov.list_known_neurons` (anonymous, always mainnet) | on load | Fallback map: 27 DFINITY Foundation, 28 Internet Computer Association. |
| `fetchCanisterStatus` (734) | `mgmt.canister_status` | on load; after controller update / upgrade / top-up | Only succeeds for controllers → cycles, memory, status, module hash, controllers; else null. |
| `fetchOfficialVersions` (778) | `factory.getAppVersions('sneed-icp-staking-bot')` → fallback `factory.getOfficialVersions()` | on load (also when logged out) | Effects derive `matchedOfficialVersion` (module hash == wasmHash), `latestOfficialVersion`, `nextAvailableVersion`. |
| `fetchConversionRate` (885) | `cmc.get_icp_xdr_conversion_rate` | on load | For cycles estimate in top-up. |
| `updateControllers` (936) | `mgmt.update_settings({controllers})` | via `handleAddController` (4192) / `handleRemoveController` (4113) | Add: rejects duplicates/invalid principal. Remove: refuses last controller; confirm via `confirmRemoveController` state (inline). |
| `handleUpgrade(version, mode)` (1034) | `fetch(wasmUrl)` → `verifyDownloadedWasm` (sha256 == wasmHash) → `mgmt.install_code({upgrade|reinstall}, emptyArgs)` | 3758, 3790, 3907, 3911, 3932, 4628 | Reinstall/unknown-WASM paths use `appConfirm`. Overrides displayed version after success (stale query cache). |
| `handleCyclesTopUp` (1123) | `topUpCanisterWithIcp({ledger, cmc})` (ICP→CMC subaccount + `notify_top_up`, pending top-up journal in utils/cyclesTopUp) | 4381 | Checks wallet balance ≥ amount + 0.0001 ICP fee. Success → "Top-Up Successful!" modal (2878-3010): "Cycles Added", "ICP Spent", "ICP ledger block", button "Done". |
| `loadChoreData(silent)` (1230) | `manager.getChoreStatuses`, `manager.getChoreConfigs`; per instance `getCollectMaturitySettings(iid)` / `getDistributionLists(iid)`; token USD via `priceService.getTokenUSDPrice` | on load (eager, for header lamps); Chores tab; after every chore action | Silent on older bots without chores API. Adaptive polling: 5 s while a conductor is non-Idle; until next scheduled run +3 s (cap 60 s); 8 s if overdue; 60 s idle. 1 s countdown tick if a chore fires within 5 min or is running. |
| `loadLogData(filter, silent)` (1334) | `manager.getLogs(filter)`, `manager.getLogConfig` | Log tab; filter/paging (6926-7256); auto-refresh every 5 s when toggled | Filter: minLevel, source, caller, fromTime, toTime, startId, limit (50), beforeId (bots < 0.9.5 ignore). |
| `loadHotkeyPermissions` (1448) | `manager.listHotkeyPrincipals`, `manager.listPermissionTypes` | Botkeys tab | Sets `botkeysSupported=false` on old bots. |
| `fetchUserPermissions` (1488) | `manager.callerPermissions` | on load | Builds the caller's permission Set; old bots → empty set. |
| `handleAddHotkeyPrincipal` (1598) | `manager.addHotkeyPermissions(principal, perms[])` | 4948 | ≥1 permission required. |
| `handleUpdateHotkeyPermissions` (1637) | diff → `addHotkeyPermissions` + `removeHotkeyPermissions` | 4843 | |
| `handleRemoveHotkeyPrincipal` (1694) | `manager.removeHotkeyPrincipal` | 4763 | Confirm via `confirmRemoveHotkey` state. |
| `handleStakeNeuron` (1722) | `manager.generateMemo` → `manager.getStakeAccount(memo)` → `journalStake` → `icrc1Transfer` (ledger, user → governance neuron subaccount, fee 0.0001) → `manager.claimNeuronFromDeposit(memo, delaySeconds)` | "🚀 Create Neuron" (7575) | Min 1 ICP; balance check; journal entry kept on unknown outcome / claim failure (PendingStakesNotice). |
| `handleSetDissolveDelay` (1829) | `manager.setDissolveDelay(id, additionalSeconds)` | "+ Add Days" (7888) | Adds to current delay. |
| `handleStartDissolving` / `handleStopDissolving` (1869/1901) | `manager.startDissolving` / `manager.stopDissolving` | 7909 / 7921 | |
| `handleAddHotKey` / `handleRemoveHotKey` (1933/1971) | `manager.addHotKey(id, p)` / `manager.removeHotKey(id, p)` | 8977 / 8947 | NNS neuron hotkeys (not botkeys). |
| `handleDisburse` (2003) | `manager.disburse(id, amount?, toAccountHex?)` | "💸 Disburse" (8728) | Empty amount = all; empty dest = bot canister; dest = 64-hex AccountIdentifier. |
| `handleSetFollowing` (2052) | `manager.setFollowing(id, topic, followees[])` once per selected topic (sequential loop) | 8645 | Empty followees = clear. Per-topic error aggregation. |
| `handleConfirmFollowing` (2123) | `manager.confirmFollowing(id)` | "✅ Confirm Following" (8399) | |
| `handleIncreaseStake` (2156) | `manager.getNeuronAccount(id)` → `icrc1Transfer` (user → neuron account) → `manager.refreshStakeFromDeposit(id)` | "➕ Add to Stake" (8062) | Not journaled (re-doable via Refresh). Waits 2 s before reload. |
| `handleManualRefreshStake` (2243) | `manager.refreshStake(id)` | "🔄 Refresh Stake from NNS" (8079) | |
| `handleToggleAutoStakeMaturity` (2282) | `manager.setAutoStakeMaturity(id, bool)` | "Enable"/"Disable" (8128/8139) | |
| `handleSpawnMaturity` (2315) | `manager.spawnMaturity(id, pct, controller?)` | "🐣 Spawn Neuron" (8273) | |
| `handleStakeMaturity` (2357) | `manager.stakeMaturity(id, pct)` | "📈 Stake Maturity" (8221) | |
| `handleMergeMaturity` (2395) | `manager.mergeMaturity(id, pct)` | "🔄 Merge into Stake" (8241) | |
| `handleDisburseMaturity` (2433) | `manager.disburseMaturity(id, pct, {owner}?)` | "💸 Disburse Maturity" (8310) | Destination principal only (no subaccount). |
| `handleSplitNeuron` (2487) | `manager.splitNeuron(id, amountE8s)` | "✂️ Split" (8784) | Min 1 ICP. |
| `handleMergeNeurons` (2526) | `manager.mergeNeurons(targetId, {id: source})` | "🔗 Merge" (8830) | Source typed by hand. |
| `handleSetVisibility` (2565) | `manager.setVisibility(id, 1|2)` | "🌐 Make Public"/"🔒 Make Private" (8888/8900) | |
| `fetchWithdrawTokenBalance` (2601) | any ledger: `icrc1_balance_of(bot)`, `icrc1_symbol`, `icrc1_decimals`, `icrc1_fee` | when withdraw section expanded / token changes | |
| `handleWithdrawToken` (2642) | ICP: `manager.withdrawIcp(amount, {owner})`; other: `manager.withdrawToken(ledger, amount, {owner})` | 4557 (Info tab, inm-b) | Whitelisted token via TokenSelector or custom ledger id; destination principal only. |
| naming (inline) | `setPrincipalNickname` (3270), `setPrincipalNameFor` (3334) | Naming section (inm-b) | |
| Inline chore calls (inm-b) | `createChoreInstance` 5285, `setChoreNextRun` 5452, `startChore` 5574, `pauseChore` 5631, `resumeChore` 5666, `stopChore` 5701, `triggerChore` 5736, `scheduleStartChore` 5794, `setChoreInterval`+`setChoreMaxInterval` 5920, `setCollectMaturityThreshold` 6071, `setCollectMaturityDestination` 6168, `removeDistributionList` 6313, `updateDistributionList`/`addDistributionList` 6744/6747, `renameChoreInstance` 6794, `deleteChoreInstance` 6824 | Chores tab | appConfirm on remove list (6307), delete instance (6818). |
| Inline log calls (inm-b) | `setLogLevel` 7060, `setMaxLogEntries` 7094, `clearLogs` 7124 | Log tab | appConfirm on max entries (7088), clear (7118). |
| Events tab (inm-b) | `<BotEventPanel>` (7284) | | Self-contained. |
| Error mapping `handleOperationError` (2710) | — | all neuron actions | GovernanceError / InvalidOperation / NotAuthorized / NeuronNotFound → banner text. |

## Trading Bot — page shell (`/trading_bot/:canisterId`)
- **Routes & params**: `/trading_bot/:canisterId`; no id → "Trading Bot" / "No canister ID provided. Navigate to a specific trading bot from your wallet or the Sneedapp page." Hero is visible logged-out; everything else needs login ("🔐 Authentication Required" / "Please log in with Internet Identity to manage this trading bot."). Controller = principal in management `canister_status` controllers; botkey permissions are enforced by the canister (the page itself does almost no permission gating, see Trading Bot parts).
- **Purpose**: console for one user-owned trading bot (DCA, range trading, rebalancing, distribution, snapshots) on ICPSwap (KongSwap listed but shut down).
- **Information shown**: hero "Sneed Trading Bot" + `PrincipalDisplay` of the canister (name/nickname, copy).
- **Actions**:
  - "Setup Wizard" (login) → opens TradingBotWizard. Auto-opens when the bot's token registry is empty (`getTokenRegistry().length === 0`); in that case the page also silently scans all whitelisted tokens' balances on the bot (8 in parallel) and calls `bot.addToken` for each non-zero one.
  - "Learn how it works" → `/help/trading_bot`.
  - "Name this bot" (controller) / "Set nickname" (others) → "Name Your Trading Bot" card: "Private Nickname — only you can see this" → "Save" (`setPrincipalNickname`); "Public Name — visible to everyone" (controllers only) → "Save" (`setPrincipalNameFor`), warning "This name will be visible to anyone who views this canister across the app."; "Hide naming options".
- **Layout below the hero** (login): (1) shared `BotManagementPanel` ("Trading Bot" section: Info / Chores / Botkeys / Events; Log tab hidden via `hideLogTab`), configured with `permissionLabels`, `multiInstanceChoreTypes` ['trade','move-funds','distribute-funds','rebalance','snapshot'], `preferredChoreTypeOrder` ['rebalance','trade','move-funds','distribute-funds','snapshot'], `renderChoreConfig` = trading chore config tabs, `renderChoreSummaryExtra` = Cumulative Progress card, `renderSwapCard` = live swap card, `extraInfoContent` = DEX settings, `cbEvents` (last 50 circuit-breaker events, for lamps); (2) `TradingBotLogs` second tab strip (Chores, Wallet [default], Performance, Logs, Circuit Breaker, Quick Trade, Recovery, Script); (3) the wizard modal.
- **Loads on entry**: `getTokenRegistry`, `listChoreInstances([])`, management `canister_status` (controllers), `getCircuitBreakerLog({limit 50})` (guarded; older bots lack it).
- **Entry points & cross-links**: Wallet Apps tab, /apps, DappCardModal, BotHealthDialog ("Log" / "View chores ›" add `?tab=log` / `?tab=chores`, which this page ignores; the upper Log tab is hidden here and logs live in the lower "Logs" tab), UpgradeBotsDialog, Sneedapp mint success.
- **Shared building blocks**: `BotManagementPanel`, `BotIcon`, `PrincipalDisplay`, naming utils (same nickname/public-name pattern as IcpNeuronManager and other principal pages), `useWhitelistTokens`.
- **Gating/state**: three separately-created bot actors (shell/wizard, BotManagementPanel, TradingBotLogs) each with their own agent; no shared state between the upper and lower consoles except via the `navigateToChore` ref.
- **Redesign notes**: two stacked tab systems on one page (upper generic console, lower trading console) with "Chores" in both; naming UI duplicated per page type; deep-link query params unsupported.

## Trading Bot — Setup Wizard (TradingBotWizard)
- **Routes & params**: modal on `/trading_bot/:canisterId` (login). Opened by "Setup Wizard" or automatically when the registry is empty. Mascot greeting differs for new vs existing bots.
- **Purpose**: guided creation of a complete strategy (tokens + chore + actions/targets + purse + funding + start) in one deploy.
- **Information shown**: "Choose a Strategy" cards: "DCA (Dollar Cost Averaging)", "Range Trade" ("…optional stop loss"), "Self-Balancing Portfolio", "AI Script (SneedScript)" ("Describe what you want in plain English to your favorite AI… Supports everything.").
- **DCA** (steps 'Tokens', 'Schedule', 'Fund & Deploy'):
  - "Select Token Pair" (Spend → Buy, `TokenSelector`).
  - "DCA Schedule": toggle "Amount to spend" / "Amount to buy" (trade size denominated in input or output token), "Amount per trade (SYM)" / "Amount to buy (SYM)", "Interval (minutes)" (default 60), sentence preview, "Risk Tolerances" collapsible ("Max slippage (%)" default 2, "Max price impact (%)" default 3), "Budget limit (SYM, optional)" (→ `maxCumulativeInput`), "Fallback Route Token" (shown when neither token is ICP; default ICP; "Route: A → X → B"), checkbox "Only trade within a price range (optional)" with "Min price"/"Max price" (input per output).
  - "Fund & Deploy": "From Wallet" / "From Main Purse" (disabled when main purse empty) with balance, "Fund bot with SYM (optional)", options "Isolate funds with own purse" (default on) and "Start chore immediately after deploy" (default on), "Summary" rows, "Deploy DCA Bot".
- **Range Trade** (steps 'Tokens', 'Ranges', 'Fund & Deploy'): token A/B; sell-A price range "Min price"/"Max price" (in B) and sell-B range; "Trade size per execution" per side (sell amount or buy amount); "Check interval (minutes)" (default 5); fallback route token; "Enable Stop Loss" → "Sell all of:" A|B, "If price drops below (B)", stop-loss "Max slippage (%)"/"Max price impact (%)" (default 100/100, "High tolerances ensure the stop loss executes…"); fund token + amount; same options; "Deploy Range Bot". Deploys up to 3 trade actions (stop loss with `haltChoreAfterExecution: true`, sell-A, sell-B).
- **Self-Balancing Portfolio** (steps 'Allocations', 'Settings', 'Fund & Deploy'): "Portfolio Allocations" rows Token + "Target %" (≥2 tokens, must total 100%); "Rebalancer Settings": "Rebalance threshold (%)" (default 5), "Check interval (minutes)" (default 60), "Denomination token (for valuation)" (default ICP), risk tolerances; funding; "Deploy Portfolio".
- **AI Script** (steps 'Guide', 'Fund', 'Export', 'Script', 'Review'): "Copy LLM Guide" (instructs pasting into ChatGPT/Claude/Gemini); optional "Send" tokens from wallet to the bot ("Next"/"Skip"); "Export & Copy State" (serialized SneedScript of current bot state) + "Copy to Clipboard"; paste AI output → `DSLContextBar` (copy extra context) → "Parse & Review"; operation review with per-op checkboxes → "Execute N Operations"; back/reset; "Done".
- **Actions / canister calls** (deploy): `addToken` (each token), `createChoreInstance('trade'|'rebalance', '<type>-<base36 time>', label)` (labels "DCA A → B", "Range A/B", portfolio label), `addTradeAction` (DCA/Range) or `setRebalanceTargets` + `setRebalanceDenominationToken` + `setRebalanceThresholdBps` + `setRebalanceMaxSlippageBps` + `setRebalanceMaxPriceImpactBps` (rebalance), `setChoreInterval`, `setTradeFallbackRouteTokens`, `enablePurse`, wallet → bot ICRC-1 transfer (from wallet funding), `fundPurse`, `startChore`. Progress text per step ("Registering tokens...", "Creating DCA trade chore...", "Funding bot from wallet...", "Starting chore..."). Non-atomic: a failure mid-way leaves a partially configured chore. Fees: ledger fee per transfer. No confirmation dialog beyond the summary.
- **Entry points & cross-links**: on success "Done" → `onClose(true, choreTypeId, instanceId)` → page calls `BotManagementPanel.navigateToChore` to open the new chore in the upper console (AI Script returns no chore id, so no navigation).
- **Shared building blocks**: `TokenSelector`, `useDSLEngine`/`DSLScriptEditor`/`DSLOperationReview`/`DSLContextBar`/`DSLCopyGuideButton`/`DSLExportButton` (from TradingBotDSLPanel), `icrc1Transfer`, token metadata cache.
- **Gating/state**: no permission pre-check; amounts converted with token decimals from ledger metadata/cache.
- **Redesign notes**: one of three editors for the same strategy config (wizard, per-chore config tabs, Script DSL); wizard can only create, never edit; "Move funds", "Distribute" and "Snapshot" chores have no wizard path except AI Script.

## Trading Bot — Script tab (SneedScript DSL console, TradingBotDSLPanel)
- **Routes & params**: `/trading_bot/:canisterId` → lower tab bar → "Script" (login; canister enforces permissions per call).
- **Purpose**: export the whole bot configuration as text, edit it (by hand or with an AI), and apply the diff as reviewed canister calls.
- **Information shown**: header "SneedScript DSL" / "Export bot state, write configuration programs, and apply changes."; editor (monospace, Tab inserts spaces, placeholder example); parse/resolve errors; review list grouped by category ('Token Registry', 'Global Settings', 'Chore Management', 'Trade / Move-Funds Actions', 'Rebalance Configuration', 'Distribution Lists', 'Circuit Breakers', 'Purse Management', 'Event System', 'Logging', 'Quick Trades / Transfers'), "N operations to execute", no-op ops shown separately, per-op result after execution, "Execution Complete — N succeeded, M failed".
- **Actions**:
  - "Export State" → `serializeBotState(bot)` (reads registry, chores, actions, rebalance, distributions, CB rules, purses, events, logging, DEX, staleness…) into the editor and copies to clipboard; exported text includes `#` comment lines with read-only stats (balances, execution counts, prices).
  - "Copy LLM Guide" → `generateLLMGuide({whitelistTokens, snsData})` (language reference + safety checklist + token directory with [SNS] tags).
  - "LLM Context — copy supplementary data for your AI" bar: "Copy All Tokens", "Copy Trade Log (50)", "Copy Bot Log (50)", "Copy CB Log (20)".
  - "Parse & Resolve" → `parseDSL` (tokenizer + parser) → `resolveOperations(ast, bot)` (reads live state, diffs, emits ordered `call` / `no-op` operations).
  - Toggle individual operations; "Execute N Operations" → calls each enabled `bot[method](...args)` sequentially (no confirmation dialog; stops nothing on failure, continues to next); "Back to Editor".
- **Entry points & cross-links**: last tab of the lower strip; same engine embedded in the wizard's "AI Script" scenario; LLM guide tells the AI to ask the user for "Copy Trade Log"/"Copy Bot Log"/"Copy CB Log" from this bar or from the Trade Log / Bot Log / CB panels.
- **Shared building blocks**: `dsl/` (parser, resolver, serializer, context-formatters, llm-guide), `useWhitelistTokens`/SNS data for the guide.
- **Gating/state**: operations whose method is missing on an old bot fail at execution ("Unknown method").
- **Redesign notes**: the most complete editor of bot config (covers everything except botkeys, controllers, cycles, upgrades); power-user feature buried as the 8th tab; non-transactional execution.

## Trading DSL (SneedScript) — what it can express
- **Literals**: integers (`10_000`), decimals (converted with token decimals), amounts (`1.5 ICP`), basis points (`100 bps`), durations (`300s`, `5m`, `1h`), booleans, strings, `none`, lists, token refs by symbol or ledger principal.
- **Statements**: `ensure <entity> <key> [in <scope>] { props }` (idempotent upsert), `remove <entity> <key> [from <scope>]`, `set <target> <property>: <value>`, `submit <action> { props }` (imperative, not idempotent), `reset_stats action "<key>" in "<chore>"`, `#` comments.
- **Entities / settings**:
  - `token` (ledger, symbol, decimals, fee); `set token X paused|frozen: bool`.
  - `chore` (type trade | rebalance | move-funds | distribute-funds | snapshot, label); `set chore` interval, max_interval (random range), task_timeout, status running|paused|stopped.
  - `action` in a trade or move-funds chore: type trade | fund_purse | reclaim | send; enabled; input/output; min/max amount; amount_mode range|percent + balance_percent; preferred_dex; source/target purse; destination; min/max balance + denomination; min/max price + denomination; max_price_impact; max_slippage; min/max frequency; trade_size_denomination; trailing_stop (bps, stop_loss|take_profit, reset on_exec|never); halt_chore_after; max_cumulative_input/output; max_executions.
  - `rebalance_target TOKEN in "<rebal>"` (target bps, paused); `set rebalance` denomination, max/min trade size, threshold, max_slippage, max_price_impact, fallback_route_tokens.
  - `distribution "<name>" in "<dist>"` (token, threshold, min/max amount, amount_mode, balance_percent, source_purse, targets of `{account|purse, share}`).
  - `circuit_breaker "<name>"` with `when ALL|ANY { … }` (nestable) over conditions `price A/B >|< amt`, `price A/B changed up|down|either <bps> in <dur>`, `balance TOKEN in <purse> >|< amt`, `value <ALL|TOKEN> in <main|"chore"> denominated_in TOKEN >|< amt` or `changed …`; `then { stop|pause|start chore "id" | all chores | all TYPE chores; pause token X [in "rebal"]; freeze token X }`; `set circuit_breaker_enabled`.
  - DEX: `set dex ICPSwap|KongSwap enabled`, `set default_slippage`, `set default_max_price_impact`.
  - Purses: `set purse "<chore>" enabled`, `set purse "<chore>" trading_purse: "<other>"|none`.
  - Events: `ensure event_subscription to "<canister>" { event_types }`, `ensure event_reaction "<name>" { enabled, subscription, event_type, action, action_params, conditions, cooldown }`, `set event_emission`.
  - Logging: `set logging trade_log|portfolio_log|max_trade_log_entries|max_portfolio_log_entries`, `set chore_logging "<chore>" …`, `remove chore_logging`.
  - Engine: `set price_staleness`, `set metadata_staleness`, `set price_history_max_size`; `set trade "<chore>" fallback_route_tokens`.
  - Imperative: `submit trade {input, output, amount, min_output, slippage, max_price_impact, dex, source_purse}`, `submit withdraw {token, to, amount}`, `submit send {token, to, amount, source_purse}`, `submit fund_purse`, `submit reclaim`.
- **Resolution**: resolver reads live state (registry, chore instances/statuses, trade & move-funds actions, rebalance targets, distribution lists, CB rules/enabled, paused/frozen tokens, purse allocations, DEX list, logging/overrides, staleness, event subscriptions/reactions) and emits calls such as `addToken`/`removeToken`, `createChoreInstance`/`renameChoreInstance`/`deleteChoreInstance`, `setChoreInterval`/`setChoreMaxInterval`/`setChoreTaskTimeout`, `start|stop|pause|resumeChore`, `add|update|removeTradeAction`, `add|update|removeMoveFundsAction`, `resetActionStats`, `setRebalance*`, `add|update|removeDistributionList`, `add|update|removeCircuitBreakerRule`, `setCircuitBreakerEnabled`, `pauseToken` etc., `setDexEnabled`, `setDefaultSlippage`/`setDefaultMaxPriceImpact`, `setTradingPurseId`, `fundPurse`/`reclaimFromPurse`, `manualSend`, `withdrawToken`, `submitOneOffTrade`, `add|update|removeEventSubscription`, `add|update|removeEventReaction`, `setEventEmissionEnabled`, `setLoggingSettings`, `set|removeChoreLoggingOverride`, `setPriceStaleness`/`setMetadataStaleness`/`setPriceHistoryMaxSize`, `setTradeFallbackRouteTokens`; unchanged `ensure`s become no-ops; chore `status` changes are ordered last (after configuration and funding).
- **LLM guide also prescribes**: ck-token mapping (USD→ckUSDC, BTC→ckBTC…), feasibility checks (main purse = balance − allocations), slippage/impact/interval sanity thresholds, suggest a circuit breaker, give every new chore its own purse and fund it, start new chores, naming conventions for instance ids/labels/action keys.
- **Not expressible**: botkeys/permissions, controllers, cycles, upgrades, naming, Recovery (ICPSwap stuck funds), snapshot chore config (none exists).

## Shared bot console (BotManagementPanel) — section header and tab bar
- **Routes & params**: component in components/BotManagementPanel.jsx; mounted only by `/trading_bot/:canisterId` (designed for "ICP Staking Bot, Trading Bot, etc." but IcpNeuronManager does not use it). Login required (host page gates). Props/extension points: `canisterId`, `createBotActor`, `botName`, `botIcon`, `appId` (factory version lookup), `permissionLabels`/`permissionDescriptions`, `multiInstanceChoreTypes`, `preferredChoreTypeOrder`, `renderChoreConfig` (per-chore UI, receives `controlTabContent` lamp/schedule data), `renderChoreSummaryExtra`, `renderSwapCard`, `extraInfoContent` (Info tab slot), `cbEvents` (circuit-breaker events for lamps), `hideLogTab`, imperative ref `navigateToChore(choreTypeId, instanceId)`.
- **Purpose**: generic console for any Sneedapp bot canister: canister ops, permissions, chore framework, log, events.
- **Information shown**: collapsible header (botIcon + botName), cycles badge "⚡ 1.20T" (controllers: `canister_status`; others: public `getCyclesBalance` via utils/botCycles, bots ≥ staking 0.9.5 / trading 0.9.4), one status lamp per chore instance (tooltip label), "Stop All" pill when any chore is enabled.
- **Tabs**: "Info" (always), "Chores" (with summary lamp; needs `ViewChores` or any `Manage…Chore…` permission), "Botkeys" (always), "Log" (needs `ViewLogs`; hidden when `hideLogTab`, which TradingBot sets), "Events" (needs `ViewEvents` or `ManageEvents`).
- **Actions**: expand/collapse; "Stop All" → `ConfirmDialog` "Stop All Chores" ("This will stop all running chores and clear their schedules…", "Stop All") → `bot.stopAllChores()`.
- **Loads on entry**: management `canister_status` (status, cycles, memory, module hash, controllers), `getVersion`, `callerPermissions`, CMC rate, user ICP balance, `getChoreStatuses` (+ `getChoreConfigs`, `getChoreTypes` if present); factory `getAppVersions(appId)` → fallback `getOfficialVersions()` (anonymous).
- **Shared building blocks**: `ChoreStatusLamp` helpers, `ConfirmDialog`, `appConfirm`, `PrincipalDisplay`/`PrincipalInput`, `TokenSelector`, `TokenIcon`, `PendingTopUpsNotice`, `BotEventPanel`, utils `cyclesTopUp`, `upgradePath.upgradeSteps`, `wasmIntegrity.verifyDownloadedWasm`, `botCycles`, `NeuronManagerSettings`.
- **Gating/state**: permission model: controller ⇒ everything; else `callerPermissions` set; `FullPermissions` ⇒ everything. Version-tolerant: missing methods (`getVersion`, `getChoreConfigs`, `getChoreTypes`, `getLogAlertSummary`, botkey methods) degrade silently.
- **Redesign notes**: this is the de-facto "bot console" pattern; only the trading bot uses it. Dismissed banners persist in localStorage `bot_dismissed_banners`.

## Shared bot console — Info tab (canister, version, upgrade, controllers, cycles, withdraw)
- **Routes & params**: BotManagementPanel → "Info". Logged-in viewers see it; upgrade/controller edits need controller; withdraw needs `WithdrawFunds`.
- **Purpose**: canister health, code verification and lifecycle, fuel, and pulling tokens out.
- **Information shown**: "Canister ID" (`PrincipalDisplay`), "Cycles", "Version:" (✓ when hash matches), "Status:" (running/stopping/stopped), "Memory: N MB", "Module Hash" with "✓ Official vX.Y.Z" or "⚠ Unverified" ("No module installed"), links "View Source →", "Download WASM →"; upgrade banner "vX.Y.Z is available" + "View release notes →"; "⚠️ Version Mismatch" (hash matches official vA but bot reports vB) or "⚠️ Unknown WASM — Proceed With Care" with module hash; "Latest official version: vX.Y.Z" + "View source code →"; non-controller notice "🔒 Upgrade/Reinstall Not Available".
- **Actions**:
  - Upgrade (controllers): "⬆️ Upgrade" → steps through every intermediate published version (`upgradeSteps`), each: fetch WASM from `wasmUrl`, verify sha256 against `wasmHash`, management `install_code({upgrade})`; success "✅ Successfully upgraded to vX (via …)"; partial failure reports the version reached. Unknown-WASM upgrade asks `appConfirm` first.
  - "🔄 Reinstall" → `appConfirm('⚠️ Reinstall will WIPE ALL CANISTER STATE…')` → `install_code({reinstall})` straight to the target.
  - "Controllers (N)" card (shown only when controllers are known, i.e. caller is a controller): list with a "you" badge, "Remove" → inline "Confirm?" / "⚠️ Remove yourself?" "Yes"/"No" → `update_settings(controllers)`; "Enter principal ID" (`PrincipalInput`) → "Add".
  - "Top Up Cycles" card (any logged-in user): "⛽ Add Cycles"/"Cancel", `PendingTopUpsNotice` (resume unfinished top-ups), "Your ICP Balance:", conversion rate, amount + "MAX", estimated cycles, "Top Up Canister" → `topUpCanisterWithIcp` (ICP → CMC subaccount + `notify_top_up`; 0.0001 ICP fee) → "⚡ Cycles Added!" dialog (cycles received, ICP spent, ledger block, "Close").
  - "💸 Withdraw Tokens" card (`WithdrawFunds`; "Withdraw ICP or any ICRC1 token from this canister", collapsed by default): "From list" (`TokenSelector`) / "Custom ledger" (principal input), bot balance, amount, destination "Principal ID", "💸 Withdraw" → `bot.withdrawIcp(amount, account)` for ICP else `bot.withdrawToken(ledger, amount, account)`; shows block height; "Fee: …". No confirmation.
  - `extraInfoContent` slot (Trading Bot: DEX settings).
- **Entry points & cross-links**: default tab. Release/source/WASM links go to external URLs from the factory registry.
- **Shared building blocks**: same logic exists in IcpNeuronManager (own copy), UpgradeBotsDialog (bulk upgrade), App Manager (/apps) and TopUpCyclesDialog (bulk top-up).
- **Gating/state**: "upgrade" vs "reinstall" semantics differ from IcpNeuronManager's copy (which jumps straight to latest).
- **Redesign notes**: Info mixes canister lifecycle, cycles, and a mini-wallet (withdraw); withdraw has no subaccount and only principal text input.

## Shared bot console — Botkeys tab
- **Routes & params**: BotManagementPanel → "Botkeys"; visible to all logged-in viewers; add/edit/remove need `ManagePermissions`.
- **Purpose**: grant other principals granular bot permissions without making them controllers.
- **Information shown**: explainer "Botkeys grant specific principals granular permissions to operate this bot canister without being a controller."; "Botkeys Not Available — This bot does not support botkeys." (old bots); "Botkey Principals (N)" list with `PrincipalDisplay` and permission badges (labels from `permissionLabels`, else CamelCase split); "No botkey principals configured. Add one below."
- **Actions**: "Edit" → checkbox per permission type (from `bot.listPermissionTypes()`) → "Save" (diff → `addHotkeyPermissions` / `removeHotkeyPermissions`) / "Cancel"; "Remove" → "Yes"/"No" → `removeHotkeyPrincipal`; add card: `PrincipalInput` "Enter principal ID or search by name", permission checkboxes with descriptions, "Select All", "Clear All", "Add Botkey Principal" → `addHotkeyPermissions(principal, perms)`.
- **Entry points & cross-links**: none outside the tab.
- **Shared building blocks**: `PrincipalInput`, `PrincipalDisplay`; permission vocabularies come from each page (Trading Bot list in the Trading Bot permission section; staking bot list in the IcpNeuronManager Botkeys section).
- **Gating/state**: list loads when tab opens (`listHotkeyPrincipals`).
- **Redesign notes**: API is named "hotkey" while UI says "Botkey"; on the staking bot page this sits next to NNS "Hot Keys" (different concept).

## Shared bot console — Chores tab (framework)
- **Routes & params**: BotManagementPanel → "Chores" (needs `ViewChores` or a chore-manage permission; actions enforced by canister).
- **Purpose**: list, schedule overview, and lifecycle of every chore instance; hosts bot-specific config via `renderChoreConfig`.
- **Information shown**:
  - Dismissible intro "Bot Chores are automated tasks that run on a schedule…".
  - "Schedule" strip: enabled chores sorted running-first then by next run, "running" / "in 4m 05s" / "overdue"; click opens that chore.
  - Chore-type tabs (from `getChoreTypes`, ordered by `preferredChoreTypeOrder`) with worst-instance lamp and instance count; instance sub-tabs with lamps.
  - Active instance: dismissible type description; compact status "Stopped | Paused | Running ● every 5 min(–10 min) ● N✓ M✗ ● <choreId> (click to copy)"; `renderChoreSummaryExtra` (Trading Bot: Cumulative Progress); last error (dismissible); then `renderChoreConfig` output (Trading Bot: Control/Actions/Purse/… tabs).
  - Empty: "No <Type> instances yet" + type description; "No chores configured for this bot."
- **Actions**: "Start" (`startChore`), "Pause" (`pauseChore`), "Resume" (`resumeChore`), "Stop" (`stopChore`), "Run Once" (stopped) / "Run Now" (`triggerChore`) — then 2 s status polling for up to 10 min while running, plus 3 s auto-refresh while any chore runs and a 1 s countdown tick; "+" (multi-instance types) → "New <Type> Instance" name → "Create" (`createChoreInstance(type, '<type>-<base36 time>', label)`) / "Cancel"; "+ Create <Type> Instance" from empty state; double-click an instance tab to rename → "OK" (`renameChoreInstance`) / "Cancel" / Enter / Escape. Interval, schedule, delete and per-chore settings are NOT in the framework — they come from `renderChoreConfig` (Trading Bot Control tab).
- **Entry points & cross-links**: `navigateToChore` (used after the Setup Wizard and from the lower "Chores" overview); header lamps.
- **Shared building blocks**: `ChoreStatusLamp` (scheduler/conductor/task lamps, CB-aware), run tracker (conductor start, current task, completed tasks).
- **Gating/state**: create/rename buttons shown regardless of permission.
- **Redesign notes**: hidden rename (double-click); the same Start/Pause/Resume/Stop/Run buttons repeat in the Trading Bot Control tab, Wallet Apps tab, DappCardModal and IcpNeuronManager's own chores UI.

## Shared bot console — Log tab (generic bot log)
- **Routes & params**: BotManagementPanel → "Log" (needs `ViewLogs`; settings need `ManageLogs`). Hidden on the Trading Bot (it uses its own Bot Log panel in the lower "Logs" tab), so in practice unused today.
- **Purpose**: browse the bot's activity log (API calls, chore actions, permission changes, errors).
- **Information shown**: "Currently storing N of M max entries"; unseen-alerts banner "N unseen errors, M unseen warnings" (from `getLogAlertSummary(lastSeen)`, lastSeen = max(localStorage `lastSeenLogId:<canister>`, backend `get_last_seen_log_id`)); entries newest-first: LEVEL, source, enhanced message (DEX ids → names, bps → %, raw amounts → token units), time, #id, rich tags (token pair with icons, amounts, DEX), caller; "Showing N of M matching · …".
- **Actions**: "Show" (filter to alert level), "Mark as seen" (`app_sneeddao_backend.mark_logs_seen` + localStorage); level threshold buttons Error/Warning/Info/Debug/Trace ("+" marks threshold); source buttons api/permissions/chore/system/log; "Refresh"; "Auto-refresh" (5 s); "Per page:" 10/25/50/100/200; "Write level:" Off…Trace (`setLogLevel`, ManageLogs); "Clear Logs" (`appConfirm('Clear all log entries?')` → `clearLogs`); paging "« Newest", "‹ Newer", "Older »".
- **Shared building blocks**: botLogIdl (header dialogs read logs with the same filter), token metadata cache.
- **Redesign notes**: three bot-log viewers exist (this, Trading Bot `BotLogPanel`, IcpNeuronManager Log tab) plus BotHealthDialog's alert list; "seen" state is split between localStorage and the backend canister.

## Shared bot event system (BotEventPanel) — Events tab on both bots
- **Routes & params**: component components/BotEventPanel.jsx, mounted as "Events" tab by BotManagementPanel (Trading Bot) and directly by IcpNeuronManager. Needs `ViewEvents` (view) or `ManageEvents` (edit; view also). Without either: "You need the ViewEvents or ManageEvents permission to access event settings." Creates its own actor from `utils/botEventIdl` (works for any bot implementing BotEventEngine).
- **Purpose**: bot-to-bot automation: a bot emits typed events; another bot (or itself) subscribes and runs reaction actions when events match conditions.
- **Information shown**:
  - Sub-tab "Source (Outgoing)": "Event Emission" Enabled/Disabled; "Registered Listeners" table (Listener principal, Event Types, Registered, Status Active/Disabled); "Event Types" table (Name, Data Keys with hints); "Event Log" (last 50: Event Type, Time, Data).
  - Sub-tab "Listener (Incoming)": "Event Subscriptions" table (Source Bot, Event Types, Reg. ID, Status); "Reaction Rules" list ("Event: …", "Action: …"); "Available Actions" / "Available Reaction Actions" list (id, name, params or "no params"); "Reaction Log" (Rule, Event, Action, Source, Time, Result with error tooltip).
  - Event catalog known to the UI: chore lifecycle (0–6, incl. error), distribution (10–11), staking-bot neuron events (100–161: stake/dissolve/delay/maturity/following/vote/hotkey/withdraw), trading (200–270: TradeExecuted/Skipped/Failed, CircuitBreakerTriggered/Enabled/Disabled, token pause/freeze, RebalanceExecuted/Skipped, PurseFunded/Reclaimed, SendExecuted/Failed, Inflow/OutflowDetected, OvercommitDetected, SnapshotTaken, CumulativeLimitReached). Reaction action ids: chore control (0–5), staking actions (100–106), trading actions (200–208: token pause/freeze toggles, fund purse, reclaim, send) — names come from the bot.
- **Actions** (ManageEvents): "Enable"/"Disable" emission (`setEventEmissionEnabled`); "Refresh"; listener "Remove" (`unregisterEventListener`); "Add Subscription": Source Bot "This bot (self)", known bots from WalletContext, or `PrincipalInput` "Or search / paste another bot's canister ID..." → fetches the source's `getEventTypes` → "Select events to listen to" → "Subscribe to N events" (`addEventSubscription`); subscription "Edit" → "Edit Subscribed Event Types" (All/None) → "Save Changes" (`updateEventSubscription`) / "Cancel"; "Remove" (`removeEventSubscription`); "+ Add Rule" → "New Reaction Rule"/"Edit Reaction Rule": Name ("e.g. Dissolve on CB trigger"), Subscription, Event Type, Reaction Action, "Cooldown (seconds, optional)", Enabled, "Action Parameters" ("+ Add"; chore/purse pickers, token selector, amount, destination), "Conditions (all must match)" ("+ Add"; data key, Equals/Not Equals/Contains/Greater Than/Less Than, value) → "Add Rule"/"Update" (`addEventReaction`/`updateEventReaction`); rule "Edit"/"Remove" (`removeEventReaction`). No confirmations on removes (ConfirmDialog imported but unused).
- **Entry points & cross-links**: "Events" tab of both bot pages; DSL `event_subscription` / `event_reaction` / `event_emission` statements edit the same data.
- **Shared building blocks**: `PrincipalInput`, `TokenSelector`, `PrincipalDisplay`, WalletContext `allBotEntries`/`appInfoMap` (known-bot list).
- **Gating/state**: version-dependent (older bots: "No event types found. This bot may not support the event system or hasn't been upgraded yet."). Source event types are fetched from mainnet (`icp-api.io`) regardless of environment.
- **Redesign notes**: genuinely shared and bot-agnostic — the one piece of the console both bots already share. Overlaps conceptually with the Trading Bot Circuit Breaker (condition → action rules) and with chores; three automation systems with different vocabularies.

## Trading Bot — lower console tab bar (`TradingBotLogs`)
- **Routes & params**: `/trading_bot/:canisterId`; rendered under `BotManagementPanel` whenever the user is logged in. No client-side controller or botkey check; each call is enforced by the bot canister (calls fail with an error message if the caller lacks the permission).
- **Purpose**: second tab strip on the page holding the trading-specific views (the upper `BotManagementPanel` holds Info / Botkeys / Chores framework).
- **Information shown**: tabs `Chores`, `Wallet` (default selected, although listed second), `Performance`, `Logs`, `Circuit Breaker`, `Quick Trade`, `Recovery`, `Script`.
- **Actions**: switch tab. On mount, loads `getChoreStatuses()` once (no refresh); this feeds the Chores overview, Performance purse labels, Circuit Breaker chore labels and Logging Settings overrides.
- **Entry points & cross-links**: page body; `Chores` cards call `botPanelRef.navigateToChore(typeId, instanceId)` and scroll to top, opening that chore in the upper BotManagementPanel.
- **Shared building blocks**: its own cached bot actor (`getReadyBotActor`), separate from BotManagementPanel's and the wizard's actor (three actors per page).
- **Gating/state**: tab bar wraps on narrow screens; nothing mobile-specific beyond that.
- **Redesign notes**: two stacked tab systems on one page (BotManagementPanel tabs above, this strip below), with chores in both: the overview here, the config up there. Chore statuses here are a one-time snapshot and go stale while the upper panel refreshes its own.

## Trading Bot — Chores overview (Chores tab)
- **Routes & params**: `/trading_bot/:canisterId` → lower tab `Chores`; login. Data needs `ViewChores` (canister-enforced).
- **Purpose**: one-glance status of every chore instance, grouped by type.
- **Information shown**:
  - Groups by chore type with icon and count: Trade, Rebalance, Move Funds, Distribute, Snapshot.
  - Per instance: summary lamp (`getChoreSummaryLamp`, which also takes recent circuit-breaker events into account), name (`instanceLabel`/`choreName`/id), state badge (`Stopped` / `Paused` / `Stopping` / `Running` / `Scheduled` / `Idle`).
  - Three mini lamps: `Sched` (scheduler), `Cond` (conductor), `Task`, each with a tooltip label.
  - Interval (⏱), next run (⏭, countdown if under 24h, else date), last completed run (✓, "Xs/m/h ago").
  - `Runs:` total count, `Failed:` count (if > 0), current `Task:` id while running.
  - Empty state: "No chore instances found."
- **Actions**: click a card to open that chore instance in the upper BotManagementPanel chore editor (scrolls to top). No start/stop here.
- **Entry points & cross-links**: lower tab; links into the upper Chores tab.
- **Shared building blocks**: `ChoreStatusLamp` (StatusLamp + `getChoreSummaryLamp`/`getSchedulerLampState`/`getConductorLampState`/`getTaskLampState`), the same lamp logic as the BotManagementPanel chore UI and the App Manager health lamps; `cbEvents` loaded by the page shell (`getCircuitBreakerLog`, last 50).
- **Gating/state**: read-only; data loaded once per page load.
- **Redesign notes**: a natural "bot dashboard" home card. It duplicates the chore list or status in BotManagementPanel's Chores tab; a redesign could merge them into one chore list with inline controls.

## Trading Bot — Wallet tab: token registry
- **Routes & params**: `/trading_bot/:canisterId` → bottom tab bar (TradingBotLogs) → "Wallet" (the **default** tab). Login required (page shows "Authentication Required" otherwise). No UI permission check; the bot canister enforces botkey permissions (relevant labels: `ManageTokenRegistry` "Manage Token Registry", `ViewPortfolio`).
- **Purpose**: Maintain the list of tokens the bot knows about and trades (order matters to other panels).
- **Information shown**:
  - "Registered Tokens (N)" as chips (symbol, falls back to cached metadata / short principal); chips marked paused/frozen.
  - Empty state: "No tokens registered. Add tokens or scan for tokens with balances."
  - Scan progress: "Scanning X/Y... Found Z so far."
- **Actions**:
  - "Refresh" → reloads registry, purse allocations, paused/frozen sets, ledger balances.
  - "Scan for Tokens" ("Scanning...") → for every app-whitelisted token not yet registered, reads `icrc1_balance_of(bot)` on its ledger (8 in parallel) and calls `bot.addToken({ledgerCanisterId, symbol, decimals, fee})` for each with balance > 0. Result: "Scan complete. Found N token(s) with balances." / "All whitelisted tokens are already registered."
  - "Manage" / "Hide" toggles edit mode: per-chip remove (`bot.removeToken`, no confirm); "Add Token" TokenSelector ("Search or paste ledger ID...", custom ledger IDs allowed) → `bot.addToken` on select, or "Add" button.
  - Drag-and-drop chip reorder → debounced (600 ms) `bot.reorderTokenRegistry(orderedPrincipals)`; failures only logged to console.
  - Also auto-registers tokens implicitly when funding from wallet (`ensureTokenRegistered` → `addToken`).
  - Page shell does the same silent whitelist scan in the background when the registry is empty (and opens the Setup Wizard).
- **Entry points & cross-links**: Default landing tab below the BotManagementPanel. Empty-balance text points users to "Register tokens in the Accounts panel" (that panel is not rendered anywhere).
- **Shared building blocks**: `TokenSelector`, `DraggableTokenChip` (react-dnd, `HTML5Backend`), `useWhitelistTokens`, `useTokenMetadata`/`getTokenMetadataSync` (site-wide token cache), `createLedgerActor`.
- **Gating/state**: `getTokenRegistry` / `getPausedTokens` / `getFrozenTokens` called only if the method exists (older bot versions). HTML5 drag backend → reorder probably does not work on touch devices.
- **Redesign notes**: Token registry is a bot-wide setting buried inside the "Wallet" tab behind "Manage"; the order is meaningful (used by other panels) but nothing explains it here. The Setup Wizard and the page shell also register tokens, so the same thing happens in three places.

## Trading Bot — Wallet tab: main account balances, valuation and per-token pause/freeze
- **Routes & params**: same tab. Login. Canister-side permission (pause/freeze, presumably token-registry/circuit-breaker class permissions; not checked in UI).
- **Purpose**: See what the bot holds on its main account, its value, and stop specific tokens from being traded/moved.
- **Information shown**:
  - "Receiving Addresses": Principal, ICRC-1 Account (encoded), Account ID (legacy, "for receiving ICP from centralized exchanges only"), each with a copy button.
  - "Main Account — Token Balances" table: Token (icon, symbol, FROZEN/PAUSED badge), Balance (read directly from each ledger for the bot's default subaccount), and when a denomination is chosen: Price (in denom), Value, % of total; "Total" row with total value; pie chart "Main Account" of value share by token.
  - Lists every registered token (even zero balance) plus any unregistered token that shows a balance.
  - Legend: "Paused — not traded by any chore", "Frozen — not traded or moved".
- **Actions**:
  - "Value in:" TokenSelector (default ckUSDC; any token; clear "×") → prices computed in the browser: token→ICP via `priceService` (ckUSDC/ckUSDT/ckEURC/… use fixed fiat pegs), divided by the denom token's ICP price.
  - Per-token Status buttons: pause/unpause (`bot.pauseToken` / `bot.unpauseToken`; "paused — it will not be traded by any chore") and freeze/unfreeze (`bot.freezeToken` / `bot.unfreezeToken`; "frozen — it will not be traded or moved by any chore"). No confirm. Labels are icon-only with tooltips ("Pause — prevent trading by all chores", "Freeze — prevent trading and all movement").
- **Entry points & cross-links**: Wallet tab. Same receiving-address trio as the user wallet's receive flow.
- **Shared building blocks**: `PieChart` (file-local), `TokenIcon`, `TokenSelector`, `fetchDenomPrices`, `priceService`, `encodeIcrcAccount`/`computeAccountId`.
- **Gating/state**: Pause/freeze sets only loaded if the bot has `getPausedTokens`/`getFrozenTokens`; the buttons always render. Tables scroll sideways on narrow screens.
- **Redesign notes**: Pause/freeze are bot-wide risk controls (close to the Circuit Breaker tab) but sit as tiny icon buttons in a balance table. The per-panel "Value in" denomination differs from the app-wide denomination (`useDenomination`) that Quick Trade uses.

## Trading Bot — Wallet tab: fund / withdraw / send from the main account
- **Routes & params**: same tab. Login. Fund = any logged-in user with the token; Withdraw/Send are canister-enforced (`WithdrawFunds` "Withdraw Funds").
- **Purpose**: Move tokens between the user's wallet, the bot's main account and outside addresses.
- **Information shown**: Form title "Fund from Wallet (to Main Account)" / "Withdraw to Your Wallet (from Main Account)" / "Send to External Account (from Main Account)"; Amount label shows "(wallet: X)" or "(bal: X)"; "From your wallet: <principal>" or "Destination: <principal>".
- **Actions** (toggle buttons "Fund", "Withdraw", "Send"; then the submit button with the same label, "Executing...", plus "Cancel"; "Max" = balance − registry fee):
  - "Fund": token picker allows any token or a custom ledger ID; the token is auto-registered on the bot (`addToken`), then the user's own ICRC-1 transfer to the bot principal (ledger fee charged to the user). Success shows the block index.
  - "Withdraw": registered or held tokens → `bot.withdrawToken(token, amount, {owner: caller})`; shows transfer block height.
  - "Send": adds "Destination Principal" and "Subaccount (optional, hex)" → `bot.manualSend(token, [] /*main*/, dest, subaccount, amount)`.
  - No confirmation dialog for any of the three.
- **Entry points & cross-links**: Wallet tab; the Setup Wizard also funds the bot.
- **Shared building blocks**: `icrc1Transfer` (utils/icrcTx), `createLedgerActor`, `TokenSelector`, `inputTextToRaw`/`rawToInputText`.
- **Gating/state**: Submit disabled until token + amount (+ destination for Send) are filled. Amount must be > 0.
- **Redesign notes**: The same three moves (fund, reclaim, to wallet) exist at main-account level here, per purse in "Purse Allocations", and again in each chore's Purse tab. The main-account "Withdraw" and purse "To Wallet" use different canister methods (`withdrawToken` vs `manualSend`).

## Trading Bot — Wallet tab: main purse and purse allocations (all chores)
- **Routes & params**: same tab. Login. Canister-enforced `ManagePurses` / `WithdrawFunds`.
- **Purpose**: See how the bot's funds are split between the shared main purse and per-chore purses, and move funds between them.
- **Information shown**:
  - "Main Purse" ("Unallocated funds available for chores without a dedicated purse, or to fund chore purses."): Token, Available (on-chain − total allocated), Allocated, On-chain; "overcommitted" flag when allocations exceed the on-chain balance.
  - "Purse Allocations": one card per purse-enabled chore with a balance, titled with the chore instance label; table Token, Balance, Price/Value/% and Total (same denomination as above).
  - Empty: "No chores have funded purses yet. Enable a purse in a chore's settings to isolate its balances."
- **Actions**:
  - "Reclaim All to Main" ("Reclaiming...") → `appConfirm("Reclaim all balances from N purse(s) (M token balance(s)) back to main purse?")` → `bot.reclaimFromPurse(instanceId, token, fullBalance)` for every non-zero balance.
  - Per chore: "Fund" (source select "Main Purse" | "My Wallet"; token; amount; "Max"; "Fund"/"Funding..."; shows the source balance). From wallet = user ICRC-1 transfer to the bot + `addToken`, then `bot.fundPurse(instanceId, token, amount)`; from main = `fundPurse` only.
  - "Reclaim" (back to main purse) → `bot.reclaimFromPurse`; "Max" = full purse balance.
  - "To Wallet" ("Sending...") → `bot.manualSend(token, [instanceId], caller, [], amount)`; "Max" = balance − fee.
- **Entry points & cross-links**: Wallet tab; the empty-state text points to each chore's settings (Purse tab).
- **Shared building blocks**: Same fund/reclaim/withdraw handlers as `PursePanel` (copied code), `appConfirm`.
- **Gating/state**: Main-purse figures are computed in the browser from ledger balances minus `getAllPurseAllocations()`.
- **Redesign notes**: A purse is a virtual allocation on the main account, not a separate account. Users need one "funds map" view (main purse vs purses vs wallet). Today this is split across the Wallet tab, each chore's Purse tab and Quick Trade's "Source Purse".

## Trading Bot — Performance (P&L, equity curve, capital flows)
- **Routes & params**: lower tab `Performance`; login. Canister permission is probably `ViewPortfolio`.
- **Purpose**: portfolio performance analytics for the whole bot account or one chore purse.
- **Information shown**:
  - `Scope:` selector (only if any purse is enabled): `Whole Account` or one button per enabled chore purse (named after the chore instance).
  - Whole-account cards:
    - `Portfolio Value` (ICP and USD, from the latest "After" snapshot, with carry-forward of tokens missing from sparse snapshots).
    - `Net Capital Deployed` (ICP and USD, from `getCapitalFlows`).
    - `Trading P&L` (value minus capital, absolute and %, ICP and USD).
  - Purse-scope cards: `Purse Value`, `First Snapshot Value`, `Purse Change` (absolute and %).
  - `Equity Curve` chart (title is `Purse Value` in purse scope):
    - toggles `Detailed` (area chart over snapshots) or `Daily OHLC` (open/close lines plus high-low bars from daily portfolio summaries; whole-account scope only);
    - denomination toggle `ICP` / `USD`;
    - when there are no purse snapshots the purse view is "(estimated from account data)": the account snapshot is scaled by the purse-to-account balance ratio.
  - Empty states: no snapshots yet; "At least 2 snapshots are needed…"; "No daily summaries yet…".
  - `Purse Token Holdings` table (purse scope): Token, Balance, Value (ICP), Value (USD). Source is the latest purse snapshot with its timestamp, or "Current purse balances" as a fallback.
  - `Capital Flows by Token` table (whole account): Token, Total Inflow, Total Outflow, Net (native units).
  - Price History section (see next capability).
- **Actions**:
  - Edit icon (title "Adjust capital deployed") on the Net Capital card opens inputs `ICP delta` / `USD delta` (negative reduces) → `Apply` → `bot.adjustCapitalDeployed(icpE8s, usdE8s)`; `Cancel`. No confirmation dialog.
  - Scope, view and denomination toggles (client-side, except selecting a purse, which loads `getPursePortfolioSnapshots(purseId, …)`).
- **Canister calls (load)**:
  - `getPortfolioSnapshots` (newest 500 "After" snapshots via `fetchNewestSnapshots` + `getPortfolioSnapshotStats().nextId`);
  - `getCapitalFlows`, `getTokenRegistry`, `getLastKnownPrices`, `getAllPurseAllocations`;
  - `getPriceHistory` (newest 5000), `getDailyPortfolioSummaries` (1000), `getDailyPriceCandles` (1000).
- **Entry points & cross-links**: lower tab only. No refresh button: data reloads only on tab re-mount or after Adjust.
- **Shared building blocks**: `utils/botPaging` (`BOT_PAGE`, `fetchNewestRows`, `fetchNewestSnapshots`, which handle old bots that paged oldest-first), recharts, token metadata cache (`getTokenMetadataSync`).
- **Gating/state**:
  - Version-dependent: price history, daily summaries, candles, purse allocations and the token registry are each fetched only if the method exists.
  - `getPortfolioSnapshots` and `getCapitalFlows` are not guarded, so an old bot shows "Failed to load performance data".
  - The purse scope appears only if `getAllPurseAllocations` returns enabled purses.
- **Redesign notes**:
  - "Capital deployed" is a manual accounting override hidden behind a small edit icon; it changes how P&L is computed and deserves explanation.
  - Price analytics are buried inside Performance.
  - Snapshot data is also browsable in Logs → `Portfolio Snapshots` (another part); performance and snapshots could sit together in an "Analytics" area.

## Trading Bot — Price history (inside Performance)
- **Routes & params**: lower tab `Performance`, bottom section; login.
- **Purpose**: show the prices the bot has seen (quotes it fetched during chore runs) and their history.
- **Information shown**:
  - `Last Known Prices` cards for each direct pair from `getLastKnownPrices`, excluding self-pairs. Each card: `BASE/QUOTE` rate and age ("Ns/m/h ago"). The default orientation puts ICP or ckUSDC as the quote.
  - Toggle `Show N derived pair(s)` / `Hide …`: cross-rates computed client-side through ICP (e.g. SNEED/ckUSDC from SNEED/ICP and ckUSDC/ICP), badge `derived`.
  - Chart for the selected pair:
    - title `X / Y`, badge `derived via ICP`, % change over the window, `H:` / `L:` / `N pts` stats;
    - `Detailed` view: area chart of "Quote Price" from `getPriceHistory`; derived pairs merge two series with carry-forward;
    - `Daily OHLC` view from `getDailyPriceCandles`: for derived pairs, high and low are best/worst-case cross rates.
  - Empty states: "No price data yet. Price history will populate after the bot fetches quotes during chore runs.", "No history for this pair yet.", "Not enough constituent pair history…", "No daily price candles for this pair yet."
- **Actions**:
  - Click a pair card to select it.
  - Flip icon (title "Flip pair direction") inverts that pair's display.
  - Switch `Detailed` / `Daily OHLC`.
  - All of this is client-side; there are no canister writes.
- **Entry points & cross-links**: only reachable inside Performance.
- **Shared building blocks**: same data feeds as Performance; the price feed is filled by the Snapshot chore ("Price Snapshots") and trade chores.
- **Gating/state**: needs `getLastKnownPrices` and `getPriceHistory` (version-dependent).
- **Redesign notes**: a market-data viewer buried at the bottom of Performance. It is the bot's own price oracle view and overlaps conceptually with app-wide token price displays (PriceService); it could be its own "Prices" view.

## Trading Bot — Logs tab (sub-tabs container)
- **Routes & params**: lower tab `Logs`; login.
- **Purpose**: groups the bot's record-keeping views.
- **Information shown**: sub-tabs `Trade Log` (default), `Portfolio Snapshots`, `Bot Log`, `Logging Settings`. Trade Log (`TradeLogViewer`), Portfolio Snapshots (`PortfolioSnapshotViewer`) and Logging Settings (`LoggingSettingsPanel`) live in lines 1-4277 and are covered by another part.
  - Logging Settings calls `get/setLoggingSettings`, `get/setChoreLoggingOverride(s)`, `get/setPriceStaleness`, `get/setMetadataStaleness` and `get/setPriceHistoryMaxSize`, so it also holds price and metadata staleness and price-history size, which are not logging.
- **Actions**: switch sub-tab.
- **Entry points & cross-links**: lower tab.
- **Shared building blocks**: none of its own.
- **Gating/state**: none.
- **Redesign notes**: "Logging Settings" mixes logging with price-cache tuning. The upper BotManagementPanel normally has a Log tab, but TradingBot hides it (`hideLogTab`) and re-implements it here as `BotLogPanel`.

## Trading Bot — Bot Log viewer (Logs → Bot Log)
- **Routes & params**: lower tab `Logs` → `Bot Log`; login. Canister permission `ViewLogs` to read, `ManageLogs` to change the level or clear.
- **Purpose**: browse the bot's general activity log: API calls, chore actions, permission changes, errors.
- **Information shown**:
  - Intro line with storage stats: "Currently storing N of M max entries at <Level> write level" (`getLogConfig`).
  - Entries, newest first:
    - level badge (Error/Warning/Info/Debug/Trace), source badge, message, time, `#id`;
    - messages are enriched: `dex N` becomes ICPSwap/KongSwap, `N bps` becomes %, raw amounts are converted using token decimals and symbol;
    - tag chips: token pair with icons (input → output), amounts in → out, DEX name, token tags with icon, bps tags as %, amount tags formatted, other `key: value`.
  - Footer: "Showing X of Y matching · Z total stored · Max: M".
  - Empty state: "No log entries found[ matching the current filters]."
- **Actions**:
  - `Show:` level buttons set a minimum-level threshold (shows `Level+`; click again to clear).
  - `Source:` filter buttons `api` / `permissions` / `chore` / `system` / `log`.
  - `Refresh`; `Auto-refresh` checkbox (5 s silent polling).
  - `Per page:` 10/25/50/100/200.
  - Paging `« Newest`, `‹ Newer`, `Older »` (by start id).
  - `Write level:` select `Off`/`Error`/`Warning`/`Info`/`Debug`/`Trace` → `setLogLevel` (no confirmation).
  - `Clear Logs` asks "Clear all log entries?" → `clearLogs()`.
  - `Copy` copies text lines `[ISO] [LEVEL] [source] message | k=v…`.
  - Dismiss on the status message.
- **Canister calls**: `getLogs(filter)` (minLevel, source, caller, from/to time, startId, limit, beforeId), `getLogConfig`, `setLogLevel`, `clearLogs`.
- **Entry points & cross-links**: Logs sub-tab only. `useBotLogNotification` and `BotLogAlertDialog` (shared, another part) alert on new errors and warnings app-wide.
- **Shared building blocks**: `useTokenMetadata`, `TokenIcon`, `CopyToClipboardButton`, `appConfirm`. It is functionally a fork of BotManagementPanel's Log tab, which the ICP Staking Bot page uses through its own built-in Log tab.
- **Gating/state**: the `caller` filter exists in the request but has no UI.
- **Redesign notes**:
  - There are three implementations of the "bot log" viewer: BotManagementPanel Log tab, this `BotLogPanel`, and IcpNeuronManager's own Log tab. A single shared "Bot console → Log" component is an obvious consolidation.
  - The write-level setting also appears in Logging Settings.

## Trading Bot — Trade Log (TradeLogViewer)
- **Routes & params**: TradingBotLogs tab "Logs" → sub-tab "Trade Log" (default sub-tab); login; canister-enforced ViewLogs/ViewPortfolio.
- **Purpose**: audit trail of every swap, purse funding/reclaim, send, and detected inflow/outflow.
- **Information shown**:
  - Header: "N entries", "Page X of Y".
  - Per entry: "#id", status badge "Success"/"Failed"/"Skipped" (or "Inflow Detected"/"Outflow Detected"), action type label, timestamp; "Sold:/Bought:" (swap) or "Funded:/To Purse:", "Reclaimed:/To Main:", "Sent:/To:"; inflow/outflow ± amount; "Price:" (input per output), "DEX:" (ICPSwap/KongSwap), "Impact:", "Chore:" (instance id), "Action: #n", "Error:".
  - Swaps: collapsible "Balance Snapshots" table Token / Before / After / Change / USD Value (before snapshot matched by chore + proximity, after by tradeLogId).
  - Empty: "No trade log entries yet. Entries are recorded when chores execute trades."
- **Actions**:
  - Filters: status (All/Success/Failed/Skipped), chore type (Trade/Rebalance/Move Funds/Distribute), action type (6 types) → "Filter".
  - Refresh icon; "Copy" (copies current page as plain-text log lines).
  - Pagination "First"/"Prev"/"Next"/"Last" (50 per page, offset-based).
  - Inflow/outflow entries: "Remove" → appConfirm "Delete entry #N and reverse its capital impact?" → `deleteTradeLogEntries([id])`.
  - "Cleanup Duplicates" → opens DuplicateCleanupDialog (below).
  - Reads: `getTradeLog(query)`, `getTradeLogStats()`, `getPortfolioSnapshots({fromTime,toTime})`.
- **Entry points & cross-links**: Logs tab; Performance tab (capital accounting) is affected by Remove/Cleanup.
- **Shared building blocks**: CopyToClipboardButton (also used in other trading-bot panels), TokenIcon, appConfirm/appAlert, DuplicateCleanupDialog, botPaging (via dialog).
- **Gating/state**: `getTradeLogStats` fetched but only used to gate the count label.
- **Redesign notes**: capital-correction tools (delete inflow, dedupe) are buried in the log viewer though they change Performance numbers; chore id shown raw instead of chore label; "snapshot" chore type not in filter.

## Trading Bot — Cleanup Duplicate Inflows dialog (DuplicateCleanupDialog)
- **Routes & params**: modal from Trade Log "Cleanup Duplicates"; login; canister-enforced.
- **Purpose**: find and delete duplicated "Detected Inflow" trade-log rows that inflate capital accounting.
- **Information shown**: "Scanning trade log for duplicate inflows..."; groups of inflows with identical token + amount (oldest first), each row tagged "Original"/"Duplicate" with timestamp/amount and checkbox; "No duplicate inflows detected. Your trade log looks clean."; result "Deleted N entries", "Capital reversed: X ICP, $Y USD".
- **Actions**: "Select duplicates", "Deselect all" (per group), per-row checkbox, "Delete Selected (N)" (pre-selects all but first per group) → `deleteTradeLogEntries(ids)`; "Cancel"/"Close". No second confirm.
- **Entry points & cross-links**: Trade Log only.
- **Shared building blocks**: `fetchAllTradeLog` (utils/botPaging, 500/page), useTokenCache, TokenIcon, portal modal.
- **Gating/state**: reads all inflows (paged) on open.
- **Redesign notes**: maintenance/repair tool; fits a "Recovery/Maintenance" section with the Recovery tab rather than inside logs.

## Trading Bot — Portfolio Snapshots (PortfolioSnapshotViewer)
- **Routes & params**: TradingBotLogs "Logs" → "Portfolio Snapshots"; login.
- **Purpose**: view balance snapshots the bot takes before/after trades (and by snapshot chores).
- **Information shown**: "N snapshots" (from `getPortfolioSnapshotStats().totalEntries`); list newest first of "Before / After" pairs (matched by chore + "Trade N" trigger prefix) with trigger text and time → expand: Token / Before / After / Change / USD Change; unpaired snapshots: "#id", phase badge (Before/After/Snapshot), trigger, time → expand: Token / Balance / ICP Value / USD Value. "More snapshots available..." when more than 100.
- **Actions**: refresh icon; click row to expand/collapse. Calls `getPortfolioSnapshots({limit:100, newestFirst:true})`.
- **Entry points & cross-links**: Logs tab; Trade Log embeds the same before/after data per swap.
- **Shared building blocks**: TokenIcon, useTokenCache.
- **Gating/state**: older bots ignore `newestFirst` and return oldest 100 (comment in code).
- **Redesign notes**: half-built paging — "More snapshots available..." has no load-more control; only latest 100 viewable. Overlaps Trade Log balance-snapshot expanders and Performance tab.

## Trading Bot — Bot & Logging Settings (LoggingSettingsPanel)
- **Routes & params**: TradingBotLogs "Logs" → "Logging Settings"; login; canister-enforced ManageLogs.
- **Purpose**: titled "Bot & Logging Settings": global bot data-freshness settings plus trade/portfolio logging switches.
- **Information shown**: "Token Metadata Staleness" (seconds + humanized; default 3600), "Price Staleness" (default 300s), "Price History Buffer Size" (entries; default 5000; ring buffer used by circuit breakers and price charts); "Trade Log" ON/OFF + "Max:" entries; "Portfolio Snapshots" ON/OFF + "Max:"; "Per-Chore Overrides" list (chore label, Trade: / Portfolio: = "Use Master"/ON/OFF).
- **Actions**:
  - "Save" → `setMetadataStaleness`, `setPriceStaleness`, `setPriceHistoryMaxSize` (non-negative ints).
  - ON/OFF toggles and "Max" "Save" (min 100) → `setLoggingSettings`.
  - Per-chore selects → `setChoreLoggingOverride(choreId, {tradeLogEnabled, portfolioLogEnabled})`.
  - Reads: `getLoggingSettings`, `getChoreLoggingOverrides`, `getMetadataStaleness`, `getPriceStaleness`, `getPriceHistoryMaxSize` (last three with method-existence fallbacks).
- **Entry points & cross-links**: Logs tab only; chore list from TradingBotLogs `getChoreStatuses`.
- **Shared building blocks**: none notable.
- **Gating/state**: version-dependent getters fall back to defaults on older bots.
- **Redesign notes**: bot-wide engine settings (staleness, price-history size) are hidden under Logs → Logging Settings; they belong in a bot "Settings" area (next to DEX Settings in Info tab). The generic bot log (level/max entries) is a separate "Bot Log" sub-tab (another part).

## Trading Bot — Circuit Breaker rules (safety automation)
- **Routes & params**: lower tab `Circuit Breaker`; login. Editing needs canister permission `ManageCircuitBreaker`.
- **Purpose**: define IF-conditions-THEN-actions safety rules that the bot evaluates and acts on automatically.
- **Information shown**:
  - Header `Circuit Breaker` with a global toggle showing `Enabled` / `Disabled` (`getCircuitBreakerEnabled`).
  - Rule list (`getCircuitBreakerRules`). Each rule card shows:
    - name and `Active`/`Disabled` badge;
    - a readable summary such as `IF … AND/OR … THEN …`, for example "PRICE SNEED/ICP drops ≥5.0% over 1 hour", "BALANCE ICP in Purse "X" < 10", "VALUE of [all tokens in account] in ICP outside [a .. b]";
    - actions such as "STOP ALL chores", "PAUSE ckBTC globally", "START all Trade chores".
  - Empty state: "No circuit breaker rules configured. Click "New Rule" to create one."
- **What a rule can express**:
  - Top-level combiner: `ALL (AND)` or `ANY (OR)` across conditions. All actions run when the rule fires ("all execute when conditions are met").
  - Condition types:
    - `Price`: token pair, Token 1 / Token 2; amounts in Token 2 units.
    - `Value`: one or more value sources, each `Specific token` (in an account), `All tokens in purse`, or `All tokens in account`; `Denomination (default: ICP)` is any token.
    - `Balance`: one token in an account.
    - `AND Group` / `OR Group`: nested child conditions (recursive tree, `Add Child`).
  - Account choices for Balance and Value: `Main account (full on-chain)`, `Main purse (minus allocations)`, or `Purse: <chore instance>` (enabled purses only).
  - Operators: `Greater than` / `Less than` (`Threshold`), `Inside range` / `Outside range` (`Min`/`Max`), `% Change`. The % Change operator takes `Change %` (stored in bps), `Direction` (`Up` / `Down` / `Either direction`) and `Over period` (value × `minutes`/`hours`/`days`).
  - Action types:
    - `Pause token in rebalance portfolio` (pick a token plus a rebalance chore);
    - `Pause token globally`, `Freeze token globally`;
    - `Stop chore` / `Pause chore` / `Start chore` (pick an instance);
    - `Stop all chores by type` / `Pause all chores by type` / `Start all chores by type` (types: Trade, Rebalance, Move Funds, Distribute Funds, Snapshot);
    - `Stop ALL chores` (default), `Pause ALL chores`, `Start ALL chores`.
- **Actions**:
  - Global toggle → `setCircuitBreakerEnabled(bool)`.
  - `New Rule` opens the editor with `Rule Name` (placeholder "e.g. BTC crash protection"; required, "Rule name is required") and an `Enabled` checkbox.
  - In the editor: `Add Condition`, `Add Action`, trash icons ("Remove condition" / "Remove action"), `Save Rule` (→ `addCircuitBreakerRule(input)` or `updateCircuitBreakerRule(id, input)`), `Cancel`.
  - Per rule: pause/play icon (Enable/Disable) → `enableCircuitBreakerRule(id, bool)`; edit icon (`Edit`); trash icon (`Delete`) asks "Delete this circuit breaker rule?" (app dialog) → `removeCircuitBreakerRule(id)`.
  - Refresh icon reloads everything.
  - Token pickers are `TokenSelector` with `allowCustom` (any ledger id).
- **Entry points & cross-links**:
  - Lower tab.
  - CB events also feed chore lamps in the Chores overview and in BotManagementPanel through the page shell's `cbEvents`.
  - The DSL `Script` tab can also express or export CB rules (see the DSL part).
- **Shared building blocks**: `TokenSelector`, `appConfirm` (`utils/appDialog`), `useWhitelistTokens` and the token registry for symbol resolution, chore instances from `listChoreInstances([])`, purses from `getAllPurseAllocations`.
- **Gating/state**:
  - The load calls `getCircuitBreakerRules`, `getCircuitBreakerEnabled`, `getCircuitBreakerLog`, `getTokenRegistry` and `listChoreInstances` without existence checks; on a bot version lacking CB the whole tab shows "Failed to load circuit breaker data".
  - The page shell does guard `getCircuitBreakerLog`.
  - `getAllPurseAllocations` is optional.
- **Redesign notes**:
  - A powerful, safety-critical feature placed as the 5th lower tab. It is effectively "alerts and automations" and could pair with the BotEventPanel event and reaction rules (another automation system on the same bot), which overlap conceptually.
  - Thresholds are entered in human token units and converted with the token's decimals; for Value conditions, denomination decimals.

## Trading Bot — Circuit Breaker event log
- **Routes & params**: lower tab `Circuit Breaker` → `Show Event Log (N)`; login.
- **Purpose**: audit trail of rule firings.
- **Information shown**: table `Time`, `Rule`, `Conditions` (summary text recorded by the bot), `Actions Taken`, newest first. Shows the last 50 events, with the total count in the button label. Empty state: "No circuit breaker events recorded."
- **Actions**:
  - `Show/Hide Event Log (N)`.
  - `Copy` copies plain-text lines `#id | ISO time | Rule: "…" | conditions | Actions: …`.
  - `Clear Log` asks "Clear the entire circuit breaker event log?" → `clearCircuitBreakerLog()`.
- **Entry points & cross-links**: inside the CB tab only.
- **Shared building blocks**: `CopyToClipboardButton` (local to TradingBot.jsx, used across its log views).
- **Gating/state**: only 50 events are fetched; there is no paging, although `totalCount` is known.
- **Redesign notes**: one of several separate log streams on this bot: Bot Log, Trade Log, Portfolio Snapshots, CB log, per-chore logs, Event log in BotEventPanel. A unified activity or audit view would help.

## Trading Bot — Quick Trade tab (one-off trades and queue)
- **Routes & params**: `/trading_bot/:canisterId` → "Quick Trade" tab. Login; canister-enforced `ExecuteOneOffTrade` "Execute Quick Trades".
- **Purpose**: Make the bot do a single swap now, using its own funds.
- **Information shown**:
  - Input/Output Token pickers, limited to the bot's token registry (as loaded at page mount), with a "Balance:" line from the selected source purse.
  - Input amount's value in the app-wide denomination (`useDenomination`, via `priceService` ICP→USD).
  - "Trade Queue" (newest first): status (Pending / Processing (pulsing) / Completed / Failed / Cancelled), "in amount in-symbol → out amount out-symbol", "from <purseId>", "via <DEX>", error.
- **Actions**:
  - Form fields: "Input Token", "Output Token" (must differ), "Input Amount", "Min Output (optional)", "Slippage (%)", "Max Impact (%)", "DEX" ("Auto (best quote)" | "ICPSwap"), "Source Purse" ("Main Purse" or any chore instance).
  - "Submit Trade" ("Submitting...") → `bot.submitOneOffTrade({inputToken, outputToken, inputAmount, minOutputAmount?, maxSlippageBps?, maxPriceImpactBps?, preferredDex?, sourcePurseId?})` → "Trade #N submitted". No confirm or quote preview.
  - Cancel (×) on Pending entries → `bot.cancelOneOffTrade(id)`.
  - "Clear history" (when finished entries exist) → `bot.clearOneOffTradeHistory()`.
- **Data/calls**: `getOneOffTradeQueue` (polls every 4 s while anything is pending or processing; refreshes balances when done); `getMainPurseBalances` / `getPurseBalances(purseId)`.
- **Entry points & cross-links**: Page tab only.
- **Shared building blocks**: `TokenSelector`, `priceService`, `useDenomination`, `formatTokenAmount`/`parseTokenAmount`.
- **Gating/state**: Fixed 2- and 4-column grids, so it is cramped on phones. The source-purse list comes from `listChoreInstances` at page load and is not limited to purse-enabled chores.
- **Redesign notes**:
  - Overlaps with the app's own swap features (wallet swap / Sneedex swap), but trades from the bot's funds.
  - The DEX dropdown lacks KongSwap although the bot and other panels support it (`DEX_LABELS`).
  - No quote preview, unlike the user-facing swap.

## Trading Bot — Recovery tab (ICPSwap stuck-funds recovery)
- **Routes & params**: `/trading_bot/:canisterId` → "Recovery" tab. Login; canister-enforced (recover call).
- **Purpose**: Find and recover tokens left in an ICPSwap pool after an interrupted swap.
- **Information shown**: "ICPSwap Pool Recovery" explainer; after the check: "Pool: SYM0/SYM1 (x% fee tier)", "Pool canister: <id>", table Token / "Deposited (unused)" / "In Subaccount" / "Total Recoverable"; "Stuck funds detected…" or "No stuck funds found in this pool."; footer "How funds can get stuck: …".
- **Actions**:
  - "Token A" / "Token B" (any tokens) → "Check Pool" ("Checking Pool...").
  - The browser does the check itself: ICPSwap factory `getPool` across fee tiers 0.3%, 0.05%, 1%, 3% (ICRC1 standard) → pool `metadata` → ledger decimals/symbol/fee → `pool.getUserUnusedBalance(bot)` → ledger balance of the pool subaccount derived from the bot principal.
  - "Recover Funds" ("Recovering...") → `bot.recoverPoolFunds(token0, token1)` → "Successfully recovered X and Y to main account!" (no confirm), then re-checks.
- **Entry points & cross-links**: Page tab only; no link from failed swap entries or the swap card.
- **Shared building blocks**: `createIcpSwapFactoryActor`, `createIcpSwapActor`, `principalToSubaccount`, `TokenSelector`, `TokenIcon` (passed `tokenId=` instead of `canisterId=`, so the icon falls back).
- **Gating/state**: ICPSwap only; nothing for KongSwap.
- **Redesign notes**: A troubleshooting tool given a top-level tab next to everyday tabs. It could instead be offered next to failed trades. Similar "unused balance / withdraw from pool" logic may exist in the wallet's ICPSwap liquidity area (check other slices).

## Trading Bot — per-chore config tabs (dispatcher)
- **Routes & params**: `/trading_bot/:canisterId` → BotManagementPanel "Chores" tab → pick a chore type/instance. Login. Canister-enforced per-chore permissions (`ManageTradeChore`, `ManageRebalanceChore`, `ManageMoveFundsChore`, `ManageDistributeFunds`, `ManageSnapshotChore`, `ManageRebalancer`, `ConfigureDistribution`, `ManageTrades`).
- **Purpose**: `renderTradingBotChoreConfig` is the trading-bot render prop that BotManagementPanel calls under each chore instance. It shows a tab strip (`ChoreConfigTabs`).
- **Information shown / tabs by chore type**:
  - `trade` ("Trade"): "Control" | "Actions" (`ActionListPanel` with `getTradeActions`/`addTradeAction`/`updateTradeAction`/`removeTradeAction`; allowed action types "Trade (Swap)", "Fund Purse", "Reclaim from Purse", "Send"; title "Trade Actions"; conditions such as balance thresholds and price ranges, plus frequency limits) | "Purse" | "Fallback" | "Logs".
  - `rebalance` ("Rebalance"): "Control" | "Config" (`RebalancerConfigPanel`: target portfolio) | "Purse" | "Logs".
  - `move-funds` ("Move Funds"): "Control" | "Actions" (`ActionListPanel` with `getMoveFundsActions`/`add…`/`update…`/`removeMoveFundsAction`; types "Fund Purse", "Reclaim from Purse", "Send"; title "Move Funds Actions") | "Logs". No Purse tab.
  - `distribute-funds` ("Distribute"): "Control" | "Config" (`DistributionConfigPanel`) | "Logs".
  - `snapshot` ("Snapshot"): "Control" | "Config" (`SnapshotChoreConfigPanel`, read-only text: "Balance Snapshots", "Price Snapshots", "Daily Archive"; "Pipeline: Metadata Refresh → Price Fetch → Balance Snapshots → Daily Archive") | "Logs".
  - Unknown chore type → nothing (BotManagementPanel default only).
  - The first tab ("Control") opens by default.
- **Actions**: Switch tabs. The actions themselves are in the sections below; ActionList, Rebalancer and Distribution panels are covered elsewhere.
- **Entry points & cross-links**: BotManagementPanel chore list; TradingBotWizard on deploy (`navigateToChore`); Chores-overview tab "navigate to chore".
- **Shared building blocks**: `ChoreConfigTabs` (supports a `badge`, unused here); the generic BotManagementPanel chore scaffolding.
- **Gating/state**: Multi-instance chore types: trade, move-funds, distribute-funds, rebalance, snapshot.
- **Redesign notes**: Each chore is a mini-app (Control + config + purse + logs) nested three levels deep (page → Chores tab → instance → sub-tab). Bot-wide tabs (Logs, Performance) repeat per-chore data (per-chore logs).

## Trading Bot — Chore types (multi-instance) and per-chore config tabs
- **Routes & params**: `/trading_bot/:canisterId`, login required (page shows "Authentication Required" otherwise); config editing permission-gated on canister side.
- **Purpose**: define which chore types the trading bot has and which config sub-tabs each instance gets (via `renderTradingBotChoreConfig`, rendered inside BotManagementPanel's Chores tab per instance).
- **Information shown**:
  - Multi-instance chore types: `trade`, `move-funds`, `distribute-funds`, `rebalance`, `snapshot` (preferred order in panel: rebalance, trade, move-funds, distribute-funds, snapshot).
  - Per-instance sub-tabs (ChoreConfigTabs):
    - trade: Control · Actions (ActionListPanel "Trade Actions") · Purse · Fallback · Logs
    - rebalance: Control · Config (RebalancerConfigPanel) · Purse · Logs
    - move-funds: Control · Actions (ActionListPanel "Move Funds Actions") · Logs
    - distribute-funds: Control · Config (DistributionConfigPanel) · Logs
    - snapshot: Control · Config (SnapshotChoreConfigPanel) · Logs
- **Actions**: see ActionListPanel / Rebalancer / Distribution sections below (Control, Purse, Fallback, Logs covered by another part).
- **Entry points & cross-links**: BotManagementPanel Chores tab → select chore instance; TradingBotWizard deploy → `navigateToChore`; ChoresOverviewPanel (TradingBotLogs "Chores" tab) navigates here.
- **Shared building blocks**: BotManagementPanel `renderChoreConfig` hook = the extension point that makes the shared bot console bot-specific.
- **Gating/state**: move-funds and distribute-funds have no Purse tab; snapshot has only Control/Config/Logs.
- **Redesign notes**: chore configuration lives three levels deep (page → Chores tab → instance → sub-tab). Trade and move-funds share one action editor with different allowed action types.

## Trading Bot — chore Control tab (status, run monitor, start/stop, schedule)
- **Routes & params**: chore instance → "Control". Login; canister-enforced chore-manage permission.
- **Purpose**: Run a chore and watch it run.
- **Information shown**:
  - "Status" grid: State (Stopped / Paused / Running); Scheduler, Conductor and Task lamps (idle/active/busy/error colours and label); Interval (or "min–max" when randomized); "Next Scheduled Run"; "Last Completed"; "Runs (Success / Fail)".
  - Conductor run card: "Conductor Running" / "Last Conductor Run" / "Activity" with start→end clock times. Conductor elapsed timer. Current task row, using readable task labels from `TASK_DISPLAY_RULES`: "Refreshing metadata", "Fetching prices", "Circuit breaker check", "Pre-trade snapshot", "Account snapshot", "Post-trade snapshot", "Pre-/Post-rebalance snapshot", "Pre-/Post-transfer snapshot", "Moving funds", "Pre-/Post-distribution snapshot", "Distributing funds", "Taking balance snapshot", "Archiving daily summary"; other tasks show the raw id. Activity slot: live swap card, "Initializing...", the current task, last task + "Done", or "Waiting for next run…". Collapsible "N completed task(s)" list with success/fail dot, label, duration, and the error in a tooltip.
- **Actions**:
  - "Start" (when stopped) → `bot.startChore`. The ▼ split button opens "Schedule first run at:" (datetime) → "Confirm" → `bot.scheduleStartChore(id, tsNanos)` (must be in the future) / "Cancel".
  - "Pause" → `bot.pauseChore`; "Resume" → `bot.resumeChore`; "Stop" → `bot.stopChore`.
  - "Run Once" (stopped) / "Run Now" (enabled) → `bot.triggerChore` (hidden while running).
  - "Delete" (only when stopped) → inline "Delete?" "Confirm"/"Cancel" → `bot.deleteChoreInstance`.
  - Next run "Set" (enabled only) → datetime → save → `bot.setChoreNextRun` ("Next run time updated.").
  - "Frequency: Every [n] minutes|hours|days" "Save" → `bot.setChoreInterval` + `bot.setChoreMaxInterval`. Rules: ≥ 1 min and ≤ 365 days; under 5 min asks `appConfirm("Intervals < 5 min may not give chores enough time. Continue?")` and shows an inline warning.
  - "Randomize interval..." link → "Max: [n] unit (clear to disable)" → randomized range (must be > base and ≤ 365 days); the link reads "Randomized range active (up to X) — edit" once set.
- **Entry points & cross-links**: Default sub-tab of every chore. BotManagementPanel's compact chore header above it also has Pause/Resume/Stop/Trigger buttons and the dismissable last-error banner, plus `ChoreCumulativeSummary`.
- **Shared building blocks**: The lamp/status data, polling (`startChorePolling`) and run tracker come from BotManagementPanel via `controlTabContent`; swap card from `renderSwapCard`; `appConfirm`.
- **Gating/state**: The polling run tracker comes from BotManagementPanel. Several controls are hidden until revealed (DOM `display` toggles): the schedule picker, the next-run picker, the randomize panel.
- **Redesign notes**: Start/Pause/Resume/Stop/Trigger appear twice (BotManagementPanel chore header and this Control tab). Useful controls are hard to find: scheduling a first run sits behind a small ▼, next-run "Set" is a tiny button, "Randomize interval..." is a dotted link. This Control tab is a good candidate for a shared "chore console" with the ICP Staking Bot.

## Trading Bot — live swap progress card
- **Routes & params**: Shown inside a chore's Control tab while a chore runs (trade and rebalance chores). Login.
- **Purpose**: Show a running or just-finished swap in real time.
- **Information shown**:
  - Status: "Swapping..." / "Rebalancing..." / Success ✓ / Failed ✗ / Skipped ⊘.
  - DEX badge (ICPSwap / KongSwap).
  - Token flow: input symbol and −amount (prefixed "~" when it is an estimate from the action's min/max midpoint) → output symbol and +amount.
  - Details: "Price" (output per input), "Impact" % (amber > 1%), "Slippage" % (amber > 2%), error message.
  - Compact states: "Preparing swap…", "Evaluating portfolio…", "Loading result…", and an info-only "Skipped" card with the reason.
- **Actions**: "×" dismisses the card for that run.
- **Data/calls**: Detects swap tasks by id (`^trade-action-` / `^rebalance-exec-`). While running: `bot.getTradeActions(choreId)` for an estimate, then polls `bot.getTradeLog({choreId, limit 3})` every 3 s. After the run: one fetch to get the result. Ignores trade-log entries older than the run start.
- **Entry points & cross-links**: `useSwapCardRenderer` is created in the page shell and passed to BotManagementPanel as `renderSwapCard`, which hands it back to the Control tab. It is not used elsewhere.
- **Shared building blocks**: `SwapProgressCard` (memoized), `TokenIcon`, token metadata cache.
- **Gating/state**: Needs `getTradeLog` and `getTradeActions` on the bot.
- **Redesign notes**: Similar in kind to swap progress in the app's own swap UI. A shared "swap result card" component across wallet swap, Sneedex swap and bots is possible.

## Trading Bot — Chore summary card: Cumulative Progress
- **Routes & params**: inside BotManagementPanel chore summary card (via `renderChoreSummaryExtra` → `ChoreCumulativeSummary`); login.
- **Purpose**: show progress of per-action budgets/limits on the chore card.
- **Information shown**: "Cumulative Progress" bars per action with limits: "Input X/Y SYM" (cumulative input spent vs max), "Output X/Y SYM" (received vs target), "Runs N/M" (executions vs max); "Action #id" label when >1; bar turns red at 100%.
- **Actions**: none (read-only). Calls `getTradeActions(choreId)` for trade and rebalance, `getMoveFundsActions(choreId)` for move-funds; refetches when run count changes.
- **Entry points & cross-links**: Chores tab summary card.
- **Shared building blocks**: `useTokenMetadata` (local hook over shared `useTokenCache`).
- **Gating/state**: hidden unless at least one action has maxCumulativeInput / maxCumulativeOutput / maxExecutions; method-existence check.
- **Redesign notes**: for rebalance it calls getTradeActions (rebalance chores have targets, not actions) — likely always empty.

## Trading Bot — Trade / Move Funds action list editor (ActionListPanel)
- **Routes & params**: chore instance → "Actions" sub-tab for `trade` and `move-funds` chores; login; canister-enforced ManageTrades / ManageMoveFundsChore-type permissions.
- **Purpose**: CRUD the ordered list of actions a trade or move-funds chore executes each run.
- **Information shown** (per action card):
  - Title `"key"` or `#id` — action type label: "Trade (Swap)", "Fund Purse", "Reclaim from Purse", "Send" (also "Detected Inflow"/"Detected Outflow" labels exist for log only); "Enabled"/"Disabled" badge.
  - Input token, Output token (swaps).
  - Amount: random range "Min:"/"Max:" (or "Min buy:"/"Max buy:" when sized by output), or "% of balance" with "Min cap"/"Max cap"; amounts in input token, in a denomination token ("X USD of ICP"), or output token; fiat/crypto signs ($, €, ₿, Ξ) for ckUSDC/ckUSDT/ckEURC/ckBTC/ckETH.
  - "Dest:" (send destination), "To Purse:", "From Purse:".
  - Conditions: "Min Bal:", "Max Bal:" (optionally denominated), "Min Price:", "Max Price:" (input/output or denom/output), "Max Impact:", "Max Slippage:".
  - Trailing stop: "Trailing stop loss/take profit: N% from peak/trough", current watermark "Peak:/Trough:", "Resets after trade"/"Never resets".
  - "Last run:" timestamp; "Halt chore after execution" flag.
  - Stats: "Input budget: spent/max", "Output target: received/max", "Executions: n/max" (or count).
  - Empty state "No actions configured yet."
- **Actions**:
  - "Add Action" → inline form → "Add Action" (calls `addTradeAction` / `addMoveFundsAction(instanceId, config)`); "Cancel".
  - "Edit" (inline form under card) → "Save Changes" (`updateTradeAction` / `updateMoveFundsAction(instanceId, id, config)`).
  - "Disable"/"Enable" toggle (update with full existing config, enabled flipped).
  - Trash icon remove (`removeTradeAction` / `removeMoveFundsAction`) — no confirmation.
  - "Reset Stats" (`resetActionStats(instanceId, actionId)`) when executions > 0.
  - On save: auto-registers input/output tokens in bot token registry (`addToken`, errors ignored).
  - Form fields: "Type" (trade chore: Trade/Fund Purse/Reclaim/Send; move-funds: Fund Purse/Reclaim/Send); "Input Token"/"Output Token" (TokenSelector, custom ledger allowed); "Amount Mode" = "Random in Range" | "% of Balance" (+ "Balance %" slider); "Min Amount"/"Max Amount" (or "(cap)"/"Buy Amount"); "Size by" = "Input (spend)" | "Output (buy)"; "Amount Denomination" token + "Clear (use native)"; Fund Purse: "Source Purse"/"Target Purse" (Main Purse or chore purse ids from `getAllPurseAllocations`); Reclaim: "Source Purse"; Send: "Source Purse (debit)" + "Destination Account" (PrincipalInput, principal or ICRC-1 account with subaccount); "Enabled" checkbox.
  - "Conditions (optional)": "Min/Max Input Balance", "Balance Denomination"; swaps only: "Price direction" toggle (⇆ A per B), "Min Price"/"Max Price", "Price Denomination", "Max Price Impact (%)", "Max Slippage (%)", "Trailing Stop" checkbox → Mode "Stop Loss"/"Take Profit", "Threshold (%)", "Reset after execution" Yes/No.
  - "Limits (optional)": "Halt chore after execution" (stop-loss/stop-buy semantics), "Max Cumulative Input" (budget), "Max Cumulative Output" (target), "Max Executions".
  - Validation: input token required; output token required for trades.
- **Entry points & cross-links**: chore instance sub-tab; same actions are expressible in the Script (DSL) tab and created by the Setup Wizard.
- **Shared building blocks**: TokenSelector, PrincipalInput, useTokenMetadata/useTokenCache, `encodeIcrcAccount`/`decodeIcrcAccount`, currency-sign formatters (`formatDenomAmount`, `denomLabel`, `denomPriceUnit`).
- **Gating/state**: purse selectors need `getAllPurseAllocations` (method-existence check). No client permission check.
- **Redesign notes**: very dense single form (≈30 fields); the "frequency limits" mentioned in the panel description have no form fields — `buildConfig` always sends `minFrequencySeconds/maxFrequencySeconds/preferredDex = []`, so saving an edit via this form silently clears frequency limits / preferred DEX set elsewhere (e.g., via DSL); toggle preserves them. Remove has no confirm. Candidate for a guided "strategy builder" (trade, DCA, stop-loss, take-profit presets).

## Trading Bot — Rebalancer configuration (RebalancerConfigPanel)
- **Routes & params**: chore instance (type `rebalance`) → "Config" sub-tab; login; canister-enforced ManageRebalancer.
- **Purpose**: set target portfolio allocation and rebalancer parameters; see live drift vs target.
- **Information shown**:
  - "Parameters": "Denomination Token", "Max Trade Size", "Min Trade Size" (in denom token), "Threshold (%)", "Max Price Impact (%)", "Max Slippage (%)".
  - "Fallback Route Tokens": ordered intermediary tokens for routing when direct swap lacks liquidity/has high impact ("Default: ICP only" when empty).
  - "Target Allocations (N tokens)": token, target %, "PAUSED" badge, total %.
  - "Portfolio Status" (auto-refresh every 30s): donut charts "Target" vs "Current"; table Token / Balance / Value (in denom) / Current % / Target % / Deviation (+over red / −under blue); "Total value"; balances from chore purse if purse enabled else main purse; prices computed in the frontend (PriceService, ICP-routed, fiat stablecoins pegged).
- **Actions**:
  - Denomination: TokenSelector "Change denomination token..." → `setRebalanceDenominationToken`.
  - "Set" buttons → `setRebalanceMaxTradeSize`, `setRebalanceMinTradeSize`, `setRebalanceThresholdBps`, `setRebalanceMaxPriceImpactBps`, `setRebalanceMaxSlippageBps`.
  - Fallback tokens: "Add fallback token...", ▲ "Move up", ▼ "Move down", trash "Remove" → `setRebalanceFallbackRouteTokens`.
  - Per-target pause/resume button (no edit mode) → `setRebalanceTargets`.
  - "Edit" targets → editor with live "Preview" donut + "Total: X%": per row TokenSelector, pause/unpause, trash, lock/unlock slider, linked % slider + text (other unlocked targets rebalance proportionally), drag-to-reorder; "Add Token", "Equal Split"; "Save" (disabled unless total = 100% and all tokens chosen) → `setRebalanceTargets` + auto `addToken` for each; "Cancel".
  - Reads: `getRebalanceSettings`, `getRebalanceTargets`, `isPurseEnabled`, `getPurseBalances` / `getMainPurseBalances`.
- **Entry points & cross-links**: chore instance; Setup Wizard (rebalance strategy); Script tab.
- **Shared building blocks**: PieChart (pure SVG donut, also used in AccountsPanel/WalletPanel), `fetchDenomPrices` (shared with AccountsPanel and WalletPanel), DraggableEditingTarget (react-dnd HTML5 backend), TokenSelector, TokenIcon, stable per-token colors.
- **Gating/state**: drag-reorder uses HTML5 DnD backend — does not work on touch/mobile; parameter inputs are uncontrolled DOM inputs read by id.
- **Redesign notes**: "Portfolio Status" duplicates information shown in Wallet/Performance tabs (balances, values, allocation); target order appears to carry no meaning beyond display. Paused targets exist in two places (list buttons and editor). Unused `canisterId` prop.

## Trading Bot — Distribution lists (DistributionConfigPanel)
- **Routes & params**: chore instance (type `distribute-funds`) → "Config" sub-tab; login; canister-enforced ConfigureDistribution.
- **Purpose**: percentage-based splitting and sending of a token from a purse to chore purses and/or external accounts.
- **Information shown**: per list: name, "#id", "Token:", "From:" (Main Purse or "Purse: <chore label>"), "Threshold:", "Amount:" (min–max random, or % of balance with max cap), "Targets: N"; "Details" expands targets (Purse: <chore label> or short principal, % or "Auto-split"); editor shows "Assigned: X% · Auto-split targets: N", "Total exceeds 100%!".
- **Actions**:
  - "Add Distribution List" → form: "Name", "Token Ledger" (TokenSelector), "Source Purse", "Threshold Amount", "Amount Mode" ("Random in Range" | "% of Balance"), "Balance %", "Min Amount", "Max Amount"/"Max Amount (cap)" → "Add List" (`addDistributionList`); "Cancel".
  - "Details"/"Collapse".
  - "Settings" → edit source purse, threshold, amount mode, %, min/max → "Save Settings" (`updateDistributionList`); "Cancel".
  - "Targets" → editor: row type "Purse" (select enabled chore purse) or "External" (PrincipalInput with subaccount), "%" (blank = auto-split), trash; "+ Purse Target", "+ External Target", "Save Targets" (`updateDistributionList`); "Cancel".
  - Trash remove list (`removeDistributionList`) — no confirmation.
  - Reads: `getDistributionLists`, `getAllPurseAllocations`, `getChoreStatuses` (for purse labels).
- **Entry points & cross-links**: chore instance; Script tab.
- **Shared building blocks**: TokenSelector, PrincipalInput, useTokenMetadata.
- **Gating/state**: purse targets only offer purses with `enabled` allocation; method-existence checks on purse/status calls.
- **Redesign notes**: threshold/min/max amounts are entered and displayed as raw integer base units (no decimal conversion) — inconsistent with the rest of the page and error-prone. Remove without confirm.

## Trading Bot — chore Purse tab (per-chore purse)
- **Routes & params**: chore instance → "Purse" (trade and rebalance chores only). Login; canister-enforced `ManagePurses` / `WithdrawFunds`.
- **Purpose**: Give a chore its own budget, or point it at another chore's budget.
- **Information shown**:
  - "Chore Purse" header with the enabled state.
  - "Trade from:" override, shown when other purse-enabled chores exist or an override is already set: "— Own purse (default) —" / "— Main purse (default) —" or another chore's label. When set: "Trades will use X's purse", that purse's balances, or "Referenced purse is empty".
  - When enabled: "Purse Balances" (Token, with "(sub N)" for subaccount allocations; Balance), or "Empty — fund this purse to begin trading"; "Main Purse (available to fund)" (Token, Available, "overcommitted" flag).
- **Actions**:
  - Toggle "Enabled"/"Disabled" → `bot.enablePurse` / `bot.disablePurse`. Disable may return Err; no confirm.
  - Refresh icon.
  - "Trade from:" select → `bot.setTradingPurseId(instanceId, [otherId] | [])` (cross-chore purse sharing).
  - "Fund from:" "Main Purse" | "My Wallet" + token + amount + "Max" + "Fund" → (wallet: `addToken` + user ICRC-1 transfer to the bot) + `bot.fundPurse`. Shows "Wallet/Main purse balance: X".
  - "Reclaim:" token + amount + "Max" + "Reclaim" → `bot.reclaimFromPurse` (back to main purse).
  - "Withdraw:" token + amount + "Max" (minus fee) + "To Wallet" → `bot.manualSend(token, [instanceId], caller, [], amount)`.
- **Entry points & cross-links**: Chore sub-tab; the Wallet tab's "Purse Allocations" offers the same moves.
- **Shared building blocks**: Same handlers and UI as WalletPanel purse cards; `TokenSelector`, `icrc1Transfer`.
- **Gating/state**: `getTradingPurseId` / `getChoreStatuses` called only if the bot has them; `getAllPurseAllocations` is required.
- **Redesign notes**: Duplicates the Wallet tab's purse section. The "Trade from" override is a powerful but hidden setting that only appears when other purses exist.

## Trading Bot — trade chore Fallback tab (fallback route tokens)
- **Routes & params**: trade chore instance → "Fallback". Login; canister-enforced trade-chore permission.
- **Purpose**: Choose the intermediate tokens used to route a swap when the direct pair lacks liquidity or has high price impact.
- **Information shown**: "Fallback Route Tokens" ordered list (1., 2., … with icon and symbol), or "Default: ICP only".
- **Actions**: "▲"/"▼" reorder, trash = remove, "Add fallback token..." TokenSelector (custom IDs allowed). Each change saves at once with `bot.setTradeFallbackRouteTokens(instanceId, tokens)`; loads via `getTradeFallbackRouteTokens`. Save errors are silently ignored.
- **Entry points & cross-links**: trade chores only (rebalance has no Fallback tab).
- **Shared building blocks**: `TokenSelector`, `TokenIcon`.
- **Gating/state**: If the load fails the list shows empty (older bots).
- **Redesign notes**: A routing setting that sits apart from DEX Settings (BotManagementPanel Info tab `DexSettingsPanel`) and Quick Trade's DEX choice. Routing and DEX settings could share one place.

## Trading Bot — Snapshot chore config (explainer)
- **Routes & params**: upper BotManagementPanel → Chores → a `snapshot` chore instance → config area (`renderTradingBotChoreConfig`, ~line 11829); login. Canister permission `ManageSnapshotChore` for chore control.
- **Purpose**: explains what the Snapshot chore does. It has no type-specific settings.
- **Information shown**:
  - Three features: `Balance Snapshots` ("Captures balances of all registered tokens in the main account."), `Price Snapshots` ("Fetches fresh quotes for all registered token pairs, updating price history and daily candles."), `Daily Archive` ("Finalizes the previous day's OHLC summaries…").
  - "Pipeline: Metadata Refresh → Price Fetch → Balance Snapshots → Daily Archive".
  - "Use the interval setting above to control how often snapshots are taken."
- **Actions**: none (interval and start/stop live in the generic chore controls above it).
- **Entry points & cross-links**: chore editor; it produces the data shown in Performance, Price history and Portfolio Snapshots.
- **Shared building blocks**: generic chore framework in BotManagementPanel.
- **Gating/state**: static.
- **Redesign notes**: this chore is the data source for all analytics; the link between "snapshot chore running" and "charts populated" is only implied by the empty-state texts.

## Trading Bot — chore Logs tab (per-chore trade and activity logs)
- **Routes & params**: any chore instance → "Logs". Login; canister-enforced `ViewLogs` "Read Logs".
- **Purpose**: History for one chore.
- **Information shown**:
  - "Trade Log" (default): #id, status badge (Success/Failed/Skipped), action type ("Trade (Swap)", "Fund Purse", "Reclaim from Purse", "Send", "Detected Inflow", "Detected Outflow"), date/time, input amount and token → output amount and token, error message.
  - "Activity Log": level icon (✗ ! ℹ ·), date/time, message (tags are not shown in the UI).
  - Empty states: "No trade log entries for this chore." / "No activity log entries for this chore."
- **Actions**:
  - Refresh.
  - "Copy" → plain-text export of the shown entries (activity lines include tags; trade lines: id, ISO time, action, status and detail).
  - "Load older": activity uses `fetchOlderLogs` (beforeId paging, cap 500); old bots fall back to a growing `limit`. Trade log uses offset paging, 25 per page.
- **Data/calls**: `bot.getLogs({source: ['chore:'+id], limit})`, `bot.getTradeLog({choreId})`.
- **Shared building blocks**: `CopyToClipboardButton` (file-local), `utils/botPaging.fetchOlderLogs`, `useTokenMetadata`.
- **Gating/state**: Version fallback for `beforeId` paging.
- **Redesign notes**: The bot-wide "Logs" tab (BotLogPanel / TradeLogViewer) shows the same streams with filters. These per-chore tabs are pre-filtered copies with less functionality. The copy export maps only action types 0–3 (inflow/outflow print as numbers).

## Trading Bot — DEX configuration (upper panel Info tab extra)
- **Routes & params**: `/trading_bot/:canisterId`, shown inside BotManagementPanel's Info tab via `extraInfoContent`; login. Canister permission `ManageDexSettings`.
- **Purpose**: choose which DEXes the bot may trade on.
- **Information shown**:
  - Title `DEX Configuration`, note "Enable or disable DEXes for trading. At least one DEX must remain active."
  - Per DEX from `getSupportedDexes()`: name, description, `Active`/`Disabled`, "(only active DEX)".
  - For KongSwap (id 1) while enabled, the warning "KongSwap has shut down. Turn it off so every trade goes through ICPSwap."
- **Actions**: toggle switch (title `Enable`/`Disable`, or "Cannot disable the only active DEX") → `setDexEnabled(dexId, bool)`. No confirmation. The button is not disabled for the last active DEX, so the canister has to refuse that.
- **Entry points & cross-links**: upper Info tab.
- **Shared building blocks**: its own actor; rendered through BotManagementPanel's `extraInfoContent` slot (the way the shared panel takes per-bot info).
- **Gating/state**: hidden entirely if `getSupportedDexes` fails or isn't loaded (returns null).
- **Redesign notes**:
  - Trading configuration sits under the generic "Info" tab and is easy to miss.
  - The permission description says "slippage, enabled DEXes", but only enable/disable is here (slippage lives in chore configs).
  - The KongSwap shutdown notice suggests this toggle is now mainly a one-time migration action.

## Trading Bot — botkey permission set (labels shown in Botkeys tab)
- **Routes & params**: `/trading_bot/:canisterId`; passed as `permissionLabels` / `permissionDescriptions` into the shared BotManagementPanel (Botkeys tab), which renders/edits them.
- **Purpose**: the trading bot's fine-grained permission vocabulary for botkey principals.
- **Information shown** (key → label — description):
  - FullPermissions → "Full Permissions" — all permissions incl. future ones
  - ManagePermissions → "Manage Permissions" — add/remove botkey principals and their permissions
  - ViewChores → "View Chores" — chore statuses, configs, settings
  - ViewLogs → "Read Logs" — read log entries, view log config
  - ManageLogs → "Manage Logs" — set log level, max entries, clear logs
  - ViewPortfolio → "View Portfolio" — balances, purses, portfolio state
  - ManagePurses → "Manage Purses" — enable/disable chore purses, fund and reclaim (key duplicated in the map; second description wins)
  - ManageTrades → "Manage Trades" — add/edit/remove trade chore actions
  - ManageRebalancer → "Manage Rebalancer" — rebalancer targets and parameters
  - ManageTradeChore / ManageRebalanceChore / ManageMoveFundsChore / ManageDistributeFunds / ManageSnapshotChore → "Manage … Chore" — start/stop/pause/resume/trigger that chore type
  - ManageTokenRegistry → "Manage Token Registry" — add/remove supported tokens
  - ManageDexSettings → "Manage DEX Settings" — slippage, enabled DEXes
  - WithdrawFunds → "Withdraw Funds" — send tokens to external accounts
  - ConfigureDistribution → "Configure Distribution" — add/edit/remove distribution lists
  - ManageCircuitBreaker → "Manage Circuit Breaker" — rules and conditions
  - ManageEvents / ViewEvents → "Manage Events" / "View Events" — event subscriptions, reaction rules, emission settings / view event types, listeners, logs
  - ExecuteOneOffTrade → "Execute Quick Trades" — submit, view, cancel one-off trades
- **Actions**: none here (granting happens in BotManagementPanel Botkeys tab).
- **Entry points & cross-links**: Botkeys tab of the shared bot console.
- **Shared building blocks**: same mechanism as ICP Staking Bot (which passes its own permission map); BotManagementPanel renders whatever map it gets.
- **Gating/state**: the trading-bot panels in this part do NOT check permissions client-side — every edit is attempted and the canister rejects unauthorized calls (error banner).
- **Redesign notes**: 23 permissions, mostly one per chore type + one per config area; a redesign could group them (View / Configure / Operate / Funds / Admin). Duplicate `ManagePurses` key is a code smell. UI does not hide controls a botkey user lacks permission for.

## Trading Bot — shared helpers inside TradingBot.jsx (lines 1-4277)
- **Purpose**: building blocks reused across trading-bot panels (relevant for a "bot console" design system).
- **Information shown / behavior**:
  - Currency signs: ckUSDC/ckUSDT → `$`, ckEURC → `€`, ckBTC → `₿`, ckETH → `Ξ`; fiat shows 2 decimals, crypto 6 sig. digits; others show "amount SYMBOL".
  - `fetchDenomPrices`: frontend-side prices of tokens in any denomination token via PriceService (ICP pivot; ckUSDC/ckUSDT pegged $1, ckEURC $1.08) — used by Rebalancer, AccountsPanel, WalletPanel.
  - `useTokenMetadata`: symbol/decimals/fee from site-wide token cache (IndexedDB) with on-demand fetch.
  - Exact amount helpers `rawToInputText` / `inputTextToRaw` (MAX buttons without float rounding).
  - CopyToClipboardButton ("Copy"/"Copied!").
  - PieChart (SVG donut with legend %, "N tokens"/"No data").
  - DraggableTokenChip (token chip with grip, frozen/paused status icons, remove ×; used in AccountsPanel/WalletPanel token lists) and DraggableEditingTarget (rebalance editor rows).
  - Management-canister `canister_status` IDL for controller detection (page shell).
- **Gating/state**: react-dnd HTML5 backend → no touch drag.
- **Redesign notes**: these are trading-bot-local copies (e.g., `shortPrincipal`, `formatTokenAmount` also re-implemented in DuplicateCleanupDialog); candidates for shared utils.

## Trading Bot — Accounts and subaccounts panel (`AccountsPanel`) — dead code
- **Routes & params**: none. Defined at line 4278 but never mounted; its mount was removed in commit 17352aa7 "trading_bot purses remove subaccounts", which replaced it with `WalletPanel`.
- **Purpose (as written)**: token registry plus per-subaccount balances plus manual fund movements.
- **Information shown (unreachable)**:
  - `Registered Tokens (N)` list with drag-to-reorder chips (`reorderTokenRegistry`) and scan progress.
  - Account switcher (Main Account / named subaccounts).
  - `Value in:` denomination TokenSelector (default ckUSDC).
  - `Receiving Addresses`: Principal, ICRC-1 Account, Subaccount hex, Account ID (with the CEX-only hint).
  - Pie chart and table: Token, Balance, Price, Value, %, Status (`FROZEN`/`PAUSED`).
- **Actions (unreachable)**:
  - Token registry: `Scan for Tokens` (whitelist scan → `addToken`), `Manage`/`Hide`, `Add` (`addToken`), remove chip (`removeToken`).
  - Token status: pause or unpause (`pauseToken`/`unpauseToken`), freeze or unfreeze (`freezeToken`/`unfreezeToken`).
  - Money movement:
    - `Fund` (ICRC-1 transfer from the user's wallet to the selected account);
    - `Withdraw` (`manualSend` to the user);
    - `Transfer to Subaccount` / `Transfer` (`manualTransfer` between own accounts);
    - `Send` (`manualSend` to any principal plus optional hex subaccount);
    - each with a `Max` button.
  - Subaccounts: `Create Subaccount` (`createSubaccount`), rename (`renameSubaccount`), delete with confirm "Delete subaccount #N?" (`deleteSubaccount`).
- **Redesign notes**: the bot API still has subaccounts (`getSubaccounts`, `createSubaccount`, `renameSubaccount`, `deleteSubaccount`, `manualTransfer`) with no live UI. Most other actions here survive in `WalletPanel` (which calls `addToken`, `removeToken`, `reorderTokenRegistry`, pause/freeze, `manualSend`, `withdrawToken`, `fundPurse`, `reclaimFromPurse`). Candidate for deletion, or a decision to expose subaccounts again.

## Trading Bot — Purse overview panel (`PursesPanel`) — dead code
- **Routes & params**: none. Defined at line 8416 and never mounted (also removed in 17352aa7); superseded by `WalletPanel`.
- **Purpose (as written)**: read-only allocation view.
- **Information shown (unreachable)**:
  - `Purse Overview` with `Refresh`.
  - `Main Purse` table: Token, Available, Allocated, On-chain, with an `overcommitted` flag when allocations exceed the on-chain balance.
  - Per-chore purse tables: Token (+ "(sub N)"), Balance.
  - Empty state: "No chores have funded purses yet. Enable a purse in a chore's settings…"
- **Calls**: `getAllPurseAllocations`, `getTokenRegistry`, ledger `icrc1_balance_of` per token.
- **Redesign notes**: delete, or confirm that WalletPanel shows the same "available vs allocated vs on-chain" breakdown, including the overcommitted warning.

## App-wide bot health indicators in the header (hooks)
- **Routes & params**: every page (components/Header.jsx notification strip); login; each indicator also needs its `/me` setting on (`notifyLowCycles`, `notifyOutdatedBots`, `notifyBotChores`, `notifyBotLogErrors`, `notifyBotLogWarnings`, stored in localStorage and synced to backend user settings).
- **Purpose**: surface bots needing attention without visiting each bot.
- **Information shown**: "⚡ N" ("N canisters low on cycles" — `useLowCyclesNotification`, canisters below their red threshold), "⬆ N" ("N apps can be upgraded" — `useOutdatedBotsNotification`, newer version for their WASM-resolved app), "🤖 N" ("N bots need attention" — combined `useBotChoreNotification` (bots whose all-chores lamp is warn/error) + `useBotLogNotification` (unseen error/warning counts; warnings counted only if `notifyBotLogWarnings`)).
- **Actions**: click → TopUpCyclesDialog (bulk top-up; another slice), UpgradeBotsDialog, BotHealthDialog. After upgrades/top-ups the header dispatches `neuronManagersRefresh`.
- **Shared building blocks**: all four hooks only read WalletContext (`lowCyclesCanisters`, `outdatedManagers`, `managerChoreStatuses`, `allBotEntries`, `botLogAlerts`) and expose `{count, list, isDialogOpen, openDialog, closeDialog}`; the hooks' own dialog state is unused by Header (it keeps its own `isBotHealthDialogOpen`).
- **Gating/state**: data is only as fresh as WalletContext's bot polling.
- **Redesign notes**: natural seed for a "Bots" notification centre; cycle thresholds are per-browser settings (`/me`) shared by staking bots ("neuron manager" thresholds) and other canisters ("canister manager" thresholds).

## Bot Health dialog (BotHealthDialog)
- **Routes & params**: modal from the header "🤖 N" indicator; login.
- **Purpose**: one list of all bots with chore problems or unseen log errors/warnings, with jump-off links.
- **Information shown**: title "Bot Health"; "All bots are healthy."; summary "N bots need attention", "N chore errors", "N chore warnings", "N log errors", "N log warnings"; per bot (collapsible): `BotIcon` by type, name (`PrincipalDisplay`), "(Trading Bot | ICP Staking Bot | app name)", "Chore: <lamp label>" + "View chores ›", up to 50 unseen Warning/Error log entries since lastSeen (level, source, message, time, #id, tags) fetched with `botLogIdl.getLogs({minLevel: Warning, startId: lastSeen+1})`, or "N unseen errors … — view in log".
- **Actions**: "Log" (navigate to bot page + `?tab=log`), "Seen" (per bot: localStorage `lastSeenLogId:<id>` + backend `mark_logs_seen`, then `refreshBotLogAlerts`), "View chores ›" (`?tab=chores`), click entry → log, "Mark all logs seen", "Close".
- **Entry points & cross-links**: URLs from factory `manageUrl` with `CANISTER_ID`, fallback hard-coded `/trading_bot/<id>` / `/icp_neuron_manager/<id>`, else `/canister?id=`. Neither bot page reads `?tab=`, so the user lands on the default tab; on the Trading Bot the generic Log tab is hidden altogether.
- **Shared building blocks**: `ChoreStatusLamp` labels/colours, `BotIcon`, utils/botLogIdl, WalletContext.
- **Redesign notes**: deep links need real routes/anchors for "bot → chores" and "bot → log".

## Upgrade Apps dialog (UpgradeBotsDialog)
- **Routes & params**: modal opened from the header "⬆ N" indicator, the Wallet Apps tab, `/apps` and PrincipalBox; login; controller of each canister required (install_code).
- **Purpose**: bulk-upgrade outdated app canisters (both bot types and other Sneedapp apps, incl. ledger-suite code from the NNS SNS-W canister).
- **Information shown**: title "Upgrade Apps"; "All apps are up to date."; warning banner for canisters with an unrecognized WASM (deselected by default, "Unknown WASM" badge); per row: checkbox, app icon/label, name, current → target version ("vA → vB", or NNS hash label), per-step progress "vX (step i of n)", status, error text (cycles/controller errors summarised); results summary.
- **Actions**: select / "Select all" / deselect; "Upgrade app" / "Upgrade N apps" → per canister step through every published version (`upgradeSteps`; WASM fetched from URL and sha256-verified, or fetched from SNS-W for NNS-sourced code), management `install_code({upgrade})` with empty args; `window.confirm` when unknown-WASM canisters are selected; "Retry failed"; "Cancel"/"Done" (closing mid-run aborts after the current step).
- **Entry points & cross-links**: header, Wallet, /apps, PrincipalBox.
- **Shared building blocks**: utils/upgradePath, utils/wasmIntegrity, WalletContext version maps, `BotIcon`.
- **Redesign notes**: third copy of upgrade logic (also BotManagementPanel and IcpNeuronManager); uses `window.confirm` while pages use `appConfirm`.

## Shared bot UI atoms and utilities
- **ChoreStatusLamp** (components/ChoreStatusLamp.jsx): lamp states off/ok/active/warn/error/cb with labels "Off", "Healthy", "Running", "Attention needed", "Error", "Circuit breaker triggered"; per-chore scheduler lamp ("Scheduled", "Stopped", "Paused", "Stopped by CB rule: X", "Overdue — scheduled time has passed", "Overdue — last run was over 3 intervals ago", "Enabled but no timer set", "Interval exceeds NNS 6-month followee confirmation deadline" for confirm-following), conductor lamp ("Idle", "… — running for N min", "Stop requested"), task lamp ("Running: <task>", "Stale — exceeded timeout", "Last task failed: …"); summaries per chore and per bot. Used by BotManagementPanel, TradingBot, IcpNeuronManager, Wallet, PrincipalBox, Apps, header hooks, BotHealthDialog.
- **BotIcon** (components/BotIcon.jsx): robot icon with badge — `staking` (brain) or `trading` (chart). Used on both bot pages, CreateIcpNeuron, Wallet, PrincipalBox, DappCardModal, SneedApp store/detail, dialogs.
- **NeuronManagerSettings** (utils/NeuronManagerSettings.js): localStorage cycle thresholds (default red < 1T, orange < 5T) for staking bots ("neuronManagerSettings") and for other canisters ("canisterManagerSettings"), edited on `/me`; `getCyclesColor`, `formatCyclesCompact`, `formatMemory`, `parseCyclesInput`. Used by bot pages, Wallet, Apps, Canister page, Sns page, TopUpCyclesDialog, ToolsRecoverTopUp.
- **botCycles** (utils/botCycles.js): public `getCyclesBalance` query so non-controllers can see a bot's cycles (staking ≥ 0.9.5, trading ≥ 0.9.4); used by BotManagementPanel, Wallet, WalletContext (IcpNeuronManager does not use it).
- **botPaging** (utils/botPaging.js): version-tolerant paging (caps since trading 0.9.4 / staking 0.9.5: trade log & logs 500, snapshots 200, price history & daily rows 1000; `newestFirst`/`offset`; `beforeId` for older logs) — `fetchNewestRows`, `fetchNewestSnapshots`, `fetchAllTradeLog`, `fetchOlderLogs`; used by TradingBot panels and DuplicateCleanupDialog.
- **botLogIdl / botEventIdl** (utils): minimal IDLs (`getLogs`, `getLogConfig`, `getLogAlertSummary`; full BotEventEngine) so app-wide code can read logs/events of any bot type without its full declarations.
- **PendingStakesNotice** (components/PendingStakesNotice.jsx; detailed in the staking-bot Neurons section): "Unfinished neuron stakes (N)" from the local stake journal, "Finish staking" (`claimNeuronFromDeposit` for `icp_bot`, SNS `ClaimOrRefresh` for `sns`), "Details", "Dismiss" → "Yes, forget it"/"Keep"; mounted on IcpNeuronManager, Wallet and SnsNeuronWizard (on error).

## Dead / leftover bot components
- `components/BotChoreHealthDialog.jsx` ("bots with unhealthy chore lamps … navigates to its chores tab") and `components/BotLogAlertDialog.jsx` ("bots with unseen log errors/warnings … Mark as seen") are not imported anywhere — superseded by the unified BotHealthDialog.
- The four notification hooks' `isDialogOpen/openDialog/closeDialog` are unused by Header for the bot-health case.

## Slice observations

### (a) Entities, with the views and actions that exist
- **Bot canister** (staking or trading; any Sneedapp app):
  - Views: id, name/nickname, type (from WASM hash), version, official-version match / unverified / mismatch, status, cycles (controller via `canister_status`; public `getCyclesBalance` only in BotManagementPanel), memory, module hash, controllers, health lamps.
  - Actions: mint (Sneedapp mint, `/create_icp_neuron`), name / nickname, upgrade (step-wise or jump), reinstall, add/remove controllers, top up cycles (single here; bulk in header/App Manager), withdraw tokens, bulk upgrade (UpgradeBotsDialog).
  - Where: both bot pages, /apps, Wallet Apps tab, PrincipalBox/DappCardModal, header dialogs.
- **Official version** (factory registry): view (version, WASM/source URLs, hash); install.
- **Botkey** (principal + permission set; API "hotkey"):
  - Views: list with badges (both pages, anyone logged in).
  - Actions: add, edit, remove (ManagePermissions).
  - Vocabulary: staking bot 28 keys + ViewLogs/ManageLogs; trading bot 23 keys (ManagePurses listed twice).
  - `Vote` / `MakeProposal` can be granted on the staking bot but no UI uses them.
- **Chore type / chore instance**:
  - Types: staking confirm-following, refresh-stake, collect-maturity, distribute-funds; trading trade, rebalance, move-funds, distribute-funds, snapshot.
  - Views: schedule strip, lamps (scheduler/conductor/task, CB-aware), status card, run monitor, overview cards, header aggregate.
  - Actions: create/rename/delete instance; start, scheduled start, pause, resume, stop, Run Once/Run Now, Stop All (trading only); interval + random max interval; next-run override; task timeout (DSL only).
  - Where: both pages (two implementations), Wallet, DappCardModal, DSL.
- **ICP neuron (NNS)** (many per staking bot):
  - Views: id list, stake, voting power, dissolve state/delay, age, maturity, staked maturity, visibility, followees by topic, hotkeys, deposit account, following-confirmation deadline.
  - Actions: create (stake from wallet; journaled), increase stake, refresh stake, auto-stake maturity, stake/merge/spawn/disburse maturity, set/clear following per topic, confirm following, add dissolve delay, start/stop dissolving, disburse, NNS hotkeys add/remove, split, merge neurons, make public/private.
  - Not available: voting or proposals.
- **Pending stake** (local journal): list, "Finish staking", dismiss (IcpNeuronManager, Wallet, SnsNeuronWizard).
- **Pending top-up** (local journal): resume (PendingTopUpsNotice inside every top-up card).
- **Token registry entry (trading)**: view chips; add, remove, scan whitelist, reorder; pause/unpause, freeze/unfreeze (Wallet tab, Circuit Breaker actions, DSL, reaction actions).
- **Main account / main purse / chore purse (trading)**:
  - Main purse = on-chain balance − allocations ("overcommitted" flag).
  - Views: balances, valuation, pie chart, allocations.
  - Actions: fund from wallet, withdraw, send, enable/disable purse, fund/reclaim/"To Wallet", "Reclaim All to Main", trade-from override.
  - Where: Wallet tab and each chore Purse tab (duplicated), wizard funding, DSL `submit`.
  - Subaccount API (create/rename/delete/manual transfer) has no live UI.
- **Trade / move-funds action** (trading): list, add/edit (~30 fields: amounts, balance and price conditions, slippage/impact, trailing stop, cumulative limits, executions), enable/disable, remove, reset stats; also created by the wizard and the DSL.
- **Rebalance target set + settings**: editor (sliders, locks, "Equal Split", 100% rule), live drift/"Portfolio Status", per-token pause, fallback route tokens.
- **Distribution list** (both bots, separate UIs): token, amount mode, threshold, min/max, targets (account or purse, share); CRUD.
- **Trade log entry / detected inflow-outflow**: filterable list, copy; delete inflow/outflow rows, "Cleanup Duplicates". Views: Logs → Trade Log, per-chore Logs, swap card.
- **Portfolio snapshot / daily summary**: Performance charts (equity curve, OHLC), Logs → Portfolio Snapshots (latest 100).
- **Capital flows / capital deployed**: Performance cards and table; hidden manual adjust (`adjustCapitalDeployed`).
- **Price quote / price history / candles**: Performance → price history (last known prices, derived cross rates, pair charts).
- **Circuit breaker rule / event**: rules engine (nested AND/OR conditions on price/value/balance, 12 action types); global/per-rule enable; event log (last 50) also drives lamps.
- **One-off trade (Quick Trade)**: submit, queue (polls every 4 s), cancel pending, clear history.
- **ICPSwap stuck balance (Recovery)**: check pools per token pair and fee tier; recover.
- **DEX setting**: enable/disable ICPSwap / KongSwap (KongSwap shut down); default slippage/impact (DSL only).
- **Bot log entry / log config**:
  - Views: three viewers (BotManagementPanel Log tab, unused on trading; Trading `BotLogPanel`; staking Log tab); per-chore activity log; header alerts.
  - Actions: filter, page, auto-refresh, set write level / max entries, clear, mark seen.
- **Logging / engine settings (trading)**: trade/portfolio log switches, per-chore overrides, buffer sizes, price/metadata staleness, price-history size.
- **Event (bot event system)**: emission toggle, listeners, event types, event log; subscriptions (self or other bot); reaction rules (conditions, action params, cooldown); reaction log. Both bots, via BotEventPanel.
- **SneedScript document**: export state, copy LLM guide and context, parse/resolve, review ops, execute (Script tab, AI Script wizard).

### (b) Overlaps with other areas of the app
- **Neuron management is implemented twice.**
  - ICP neurons in IcpNeuronManager vs SNS neurons in /neuron (Neuron.jsx), wallet TokenCard neuron rows, SnsNeuronWizard and LockDissolvePanel.
  - Both cover the same verbs (stake, dissolve, delay, disburse, split, maturity, followees, hotkeys/permissions) with different UI and wording.
  - ICP neuron voting exists only as hotkey voting on `/icp_proposal` (IcpHotkeyNeurons). Nothing links it from the bot page.
- **Bot-owned neurons are also shown in Wallet/WalletContext and DappCardModal.** Those views (stake, maturity, dissolve per neuron, totals) are richer than the bot page's neuron selector, which shows only "Neuron #id".
- **The bot console exists twice** (BotManagementPanel vs IcpNeuronManager's inline copy). Only BotEventPanel and ChoreStatusLamp are shared. Several pieces are repeated across the app:
  - Upgrade logic exists three times (both consoles plus UpgradeBotsDialog), with different step semantics.
  - Cycles top-up exists five times (both consoles, CreateIcpNeuronWizard, TopUpCyclesDialog, App Manager/Wallet).
  - Controller editing is in both consoles and the App Manager.
- **Chore controls are spread out.** They appear on both pages, in the Wallet Apps tab, in PrincipalBox → DappCardModal, and in the DSL. On the Trading Bot itself they also repeat between the upper chore header and the Control tab.
- **Wallet-like functions inside bots**: withdraw (Info), fund/withdraw/send (Trading Wallet tab), Quick Trade (a swap with no quote preview) and Recovery (ICPSwap pool balances). These overlap the user Wallet, SendTokenModal, Sneedex/Swap and any ICPSwap tooling.
- **Valuation and price views** (Trading Wallet "Value in", Performance, Rebalancer portfolio status, price history) overlap the app's PriceService and wallet portfolio valuation. The Wallet tab uses its own local denomination picker while Quick Trade uses the global one.
- **Distribution lists** exist in both bots with separate UIs.
- **Naming** (nickname and public name) is repeated per page type, the same pattern as other principal pages.
- **Minting** has three paths: Sneedapp mint for any app, `/create_icp_neuron` for the staking bot, and the Trading Bot via Sneedapp only. Bot lists appear on `/create_icp_neuron`, `/apps` and the Wallet Apps tab.
- **Automation comes in four flavours**: chores, circuit breaker rules, event reaction rules, and SneedScript `ensure` programs.
- **Bot health is spread across three surfaces**: header indicators and dialogs, per-page lamps, and App Manager lamps.

### (c) Dead, hidden, half-built or confusing
- **Dead code**:
  - TradingBot `AccountsPanel` and `PursesPanel` (~1,300 lines). The Wallet empty state still says "Register tokens in the Accounts panel".
  - `BotChoreHealthDialog` and `BotLogAlertDialog`.
  - On CreateIcpNeuron: `caller_is_admin` and several unused state variables.
  - The beta gate on CreateIcpNeuron is past its end date (2026-01-24).
  - BotManagementPanel's Log tab is unused, because its only consumer hides it.
- **Deep links are broken.** `?tab=chores` and `?tab=log` (BotHealthDialog, Wallet, DappCardModal) are ignored by both pages. On the Trading Bot the generic Log tab is hidden, and logs live in the lower "Logs" tab.
- **IcpNeuronManager bugs**:
  - After any refresh, the neuron stats may belong to `neurons[0]` rather than the selected neuron.
  - Every action blanks the whole page with "Loading manager...".
  - Top-up, withdraw and controller messages render inside the Neurons section, so users without neuron access never see them.
- **No client-side permission gating on the Trading Bot panels.** Users see every tab and get canister error strings. On the staking bot, neuron tabs are gated, but some buttons (Increase/Refresh Stake, "Set" next run, "+" instance) are not.
- **Missing confirmations on irreversible or fund-moving actions**: disburse, split, merge, start dissolving, make public, remove NNS hotkey, withdraw/send, token removal, purse disable, pause/freeze, recover, action/distribution removal, event-rule removal, and DSL "Execute N Operations".
- **Data-loss bug**: the trading action editor clears frequency limits and preferred DEX on save, which wipes values set via DSL.
- **Half-built features**:
  - Portfolio Snapshots stop at 100 ("More snapshots available...").
  - The circuit-breaker event log is capped at 50.
  - The staking bot persists `minDistributionAmount` but has no input for it.
  - Trading distribution amounts are entered in raw base units.
  - Quick Trade has no KongSwap option and no quote preview, and its source-purse list goes stale.
  - The Recovery tab's token icons are broken (wrong prop).
- **Buried capabilities**:
  - Engine settings sit under Logs → Logging Settings.
  - "Capital deployed" is changed through a hidden edit icon.
  - Hidden controls: "Schedule first run at" (▼), next-run "Set", "Randomize interval...", rename by double-click, token "Manage".
  - The "Trade from" purse override only appears when other purses exist.
  - The Script tab is the 8th tab, yet it is the most complete editor.
- **Confusing terminology and layout**:
  - "Hot Keys" (NNS) vs "Botkeys" (bot permissions; API `*Hotkey*`) vs SNS neuron hotkeys/permissions.
  - Two tab systems on the Trading Bot page, both with "Chores".
  - The lower strip defaults to "Wallet" although "Chores" is listed first.
  - Info tabs mix canister lifecycle, cycles and a mini-wallet.
  - "Upgrade Available" (next version) vs "Install Official Version" (latest) on the staking bot.
  - Merge Maturity (deprecated NNS command) is still offered.
  - Disburse takes a 64-hex account id while the rest of the page uses principals.
- **Version-dependent behaviour** is handled by checking whether a method exists, and only partially. The Trading CB and Performance calls, and the staking page's cycles display for non-controllers, are not version-aware.
- **Mobile**: drag-to-reorder (rebalance targets, token chips; react-dnd `HTML5Backend`) does not work on touch, and Quick Trade grids are not responsive. Neither bot page has a mobile-specific layout.

### What a unified "bot console" would need (both bots)
- Identity header: name, type, version and verification, cycles, lamps, Stop All.
- Canister tab: status, upgrade (multi-step), reinstall, controllers, cycles top-up.
- Funds tab: balances, withdraw/send, and for trading the purses.
- Botkeys tab, with a shared permission vocabulary and descriptions.
- Chores tab: a framework with per-type config slots. Staking needs collect-maturity and distribution; trading needs actions, rebalance, distribution, purse, fallback and snapshot.
- Log, with one viewer, alerts and mark-seen.
- Events.
- Automation rules: circuit breaker (trading).
- Bot-type modules: Neurons (staking); Performance, Price history, Quick Trade, Recovery, Script (trading).
- Deep-linkable routes for every tab.

### Area notes (detail from the per-area review, kept for reference)

**ICP Staking Bot — bot console (Info / Chores / Botkeys / Log / Events)**

- **Entities and their views/actions (this part)**:
  - **Bot canister**:
    - View: id, version, verification status, cycles, memory, status, module hash, controllers.
    - Actions: name, nickname, upgrade, reinstall, add/remove controllers, top up cycles, withdraw tokens.
  - **Official version** (sneedapp factory registry):
    - View: version, wasm/source URLs, matched/unverified/mismatch states.
    - Actions: install (upgrade or reinstall).
  - **Botkey** (principal plus permission set):
    - View: list with badges (anyone logged in).
    - Actions: add, edit, remove (ManagePermissions).
  - **Permission type**: a dynamic list from the canister (`listPermissionTypes`); 28 known keys plus ViewLogs and ManageLogs.
  - **Chore type and instance** (confirm-following, refresh-stake, collect-maturity, distribute-funds):
    - View: schedule strip, lamps, status card.
    - Actions: start, scheduled start, pause, resume, stop, run once/now, interval, randomized max interval, next-run override, create/rename/delete instance.
  - **Collect-maturity settings**: threshold and destination account.
  - **Distribution list**: name, token, amount mode, %, threshold, max per round, targets (account, % share). Actions: add, edit, remove.
  - **Log entry / log config**:
    - View: entries with filters and paging.
    - Actions: write level, max entries, clear.
  - **Event system**: delegated to BotEventPanel.
  - **Pending top-up** (journal): resume via PendingTopUpsNotice.
- **Overlaps with other areas**:
  - The whole Bot section duplicates `components/BotManagementPanel.jsx` (TradingBot).
  - Cycles top-up also exists in /apps App Manager, the Wallet Apps tab and BotManagementPanel.
  - Controllers editing and WASM upgrades also exist in the App Manager and UpgradeBotsDialog.
  - Principal naming duplicates naming elsewhere (users, neurons).
  - Distribution lists duplicate the trading bot's DistributionConfigPanel.
  - Withdraw is a mini-wallet send.
  - Botkeys vs NNS hotkeys (neuron section) vs SNS neuron hotkeys are three "who can act" concepts.
- **Dead, hidden, half-built or confusing**:
  - **Page-level errors and successes for top-up, withdraw and controller changes render inside the Neurons section** (around line 7404). Users without neuron access (the Neurons section is hidden) or with it collapsed never see them.
  - `minDistributionAmount` has no input, though it is persisted.
  - ViewLogs and ManageLogs have no friendly label or description.
  - Non-controllers get no cycles figure (no public fallback here).
  - The "Set" next-run picker and "+" instance creation show without a client-side permission check; the backend enforces.
  - "Upgrade Available" steps to the next version, while "Install Official Version" jumps to the latest. This is inconsistent with BotManagementPanel's multi-step path.
  - The Withdraw destination takes no subaccount or name search, unlike the other principal inputs.
  - A leftover comment notes that the "Refresh" button for chores was removed in favor of auto-refresh.
  - DOM-id-toggled hidden panels (`style.display` via getElementById) are used for schedule start, next run and randomized range.
  - The Info tab mixes canister ops, wallet ops (withdraw) and fuel (cycles).
  - The No Access card renders below an accessible Bot section.

**ICP Staking Bot — Neurons section and handlers**

- **Entities**:
  - *Staking bot canister*: views — validity check, version, official-version match, cycles/controllers (controller-only); actions — upgrade/reinstall, controllers add/remove, cycles top-up, withdraw ICP/tokens, naming (handlers here, UI in inm-b).
  - *ICP (NNS) neuron* (many per bot): views — ID list, summary stats (stake, VP, delay, age, maturity, state), overview (visibility, controller, maturity, staked maturity, hotkey/followee counts), deposit account, followees by topic, activity/confirm deadline; actions — create (stake), increase stake, refresh stake, auto-stake toggle, stake/merge/spawn/disburse maturity, set/clear following per topic, confirm following, add delay, start/stop dissolving, disburse, NNS hotkeys add/remove, split, merge, visibility.
  - *Known neuron* (NNS registry): view-only, used as followee picker labels.
  - *NNS topic*: hardcoded 19 topics with critical flags.
  - *Botkey / permission*: caller permission set gates tabs and sections.
  - *Pending stake* (local journal): list, finish, dismiss.
  - *Chore* (handlers only here): statuses/configs, collect-maturity settings, distribution lists — UI in inm-b.
- **Overlaps**:
  - SNS neuron management (Neuron.jsx `/neuron`, TokenCard.jsx wallet cards, SnsNeuronWizard) implements the same verbs (dissolve, disburse, split, stake/disburse maturity, followees, permissions/hotkeys) with separate UIs and different terminology.
  - ICP neuron read views for the same bot in WalletContext/Wallet.jsx (getNeuronIds/getNeuronInfo/getFullNeuron) and DappCardModal (per-neuron stake/maturity/dissolve, totals) — richer neuron list than this page's selector.
  - Chore start/stop/pause/resume/trigger also available from Wallet.jsx (2393-2397) and PrincipalBox.jsx (1022-1026).
  - Manual NNS voting exists only via IcpHotkeyNeurons on `/icp_proposal` (requires adding own principal as NNS hotkey here).
  - Staking (journal + claim) duplicated with CreateIcpNeuronWizard and SnsNeuronWizard; PendingStakesNotice shared with Wallet and SnsNeuronWizard.
  - Management-canister IDL, controller/upgrade/top-up logic duplicated between IcpNeuronManager.jsx and the shared BotManagementPanel used by TradingBot (staking bot does not use BotManagementPanel).
- **Dead / hidden / half-built / confusing**:
  - `?tab=chores` / `?tab=log` deep links (Wallet, DappCardModal, BotHealthDialog) are ignored — page has no query-param handling; `navigate` imported but unused.
  - `fetchManagerData` re-fetches info for `neurons[0]` while keeping a different `selectedNeuronId` → with multiple neurons, after any action/refresh the stats shown can belong to the first neuron while the header says another (bug). It also flips the page-wide `loading` flag, so every action blanks the page to "Loading manager...".
  - No confirmation dialogs for irreversible neuron actions (disburse, split, merge, start dissolving, make public, remove hotkey); appConfirm only used for upgrade/reinstall, chore-instance delete, distribution-list remove, log settings.
  - `Vote` / `MakeProposal` permissions grantable with no UI to use them.
  - Merge Maturity is a deprecated NNS operation but still offered.
  - Client-side dissolve-delay validation commented out; hint says "Min 183 days to vote, max 2 years" (consistent with the code comment in CreateIcpNeuronWizard: 730 days / 2 years is the NNS maximum since the 2026 tokenomics change; but nothing validates the input).
  - "Confirm Following" button only appears when followees already exist; the activity status is hidden otherwise.
  - Two "hotkey" concepts on one page ("Hot Keys" tab = NNS; "Botkeys" tab = bot permissions, API named `*Hotkey*`).
  - Increase Stake and Refresh Stake are not permission-gated in UI (rely on canister); Increase Stake transfer is not journaled.
  - Hardcoded NNS_TOPICS list may drift; Known neurons always fetched from mainnet regardless of network.

**Trading Bot — chore config editors, Trade Log, Snapshots, Logging Settings**

- **Entities**:
  - Chore instance (trade / rebalance / move-funds / distribute-funds / snapshot): config sub-tabs; summary card progress.
  - Action (trade-chore or move-funds-chore): list, add, edit, enable/disable, remove, reset stats; types Trade (Swap) / Fund Purse / Reclaim from Purse / Send; conditions, trailing stop, limits.
  - Rebalance target set + rebalance settings + fallback route tokens: view, edit, pause per token, live drift view.
  - Distribution list + targets (purse or external account, % or auto-split): CRUD.
  - Purse (Main Purse vs chore purse by instance id): selected as source/target in actions and distributions; balances read for rebalance.
  - Trade log entry (incl. Detected Inflow/Outflow reconciliation rows): list/filter/page/copy; delete inflow/outflow; bulk dedupe.
  - Portfolio snapshot (Before/After/Snapshot): view latest 100, paired diffs.
  - Logging/engine settings: trade/portfolio logging master + per-chore overrides + buffer sizes; metadata/price staleness; price history size.
  - Token registry: implicitly written (`addToken`) whenever actions/targets are saved.
  - Botkey permissions: 23-key vocabulary.
- **Overlaps with other app areas**:
  - Token pickers/amount formatting overlap Wallet/Swap; frontend price computation (PriceService) overlaps wallet valuation.
  - Send action destination (PrincipalInput, ICRC-1 account) mirrors wallet send.
  - Portfolio Status (Rebalancer), Portfolio Snapshots, Trade Log balance snapshots and Performance/Wallet tabs all show bot balances/values in different shapes.
  - Same strategies are editable via Setup Wizard and Script (DSL) tab — three editors for one config.
- **Dead / hidden / half-built / confusing**:
  - Action form always clears `minFrequencySeconds`, `maxFrequencySeconds`, `preferredDex` on save although the description advertises "frequency limits".
  - Distribution amounts in raw base units.
  - Portfolio Snapshots "More snapshots available..." without paging control.
  - `getTradeLogStats` result unused beyond a label; `ChoreCumulativeSummary` calls getTradeActions for rebalance chores.
  - Duplicate `ManagePurses` permission key; `canisterId` prop unused in RebalancerConfigPanel; price-direction ternary in edit form has identical branches.
  - No client-side permission gating; removes (action, distribution list) without confirmation while trade-log deletes do confirm.
  - Engine settings (staleness, price history size) buried under Logs → Logging Settings.
  - Drag-and-drop (rebalance editor, token chips) unusable on touch devices.

**Trading Bot — lower tab bar, Performance, Circuit Breaker, Bot Log, DEX, dead panels**

- **Entities and their views/actions in this part**:
  - **Chore instance**: overview cards with lamps, state, timing and run counts; click opens the editor. Also a CB action target (start/stop/pause one, by type, or all).
  - **Chore purse** (per-chore isolated balance):
    - Performance scope selector and purse value chart/holdings;
    - CB Balance/Value source ("Purse: X", "Main purse (minus allocations)");
    - dead PursesPanel.
  - **Portfolio snapshot / daily portfolio summary**: Performance equity curve (detailed / daily OHLC), P&L cards.
  - **Capital flows / capital deployed**: Net Capital card, `Capital Flows by Token` table, manual `adjustCapitalDeployed`.
  - **Price quote / price history / daily price candle**: Last Known Prices cards (direct and derived), pair chart (detailed / OHLC), flip.
  - **Circuit breaker rule**: list, global enable, per-rule enable, create/edit (nested AND/OR condition tree, multiple actions), delete.
  - **Circuit breaker event**: last-50 table, copy, clear; also feeds chore lamps.
  - **Bot log entry**: filter (level threshold, source), paging, auto-refresh, copy, set write level, clear.
  - **DEX setting**: list and enable/disable (ICPSwap, KongSwap).
  - **Token** (registry entry / paused / frozen): CB actions pause or freeze globally or in a rebalance portfolio. Registry management is only in the dead AccountsPanel here; it is live in WalletPanel (another part).
  - **Subaccount**: API exists but has no live UI (dead AccountsPanel only).
- **Overlaps**:
  - Bot Log viewer is implemented three times: `BotLogPanel` here, BotManagementPanel Log tab (hidden on this page via `hideLogTab`), and IcpNeuronManager's own Log tab.
  - Chores overview duplicates status shown in BotManagementPanel's Chores tab; the lamp helpers are shared with `ChoreStatusLamp` and the App Manager.
  - Circuit Breaker (conditions → actions) overlaps conceptually with BotEventPanel reaction rules and with the DSL Script tab; that makes three ways to automate on one bot.
  - Price history and portfolio valuation overlap with app-wide price displays (PriceService) and the user Wallet's value views; denominated values use the shared `fetchDenomPrices`.
  - Portfolio snapshots appear in both Performance (charts) and Logs → Portfolio Snapshots (list, another part).
- **Dead, hidden, half-built or confusing**:
  - `AccountsPanel` and `PursesPanel` are unmounted dead code (~1,300 lines). Subaccount features (`createSubaccount`, `renameSubaccount`, `deleteSubaccount`, `manualTransfer`) are now unreachable.
  - The lower tab bar's default tab is `Wallet`, but `Chores` is listed first.
  - Chore statuses in the lower console load once and never refresh.
  - Performance has no Refresh button. The CB tab and the Performance core calls are not version-guarded, so older bots show raw "Failed to load…" errors instead of a "not supported in this version" state.
  - CB event log is capped at 50 with no paging. The Bot Log `caller` filter is supported by the API but has no UI.
  - DEX toggle: the UI does not block disabling the last active DEX (it relies on the canister). KongSwap shutdown is shown as a warning, not handled automatically.
  - `PERMISSION_LABELS` / `PERMISSION_DESCRIPTIONS` define `ManagePurses` twice (the later description wins). `ManageDexSettings` mentions slippage, which is not in the DEX panel.
  - "Logging Settings" also holds price staleness, metadata staleness and price-history size (non-logging settings).
  - "Capital deployed" manual adjustment is a hidden edit icon with no confirmation, yet it changes the P&L figures.
  - No client-side permission gating anywhere in these panels: users without the needed botkey permissions see the tabs and get error strings from the canister.

**Trading Bot — Wallet, chore Control/Purse/Fallback/Logs, Quick Trade, Recovery**

- **Entities (views → actions)**:
  - Token registry entry: Wallet chips → add, remove, scan, reorder.
  - Token risk state (paused/frozen): Wallet balance table badges → pause/unpause, freeze/unfreeze.
  - Main account (on-chain, default subaccount): balances, valuation, pie chart, receiving addresses → fund (from wallet), withdraw (to caller), send (external).
  - Main purse (virtual = on-chain − allocations): Wallet "Main Purse" table, Purse tab context → source for funding purses.
  - Chore purse (virtual allocation, optional subaccount number): Wallet "Purse Allocations", chore Purse tab → enable/disable, fund (main/wallet), reclaim, to wallet, reclaim all, trade-from override.
  - Chore instance: Control tab (state, lamps, schedule, run counts, conductor run and task history) → start, schedule-start, pause, resume, stop, run once/now, delete, set next run, set interval, randomize max interval.
  - Chore run / task: Control run card, swap card → dismiss.
  - Trade-log entry: swap card, per-chore Trade Log → copy, load older.
  - Activity-log entry: per-chore Activity Log → copy, load older.
  - One-off trade: Quick Trade queue → submit, cancel (pending), clear history.
  - Fallback route token list (per trade chore) → add, remove, reorder.
  - ICPSwap pool stuck balance: Recovery → check, recover.
- **Overlaps with other app areas**:
  - Fund/withdraw/send and receiving addresses mirror the user Wallet's send/receive.
  - Quick Trade mirrors wallet/Sneedex swap (fewer options, no quote).
  - Per-token value/denomination mirrors wallet portfolio valuation, but with its own local "Value in" picker, while Quick Trade uses the global `useDenomination`.
  - Recovery mirrors any ICPSwap pool-balance tooling elsewhere.
  - Chore Control duplicates BotManagementPanel's own chore header controls and is the counterpart of the ICP Staking Bot's chores UI.
- **Dead / hidden / half-built / confusing**:
  - `AccountsPanel` (line 4278) and `PursesPanel` (line 8416) are defined but never rendered. The WalletPanel empty state still says "Register tokens in the Accounts panel".
  - Purse fund/reclaim/withdraw code and UI are duplicated between WalletPanel and PursePanel.
  - Quick Trade's DEX select has no KongSwap option. Its "Source Purse" lists all chore instances (stale after creating chores) rather than purse-enabled ones.
  - `ChoreConfigTabs` badge support is unused, and `dismissedErrors` is passed to ControlTabPanel but unused there.
  - `PERMISSION_LABELS` lists 'ManagePurses' twice.
  - The trade-log copy export does not name the Detected Inflow/Outflow action types.
  - Activity-log tags appear only in the copy export.
  - Hidden controls: ▼ "Schedule first run at", next-run "Set", "Randomize interval...", "Manage" for the token registry, per-token pause/freeze icon buttons, the "Trade from" override (only appears when other purses exist).
  - No confirmations for withdraw/send, token removal, purse disable, pause/freeze or recover. Only "Reclaim All to Main", chore delete and short intervals ask first.
  - Drag-reorder of tokens uses `HTML5Backend` (likely not touch-friendly). Quick Trade grids are not responsive.
  - Errors are swallowed silently: token reorder persistence, fallback-token saves.
  - Version handling relies on method-existence checks (`getTokenRegistry`, `getPausedTokens`, `getFrozenTokens`, `getTradingPurseId`, `getChoreStatuses`) and the `beforeId` log-paging fallback.
