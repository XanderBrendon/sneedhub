# Admin area and Sneed Drive

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Sources read: `pages/Admin.jsx`, every file in `pages/admin/`, `pages/AdminNames.jsx`, `pages/AdminSnsJailbreak.jsx`, `pages/SmsAdmin.jsx`, `pages/Drive.jsx`, `hooks/useAdminCheck.js`, `utils/DriveService.js`, `utils/driveFlag.js`, `components/SpecialPricesCard.jsx`, `components/LaunchGate.jsx`, the admin menu in `components/Header.jsx`, the routes in `App.jsx`, plus every admin check found in non-admin pages. SKILL.md does not mention the admin area or Drive at all.

## How admin is determined (`hooks/useAdminCheck.js`) and the gate matrix

- **"Site admin"** means `app_sneeddao_backend.caller_is_admin()`. It is one global flag that lives on the backend canister. It is the only thing `useAdminCheck` checks.
  - The host is hard-coded to `https://ic0.app`.
  - Not logged in: after a 1 s wait it sets "Please connect your wallet first." and redirects to `redirectPath?from=<path>`.
  - Not an admin: it sets "You do not have admin privileges." and redirects after 2 s. The default `redirectPath` is `/`, and `redirectPath: null` turns redirecting off (silent check).
- **Every product canister keeps its own admin list.**
  - The site-admin check only lets you into the page. The canister itself decides whether a write is allowed.
  - Some pages run a second check against the canister's own list. Others only warn ("You are not on sneed_mon's admin list…").
- Gate per page:

| Page | Front-end gate | Redirect | Renders before the check? | Second, canister-level check |
|---|---|---|---|---|
| /admin | backend admin | /wallet | no | – |
| /admin/users/bans, /admin/words | backend admin (return value ignored) | / | **yes, flashes for ~2 s** | backend |
| /admin/partners | **none** (reads use an anonymous actor) | – | anyone can view | backend on write |
| /admin/projects | login only ("Access Denied… connect your wallet") | – | any logged-in user | backend on write |
| /admin/names | **none** | – | anyone | backend / naming canisters on write |
| /admin/donations, /admin/feedback | backend admin | /admin | no | backend |
| /admin/forum | backend admin **and** forum `is_admin(me)` | /wallet | no | forum canister |
| /admin/sms | **SMS canister `is_admin_query` only** (no backend check) | – | no | SMS |
| /admin/sneedlock, /admin/neuron-manager-factory, /admin/pad, /admin/profits | backend admin (return value ignored), login required | / | **yes, flashes** | each canister's own list |
| /admin/sneedapp | backend admin | / | no | sneedapp |
| /admin/sneedex, /admin/premium | backend admin **or** that canister's admin list | /admin | allowed, but the redirect still bounces anyone who is only a canister admin | canister |
| /admin/canisters | backend admin | /wallet | no | backend |
| /admin/network, /admin/tokens, /admin/admins, /admin/sns_jailbreak | backend admin | /admin | no | each target canister |
| /admin/rewards | **RLL canister `caller_is_admin` only** (no backend check) | /admin | no | RLL |
| /admin/rll | backend admin; the RLL `caller_is_admin` then switches the actions on | /admin | no | RLL |
| /admin/mon | backend admin; warns when you are not on sneed_mon's list | /admin | no | sneed_mon |
| /drive | backend admin **plus** a per-browser localStorage flag | / | no | sneed_drive (owner and grants) |

## Navigation: Header admin menu vs dashboard vs URL-only

- The Header shows an "Admin" section only when the user is logged in and `isAdmin === true`. It does a silent `useAdminCheck` on every page. On `/admin*` and `/drive` the section title "Admin" shows even while that check is still loading.
- **Header admin submenu:** Dashboard, User Bans, Word Blacklist, Partners, Names, Donations, Feedback, Projects, SneedLock, Sneedapp, Sneed Pad, Sneedex, "RLL Rewards" (→ /admin/rll), Sneed Monitor, SNS Jailbreak. "Sneed Drive" (→ /drive) is added only when the Drive beta flag is on in this browser.
- **Reachable only from dashboard cards or by URL, not from the Header:**
  - /admin/forum, /admin/sms, /admin/neuron-manager-factory ("ICP Staking Bot Factory")
  - /admin/premium, /admin/canisters, /admin/network, /admin/profits
  - /admin/tokens ("Token Whitelist"), /admin/admins, /admin/rewards ("Rewards (RLL)")
- **Dashboard cards:**
  - Every one of the 24 `/admin/*` sub-routes has a card.
  - There is also a "Sneed Drive (beta)" card. It holds a switch, not a link, and shows an "Open Sneed Drive" link once the switch is on.
- **Cross-links between admin pages:**
  - /admin/mon → /admin/admins
  - /admin/rll → /admin/rewards and /rll
  - /admin/donations → /donate
  - /admin/feedback → /feedback
  - /admin/pad launch rows → /studio/:ledger
- **Links into admin from user pages:**
  - /sneedex shows an admin card linking to /admin/sneedex, but only to people on the Sneedex canister's admin list.

## /admin: Admin Dashboard
- **Who:** backend admin (redirects to /wallet).
- **Purpose:** a launcher grid of cards, one per admin page.
- **Information shown:** card titles and one-line descriptions only. There are no stats.
- **Actions:**
  - A link per card.
  - The "Sneed Drive (beta)" switch, which writes localStorage `sneedDriveBeta=1`, then "Open Sneed Drive".
- **Redesign notes:**
  - The only place that lists all admin pages.
  - Card names differ from the Header names, e.g. "ICP Staking Bot Factory" vs nothing in the Header, "Rewards (RLL)" vs "RLL Claims & Cycle", and "Token Whitelist" vs the /admin/tokens route.

## /admin/users/bans: User Bans
- **Who:** backend admin (soft gate).
- **Canister:** `app_sneeddao_backend`.
- **Purpose:** ban and unban principals. Bans block setting names: the backend is still the moderation authority for the naming canisters through `names_moderate`.
- **Information shown:**
  - Table of banned users: Principal ID, Expiry, Actions.
  - Clicking a row expands the user's "Ban History": each entry's timestamp, Banned/Unbanned, reason, and expiry.
- **Actions:**
  - "Ban User" (`ban_user(principal, hours, reason)`). Duration units: Hours, Days, Weeks, Months, Years.
  - "Unban" (`unban_user`).
- **Redesign notes:**
  - Pairs with Word Blacklist and Names as one "moderation" group.
  - No search, and no link to the user's profile.

## /admin/words: Word Blacklist
- **Who:** backend admin (soft gate).
- **Canister:** backend.
- **Purpose:** maintain the list of blocked words. A public name containing one is refused and auto-bans the user; a nickname containing one is just refused (per HelpNames).
- **Information shown:** the list of blacklisted words.
- **Actions:**
  - "Add Word" (`add_blacklisted_word`).
  - "Remove" (`remove_blacklisted_word`).
- **Redesign notes:** part of moderation, together with Bans and Names.

## /admin/partners: Manage Partners
- **Who:** no front-end gate. The backend enforces writes.
- **Canister:** backend.
- **Purpose:** keep the ecosystem-partner directory shown on the public /partners page.
- **Information shown:** partner cards with name, logo, description, and links.
- **Actions:**
  - "Add New Partner" opens the form: Partner Name, Logo URL, Description, optional Index (display order), and Links (title + URL; "Add Link", "Remove").
  - "Add Partner" / "Update Partner" (`add_partner` / `update_partner`), "Cancel".
  - "Edit".
  - "Delete" (`remove_partner`, with a `window.confirm`).
- **Redesign notes:**
  - Almost the same model and UI as Projects.
  - Name clash with the "Partners" list inside SpecialPricesCard, which means price partners on each canister.

## /admin/projects: Manage Projects
- **Who:** login only. The backend enforces writes.
- **Canister:** backend.
- **Purpose:** keep the project directory shown on the public /projects page.
- **Information shown:** "Existing Projects" with name, type, logo, description, and links.
- **Actions:**
  - The form: Project Name, Logo URL, Project Type (Product / Project / Fork), Description, Index, Links.
  - "Add Project" / "Update Project" (`add_project` / `update_project`), "Cancel".
  - "Edit".
  - "Delete" (`remove_project`, with a confirm).
- **Redesign notes:** could be merged with Partners into one "Directory" admin with a type field.

## /admin/names: Admin - Names & Nicknames (`pages/AdminNames.jsx`)
- **Who:** **no front-end admin check at all.**
- **Writes go to:** the naming-service canisters when `namingServiceEnabled`, otherwise the backend.
- **Purpose:** moderate the public names of principals and neurons, and set the limits on nicknames.
- **Tab "📝 Names":**
  - **Information shown:** every principal name and neuron name from NamingContext (neuron names belong to the selected SNS), each marked VERIFIED or UNVERIFIED.
  - Search "by name or ID".
  - Filters: All Types / Principals / Neurons, and All Status / Verified / Unverified.
  - 25, 50 or 100 per page, with Previous / Next.
  - **Actions:**
    - "Edit" → "Save" / "Cancel". Saving sets the name for that principal or neuron (`setPrincipalNameFor` / `setNeuronName`), with the 32-character limit and charset check.
    - The VERIFIED / UNVERIFIED button toggles verification (`verify*` / `unverify*`).
- **Tab "⭐ Nicknames Config"** (backend):
  - **Information shown:** "Sneed Premium Canister ID", "Max Neuron Nicknames", "Max Principal Nicknames", and the premium versions of both limits.
  - **Actions:** "Update Configuration" (`set_nickname_premium_canister` + `update_nickname_limits`).
- **Redesign notes:**
  - The premium canister ID can be set here and in 6+ other places (see Slice observations).
  - Nickname limits stay on the backend even though nicknames now live in the `sneed_pid_nicknames` / `sneed_neuron_nicknames` canisters. Whether those limits are still enforced is not verified in the UI.
  - The naming canisters' own admin methods have no UI: `admin_set_config`, rate limits, imports, ICRC-3 settings, and `admin_clear_avatar` (`NamingService.adminClearAvatar` has no caller).

## /admin/donations: Donations
- **Who:** backend admin.
- **Canister:** backend (`utils/donations.js`).
- **Purpose:** choose where donations go, and moderate the thank-you wall on /donate.
- **Information shown:**
  - The current donation account (principal + subaccount), or "None set: the donate dialog says donations are not open".
  - The wall: total count, then the latest 100, newest first. Each row has donor, amount + token, #id, block, message, and whether it is hidden. Hidden donations still count in the totals.
- **Actions:**
  - Edit the Principal and the optional Subaccount (64 hex).
  - "Save" (`admin_set_donation_account`, confirmed in a modal).
  - "Stop donations" (sets it to none).
  - "Hide" / "Show" per donation (`admin_set_donation_hidden`).
- **Redesign notes:** a "where funds go" setting; one of about 8 fee/recipient-account settings spread over the admin pages.

## /admin/feedback: Feedback
- **Who:** backend admin.
- **Canister:** backend settings (`utils/feedback.js`).
- **Purpose:** decide where reports from the /feedback page go.
- **Information shown:**
  - The Bugs topic and the Ideas topic: dropdowns over the Sneed DAO forum's topics (shown as "parent › child"), or None.
  - Recipients: at most 5 principals. They get a Sneed SMS for every report and receive security reports exclusively.
  - "This backend has no feedback settings yet (it needs the upgrade)" when the backend has none.
- **Actions:**
  - "Add a recipient", and an × button per recipient.
  - "Save" (`admin_set_feedback_config`).
- **Redesign notes:** connects the forum and SMS. Could sit with the Forum and SMS settings.

## /admin/forum: Forum Administration
- **Who:** backend admin **and** an admin of the forum canister (`sneed_sns_forum.is_admin`). Without the second it shows "You are not an admin in the forum canister…".
- **Purpose:** full CRUD and moderation of the forum hierarchy, plus forum settings.
- **Tabs:**
  - **Forums / Topics / Threads / Posts**
    - Lists come from the `*_admin` getters, so deleted items show with "[DELETED]".
    - Drill down with "Select".
    - Buttons: "Create Forum" / "Create Topic" / "Create Thread" / "Create Post", "Create", "Update", "Edit", "Delete", "Undelete", "Cancel".
    - Topics also have "Set as Proposals Topic" / "Current Proposals Topic" (`set_proposals_topic`). Proposal discussions are created under that topic.
  - **Stats**
    - "Forum Statistics & Admin Management": Total Forums, Topics, Threads, Posts, Votes.
    - Forum Admins: "Add Admin", "Remove", "Refresh".
  - **Text limits:** the maximum lengths of post title and body, thread title and body, topic title and description, and forum title and description. "Update Text Limits" (`update_text_limits`).
  - **Premium:** Sneed Premium Canister ID, Premium Post Body Max Length, Premium Thread Body Max Length. "Update Premium Config".
- **Redesign notes:**
  - Inline moderation also exists on /thread, /post and /proposal (see the outside-/admin section). Those pages use the **backend** admin flag, while this page needs the **forum** admin list.
  - Forum admins are also managed on /admin/admins.

## /admin/sms: SMS Administration (`pages/SmsAdmin.jsx`)
- **Who:** an admin of the SMS canister only (`sneed_sms.is_admin_query`). There is no backend check, and the Header shows no admin menu to someone who is only an SMS admin.
- **Purpose:** messaging limits, premium limits, and who may send system notifications.
- **Information shown:**
  - Stats: total messages, total users.
  - Config: Rate Limit (minutes between messages), Max Subject Length, Max Body Length, Max Recipients.
  - Premium config: Sneed Premium Canister ID, and the premium versions of max subject, max body, rate limit, and max recipients.
  - "🔔 System Notifications": the System Sender Principal, and the list of Authorized System Senders.
- **Actions:**
  - "Update Configuration" (`update_config`).
  - "⭐ Update Premium Configuration" (`update_premium_config`).
  - "Save" for the system sender (`set_system_sender_principal`).
  - "Add Sender" / "Remove" (`add_authorized_sender` / `remove_authorized_sender`).
- **Redesign notes:**
  - The system sender is also editable on /admin/network.
  - SMS admins can read private messages: user pages (SmsConversations) disclose this.

## /admin/sneedlock: SneedLock Administration
- **Who:** backend admin (soft gate), login required.
- **Canister:** `sneed_lock`, which has its own admin list.
- **Purpose:** lock-product logs, fees, recipient, and admins.
- **Tabs:**
  - **Legacy Queue:** "Legacy Claim Queue (retired, read-only)". It is always empty, because claims are synchronous now.
  - **Completed / Failed:** paged claim requests (`get_claim_requests_page`) with each request's status and details. "Clear All" (`admin_clear_completed_claim_requests` / `admin_clear_failed_claim_requests`), and "Refresh".
  - **Payments:**
    - "Payment Statistics": Total Payments, Total ICP Collected.
    - "Payment Log" (`admin_get_payment_log`): id, payer, lock type, amount, ICP tx, time. "Load Payment Log", "Refresh", "← Newer" / "Older →".
  - **Info Logs / Error Logs:** ranged log entries, with "← Newer" / "Older →".
  - **Admin Functions:** "ICPSwap Pool Balance Recovery". Enter a swap pool canister ID, then "Check Balance" (`admin_get_pool_unused_balance`) shows the pool's unused token0 and token1. It only checks; there is no action to recover the funds.
  - **Settings:**
    - "SneedLock Admins": "Add Admin" / "Remove Admin".
    - "🔶 Sneed Premium Integration": "Set Premium Canister".
    - "💰 ICP Lock Fees": the token and position lock fees, regular and premium, in e8s. "Update ICP Fees" (`admin_set_lock_fees_icp`).
    - SpecialPricesCard: admin and partner prices for the token and position lock fees.
    - "🏦 Fee Recipient Account": principal + subaccount. "Update Fee Recipient" (`admin_set_icp_fee_recipient`).
    - "📊 Fee Collection Statistics": Token Lock Fees, Position Lock Fees, Total Collected.
- **Redesign notes:**
  - The fee statistics also appear on /admin/profits and the public /revenue page.
  - The Legacy Queue tab is dead weight.

## /admin/neuron-manager-factory: ICP Staking Bot Factory Admin
- **Who:** backend admin (soft gate), login required.
- **Canister:** `declarations/sneedapp`. This is the **same canister as /admin/sneedapp**.
- **Purpose:** the payment and fee settings of the original staking-bot factory, its manager WASM and versions, and its creation log.
- **Tabs:**
  - **Payment Config:**
    - Factory Balances: cycles, ICP, manager count.
    - Payment Requirement: "Enable Payment" / "Disable Payment".
    - Current CMC Conversion Rate.
    - Fee Settings: Creation Fee (ICP), Target Cycles (Trillion).
    - Fee Destination: principal + subaccount. "Update Configuration".
    - Canister Creation Cycles: "Update Cycles".
    - "👑 Sneed Premium Discount": Premium Creation Fee and Sneed Premium Canister ID. "Update Premium Configuration".
  - **Admins:**
    - "Current Admins": "Add Admin" / "Remove Admin".
    - "Sneed Governance": "Set Governance" / "Clear Governance".
  - **Operations:**
    - "Top Up Factory Cycles": ICP Amount to Convert, then "Top Up Cycles" (`adminTopUpCycles`).
    - "Withdraw ICP" (`adminWithdrawIcp`).
    - "Manager WASM Module": Select WASM File + version, then "Upload WASM" (`setManagerWasm` + `setCurrentVersion`). "Clear WASM".
  - **Managers:** "All Registrations" shows every owner → canister (`getAllRegistrations`). "Load Registrations".
  - **Versions:**
    - "Official Versions": add or edit Major / Minor / Patch, WASM Hash (SHA256), WASM URL, Source URL.
    - "Add Version" / "Update Version", "Edit", "Remove", "Cancel".
  - **Creation Log:**
    - Filters: Caller Principal, Canister ID, From Date, To Date. "Search", "Reset", "Load / Refresh".
    - Table: #, Canister ID, Creator, Created At, ICP Paid, Profit, Cycles. It uses `getMergedLog` and falls back to `getCreationLog`.
  - **📊 Stats:** "Factory Statistics" with ICP Breakdown, Cycles Breakdown, and Per-Canister Averages. "Load Statistics".
- **Redesign notes:**
  - **Heavy overlap with /admin/sneedapp.** Payment config, fees, creation cycles, premium discount, admins, governance, and factory stats all appear on both pages.
  - Only here: the manager WASM, official versions, all registrations, the creation log, cycle top-up, and ICP withdrawal.
  - The single "manager WASM" model is probably the legacy path, superseded by per-app versions on /admin/sneedapp (not verified).
  - Not in the Header menu.

## /admin/sneedapp: Sneedapp Admin
- **Who:** backend admin.
- **Canister:** `sneedapp` (the app store and bot factory).
- **Purpose:** run the app store: publishers, apps, versions and WASMs, the mint log, revenue, and network/payment settings.
- **Tabs:**
  - **Publishers:**
    - List: name, verified flag, owners, families, payment account, DAO cut.
    - "Verify" / "Unverify".
    - Edit Name, Website URL, Description, and Default Payment Account (ICRC-1). "Save", "Cancel".
    - "Set DAO Cut %:" → "Set".
    - Owners: add a "Principal to add" with "Add", or remove one.
    - Families: add a "Family tag", or remove one.
    - "Create Publisher" (Name, Website URL, Description, Default Payment Account).
    - Calls: `createPublisher`, `updatePublisher`, `add/removePublisherOwner`, `add/removePublisherFamily`, `verify/unverifyPublisher`, `setPublisherDaoCut`.
  - **Apps:**
    - "Registered Apps" list.
    - Edit Name, Icon URL, Description, Mint Price (ICP), Premium Price (ICP), View URL, Manage URL, Custom Mint URL, Families.
    - "Enabled" / "Disabled" toggle (`setAppEnabled`), DAO cut (`setAppDaoCut`), "Edit".
    - "Add New App": Publisher, App ID (slug), Name, …
    - Removal via `removeApp`.
  - **Versions:**
    - Select an app; "Versions" list.
    - "Add New Version": Major, Minor, Patch, WASM Hash, Release Date, WASM URL, Source URL, Release Notes.
    - WASM per version: "Upload", "Download & Upload" (fetches the WASM URL, then uploads), "Clear" (`uploadAppVersionWasm` / `clearAppVersionWasm`).
    - Edit or remove a version.
  - **Mint Log:**
    - "Immutable Mint Log (N entries)": #, Canister, Minter, App, Publisher, Version, ICP Paid, DAO Cut, Pub Revenue, Premium, Date.
    - "Search", "Previous" / "Next".
  - **Revenue:**
    - "Revenue Overview": Sneed DAO revenue (Total Revenue, From DAO Cut, Direct (Publisher 0)).
    - "Publisher Stats": Publisher, Total Revenue, Withdrawn, DAO Cut, Mints.
  - **Network:**
    - Factory Balances, Payment Requirement ("Enable Payment" / "Disable Payment"), CMC Conversion Rate.
    - Fee Settings: Creation Fee, Target Cycles, Fee Destination principal and subaccount.
    - Canister Creation Cycles: "Update Cycles".
    - Premium Discount: "Update Premium Config".
    - SpecialPricesCard: "Mint fee (Sneed DAO apps)".
  - **Admins:** "Current Admins" with add / "Remove", and "Sneed Governance" "Set" / "Clear".
  - **Stats:** Factory Statistics with ICP / Cycles Breakdown and Per-Canister Averages. Identical to the factory page.
  - **Migration:**
    - "Register Canisters for Users": User Principal, Canister ID, App ID (`adminRegisterCanister`).
    - "Bulk Register (JSON)": a JSON array of `[user, canisterId, appId]` (`adminBulkRegisterCanisters`).
- **Redesign notes:**
  - Duplicates most of /admin/neuron-manager-factory.
  - Publishers manage some of the same things themselves on /sneedapp/publisher/:id: owners, families, app versions and WASM uploads, enabling apps, and withdrawals.

## /admin/pad: Sneed Pad admin
- **Who:** backend admin (soft gate), login required.
- **Canister:** `sneed_pad`, read through a hand-written IDL. Its own admin list is shown read-only in the header line.
- **Availability:** staging only. Its canister ID is in `canister_ids.json` for staging only, so on prod the page says "not configured in this build".
- **Purpose:** the token launchpad's fees, cycles, ledger WASM source, launches, and rescue.
- **Information shown:**
  - Canister ID, cycle balance, admin list.
  - "Ledger code":
    - Source: following the NNS or pinned.
    - Last check, last change, last error.
    - Ledger and index WASM hashes and sizes.
  - "Launches (N, x failed…)", paged with "Load older launches". Each row has logo, #id, name and symbol, creator, created time, step trail, ledger link (to /studio), index ID, and failure reason.
  - "Log (last N lines)": the text log.
- **Actions:**
  - "Check the NNS now" (`refresh_wasms`).
  - "Settings":
    - Launch fee, Sneed Premium fee, Sneed DAO handover fee (ICP).
    - Ledger cycles and Index cycles (T), Pad's own reserve (T).
    - Fee destination.
    - "Paused (no launches)", "Follow the NNS's latest ledger suite".
    - "Custody canisters a creator may hand control to" (guardians).
    - "Save settings" (`admin_set_config`).
  - SpecialPricesCard: launch fee, index fee, DAO handover fee.
  - "Rescue a canister": Canister + New controllers, "Use my principal", "Set controllers" (`admin_set_canister_controllers`). Failed launch rows have "Rescue ledger" / "Rescue index" buttons that prefill this form.
  - "Registration neuron and ICP":
    - Neuron ID (64 hex) → "Set registration neuron".
    - Withdraw ICP left by failed steps: to + amount → "Withdraw ICP".
- **Redesign notes:**
  - The regular index fee and premium index fee are saved back unchanged. There are no inputs for them.
  - The user-facing pad (/tools/pad) is behind the "coming soon" gate, where only site admins see it.

## /admin/sneedex: Sneedex Admin
- **Who:** backend admin, or an admin of the Sneedex canister. The `useAdminCheck` redirect still bounces anyone who is only a Sneedex admin.
- **Canister:** `sneedex`.
- **Purpose:** marketplace fees, recipients, asset types, timers, integrations, and NFT rescue.
- **Information shown:**
  - "Marketplace Overview": Total Offers, Active Offers, Completed, Total Bids, Fee Rate.
  - "Payment Statistics": Creation Fee Payments, Creation Fees Collected, Cut Payments.
  - "Cuts Collected by Token".
  - "Payment Logs": the creation-fee log and the cut log, with Previous / Next.
- **Actions:**
  - "Marketplace Fee Rate" → "Save".
  - "Default Fee Recipient Account" → "Save Recipient".
  - "Offer Creation Fee" → "Save Fee".
  - "👑 Sneed Premium Settings": premium creation fee, premium auction cut (bps), Sneed Premium Canister ID → "Save Premium Settings".
  - "Min Bid Increment Defaults": Range Min, Target, Range Max (USD), Fallback → "Save Increment Settings".
  - "Per-Token Fee Recipient Overrides": "Add Override", remove one.
  - "Sneedex Admins": "Add Admin", remove one.
  - "General Configuration": Minimum Offer Duration, Max Assets Per Offer → "Save Configuration".
  - "Asset Types": add, or "Deactivate" (Active / Inactive).
  - "Wallet Registration Integration": Backend Canister ID, Staking Bot Factory Canister ID, Sneed SMS Canister ID, each with "Set" / "Clear".
  - "Expiration Auto-Processing Timer": "Start Timer" / "Stop Timer", Check Interval → "Set", "Trigger Now".
  - "Return a misplaced NFT": collection, token ID, return-to principal (`adminReturnMisplacedNft`).
  - SpecialPricesCard: offer creation fee, auction cut.
- **Redesign notes:**
  - The integration IDs duplicate /admin/network.
  - The premium canister ID duplicates /admin/premium and /admin/network.

## /admin/premium: Sneed Premium Admin
- **Who:** backend admin, or an admin of the Premium canister. Same redirect caveat as Sneedex.
- **Canister:** `sneed_premium`. It also reads and writes backend, SMS, forum, lock, and Sneedex.
- **Purpose:** membership tiers, members, promo codes, payments, and spreading the premium canister ID to the other canisters.
- **Information shown:**
  - "Configuration Overview": canister ID, ICP Ledger, Sneed Governance, Active Members.
  - "Propagate Premium Canister ID": the status of each consumer (backend nicknames, SMS, Forum, Lock, Sneedex). "Refresh Status".
  - "Premium Settings Overview": a read-only table of each consumer's settings (Setting, Regular, Premium). It covers Backend Nicknames, Backend Canister Groups, Sneed SMS, Sneed Forum, Sneed Lock, and Sneedex. "Refresh Settings".
  - "Payment Statistics & Logs":
    - "All Claims (N)": ID, Time, Type, Claimant, Tier / Code, Duration, Details.
    - "ICP Payments (N)": ID, Time, Payer, Amount, Net Amount, Tier, Duration, ICP Tx.
    - Previous / Next.
  - "Current Members (N)", with each member's expiry.
  - "Promo Codes (N)".
- **Actions:**
  - "Propagate to All Canisters" (`set_nickname_premium_canister`, SMS/forum `update_premium_config`, lock `admin_set_sneed_premium_canister_id`, `sneedex.setSneedPremiumCanisterId`).
  - "Payment Recipient" → "Save Recipient".
  - "Voting Power Claim Settings": the minimum claim interval → "Save".
  - "ICP Payment Tiers": Tier Name, ICP Amount, Duration → "Add Tier". Edit ("Save Changes", Active toggle) or remove.
  - "Voting Power Tiers": the same, with Minimum Voting Power in SNEED.
  - "Premium Admins": "Add Admin", remove one.
  - "Grant Membership Manually": principal + duration → "Grant Membership" (`extendMembershipAdmin`). Revoke from the member list (`revokeMembership`).
  - "Create Promo Code": Membership Duration, Max Claims, optional Expiration Date, admin-only Notes → "Generate Promo Code".
  - Per code: deactivate or reactivate, delete, and "Claims" (the claims modal).
  - SpecialPricesCard: "Share of every ICP tier price" (bps).
- **Redesign notes:**
  - The settings overview only reads; every value in it is edited on another page (Names, Canister Groups, SMS, Forum, Lock, Sneedex).
  - Propagation is duplicated on /admin/network.
  - Not in the Header menu.

## /admin/canisters: Canister Groups Admin
- **Who:** backend admin (redirects to /wallet).
- **Canister:** backend.
- **Purpose:** the limits on the folders users create on /canisters.
- **Information shown and edited:**
  - Regular: Max Folders, Max Canisters per Folder, Max Total Canisters.
  - Premium: the same three limits.
- **Actions:** "Update Configuration" (`update_canister_groups_limits`).
- **Redesign notes:** a one-form page. Could be merged into a "Limits and quotas" page with the nickname, SMS, and forum limits.

## /admin/network: Network Configuration
- **Who:** backend admin.
- **Purpose:** wire the canisters together by setting cross-canister IDs.
- **Information shown:**
  - A canister-ID reference panel, with copy buttons, for Backend, Sneed Premium, Sneed Lock, SNS Forum, SMS, Staking Bot Factory (= sneedapp), Sneedex, and Rewards (RLL), plus the external ICP Ledger, Sneed Governance, and Sneed Ledger.
  - A card per canister showing its current settings. Each value is labelled with the known canister it points to.
- **Actions** (edit inline per row, then ✓ Save or Cancel):
  - Backend: premium ID.
  - SneedLock: premium ID.
  - Forum: premium ID.
  - SMS: premium ID, and the System Sender Principal. "Authorized System Senders" is read-only here.
  - Premium: the ICP Ledger ID and the Sneed Governance ID.
  - Factory: Sneed Governance, premium ID.
  - Sneedex: premium ID, Backend ID, Factory ID, SMS ID.
  - "Propagate Premium ID", "Refresh".
- **Redesign notes:**
  - Each of these values is also editable on the canister's own admin page.
  - Pad, Monitor, Drive, Guardian, the naming canisters, and Notifications are missing here.
  - Not in the Header menu.

## /admin/profits: Profits Dashboard
- **Who:** backend admin (soft gate), login required.
- **Purpose:** read-only revenue per product, with USD estimates from PriceService.
- **Information shown:**
  - "Total Profits Summary".
  - SNS Jailbreak: scripts created, revenue, unique users, premium vs regular.
  - Sneedapp: canisters created, creation fees, ICP used for cycles, total ICP profit, DAO revenue (tracked), and "Revenue by App".
  - Sneedex: creation fee payments, cut payments, and "Marketplace Cuts by Token".
  - Sneed Lock: token, position, and total lock fees.
  - Sneed Premium: ICP payments, total ICP collected, VP claims, promo claims.
- **Actions:** "Refresh All Data".
- **Redesign notes:**
  - Covers the same five products as the **public /revenue page**, so the two largely overlap.
  - Neither page includes Pad, Monitor, Guardian, or donations.

## /admin/tokens: Token Whitelist Management
- **Who:** backend admin.
- **Canister:** backend.
- **Purpose:** the token whitelist that feeds token dropdowns across the app (WhitelistTokensContext).
- **Information shown:**
  - "Daily token list maintenance": token count, whether the timer is scheduled, interval, last import, last metadata refresh, last error.
  - "Whitelisted Tokens": the list with logos.
- **Actions:**
  - "Run daily maintenance now" and "Refresh status". These use a hand-written IDL ("not in the checked-in declarations yet").
  - "Import Tokens from ICPSwap" (`import_whitelist_from_swaprunner`).
  - "Refresh Metadata" for one token: chosen from the whitelist or a custom ledger ID (`refresh_token_metadata`).
  - "Refresh All Tokens" / "Stop Refresh", with progress.
  - "Add Token Manually": "Verify" (reads `icrc1_metadata`), then "Add to Whitelist".
  - Remove a token (confirmed).
- **Redesign notes:**
  - The RLL canister keeps a **separate** token whitelist (/admin/rewards "Import from ICPSwap token list").
  - Not in the Header menu.

## /admin/admins: Admin Management
- **Who:** backend admin.
- **Purpose:** one place for the admin lists of 15 canisters:
  - Backend, Sneed Lock, Forum, SMS, Premium, Sneedex, "Sneedapp (app store & bots)", Rewards (RLL), Sneed Monitor.
  - The 6 naming canisters: Names, Neuron Names, Nicknames, Neuron Nicknames, Avatars, Profiles (dapps).
- **Information shown:** per canister, the admin count and the list of admins.
- **Actions:**
  - Per canister: "Remove", or add a principal with "Add".
  - "Bulk Admin Operations": tick canisters ("Select All"), enter a principal, then "Add to Selected" / "Remove from Selected".
  - "Refresh All".
- **Redesign notes:**
  - Admin lists are also managed on SneedLock, the Factory, Sneedapp, Sneedex, Premium, Forum, Rewards, and SneedMon. That is 8 duplicate UIs.
  - Not covered here: Sneed Pad (its list is shown read-only on /admin/pad), Guardian deployments (read-only on /guardian), Notifications, and Drive (owner-based).
  - Not in the Header menu.

## /admin/sns_jailbreak: SNS Jailbreak Settings (`pages/AdminSnsJailbreak.jsx`)
- **Who:** backend admin.
- **Canister:** backend.
- **Purpose:** the fee for the SNS jailbreak script tool (/tools/sns_jailbreak).
- **Information shown:**
  - "Payment Statistics": Scripts Sold, Total Revenue (ICP), Unique Buyers, Premium / Regular Purchases, Premium / Regular Revenue.
  - "Payment Logs (N total)": #, User, SNS, Amount, Type, Date.
- **Actions:**
  - "Fee Configuration": Premium Member Fee, Regular User Fee.
  - "Fee Recipient Account (ICRC-1)": owner + subaccount.
  - "Save Settings" (`set_jailbreak_fee_settings`).
  - SpecialPricesCard for the jailbreak fee.
  - "Refresh".
- **Redesign notes:** the route uses an underscore (`sns_jailbreak`) while the other admin routes use hyphens or single words.

## /admin/rewards: Rewards (RLL) Administration
- **Who:** an admin of the **RLL canister** only (`rll.caller_is_admin`). There is no backend check. It redirects to /admin.
- **Purpose:** the full set of RLL tools: loops, imports, balances, events, tokens, resets, and admins.
- **Collapsible sections:**
  - "Main Loop Status": Frequency, Last Cycle Started / Ended, Next Scheduled. "Start Main Loop" / "Stop Main Loop".
  - "Distribution Cycle Status": "Run Distribution Cycle Now".
  - "Overview Statistics": Imported Neurons, Unique Owners, Imported Proposals, Balance Entries, Highest Closed Proposal.
  - "Token Balances & Reconciliation":
    - "Load Token Balances": Token, Total Owed, Ledger ID.
    - "Run Reconciliation": Token, Local Total (Owed), Server Balance (Held), Remaining (Available), Underflow.
  - "User Balances": "Load All User Balances" shows User Principal and Token Balances, with a principal filter and paging.
  - "Neuron Statistics": "Load Neuron Statistics".
    - Total and Active Neurons, Total Stake, Total Voting Power.
    - Dissolve state: Not Dissolving, Dissolving, Dissolved.
    - Permissions: Total Hotkeys, Multi-Hotkey Neurons.
  - "Event Statistics": Last 24 Hours and All Time. "Load Event Statistics".
  - "Distribution Events": Timestamp, Token, Amount, Proposal Range. "Load Distribution Events", "Clear Events".
  - "Claim Events": Seq, Timestamp, Hotkey, Token, Amount, Status. Hotkey/principal filter. "Load All Claim Events", "Clear Events".
  - "Error Claim Events": Seq, Timestamp, Hotkey, Token, Amount, Error. "Load Error Claims".
  - "Token Management":
    - Known Tokens and Whitelisted Tokens: Token, Symbol, Name, Decimals, Fee, Ledger ID. "Load Tokens".
    - "Import from ICPSwap token list" (`import_whitelisted_tokens_from_swaprunner`).
  - "Distribution Limits & Totals": Minimum Distributions, Maximum Distributions, Total Distributed. "Load Distribution Limits".
  - "Import Controls":
    - Neuron Import: "Import All Neurons", "Import New Neurons Only".
    - Proposal Import: "Import All Proposals", "Import New Proposals Only".
  - "Danger Zone": "Clear All Balances" (`clear_balances`).
  - "Admin Management": "Current Admins (N)" with remove, and "Add Admin".
- **Redesign notes:**
  - Overlaps /admin/rll on the main loop, the distribution cycle, imports, reconciliation, event statistics, and distribution limits.
  - `handleLookupUserBalances` (a per-user `get_user_distributions` lookup) is dead code: no input is wired to it.
  - Not in the Header menu.

## /admin/rll: RLL rewards (claims and cycle)
- **Who:** backend admin to see the page. Actions also need RLL `caller_is_admin`. Without it a banner says "Your principal is not an admin of the RLL canister… its tools are off."
- **Purpose:** the operational view of the RLL. Its header comment says the admin buttons moved here from the old public /rll page.
- **Information shown:**
  - "Claims": every claim attempt in one table.
    - Status filter: All, Paid, Failed, Pending (no outcome), each with its count.
    - Filter "by caller, token or #".
    - "group by error" groups failed claims into rows like "N× <error> latest …".
    - Paged.
  - "Distribution cycle":
    - Main loop Running or Stopped, Stage, Every, Next scheduled, Last cycle, Loop started / stopped.
    - Imported neurons, owners, proposals; highest closed proposal.
    - Stage chips (importing whitelist → checking balances → importing neurons → importing proposals → distributing tokens).
    - Progress bars for canister token balances, wallet token balances, and token distribution.
    - Auto-refreshes every 5 s while a cycle runs, otherwise every 30 s.
  - "Known tokens": Token, total distributed, claimable, ledger balance, remaining (or underflow), min and max per cycle.
  - "Event statistics": running counters for 24 h and 30 d.
  - "Claim incidents (open of total)", shown to RLL admins only: transfers whose outcome is unknown, listed with id, claim seq, amount, recipient, and resolved / landed / re-credited.
- **Actions:**
  - "Start distribution cycle", "Stop distribution cycle".
  - "Import all neurons", "Import all proposals".
  - "Start main loop", "Stop main loop".
  - Each of these asks for confirmation.
  - "Set limits" / "Remove limits" per token: a TokenSelector plus "Minimum per cycle" and "Maximum per cycle".
- **Redesign notes:**
  - Incidents can be viewed but **not resolved**: there is no `resolve_claim_incident` button.
  - The Header labels this page "RLL Rewards", while the dashboard's "Rewards (RLL)" card goes to /admin/rewards. Easy to confuse.

## /admin/mon: Sneed Monitor
- **Who:** backend admin. It warns when you are not on sneed_mon's own admin list.
- **Canister:** `sneed_mon`.
- **Availability:** staging only. On prod: "Sneed Monitor has no canister on this network yet".
- **Purpose:** the paid proposal watcher: operations, economics, pricing, config, the notifications app, targets, and admins.
- **Information shown:**
  - "Status":
    - State (paused / round running / idle); users, watches, targets; paused users; open proposals.
    - Rounds and interval; last round (time, duration, ticks); calls; sent / deduplicated.
    - Skipped (not allowed, rate limited, other); errors; cycles; notifications app ID.
    - Last error, and the backlog breakdown.
  - "Economics": a table over Last 24 h, Last 30 days, and Total with Revenue (ICP and USD), Cycles burned (T and USD), Revenue / cost ratio, Checks, Rounds.
  - "Accounting": user balances, fees available, obligations, fees collected, fees forwarded (with transfer count), fees withdrawn, deposits / withdrawals, accounts (paused), ledger.
  - "Targets (N)": name, watchers (active), poll frequency, open proposals, listed time, errors.
- **Actions:**
  - "Resume" / "Pause", "Run a round now", Interval → "Set".
  - "Withdraw fees": amount, "All available", recipient → "Withdraw" (confirmed).
  - "Fees: destination and sweep threshold": "Save destination" (confirmed), "Save threshold".
  - "Prices (tiers)": base daily price; per tier every_min × multiplier, with a preview; "Tier" adds a row; "Save tiers" (confirmed).
  - SpecialPricesCard: base daily price.
  - "Core config" → "Save core". "Tuning and watch defaults" → "Save tuning". "Billing" → "Save billing" (confirmed). These are generic key/value editors over the canister's records.
  - "Sneed Notifications": "Set canister" (confirmed), "Register app", "Update app (name, topics)", App id → "Set app id".
  - "Admins": "Remove" (confirmed), "Add admin".
- **Redesign notes:**
  - The user-facing part is SneedMonitorSection on /notifications, which is not admin-facing. It only shows a user their own admin or partner price.
  - The admin list duplicates /admin/admins.

## /drive: Sneed Drive (beta), phase 0
- **What it is:** a notes-first drive in one canister (`sneed_drive`).
  - The canister has a single owner (or its controllers while no owner is set). Everyone else sees only what is shared with them.
  - Small files are stored inline, up to 64 KiB per version.
  - Every save makes a version. There is a trash, tags on any node, and server-side search over names, tags, and text.
  - Per-node grants (read / write / manage) are inherited down the folder tree.
  - Every write is a block in an ICRC-3 log that only the owner reads.
  - Content is stored as plain text: "encryption comes in a later phase".
- **How it is flagged:**
  - Three conditions must all hold:
    1. backend admin (`useAdminCheck`, redirects to /);
    2. localStorage `sneedDriveBeta === '1'` in this browser (`utils/driveFlag.js`);
    3. a build-time `CANISTER_ID_SNEED_DRIVE`. It exists for staging only, so on prod the page shows "Sneed Drive has no canister on this network yet".
  - The switch "Sneed Drive (beta): Admins only · in this browser only · adds Drive to the Admin menu" appears on /admin and on /drive while the flag is off.
  - When the flag is on, the Header admin menu gains "Sneed Drive". The canister enforces access itself.
- **Information shown:**
  - The canister ID.
  - Sidebar:
    - The folder tree: "My drive" if you own the root, otherwise "Drive", or the nodes "Shared with you".
    - Tags with counts.
    - "Trash".
    - Owner only: "Drive details" with files, folders, versions, bytes used of the maximum, items in trash, shares, and version.
  - Folder view: items with name, modified time, item count or size, and tags.
  - Search results: "N found", with snippets.
  - Note view:
    - The title, the editor (Write / Preview for markdown), tags, and the save state ("Unsaved changes" / "Saved").
    - "History (N)": versions with time and size; "current" marks the head.
    - "Details": file name, MIME type, size, node, version, created and changed by whom, "Your access".
  - Trash: items with their deleted time.
- **Actions:**
  - Search: "Search", "Clear". Search options: "Search inside notes", "Only in this folder", "Include the trash". Clicking a tag filters by it.
  - Folders: "New note" (`create_file`), "New folder" → "Create" (`create_folder`), "Share", "More" → "Rename", "Move folder to trash" → "Yes, move it" / "No".
  - Note editing:
    - "Save" (`save_file` with an `expected_head` conflict check). Ctrl/Cmd+S also saves.
    - On a conflict: "Load theirs (drops your edits)" or "Save mine anyway".
    - Rename by editing the title. Edit tags (`set_tags`).
    - "Move to trash" → "Yes, move it".
    - "Move to another folder…" → "Move" (`move_node`).
  - History: open a version, "Restore this version" (`restore_version`), "Older versions".
  - Sharing:
    - Enter a principal or name and pick a role: "Can read", "Can edit", "Can edit and share". Then "Share" (`grant`).
    - Change a role, or "Remove" (`revoke`).
    - "Access from folders above (N)" lists inherited grants.
  - Trash:
    - "Restore" (`restore`).
    - Owner only: "Delete for good" → "Yes, delete" (`purge`), and "Empty trash" → "Yes, delete" (`empty_trash`).
  - Owner settings: "Keep the newest versions per note", "Delete trash for good after days" → "Save retention" (`set_settings`).
- **Redesign notes:**
  - Not an admin tool. It is a user product hidden behind an admin flag (a soft launch), so it belongs with the user features once it launches.
  - Binary files cannot be uploaded or viewed: "This file is not text; it can be read with the API only".
  - Canister methods with no UI: `get_path`, `resolve_path`, `get_log` (ICRC-3), `set_owner`, `purge_versions`, the ICRC-3 admin methods, and the `max_nodes` / `max_versions_per_node` / `max_total_bytes` settings.

## Shared admin component: SpecialPricesCard ("Admin & partner prices")
- **Used on:** SneedLock, Sneedex, Sneedapp (Network tab), Premium, Pad, SneedMon, SNS Jailbreak admin, and **/guardian** (a non-admin page, for deployment admins).
- **Information shown:**
  - The regular and premium prices, for reference.
  - The admin price, which applies to that canister's admins.
  - "Partners (N)": principal, public label, a per-fee override, and an optional expiry.
- **Actions:** "Save admin prices" (`set_admin_prices`); "Edit", "Save partner", "Cancel" (`set_partner_prices`); remove a partner (`remove_partner_prices`).
- **User side:** a PriceTierBadge ("Admin price" / "Partner price") appears on LockWizard, SnsJailbreak, SneedPad, Premium, studio IndexPanel, and SneedMonitorSection.
- **Redesign note:** the same component repeats on 8 pages. A natural candidate for a single "Pricing" admin view across canisters.

## Admin controls outside /admin

- **/thread, /post (`components/ThreadViewer.jsx`) and /proposal (`components/Discussion.jsx`)**
  - Backend admins (silent `useAdminCheck`, `redirectPath: null`) get the post owner's controls on every post and thread:
    - "Edit Thread"
    - post "Edit" / "Delete" (inline and in the post menu)
  - The forum canister decides whether the edit succeeds; it uses its own admin list, not the backend flag.
- **"Coming soon" gate (`components/LaunchGate.jsx`, `useSiteAdmin` = backend `caller_is_admin`, cached per principal)**
  - Site admins see the real feature under an "Admin preview: … shows "coming soon" to everyone else" banner. Everyone else sees a coming-soon panel.
  - Whole routes: /guardian ("Sneed custody"), /tools/pad ("Token launcher").
  - Tabs in /studio (TokenStudio): "Hand over to Sneed custody", "Index launcher", "Token launcher".
  - A `ComingSoonRibbon` marks cards on /sneedapp.
- **/guardian (`pages/Guardian.jsx`)**
  - Admins and approvers of each custody deployment (`cfg.admins` / `cfg.approvers`, the canister's own lists, "trial period") get:
    - "Run status now" (`admin_run_status`)
    - "Run auto-upgrade now" (`admin_run_auto_upgrade`)
    - "Re-read blessed versions"
    - "set" the approve_op generic function ID (`admin_set_approve_function_id`)
    - SpecialPricesCard for the op fee
  - Approvers also get "Approve (you are an approver)", "Go-ahead", "Veto", "Execute", and "Cancel" on ops. An admin's approval alone meets the threshold.
- **/sneedex (`pages/Sneedex.jsx`)**
  - People on the Sneedex canister's admin list (`getConfig().admins`) see an admin card linking to /admin/sneedex.
- **/sneedex/create (`pages/SneedexCreate.jsx`)**
  - Backend admins may press Create while not all assets are verified as ready (the `allAssetsReady` check is bypassed). Payment checks still apply.
- **/nft (`pages/Nft.jsx`)**
  - Admins of a collection canister (not site admins) get the collection's "Mint" control. The check is skipped for EXT collections.
- **/sneedapp/publisher/:publisherId (`pages/SneedAppPublisher.jsx`)**
  - The publisher-owner console (owners, not site admins).
  - It duplicates the admin publishers and versions work: owners, families, `updatePublisher`, adding and removing app versions, WASM uploads, `setAppEnabled`, and app/publisher fund withdrawals.
- **Dead admin check:** /create_icp_neuron (`pages/CreateIcpNeuron.jsx`) computes `isAdmin` "for showing total managers" but never renders anything with it.
- **Disclosure on a user page:** SmsConversations says "Private, not encrypted: Sneed SMS admins can read messages".
- **Public counterparts of admin pages:**
  - /partners, /projects → /admin/partners, /admin/projects
  - /donate → /admin/donations
  - /feedback → /admin/feedback
  - /rll → /admin/rll and /admin/rewards
  - /revenue → /admin/profits
  - /tools/sns_jailbreak → /admin/sns_jailbreak
  - /canisters folders → /admin/canisters
  - /names and nicknames → /admin/names
  - /sms → /admin/sms
  - forum pages → /admin/forum
  - /notifications Monitor section → /admin/mon
  - /tools/pad and /studio → /admin/pad

## Slice observations

**(a) Entities administered, and where**

- **Admin lists**
  - Bulk management: /admin/admins (15 canisters).
  - Per-page duplicates: SneedLock, Factory, Sneedapp, Sneedex, Premium, Forum, Rewards, SneedMon.
  - Read-only: Pad, Guardian.
  - No UI: SMS admins outside /admin/admins, Notifications, Drive (owner).
- **Moderation**
  - User bans: /admin/users/bans.
  - Blocked words: /admin/words.
  - Public names and verification: /admin/names.
  - Forum content: /admin/forum, plus inline on /thread, /post, /proposal.
  - The donation wall: /admin/donations.
- **Directories**
  - Partners: /admin/partners.
  - Projects: /admin/projects.
  - Token whitelist: /admin/tokens.
  - RLL token whitelist: /admin/rewards.
  - Sneedapp publishers and apps: /admin/sneedapp.
  - Sneedex asset types: /admin/sneedex.
- **Fees and prices**
  - Lock: /admin/sneedlock.
  - Sneedex: /admin/sneedex.
  - Factory and Sneedapp: both pages.
  - Pad: /admin/pad.
  - Monitor tiers: /admin/mon.
  - Jailbreak: /admin/sns_jailbreak.
  - Premium tiers: /admin/premium.
  - Admin and partner prices: SpecialPricesCard on 8 pages.
- **Fee destinations and recipient accounts** (one setting on each of 8 pages)
  - Donations, SneedMon, Pad, Factory/Sneedapp, Sneedex (with per-token overrides), SneedLock, Premium, Jailbreak.
- **Limits and quotas**
  - Nicknames: /admin/names.
  - Canister folders: /admin/canisters.
  - SMS: /admin/sms.
  - Forum text and premium limits: /admin/forum.
  - Sneedex general config: /admin/sneedex.
  - Drive retention: /drive.
- **Cross-canister wiring**
  - The premium canister ID can be set in **9 places**: /admin/network, /admin/premium (Propagate), /admin/names, /admin/sneedlock, /admin/sneedex, the Factory page, /admin/sneedapp, /admin/forum, /admin/sms.
  - Integration IDs: /admin/network and /admin/sneedex.
  - System sender: /admin/network and /admin/sms.
  - Governance ID: Factory, Sneedapp, /admin/network.
- **Operations and jobs**
  - RLL loop, cycle, and imports: /admin/rewards and /admin/rll.
  - Monitor rounds: /admin/mon.
  - Sneedex expiration timer: /admin/sneedex.
  - Token-list maintenance: /admin/tokens.
  - Pad WASM refresh: /admin/pad.
  - Guardian status and auto-upgrade: /guardian.
- **Code and WASMs**
  - Factory manager WASM and official versions: /admin/neuron-manager-factory.
  - Per-app version WASMs: /admin/sneedapp (and publishers on their own console).
  - Pad ledger and index suite: /admin/pad.
- **Logs and ledgers**
  - SneedLock info, error, and payment logs.
  - Pad text log and launches.
  - Factory creation log.
  - Sneedapp mint log.
  - Sneedex payment logs.
  - Premium claim and ICP payment logs.
  - Jailbreak payment logs.
  - RLL claim, distribution, and error events, plus incidents.
- **Treasury and rescue**
  - Pad "Rescue a canister" and "Withdraw ICP".
  - Factory "Top Up Cycles" and "Withdraw ICP".
  - SneedMon "Withdraw fees".
  - SneedLock pool-balance check.
  - Sneedex "Return a misplaced NFT".
  - RLL "Clear All Balances" and clearing events.
- **Revenue and analytics**
  - /admin/profits.
  - Per-page stats on Lock, Sneedex, Sneedapp, Factory, Premium, Jailbreak, and Mon.
  - The public /revenue page.
- **Membership**
  - Premium members, manual grants, revocations, and promo codes: /admin/premium.

**(b) Overlaps**

- **/admin/neuron-manager-factory and /admin/sneedapp manage the same `sneedapp` canister.** Payment config, fees, creation cycles, premium discount, admins, governance, and statistics are all duplicated. /admin/network calls the same canister "Staking Bot Factory".
- **/admin/rewards and /admin/rll** both control the RLL loop and cycle, imports, reconciliation, event statistics, and token limits. They gate access differently (RLL-only vs backend + RLL), and their nav labels are swapped-looking ("RLL Rewards" vs "Rewards (RLL)").
- **/admin/network** repeats settings that each canister's own page also edits.
- The Premium "Settings Overview" mirrors limits edited on 6 other pages.
- **/admin/profits** largely duplicates the public /revenue page and the per-page stats.
- **Admin lists** are managed on 9 pages.
- **SpecialPricesCard** is on 8 pages.
- **Partners and Projects** are near-identical directory editors.
- **"Partners"** also means price partners inside SpecialPricesCard. Name clash.
- **Two token whitelists**: the backend's (/admin/tokens) and the RLL's (/admin/rewards). Each has its own "Import from ICPSwap".
- **Forum moderation** is split between /admin/forum and inline controls that use a different admin flag.
- **Publisher self-service** (/sneedapp/publisher/:id) overlaps the admin Publishers and Versions tabs.

**(c) Dead, hidden, half-built, or confusing**

- **Hidden from the Header** (dashboard or URL only): forum, sms, neuron-manager-factory, premium, canisters, network, profits, tokens, admins, rewards.
- **Inconsistent gates:**
  - Partners, Projects, and Names have no admin gate.
  - UserBans, WordBlacklist, SneedLock, Factory, Pad, and Profits render the page for ~2 s for non-admins, then redirect to `/`.
  - Sneedex and Premium allow canister-only admins in their render logic, but the hook redirects them anyway.
  - SMS and Rewards skip the backend check entirely.
  - Redirect targets differ: `/`, `/wallet`, `/admin`.
- **Staging-only canisters**, so these show "not configured" on prod: Sneed Pad (/admin/pad), Sneed Monitor (/admin/mon), Sneed Drive (/drive), and Guardian (/guardian, which is also behind coming-soon).
- **Sneed Drive** is a user product soft-launched behind an admin flag stored in the browser, not an admin tool. Phase 0: text only, no encryption, one owner per canister.
- **Dead code or no-ops:**
  - SneedLock's "Legacy Claim Queue (retired, read-only)" tab is always empty.
  - Rewards' `handleLookupUserBalances` is never wired to an input.
  - CreateIcpNeuron's `isAdmin` is never used.
  - `NamingService.adminClearAvatar` has no UI caller.
  - Pad's index fees cannot be edited (they are round-tripped unchanged).
- **Half-built:**
  - RLL claim incidents are listed but cannot be resolved in the UI.
  - SneedLock pool recovery only checks the balance.
  - Feedback warns "needs the upgrade" on older backends.
  - Tokens maintenance uses a hand-written IDL and a hard-coded `icp-api.io` host.
  - The naming canisters' config, rate limits, imports, and ICRC-3 admin have no UI.
  - `useAdminCheck` hard-codes `https://ic0.app`.
- **Naming and route inconsistencies:**
  - Routes mix `sns_jailbreak` (underscore), `neuron-manager-factory` (hyphen), `users/bans` (nested), and `mon`/`pad` (short).
  - The Header says "Sneedapp" while the dashboard says both "Sneedapp" and "ICP Staking Bot Factory" for the same canister.
  - /admin/names lives in `pages/` rather than `pages/admin/`, as do SmsAdmin and AdminSnsJailbreak.
