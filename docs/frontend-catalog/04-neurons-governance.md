# Stake & govern: neurons, proposals, voting, liquid staking, SNS Jailbreak, rewards

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Source root: `src/app_sneeddao_frontend/src`. Paths below are relative to it. "Neuron" means SNS neuron unless it says NNS/ICP. Permission numbers are SNS `NeuronPermissionType`: 1 Configure Dissolve State, 2 Manage Principals, 3 Submit Proposal, 4 Vote, 5 Disburse, 6 Split, 7 Merge Maturity, 8 Disburse Maturity, 9 Stake Maturity, 10 Manage Voting Permission (0 = Unspecified/legacy).

---

## Global SNS selector (`components/SnsDropdown.jsx`, `contexts/SnsContext.jsx`)
- **Routes & params**: any page that renders `<Header showSnsDropdown>`. In this slice: `/neuron`, `/neurons`, `/proposals`, `/proposal`, `/sns`. Elsewhere: `/feed`, `/forum`, `/topic/:id`, `/thread`, `/post`, `/tips`, `/me`, `/names`, `/principal`, `/canister`, `/transaction(s)`, `/admin/names`. Public.
- **Purpose**: pick the "current SNS" that most SNS pages follow.
- **Information shown**: the selected SNS's logo in the header (Sneed logo for Sneed); open list = every SNS (logo + name), Sneed first, then A–Z. No search box. "Loading SNSes..." while loading.
- **Actions**: click an SNS → sets the global selection, writes `?sns=<root>` to the URL (deletes it for Sneed), calls the page's `onSnsChange`.
- **State rules**: initial value from `?sns=` → else `localStorage.selectedSnsRoot` → else Sneed (`fp274-…`). The selection persists across pages and sessions. URL `?sns=` always wins on load. If no `?sns=` and the selection isn't Sneed, the dropdown writes `?sns=` into the URL. `SnsProvider` prefetches the full SNS list in the background on every page (idle callback). `snsReady` turns true when the selected SNS's canister ids are cached.
- **Redesign notes**:
  - Picking an SNS on `/sns` (directory) or on `/sns_neuron_wizard` also changes the global selection. So browsing the directory silently changes what `/neurons`, `/proposals`, `/forum`, etc. show.
  - The dropdown has no search, but the `/sns` directory and the wizard's picker do: three different SNS pickers.

## Header VP bar and the "votes waiting" badge (`components/Header.jsx`, `hooks/useVotableProposalsNotifications.js`, `utils/VotableProposalsUtils.js`)
- **Routes**: header of every page that shows the SNS dropdown. Needs login, at least one neuron in the selected SNS, VP > 0, and the setting `showVpBar` on.
- **Information shown**:
  - Your hotkeyed VP in the selected SNS ("X VP hotkeys (n)"; tooltip "Hotkeyed and owned neurons - can vote on SNS proposals").
  - "reachable (+n)": VP of extra neurons found under the same owner but not hotkeyed ("for forum voting").
  - SNS logo badge. Labels are shortened on narrow screens.
- **Actions**:
  - Refresh (re-fetches every neuron in WalletContext; tooltip "after transfer, Sneedex claim, or permission change").
  - Close/hide (writes `localStorage.showVpBar=false`, "re-enable in Settings").
- **Votable badge**: a count of open SNS and NNS proposals that your neurons can still vote on. Refreshes every 5 minutes and on the `votableProposalsRefresh` event (fired after votes). Click → `/active_proposals`. Turned on by the `notifyVotableProposals` setting.
- **Shared**: `WalletContext.neuronCache` (all SNSes plus NNS), `calculateVotingPower`, governance `get_nervous_system_parameters`.

## SNS directory and SNS details (`/sns`, `pages/Sns.jsx`)
- **Routes & params**: `/sns[?sns=<root>]`. Public. Has the SNS dropdown in the header. In the nav under Sneed Hub → "SNS".
- **Purpose**: browse every SNS DAO and inspect one: token, price, governance parameters, permissions, canisters.
- **Information shown**:
  - Directory list: count "N SNS DAOs"; per SNS: logo, name, token price (loaded one by one, or "No price"). "Selected" badge on the current one.
  - Details hero: name, token symbol, price in ICP and USD (DenominationContext).
  - Description and website URL (from `get_metadata`).
  - Token Information: Symbol, Name, Decimals, Total Supply, Tx Fee.
  - Token Price: Price in ICP, Price (USD), FDV ("Prices from ICPSwap"). "Price data not available (no liquidity pool found)" when there is no pool.
  - Governance Parameters: Min Stake, Max Dissolve, Min Dissolve to Vote, Dissolve Bonus, Age Bonus, Max Age for Bonus.
  - Advanced Parameters (collapsible): Proposal Reject Cost, Initial Voting Period, Wait for Quiet Extension, Max Neurons, Max Principals/Neuron, Max Followees/Function, Max Active Proposals, Maturity Modulation.
  - Voting Rewards (collapsible): Initial Reward Rate, Final Reward Rate, Transition Duration, Reward Round Duration.
  - Neuron Permissions (Hotkeys) (collapsible): "Grantable Permissions (Hotkeys can have)", "Not Grantable (Reserved for controllers)", "Claimer Permissions (New neurons get)". Also "This SNS has not configured any grantable hotkey permissions".
  - Canisters (collapsible): "Overall Health" lamp + count. System: Root, Governance, Ledger, Swap, Index. Dapps (Dapp 1..n). Archives (Archive 1..n). Each row: health lamp, cycles, memory, copy id, link `/canister?id=`, IC Dashboard link. Cycles and status come from root (`fetchSnsCyclesFromRoot`).
  - External Links: NNS dApp project page, IC Dashboard SNS page.
- **Actions**:
  - Search box "Search SNS...".
  - Sort: "Price: High to Low" (default), "Price: Low to High", "Newest", "Oldest", "A-Z", "Z-A".
  - Click an SNS to select it (sets the global SNS and `?sns=`).
  - Collapse the list or details panel.
  - Copy canister ids.
  - "View Price Chart" (ICPSwap token page, external).
  - "Explore This SNS" tiles: Proposals (`/proposals?sns=`), Forum (`/forum?sns=`), Neurons (`/neurons?sns=`), Transactions (`/transactions?sns=`).
- **Canister calls (all reads)**: governance `get_metadata`, `get_nervous_system_parameters`; ledger `icrc1_metadata`, `icrc1_total_supply`; root `list_sns_canisters`; root cycles/status; PriceService (ICPSwap).
- **Entry points**: nav "SNS"; Hub tile; SKILL guide.
- **Gating/mobile**: `isMobile` (≤768px) changes the layout (list and details collapse).
- **Redesign notes**:
  - This is the only place showing SNS governance parameters, permission policy and canister health. `/sns_neuron_wizard` and `/tools/sns_jailbreak` re-derive pieces of the same data (min stake, dissolve limits, grantable permissions).
  - No link from here to the SNS staking wizard.

## SNS neuron explorer and staker index (`/neurons`, `pages/Neurons.jsx`)
- **Routes & params**: `/neurons?sns=<root>[&tab=stakers]`. Public; a few extras need login (the VP column needs nervous system parameters, which are only fetched with an identity). Nav: Sneed Hub → "Neurons".
- **Purpose**: list and analyze every neuron of an SNS; a "Stakers" view indexes them by principal.
- **Information shown (tab "Neurons")**:
  - Title "<SNS> Neuron Explorer".
  - Stat cards: Total Active Stake (and % of total supply, neuron count "with stake"); Not Dissolving; Dissolving; Dissolved. Each card shows stake, % of supply and counts.
  - Loading progress ("Found N neurons (Total: M)").
  - Rows:
    - Neuron (`NeuronDisplay`: name / nickname / verified tick / colour).
    - Principals with role badges (Full Owner / Hotkey / Voter / Manager / Financial / Custom). Named ones are shown first; the rest sit behind "...N".
    - Stake, Status (Locked / Dissolving / Dissolved + dissolve state text), Voting Power, "View Details".
  - Promo box: "With Liquid Staking, you can create SNS neurons that remain transferable and tradable on Sneedex!" → `/liquid_staking`.
- **Information shown (tab "Stakers")**:
  - Title "<SNS> Staker Explorer".
  - Stats: Total Users, Total Neurons ("of N on-chain"), Total Staked.
  - Rows: User (`PrincipalDisplay` → profile), Neurons, Total Stake, Owned Stake, "View Profile".
- **Actions (Neurons tab)**:
  - Search "Search by neuron ID, name, or nickname..." (a `NeuronInput`).
  - State filter: "All States" / "Not Dissolving" / "Dissolving" / "Dissolved".
  - "Per page" 10/20/50/100.
  - "Show only named neurons"; "Show only verified names".
  - "Sort by:" Stake / Name / Lock Status / Voting Power.
  - "Refresh" (clears the IndexedDB cache and re-fetches every neuron).
  - "Export CSV" of filtered neurons. Columns: Neuron ID, Name, Nickname, Is Verified, Stake (E8s), Stake (Tokens), Dissolve State, Voting Power Multiplier, Owner Principals, Created At, Age (Days), Followees Count.
  - Row click or "View Details" → `/neuron?sns=&neuronid=`. Right-click or long-press on a neuron → context menu.
  - Previous / Next pages.
- **Actions (Stakers tab)**:
  - Search "Search by name or ID...".
  - "Show:" All Users / Owners Only / Hotkeys Only.
  - Per page; "Show only named users".
  - "Sort by:" Total Stake / Neurons / Owned / Name.
  - "Refresh"; Previous / Next.
- **Canister calls**: governance `list_neurons` (paged, all neurons, via `useNeuronsCache` → IndexedDB `NeuronsDB`); ledger `icrc1_total_supply`; `get_nervous_system_parameters`.
- **Shared**: `useNeuronsCache` (also used by `/hub`), `indexNeuronsForUsers`, `NeuronDisplay`, `NeuronInput`, `PrincipalDisplay`.
- **Redesign notes**:
  - The Stakers tab is a "holders/users of this SNS" view hidden inside a neuron page. The code comment says it mirrors `/users`, but `/users` no longer does this.
  - Whole-SNS download happens client-side and can be slow for big SNSes.

## SNS neuron detail and management (`/neuron`, `Neuron.jsx`)
- **Routes & params**: `/neuron?neuronid=<hex>&sns=<root>`. Public to view. Actions need login plus the matching permission on that neuron. Not in the nav (reached via links). With no `neuronid` it shows only the search box.
- **Purpose**: one SNS neuron: full state, permissions, following, voting history, stake-account transactions; manage dissolve, principals and followees.
- **Information shown**:
  - Header: "<SNS> Neuron Details", "Help" → `/help/neurons`.
  - Promo "Did you know? Create transferable SNS neurons with our Liquid Staking wizard!" → `/liquid_staking`.
  - Identity: public name (✓ if verified), your private nickname, neuron id (copy), "Dashboard" link (IC dashboard).
  - Principals strip with role icons (Full Owner / Hotkey / Voter / Manager / Financial / Custom).
  - Stake Account: "Stake Account (send tokens here to increase stake)". Toggles between the encoded ICRC-1 account and principal + subaccount (hex), each with copy.
  - Stats:
    - Stake (after fees, with "Fees deducted"); Maturity (available + staked + disbursing, "Disbursing: x"); Total value "Stake + Maturity".
    - Voting Power; Age with "Age Bonus +x% (max y%)".
    - Dissolve Delay or "Time to Dissolve" with "Dissolve Bonus +x%"; total bonus % (multiplicative).
    - Created date and time; "Maturity auto-staking" Enabled/Disabled.
    - Vesting (remaining time, % progress, or "Complete ✓"); "From NNS neuron <id>" (source NNS neuron).
    - Dissolve status line: "Locked for N days" / "Dissolving (N days left)" with "Dissolves on <date>" / "Dissolved" / "Ready to disburse".
  - Tab "Details":
    - "Permissions (n)" (collapsible): each principal (`PrincipalDisplay`, "You" tag) and its permission chips (labels and icons from `PERMISSION_INFO`).
    - "Followees" (collapsible, auto-opens if any): per topic (Gov, Community, SNS Framework, Dapp, App Logic, Treasury, Critical), the followed neurons (`NeuronDisplay`). "No followees configured" when empty.
    - "Voting History (n)" (Sneed only; from the RLL canister `get_neuron_voting_history`): proposal # link, title, Yes/No/not voted, VP.
  - Tab "Transactions": `TransactionList` of the neuron's governance subaccount (ledger transfers into or out of the stake account).
- **Actions**:
  - Search: `NeuronInput` "Enter neuron ID or search by name/nickname" (tabs Private / Public / All). A valid hex id navigates by itself after 500 ms. "Search different neuron".
  - "Set Nickname" (login). Inline edit, max 32 characters, letters/numbers/spaces/`-_.'`. Backend `set_neuron_nickname` (private, stored on-chain unencrypted).
  - Dissolve (needs permission 1):
    - "Increase Delay" → dialog "Increase/Set Dissolve Delay": days input; shows the current delay and min/max ("Min for voting", "Max additional"); validation dialogs "Invalid Input" / "Value Too High". → `manage_neuron Configure IncreaseDissolveDelay`.
    - "Start Dissolving" (shown when locked) / "Stop Dissolving" (when dissolving) → `Configure StartDissolving/StopDissolving`. No confirmation.
  - Principals (needs permission 2):
    - "Add Principal" form: `PrincipalInput` "Enter principal ID or search by name"; presets "Full Owner" / "Hotkey" (3+4); 11 permission checkboxes; "Add Principal" / "Update" / "Cancel".
    - Edit ✎ per principal.
    - Remove 🗑 per principal, with confirm dialogs: "Remove Principal?"; "Remove Yourself?" → "Yes, Remove Me"; blocks "Cannot Remove Principal" for the last principal or the last Manage-Principals holder.
    - Sends only the difference: `AddNeuronPermissions`, then `RemoveNeuronPermissions` (`utils/neuronPermissionPlan.js` refuses to drop the last Manage Principals). Afterwards refreshes the WalletContext and NeuronsContext caches.
  - Followees (needs permission 10):
    - Remove ✕ per followee.
    - "Add Followees": pick topics (checkboxes for the 7 topics); build a list of neurons (`NeuronInput` "Search neuron..." + "Add"); submit "Add N neurons to M topics". → `manage_neuron SetFollowing` (one call for all topics). Success dialog "Followees Added".
  - Voting history: "Hide Yes" / "Hide No" / "Hide Not Voted"; sort "Sort by Proposal ID" / "Sort by Date" / "Sort by Voting Power"; proposal links → `/proposal?proposalid=&sns=`.
- **Not available here** (only on the wallet token card): disburse, disburse maturity, split, increase stake (ClaimOrRefresh), auto-stake toggle, send/transfer neuron, "Sell on Sneedex". The page advertises the stake account but has no "refresh stake" button.
- **Entry points**: rows in `/neurons`; neuron context menu "View Neuron" (everywhere `NeuronDisplay` appears); `/me` My Neurons; wallet token card neuron link; wizard success "View Neuron"; Jailbreak "View neuron details" / "View Neuron"; Jailbreak list; `/proposal` proposer and voter links; Sneedex offer pages.
- **Shared**: `NeuronInput`, `NeuronDisplay`, `PrincipalInput`, `PrincipalDisplay`, `ConfirmDialog`, `TransactionList`, `TokenIcon`, `getNeuronFromCache`/`updateNeuronInCache` (IndexedDB), `VotingPowerCalculator`.
- **Legacy code**: unused state left in for "backwards compat" (`bulkMode`, `bulkFolloweeInput`, single-topic `addFollowee`, `bulkAddFollowees`, `bulkAddToMultipleTopics`). The UI only uses the unified multi-topic adder. Legacy non-topic `followees` are not shown.

## Neuron identity widgets used app-wide (`components/NeuronDisplay.jsx`, `NeuronContextMenu.jsx`, `NeuronNicknameDialog.jsx`, `NeuronInput.jsx`)
- **NeuronDisplay**: shows a neuron as name (verified ✓) / nickname / short id, colour-coded (setting), optional copy button, links to `/neuron`.
  - Right-click or long-press (touch) opens **NeuronContextMenu**: "Set Nickname" / "Edit Nickname", "View Neuron", "Copy Neuron ID".
  - "Set Nickname" opens **NeuronNicknameDialog** ("Private Neuron Nickname", max 32 characters, "Only visible to you. Leave empty to remove.", "Save Nickname") → backend `set_neuron_nickname`.
  - Mounted in: `/neuron` (followees), `/neurons`, `/me`, `/principal`, wallet `TokenCard`, `/sneedex_create`, `/sneedex_offer`, `/tools/sns_jailbreak_list`.
- **NeuronInput**: neuron id field with autocomplete over names and nicknames, tabs "Private" / "Public" / "All", "No matches in …" hint. Mounted in `/neuron` (twice), `/neurons`, `/names`.
- **formatNeuronDisplayWithContext** (`utils/NeuronUtils.js`): the link-style display used on `/proposals`, `/proposal`, `/active_proposals` (proposer, voters).
- **Redesign note**: the private nickname can be set from 3 places (`/neuron` inline, any `NeuronDisplay` context menu, `/names` address book). The public neuron *name* can only be set on `/me` → My Neurons ("Set Name").

## SNS proposal list (`/proposals`, `pages/Proposals.jsx`)
- **Routes & params**: `/proposals?sns=<root>`. Public to browse; voting needs login and hotkey neurons. Nav: Sneed Hub → "SNS Proposals".
- **Purpose**: browse, filter, export and quick-vote on an SNS's proposals.
- **Information shown**:
  - Title "<SNS> Proposals", "Review and vote on governance proposals", and "Create proposal →" (external, ic-toolkit.app).
  - Stats: "N proposals loaded", "N open for voting".
  - Each card:
    - Id link; status badge with icon; time remaining while open; action type label; title → `/proposal`; `VoteBar` (yes / no / not-voted % of all VP).
    - Proposer: `ProposerAvatar` (SNS logo + proposer's avatar), neuron display, proposer's principals with role badges (named first, "...N").
    - Treasury transfer details for `TransferSnsTreasuryFunds`: amount, token (ICP or SNS), target principal (named), memo.
    - Collapsible Markdown "Summary".
    - "Created <relative>", "N day voting".
    - External links: NNS dApp, IC Dashboard, ic-toolkit.
  - When logged in and open: your eligibility ("N neurons • X VP", or "Voted with N neurons • X VP", or "No eligible neurons").
- **Actions**:
  - "Filter by proposer..." (neuron id / name / nickname).
  - Type filter (labelled as topics but filters by action type): "All Topics", Motion, Manage Nervous System Parameters, Upgrade SNS to Next Version, Execute Generic Nervous System Function, Manage SNS Metadata, Transfer SNS Treasury Funds, Register Dapp Canisters, Deregister Dapp Canisters, Upgrade SNS Controlled Canister, Manage Dapp Canister Settings, Mint SNS Tokens, Manage Ledger Parameters, Add/Remove Generic Nervous System Function.
  - "Clear Filters".
  - "Load All" (fetches every proposal; becomes "All Loaded").
  - "Export CSV". Columns: id, title, topic, action type, status, proposer id/name/nickname, created/decided/executed/failed timestamps, voting period, treasury transfer amount/e8s/token/target/target name/nickname/memo, summary, raw payload, NNS URL, Dashboard URL.
  - "Show N per page"; "Load More Proposals".
  - Quick vote ✓ (Adopt) / ✗ (Reject), icon-only with tooltip "Adopt with N neurons (X VP)". Uses all your eligible hotkey neurons. `confirmVote` dialog, then `manage_neuron RegisterVote` per neuron.
- **Canister calls**: governance `list_proposals` (paged; filters are applied client-side to loaded pages only), `get_nervous_system_parameters`, `get_neuron` for proposer permissions (`getOrFetchNeuron`).
- **Shared**: `VoteBar`, `ProposerAvatar`, `confirmVote`, `NeuronsContext.getHotkeyNeurons`, `ProposalUtils`.
- **Redesign notes**:
  - Filters apply only to proposals already loaded (so "Load All" first).
  - The "topic" filter is really an action-type filter.
  - Creating SNS proposals sends you off-site to ic-toolkit.app.
  - No link to `/help/proposals`.

## SNS proposal detail, voting and discussion (`/proposal`, `Proposal.jsx`)
- **Routes & params**: `/proposal?proposalid=<n>&sns=<root>`. Public; voting needs login. Not in the nav.
- **Purpose**: read one proposal, vote with your neurons, see ballots, discuss it in the forum.
- **Information shown**:
  - Header "Proposal Details", "<SNS> Governance".
  - Proposal card (collapsible): "Proposal #n", status and icon, topic, title.
  - Created; "⏱️ Voting Ends" (time remaining) while open; Voting Period (days).
  - "Proposed by": `ProposerAvatar`, neuron display, proposer's principals with role badges.
  - Markdown Summary. Links: NNS, Dashboard, Toolkit, "Proposal URL" (the proposal's own url).
  - Payload (`payload_text_rendering`, plain text).
  - **Sneed only**: `ProxyPayloadView`. Decodes Sneed-proxy `ExecuteGenericNervousSystemFunction` payloads in the browser: action label, target canister and method, note, decoded arguments (current → proposed via `CandidValue`), and a check of the argument SHA-256 against the validator text ("do not trust the decoded values" when it doesn't match), plus the payload hash. Renders nothing until the proxy is deployed.
  - Voting bar: Yes % / VP, No % / VP, threshold markers (critical: "67% Threshold" and 20% minimum participation; normal: "50% Threshold" and 3%), "Current".
  - "Voting History (n)" (collapsible): ballots (vote, voter neuron, voter principals with role badges, VP, time). Comes from `ballots`, or for Sneed from RLL `get_proposal_ballots` when governance has cleared them.
- **Actions**:
  - "Previous" / "Next" (id ±1); "Proposal ID" field + "Go".
  - Changing SNS in the header refetches the same id in the new SNS.
  - Voting (login, open proposal): inline `HotkeyNeurons` panel (see next section).
  - Voting history: "Hide Yes" / "Hide No" / "Hide Pending", sort "By Date" / "By VP".
  - "Discussion" (collapsible). If a forum thread exists, an embedded `ThreadViewer` (read, post, vote on posts, tip; "Discussion for Proposal #n") plus a link to `/thread?threadid=&sns=`. If not, the `Discussion` component lets you write the first comment, which creates the thread (`forum.create_proposal_thread_with_auto_setup`).
- **Canister calls**: governance `get_proposal`, `get_nervous_system_parameters`, `get_neuron` (proposer and voter permissions); RLL `get_proposal_ballots`; forum `get_proposal_thread`. After a vote, refreshes silently and fires `votableProposalsRefresh`.
- **Entry points**: `/proposals`, `/active_proposals`, `/neuron` voting history, forum threads linked to proposals, ProposeDialog/MrSneed success links, external.
- **Shared**: `HotkeyNeurons`, `ThreadViewer`, `Discussion` (forum slice), `ProposerAvatar`, `ProxyPayloadView`, `CandidValue`.
- **Redesign notes**:
  - Mixes three concerns (governance detail, voting, forum).
  - Unlike the list, the detail page has no special rendering of treasury transfers (only the text payload).

## Inline SNS voting panel (`components/HotkeyNeurons.jsx`)
- **Where**: only `/proposal` (`inlineInProposal`). No other mount exists.
- **Information shown**:
  - "Vote with your neurons:" with "Adopt" / "Reject" (all eligible neurons).
  - "Show/Hide my neurons (n)": per neuron its VP, voted state, and its own "Adopt" / "Reject".
  - Messages: "No eligible neurons available. Either you've already voted or your neurons have no voting power." / "No eligible neurons available. Add your principal as a hotkey to your neurons to vote."
- **Actions**: vote with all, or with one neuron → `confirmVote` → governance `manage_neuron RegisterVote`. Result modal (`InfoModal`): "Vote submitted" / "Voting failed" / "No eligible neurons" with diagnostics (total neurons, with hotkey, already voted).
- **Dead or half-built**: the non-inline mode is never mounted:
  - "Your Hotkey Neurons" card with stats Total Voting Power / Distribution VP / Your Share (always 0 now: NeuronsContext stopped calling RLL).
  - "Add Your Principal as a Hotkey" how-to with copy-principal and "Check Hotkey Status".
  - Per-neuron details (Owner, Created, Dissolve State, Maturity, Staked Amount, Voting Power, Hotkey Access).
  - "Adopt All" / "Reject All", "Retry Adopt" / "Retry Reject".
  - The `forceSneedSns` prop.
  - So users without hotkey neurons get no in-app how-to on `/proposal`, although SKILL.md says the instructions are there.

## Cross-DAO voting inbox (`/active_proposals`, nav "Voting", `pages/ActiveProposals.jsx`)
- **Routes**: `/active_proposals`. Needs login ("Please log in to see your active proposals"). Nav: Sneed Me → "Voting". Also reached from the header votable badge and the `/me` tile "Voting".
- **Purpose**: every open proposal across all SNSes where you have neurons, plus the NNS, in one list.
- **Information shown**:
  - "Active Proposals — Vote on open proposals across all your SNS and ICP neurons".
  - Groups per DAO (collapsible): logo, name, "N proposals", "N votable", "N neurons with vote access". NNS group: "Internet Computer (NNS)".
  - Per proposal: status, time remaining, title → `/proposal` or `/icp_proposal`, `VoteBar`.
  - Proposer: SNS → `ProposerAvatar` + name + `ProposerPrincipals`; NNS → link "NNS neuron <id>" to the IC dashboard.
  - Eligibility: "Checking eligibility..." / "No eligible neurons" / "Voting..." / "Voted!" / "Vote failed"; per-group "All votes cast!" / "Some votes failed".
- **Actions**:
  - "Show only votable" (default on).
  - "Refresh".
  - Per proposal "Adopt" / "Reject" (all eligible neurons) and "Details".
  - Per DAO "Adopt All" / "Reject All": bulk vote on every votable proposal of that DAO, oldest first, with one `confirmVote` listing ids.
  - SNS: `manage_neuron RegisterVote`. NNS: `voteOnNnsProposal` (NNS governance `manage_neuron RegisterVote`). Session expiry is detected (`reportIfAuthExpired`).
- **Data**: WalletContext `neuronCache` (all SNSes plus NNS neurons readable by you); per SNS `list_proposals` limit 50 then filters to open; NNS `list_proposals` (open only).
- **Redesign notes**:
  - The only multi-DAO view. It sits under "Sneed Me" while the per-SNS and NNS lists sit under "Sneed Hub".
  - It uses a different neuron store than `/proposals`.

## NNS proposal list (`/icp_proposals`, `pages/IcpProposals.jsx`)
- **Routes**: `/icp_proposals`. Public; quick vote needs login and NNS neurons where your Sneed Hub principal is hotkey or controller. Nav: Sneed Hub → "ICP Proposals".
- **Information shown**:
  - "ICP Governance Proposals — NNS proposals for the Internet Computer", link "Looking for SNS proposals? Go to /proposals".
  - Stats (loaded / open).
  - Cards: id, status, topic, title → `/icp_proposal`, tally, proposer (known-neuron name + id), Markdown summary (collapsible), created / deadline, links NNS dApp and IC Dashboard, eligibility.
- **Actions**:
  - "Search by title or proposer..." (client-side).
  - Topic select ("All Topics" + NNS topics; client-side).
  - Status select ("All Statuses", Open, Rejected, Adopted, Executed, Failed; server-side).
  - "Clear Filters", "Load All", "Export CSV". Columns: Proposal ID, Title, Topic, Status, Proposer Neuron ID, Proposer Known Name, Created At, Deadline, Yes/No/Total Votes, Summary, NNS URL, Dashboard URL.
  - Per page; "Load More Proposals".
  - Quick ✓ / ✗ vote with all eligible NNS neurons (`confirmVote` → `voteOnNnsProposal`).
- **Canister calls**: NNS governance `list_proposals` (`omit_large_fields`, `before_proposal` paging), `list_known_neurons`, `list_neurons` (yours).
- **Redesign note**: a near-copy of `/proposals` (same layout, filters, CSV, quick vote) for a different backend.

## NNS proposal detail and vote (`/icp_proposal`, `pages/IcpProposal.jsx`, `components/IcpHotkeyNeurons.jsx`)
- **Routes**: `/icp_proposal?proposalid=<n>`. Public; voting needs login.
- **Information shown**:
  - "Internet Computer Governance". Back button "Back to ICP Proposals".
  - Created, Deadline, Time Remaining, Reward Status, Topic, Decided, Executed, Failed, "Total Potential VP", "Proposed by" (known name + id, copy), "Failure Reason".
  - Voting bar (50% threshold).
  - Summary (Markdown, collapsible).
  - "Action Details" (collapsible): Motion → "Motion Text"; ExecuteNnsFunction → "NNS Function ID", "Payload Size", "Payload (hex)" (copy, cut at 2000 characters); CreateServiceNervousSystem → SNS Name, Token Name, Token Symbol and a link; other types → raw JSON.
  - "Proposal URL". External: IC Dashboard, NNS Dapp.
  - "Voting History (n)" with adopt / reject / not-voted counts.
- **Actions**:
  - "Previous" / "Refresh" / "Next".
  - "View ICP Proposals" when no id is given.
  - Voting history "Hide Adopt" / "Hide Reject" / "Hide Pending", sort "By VP" / "By Vote", click a neuron id to copy it.
  - Voting via inline `IcpHotkeyNeurons`: "Vote with your ICP neurons:" "Adopt" / "Reject", "Show my neurons" with per-neuron Adopt/Reject → `confirmVote` → NNS `manage_neuron RegisterVote`. Message "No ICP neurons found. You need NNS neurons where you are the controller or hotkey to vote."
- **Dead or half-built**: `IcpHotkeyNeurons`' non-inline mode ("Add Your Principal as a Hotkey" how-to, "Check Neuron Status", stats, Adopt All / Reject All) is never mounted.
- **Note**: there is no NNS neuron detail page. NNS neurons can be viewed and managed only on the wallet ICP token card (and inside the ICP staking bot, another slice). Proposer links go to the IC dashboard.

## Liquid staking hub (`/liquid_staking`, `pages/LiquidStaking.jsx`)
- **Routes**: `/liquid_staking`. Public, static marketing/explainer. Nav: Liquid Staking → "Overview" (section also holds "ICP Liquid Staking" → `/create_icp_neuron` and "SNS Liquid Staking" → `/sns_neuron_wizard`).
- **Information shown**:
  - "Liquid Staking — Sneed DAO's Core Innovation".
  - ICP card ("Featured"): why NNS neurons aren't transferable; that the staking bot canister owns the neuron; "Important: You must create new neurons through your Staking Bot app canister. Existing neurons in your NNS wallet cannot be transferred into a manager."; 4 bullet features.
  - SNS card ("Native Support"): SNS neurons made via Sneed are natively transferable; "Existing neurons? Use the Jailbreak Wizard".
  - Sneedex section; "Why Liquid Staking Changes Everything" (Traditional vs Liquid lists); "Your Assets Are Always Safe".
- **Actions (links only)**:
  - Hero: "Start ICP Liquid Staking" → `/create_icp_neuron`; "Stake SNS Tokens" → `/sns_neuron_wizard`; Jailbreak → `/tools/sns_jailbreak`.
  - ICP card: "Create ICP Staking Bot" → `/create_icp_neuron`; "Learn More" → `/help/icp-neuron-manager`.
  - SNS card: "SNS Staking Wizard" → `/sns_neuron_wizard`; Jailbreak → `/tools/sns_jailbreak`.
  - Sneedex: "Browse Marketplace" → `/sneedex_offers`; "Create an Offer" → `/sneedex_create`; "How Sneedex Works" → `/help/sneedex`.
- **Entry points**: nav; promos on `/neuron` and `/neurons`; Hub tile; `/products`.
- **Redesign note**: no live data; doesn't link to `/help/liquid-staking`.

## SNS liquid staking wizard (`/sns_neuron_wizard`, `pages/SnsNeuronWizard.jsx`)
- **Routes**: `/sns_neuron_wizard`. Needs login ("Please log in to stake your SNS tokens" + login button). No URL params (uses the global SNS). Nav: Liquid Staking → "SNS Liquid Staking".
- **Purpose**: stake any SNS token into a new neuron owned by your Sneed Hub principal (so it stays transferable and sellable).
- **Steps** ("Select SNS" → "Configure" → "Stake"; completed steps can be clicked to go back):
  1. "Select an SNS to stake in:" — a searchable dropdown "Search SNSes..." (also sets the global SNS); "Your Balance:"; "Continue".
  2. "Configure Stake" — shows Min stake / Min dissolve / Max dissolve (from `get_nervous_system_parameters`); amount with "MIN" / "MAX" (MAX leaves room for the transfer fee); "Dissolve Delay (days):" with "MIN" / "MAX"; "Back" / "Continue".
  3. "Confirm & Stake" — review SNS / Stake Amount / Dissolve Delay; "Back" / "Stake <amount> <SYMBOL>".
- **Actions/calls**:
  - Silently registers the ledger in your wallet (backend `get_ledger_canister_ids` / `register_ledger_canister_id`).
  - Finds an unused nonce (governance), computes the neuron subaccount, `icrc1_transfer` to governance plus the subaccount (fee = token fee), journals the stake in localStorage (`stakeJournal`).
  - `manage_neuron ClaimOrRefresh` (memo and controller), then optional `Configure IncreaseDissolveDelay`.
  - Progress screen "Creating Your Neuron".
  - Success: "You've staked X in Y with N-day dissolve delay", "Stake Another", "Go to Wallet", "View Neuron" (→ `/neuron`).
  - On failure: `PendingStakesNotice` ("Finish staking" retries the claim).
- **Shared**: `PendingStakesNotice` (also on `/wallet` and the ICP neuron manager), `stakeJournal`, `icrcTx`.
- **Redesign notes**:
  - Duplicates the wallet token card's "Create New Neuron" dialog (same nonce / transfer / ClaimOrRefresh / dissolve flow, separate code).
  - The page title says "SNS Neuron Wizard" while the step 1 hero says "Liquid SNS Staking" and the login screen "Liquid SNS Staking Wizard".

## SNS Jailbreak wizard (`/tools/sns_jailbreak`, `pages/SnsJailbreak.jsx`)
- **Routes**: `/tools/sns_jailbreak`. Needs login ("Please log in to access the SNS Jailbreak Wizard"). In the nav only when logged in: Tools → "SNS Jailbreak". Premium-affected fee.
- **Purpose**: generate a browser-console script that, run on nns.ic0.app as the neuron's NNS-dapp controller, adds a chosen principal (default: your Sneed Hub principal) as a full controller of an SNS neuron. The neuron then becomes manageable, transferable and sellable from Sneed Hub.
- **Steps** ("SNS" → "Neuron" → "Principal" → "Generate" → "Done"):
  1. "Select an SNS". Only SNSes whose governance lets hotkeys be granted permissions 1–10 are offered. "Why are some SNSes not supported?" lists the unsupported ones with "Cannot grant: …". Info box "What is SNS Jailbreak?" (process, requirements: desktop browser console, Ctrl+Shift+J / Cmd+Option+J; "Is it safe?" → GitHub source).
  2. "Select Neuron":
     - "My Neurons" lists neurons where you already have any permission (i.e. you're a hotkey via NNS dapp).
     - "Enter Manually": "Neuron ID (hex string)", fetched to preview.
     - Help: "Option 1: Enter manually / Option 2: Add your Sneed Wallet as a hotkey in NNS".
     - "View neuron details" → `/neuron`.
  3. "Select Controller": `PrincipalInput` "Enter a principal ID...", "Use my principal".
  4. "Generate" — Summary (SNS, Neuron, Adding Controller), then Payment:
     - "Free! No payment required.", or "Script Fee:" with Premium Savings / "Premium Discount" / "Go Premium & Save!" → `/premium` "Learn More"; `PriceTierBadge` for special prices.
     - "Your Wallet Balance:" with refresh; "Deposited balance:" with "Use deposited balance" and "Withdraw".
     - "Insufficient ICP…" warning.
     - "Pay X ICP & Generate Script" or "Generate Script (Free)".
     - Progress: "Transferring ICP to payment account..." / "Creating script and processing payment..." / "Payment complete!".
     - "Script ready! You've already generated this script before." → "View all my scripts".
     - "Security Notice"; the script (fetched from GitHub raw `resources/sns_jailbreak/base_script.js`, "Retry") with copy.
     - Instructions (Open NNS App / Log In / Open Console / Paste & Run / Verify); "Back" / "I've Done It".
  5. "Done" — re-reads the neuron to verify: "Neuron Jailbroken!" / "Partially Verified" (missing ManagePrincipals / Vote / Disburse) / "Verification Failed"; "What You Can Do Now" list; "Click here to re-check", "Back to Script", "View Neuron", "Jailbreak Another". Also silently adds the SNS token to your wallet (backend `register_user_token`).
- **Canister calls**:
  - Governance `get_nervous_system_parameters` (support check), `list_neurons` / `get_neuron`.
  - Backend: `get_my_jailbreak_configs`, `get_my_jailbreak_fee`, `get_jailbreak_fee_settings`, `get_jailbreak_payment_subaccount`, `get_jailbreak_payment_balance`, `withdraw_jailbreak_payment`, `save_jailbreak_config` (charges the fee from the payment subaccount).
  - ICP ledger transfer from wallet to the payment subaccount when needed; special-price lookup.
- **Entry points**: Tools menu; `/tools/main` card; `/liquid_staking` (×3); `/sneedex`, `/sneedex_offers`, `/sneedex_create` (neuron needs Manage Principals); `/premium`; Hub tile; "Learn more" → `/help/sns_jailbreak`; "View my saved scripts" → `/tools/sns_jailbreak_list`.
- **Gating/mobile**: needs a desktop browser console (stated). Fee tiered by Premium or special prices. Admin config at `/admin/sns_jailbreak`.
- **Redesign note**: it lives in "Tools", but conceptually it is "import an existing SNS neuron into Sneed Hub" and belongs with neuron management and liquid staking.

## My jailbreak scripts (`/tools/sns_jailbreak_list`, `pages/SnsJailbreakList.jsx`)
- **Routes**: `/tools/sns_jailbreak_list`. Needs login. Nav (logged in): Tools → "My Scripts".
- **Information shown**: "My Jailbreak Scripts". Per saved config: SNS logo and name, Neuron (`NeuronDisplay`), Controller (`PrincipalDisplay`), date. Empty state "No saved scripts yet".
- **Actions**:
  - "Script" (opens a "Generated Script" modal: neuron, adding controller, script loaded from GitHub, "Retry", "Copy and paste this script into the NNS app browser console", copy).
  - "View Neuron" → `/neuron`.
  - "Delete" (confirm "Are you sure you want to delete this saved script?" / "Cancel") → backend `delete_jailbreak_config`.
  - "Back to SNS Jailbreak Wizard", "Create Jailbreak Script".
- **Calls**: backend `get_my_jailbreak_configs`, `delete_jailbreak_config`. Re-generating is free (already paid).
- **Redesign note**: the nav label "My Scripts" is ambiguous out of context.

## Sneed voting rewards (`/rewards`, `pages/Rewards.jsx`)
- **Routes**: `/rewards`. The page is public, but it shows content only after login ("Unlock Your Rewards… Hotkey your neurons to the app to automatically accumulate rewards from voting.", "Connect Wallet"). Nav: Sneed Me → "Rewards"; `/me` tile "My Rewards". Not the same as `/rll` (nav Sneed DAO → "Rewards").
- **Purpose**: see and claim the RLL voting rewards earned by Sneed neurons where you are a hotkey.
- **Information shown**:
  - "Sneed Rewards — Claim your voting rewards earned through DAO participation".
  - "N tokens to claim".
  - Unclaimed Rewards (USD total, "N tokens available"); Total Claimed ("Lifetime earnings", USD).
  - "Available Rewards": per token logo, symbol, amount, ≈ USD.
  - "No Rewards Available — Participate in Sneed DAO governance…".
  - "Claim History" (collapsible): claim events grouped by sequence with status, amount, USD, relative time, and a tx link → `/transaction?sns=&id=&ledger=`. Empty: "No claim history yet".
- **Actions**: per token claim → RLL `claim_full_balance_of_hotkey(token, fee)`. Refuses with a notification when the balance ≤ the token fee. "Claiming..." state; then re-reads balances.
- **Calls**: Sneed governance `list_neurons` (your hotkey neurons); RLL `balances_of_hotkey_neurons`, `get_claim_events_for_hotkey`; ledger `icrc1_metadata`; PriceService.
- **Duplicates**: the same claim exists on the wallet SNEED/other token cards ("Rewards" → Claim) and in the wallet "collect all collectibles" flow (`type: 'reward'`). Balances are cached by `hooks/useRewardsCache.js` (IndexedDB) for the wallet, not for this page.
- **Redesign notes**:
  - Two "Rewards" pages in two nav sections (`/rewards` claim vs `/rll` tokenomics/distribution).
  - No hotkey how-to for logged-in users with no rewards.
  - Admin: `/admin/rewards` (RLL import, distribution and main loop controls).

## Proposing changes to Sneed DAO from Sneed Hub (`governance/`, `ProposeDialog`, `ProxyPayloadView`)
- **What exists**:
  - `governance/registry.js`: a table of proposable settings routed through the not-yet-deployed **Sneed proxy** canister, by class (business / dapp / community / critical / treasury, each a generic function 6000–6004).
  - Actions:
    - Sneed Lock: lock fees ICP (4 values), max lock length.
    - Sneedex: marketplace fee rate, offer creation fee, Premium offer creation fee, Premium auction cut.
    - Premium: min claim interval.
    - Sneedapp: creation fee, Premium creation fee.
    - RLL: token min/max distribution.
  - Each entry names the page that owns the setting (`/sneedlock_info`, `/sneedex`, `/premium`, `/sneedapp`, `/rll`).
  - `governance/useProposeAction.js`: per action:
    - `canApply` (you are an admin of the target canister → apply directly).
    - `neurons` (your SNEED neurons with Submit Proposal; excluded ones with a reason: dissolve delay too short, or stake ≤ reject cost).
    - `functionId`; `draft` (title and Markdown summary with a current → proposed table, why, route, hashes, expiry); `validate` (the proxy validator text); `propose` (`manage_neuron MakeProposal ExecuteGenericNervousSystemFunction`).
  - `components/ProposeDialog.jsx`: the UI for that.
    - Fields: "Why (at least 20 characters)", "Note on chain (optional, 500 characters)", "Expires after (days, 0 = never)" (default 14), editable "Title" / "Summary (Markdown)", "Regenerate title and summary".
    - Preview "What voters will see (the proxy's validator)"; "From your neuron" select; "File proposal".
    - Success links to `/proposal?proposalid=&sns=Sneed`.
    - Warns when the function is not registered yet ("Proposals for this setting are not enabled yet").
- **Where "propose" actually appears in the app today**:
  - **Nowhere for the registry actions**: `ProposeDialog` and `useProposeAction` are not imported by any page or admin page. The pipeline is built but unwired.
  - `/mr-sneed` (`pages/MrSneed.jsx`) has its own local `ProposeDialog` (via `utils/mrSneed.js`): "Propose a change" (personality), "Propose removal" (memory), "Propose new writers", "Propose new limits". This is the only live in-app proposal filing.
  - `/guardian` (custody) and `/tools/pad` (token launcher) file `MakeProposal` but sit behind `ComingSoonPage`.
  - `/proposals` "Create proposal →" goes off-site (ic-toolkit.app).
  - Reading side: `/proposal` shows `ProxyPayloadView` for Sneed proposals (decodes proxy calls; inert until the proxy exists).
  - The ICP staking bot only names "MakeProposal" as a botkey permission (other slice).
- **Broken link**: drafted summaries include `https://app.sneeddao.com/proxy/verify?c=…&m=…&a=…`, but there is no `/proxy/verify` route (and no catch-all route).

## Where neuron management lives outside this slice's pages (mapping only)
- **Wallet token card → "Neurons" tab** (`TokenCard.jsx`, mounted in `/wallet`, `TokenCardModal`, `/lock`, `/tokenlock`, `/tokenlocksoverview`). Its own `manageNeuron` (not shared with `/neuron`):
  - SNS create neuron ("Create New Neuron": nonce check "Check Free", transfer, ClaimOrRefresh, optional dissolve delay; "Create Neuron").
  - Increase stake (transfer + ClaimOrRefresh).
  - Set/increase dissolve delay ("Set Dissolve Delay", "Confirm"); start/stop dissolving.
  - "Disburse to Wallet"; "✨ Disburse Maturity" (100%, confirm).
  - Auto-stake maturity toggle (`ChangeAutoStakeMaturity`).
  - "Split Neuron" (needs stake > minimum).
  - "Send Neuron" (add full permissions for the recipient, then remove your own).
  - Link to `/neuron`. Filters such as "All neurons hidden by filter".
  - NNS (ICP card): start/stop dissolving, disburse, disburse maturity (SpawnOrMergeMaturity), add/remove NNS hotkey ("Remove hotkey …?" confirm), Controller/Hotkey badge.
- **Wallet "collect all" collectibles** (`Wallet.jsx`, header collectibles badge, `hooks/useCollectiblesNotifications.js`): disburse SNS maturity per neuron and claim RLL rewards in bulk.
- **/me → "My Neurons" tab** (`pages/Me.jsx`):
  - Neurons per SNS (SNS tabs), grouped "My Neurons" / "Hotkey access to" (owner groups).
  - "Include reachable" toggle (localStorage), hide-empty toggle.
  - Per neuron: Created, Dissolve State, Maturity, Voting Power, Permissions.
  - "Set Name" (public neuron name, backend `set_neuron_name`; only place to set it) and "Set Nickname"; link to `/neuron`; "/help/neurons".
  - No dissolve or stake actions.
- **/principal** (another user's page): that principal's neurons per SNS (read-only `NeuronDisplay`).
- **/sneedex_create, /sneedex_offer**: sell or receive SNS neurons (adds Manage Principals for escrow; needs the SNS to allow it → Jailbreak link).
- **ICP staking bot** (`/icp_neuron_manager/:id`, `/create_icp_neuron`): NNS neurons inside a bot canister: stake, follow, dissolve, maturity, hotkeys, botkeys (other slice).

## Data plumbing shared by the slice
- **Three neuron stores, all writing the same IndexedDB `NeuronsDB`**:
  - `contexts/NeuronsContext.jsx`: your neurons for the selected SNS, "hotkey neurons". Used by `/proposals`, `/proposal`, `HotkeyNeurons`, `useSneedMembership`, `/neuron` (refresh), wizard.
  - `contexts/WalletContext.neuronCache`: your neurons for every SNS plus NNS, fetched on login. Used by the header VP bar, the votable badge, `/active_proposals`, `/me`, wallet, forum voting.
  - `hooks/useNeuronsCache.js`: every neuron of an SNS. Used by `/neurons`, `/hub`; single-neuron `getOrFetchNeuron` for `/neuron`, `ProposerAvatar` and `ProposerPrincipals`.
- `utils/voteConfirm.js`: one confirmation step before every SNS or NNS vote ("Your vote: Adopt on proposal #n…", bulk lists ids). Used by `/proposals`, `/active_proposals`, `/icp_proposals`, `HotkeyNeurons`, `IcpHotkeyNeurons`.
- `utils/ProposalUtils.js` (status, accepting votes, deadline, reward status); `utils/NnsUtils.js` (NNS governance actor, topics, statuses, known neurons, vote, add/remove hotkey, NNS neuron VP and dissolve state); `utils/NeuronUtils.js` (hex ids, permissions, owner indexing, display formatting, fetching a principal's neurons); `utils/neuronPermissionPlan.js`.
- `hooks/useOptimizedSnsLoading.js`: cache-first SNS list for `/proposals` and `/proposal`.
- `hooks/useRewardsCache.js`: RLL balances cache (wallet only).

## Slice observations

**(a) Entities, with their views and actions**

| Entity | Views (where) | Actions (where) |
|---|---|---|
| SNS (DAO) | `/sns` directory and details (params, permissions policy, canisters, price); header dropdown; `/active_proposals` groups | Select as current SNS (`/sns`, dropdown, wizard); links out to Proposals / Forum / Neurons / Transactions |
| SNS neuron (any) | `/neurons` list + CSV; `/neuron` detail; `NeuronDisplay` everywhere; `/principal` | Private nickname (`/neuron`, context menu, `/names`); copy id; view |
| SNS neuron (yours) | header VP bar; `/me` My Neurons; wallet token card Neurons tab; `HotkeyNeurons` on `/proposal` | Create (`/sns_neuron_wizard`, token card); increase stake, disburse, disburse maturity, auto-stake, split, send (token card only); dissolve delay / start / stop (`/neuron` and token card); principals/hotkeys (`/neuron`; token card for send); followees (`/neuron` only); public name (`/me` only); sell (`/sneedex_create`); import from NNS dapp (`/tools/sns_jailbreak`) |
| NNS neuron (yours, hotkey or controller) | wallet ICP token card; `IcpHotkeyNeurons` on `/icp_proposal`; `/active_proposals` | Vote (`/icp_proposals`, `/icp_proposal`, `/active_proposals`); dissolve, disburse, maturity, hotkeys (token card); no detail page |
| Hotkey / permission | `/neuron` Permissions; role badges on `/neurons`, `/proposals`, `/proposal`; `/sns` grantable-permission policy | Add / update / remove principal, presets "Full Owner" / "Hotkey" (`/neuron`); NNS add/remove hotkey (token card); Jailbreak adds a full controller |
| Followees | `/neuron` Followees | Add many neurons to many topics, remove (`/neuron`) |
| SNS proposal | `/proposals`, `/proposal`, `/active_proposals`; `/neuron` voting history | Filter / export CSV / load all; discuss (forum thread) |
| NNS proposal | `/icp_proposals`, `/icp_proposal`, `/active_proposals` | Filter / export CSV |
| Vote / ballot | `/proposal` voting history (+ RLL archive for Sneed), `/icp_proposal` voting history, `/neuron` voting history (Sneed) | Vote with all / per neuron (`/proposal`, `/icp_proposal`); quick vote (`/proposals`, `/icp_proposals`, `/active_proposals`); bulk "Adopt All" / "Reject All" per DAO (`/active_proposals`) |
| Sneed DAO proposal (filing) | — | Mr Sneed settings (`/mr-sneed`); registry/proxy settings (built, unwired); off-site for anything else |
| Proxy payload | `/proposal` (Sneed) `ProxyPayloadView` | — |
| Voting reward (RLL) | `/rewards`; wallet token card reward line; wallet collectibles | Claim per token (`/rewards`, token card, collect-all) |
| Jailbreak config / script | `/tools/sns_jailbreak` step 4, `/tools/sns_jailbreak_list` | Pay and generate, re-show script, verify, delete config, withdraw deposited ICP |
| Pending stake (journal) | `PendingStakesNotice` on `/wallet`, `/sns_neuron_wizard` (on error), ICP neuron manager | "Finish staking" (ClaimOrRefresh retry) |

**(b) Overlaps with other areas**
- **Neuron management is split** between `/neuron` (dissolve, principals, followees) and the wallet `TokenCard` (create, stake, dissolve, disburse, maturity, split, send, NNS ops). They have separate implementations, overlap on dissolve, and are reached from different sections. `/me` My Neurons adds naming but no management.
- **Proposal voting UI exists in 5 places with 3 implementations**: list quick vote (`/proposals`, `/icp_proposals`), the cross-DAO inbox (`/active_proposals`), and detail panels (`HotkeyNeurons`, `IcpHotkeyNeurons`). `/proposals` ↔ `/icp_proposals` and `/proposal` ↔ `/icp_proposal` are parallel copies for SNS vs NNS.
- **Neuron creation is duplicated**: the `/sns_neuron_wizard` and token card "Create New Neuron" run the same flow in two separate implementations. Liquid staking also spans `/create_icp_neuron` (staking bot slice) and Sneedex (selling).
- **Reward claiming in 3 places**: `/rewards`, the wallet token card, wallet collect-all. "Rewards" nav labels point to `/rewards` (Sneed Me) and `/rll` (Sneed DAO).
- **Forum overlap**: `/proposal` embeds the forum thread (`ThreadViewer` / `Discussion`). Header "reachable" VP is for forum voting.
- **Names overlap**: neuron nickname and name editing is spread across `/neuron`, `NeuronDisplay` menus, `/names`, `/me`.
- **SNS data overlap**: `/sns`, `/sns_neuron_wizard` and `/tools/sns_jailbreak` each load governance params and permission policy separately. `/neurons` "Stakers" overlaps user/holder lists (`/users`, `/principal`).
- **Nav placement is scattered**: SNS Proposals, ICP Proposals, Neurons, SNS under "Sneed Hub"; Voting and Rewards under "Sneed Me"; staking under "Liquid Staking"; Jailbreak and My Scripts under "Tools" (logged-in only); help pages under Help.

**(c) Dead, hidden, half-built or confusing**
- `components/ProposeDialog.jsx` and `governance/useProposeAction.js` are not mounted anywhere. The registry's in-place "Propose" buttons don't exist yet. The Sneed proxy isn't deployed (`ProxyPayloadView` is inert), and the drafted `/proxy/verify` link has no route.
- `HotkeyNeurons` and `IcpHotkeyNeurons` non-inline modes are dead (hotkey how-to, stats, Adopt All / Reject All). Their "Distribution VP" / "Total Voting Power" read fields that are now always 0. Prop `forceSneedSns` is unused. The result: there is no in-app "add your principal as a hotkey" guide on the voting pages (only the help pages); SKILL.md claims otherwise.
- `/neuron` has leftover "backwards compat" state and functions for single-topic and bulk following.
- `/neuron` advertises a "Stake Account (send tokens here to increase stake)" but has no refresh/claim button, so a direct transfer won't update the stake until the token card's "Increase Stake" (ClaimOrRefresh) is used.
- `/neuron` lacks disburse, split, maturity, send and sell, which users may expect on a neuron detail page.
- `/neuron` voting history and the `/proposal` ballot archive work only for Sneed (RLL canister). Other SNSes show none for closed proposals.
- `/proposals` "topic" filter is an action-type filter, and every filter applies only to loaded pages.
- `/sns` selection and the wizard's SNS picker change the global current SNS without saying so.
- The public neuron name ("Set Name") can only be set on `/me`. The private nickname can be set in 3 places.
- The NeuronsContext / WalletContext / useNeuronsCache stores can disagree. The header "Refresh neurons" button exists because caches drift after transfers, Sneedex claims or permission changes.
- Naming inconsistencies: "Voting" (nav) vs "Active Proposals" (page title); "SNS Liquid Staking" (nav) vs "SNS Neuron Wizard" / "Liquid SNS Staking" (page); "My Scripts" (nav) vs "My Jailbreak Scripts"; two "Rewards".
- No help cross-links from `/proposals`, `/active_proposals`, `/icp_proposals`, `/liquid_staking`, `/sns_neuron_wizard`. Help pages exist: `/help/proposals`, `/help/liquid-staking`, `/help/neurons`, `/help/sns_jailbreak`.
- There is no NNS neuron detail route. NNS proposers and voters link out to the IC dashboard.
