# Community & messaging: forum, feed, tips, Sneed SMS, notifications

Part of the [Sneed Hub frontend functionality catalog](README.md). Snapshot: commit `ecc0eadf`, read from the code. One section per capability: routes and auth, information shown, actions (button labels as in the code), entry points, shared building blocks, gating, and redesign notes. Slice observations at the end list entities, overlaps, and anything dead, hidden or confusing.

Sources read: `pages/Forum.jsx`, `Topic.jsx`, `Thread.jsx`, `Post.jsx`, `Feed.jsx`, `Posts.jsx`, `Tips.jsx`, `SMS.jsx`, `Message.jsx`, `Notifications.jsx`, `NotificationInbox.jsx`, `NotificationView.jsx`; `components/ThreadViewer.jsx`, `Discussion.jsx`, `Poll.jsx`, `TipModal.jsx`, `TipDisplay.jsx`, `ReactionBar.jsx`, `MentionTextarea.jsx`, `EmojiPicker.jsx`, `MarkdownBody.jsx`, `MarkdownButtons.jsx`, `FeedItemCard.jsx`, `SmsConversations.jsx`, `MessageDialog.jsx`, `NotificationCard.jsx`, `PushOpenHandler.jsx`, `SneedMonitorSection.jsx` (skimmed); `contexts/ForumContext.jsx`, `SmsContext.jsx`; `utils/forumModel.js`, `forumReactions.js`, `reactionsModel.js`, `mentions.js`, `smsCache.js`, `smsConversations.js`, `NotificationsService.js`, `notificationInbox.js`, `notificationsWithdraw.js`; `hooks/use{Mention,Reply,Tip,Sms,Collectibles,VotableProposals}Notifications.js`, `useTextLimits.js`, `useStickyCount.js`; `public/sw.js`; relevant parts of `components/Header.jsx`, `Proposal.jsx`, `pages/Me.jsx`, `pages/Hub.jsx`, `pages/Principal.jsx`, `utils/PrincipalUtils.js`.

Canisters involved: `sneed_sns_forum` (forum, topics, threads, posts, votes, polls, tips, reactions, mentions, read marks, feed), `sneed_sms` (direct messages), `sneed_notifications` (notification centre), `sneed_mon` (Sneed Monitor; staging only), SNS governance (proposal titles for proposal threads; neuron data via wallet cache), ICRC-1 ledgers (tip transfers; ICP ledger for notification top-ups), Sneedex (auctions merged into the Feed), `app_sneeddao_backend` (admin check, user settings).

---

## Forum home (per-SNS forum index)

- **Routes & params**: `/forum` (`?sns=<root>` read globally by SnsContext at load; header SNS dropdown switches it). Public; topic statistics only when logged in.
- **Purpose**: entry page of one SNS's forum: list of its root topics.
- **Information shown**:
  - SNS logo + "`<SNS name>` Forum" (fallback forum title / "Community Forum"), tagline, "N Topics" count, "Log in to participate" pill when logged out.
  - Three static feature cards: "Voting Power" (hotkey neurons for posts and polls), "Direct Messages", "Governance".
  - "Discussion Topics": a card per root topic: icon (by title keyword), title, description (fallback "Explore discussions in this topic"), "N new" unread badge, "N threads", relative creation time (full date on hover), "N subtopics", subtopic pills with their own unread counts.
  - Empty: "No topics available yet".
- **Actions**:
  - Open a topic (card → `/topic/:id`, no `sns` param in link) or a subtopic pill (full page reload via `window.location.href`).
  - Bootstrap actions: see "Forum bootstrap" below.
- **Entry points**: header "Sneed Hub" menu → Forum; header quick-link icon "Go to Forum"; Feed SNS logo click (`/forum?sns=`); FeedItemCard SNS logo; breadcrumb links on topic/thread/post pages.
- **Shared building blocks**: Header (SNS dropdown), `get_forum_by_sns_root`, `get_topics_by_forum`, `get_topic_statistics` (one call per topic).
- **Gating/state**: stats (`thread_count`, `total_unread_posts`) fetched only with identity. Scroll reset on non-back navigation.
- **Redesign notes**: one forum per SNS, chosen implicitly by global SNS selector; feature cards are marketing filler; topic links drop `sns`.

## Forum bootstrap: create forum and special topics

- **Routes & params**: on `/forum` (forum/General/Governance) and `/topic/:id` of the "Governance" topic (Preproposals). Login (buttons render for logged-out users but silently no-op).
- **Purpose**: let anyone seed a forum structure for an SNS that lacks one.
- **Information shown**: "Create Your Forum — This SNS doesn't have a forum yet. Be the first to create one!"; prompts "💬 Create General Topic?", "🏛️ Create Governance Topic?", "💡 Create Preproposals Topic?" with explanations.
- **Actions**:
  - "Create Forum" → `create_sns_forum_setup(snsRoot)` then reloads topics.
  - "Create General Topic" / "Create Governance Topic" → `create_special_topic({General|Governance})`, navigates to the new topic.
  - "Create Preproposals Topic" (shown on a topic titled "Governance" without a "Preproposals" subtopic) → `create_special_topic({Preproposals})`.
  - "Maybe Later" dismisses (not persisted; prompt returns on reload).
- **Gating/state**: no client-side role check (canister decides); prompts shown whenever the named topic is missing.
- **Redesign notes**: only these special topics can be created by users; arbitrary topic creation/rename/delete lives in `/admin/forum`. The "Proposals" topic is created automatically by proposal threads (`create_proposal_thread_with_auto_setup`).

## Topic page: subtopics and thread list

- **Routes & params**: `/topic/:topicId` (`?sns=`; page writes `sns` into URL if non-Sneed SNS selected, and switches global SNS to `?sns`). Public browse; unread info login-only. Changing SNS in header → `/forum`.
- **Purpose**: browse one topic's subtopics and threads; start a thread.
- **Information shown**:
  - Breadcrumb "`<SNS>` Forum › `<topic>`"; topic icon, title, description, "N threads", "N subtopics", "Created <relative>".
  - Subtopic cards: title, "N new", description, "N threads".
  - Thread list ("Threads (N)"): title (fallback "Thread #id"), unread badge ("N new"/"NEW"), body preview (Markdown), for topic "Proposals": "📋 Proposal #N: <title>" link to `/proposal` (fetched from SNS governance; logged-in only), creator (PrincipalDisplay with names/nicknames), relative time, "N replies" (from listing or per-thread `get_posts_by_thread`).
  - Empty: "No threads yet — Be the first to start a discussion!".
- **Actions**:
  - Sort select: "Hot" (default; `get_threads_by_hot`), Advanced: "Recent Activity", "Oldest Activity", "Newest", "Oldest". Persisted `localStorage.topicThreadSort`.
  - Page size select 5/10/20/50 per page (`localStorage.topicThreadsPerPage`); "Previous"/"Next", "Page X of Y" (client-side pagination over full list).
  - Click thread → `/thread?threadid=` (no `sns`).
  - Start a thread (next section).
- **Shared building blocks**: MarkdownBody, PrincipalDisplay (context menu incl. Send Message), `orderTopicThreads`, `getPostsByThread`.
- **Gating/state**: unread counts via `get_threads_by_activity_with_unread_counts` / `get_user_thread_reads_for_topic` need identity.
- **Redesign notes**: thread scores, reactions and tips are not shown in lists (only inside a thread). N+1 calls for post counts and proposal titles.

## Start a new thread (optionally with a poll)

- **Routes & params**: form at bottom of `/topic/:id` (hidden in the "Proposals" topic). Login (form visible logged out; submit no-ops).
- **Purpose**: create a discussion thread, optionally with a neuron-weighted poll.
- **Information shown**: "Start a New Thread"; character counters "n/limit characters" for title and body; "⭐ PREMIUM" tag when the premium body limit applies.
- **Actions**:
  - Fields: "Thread title", body (MentionTextarea "What would you like to discuss?") with EmojiPicker + MarkdownButtons.
  - "Include a Poll" checkbox → "Poll Details": "Poll title" (required), "Poll description (optional)", options (2–10; "Option N", optional description; "✕" remove; "+ Add Option"), "End Date"/"End Time" (default now+5 days), "VP Power" (Equal (0) / Square Root (0.5) / Linear (1) / Quadratic (2)), "Allow voters to change their votes" (default on).
  - "Create Thread" / "Create Thread & Poll" → `create_thread` then `create_poll` (if poll fails: "Thread created successfully, but poll creation failed"); navigates to the new thread. "Clear Poll".
  - After save: `smsMentioned` sends Sneed SMS to newly @-mentioned opted-in users (from the poster).
- **Limits**: thread title/body max from `get_text_limits` (fallback 200/10000); premium raises thread/post body max (`get_premium_config`); mentions max 10.
- **Shared building blocks**: MentionTextarea, EmojiPicker, MarkdownButtons, `useTextLimits`, Poll creation logic (duplicated from `Poll.jsx`).
- **Redesign notes**: poll form duplicated here and in Poll.jsx (different validation: here poll body optional, Poll.jsx requires it and checks end date in the future).

## Thread view (ThreadViewer) — read a discussion

- **Routes & params**: `/thread?threadid=<n>&sns=<root>` (ThreadViewer rewrites `sns` to the thread's SNS from `get_thread_context`). Public read. Also embedded on `/proposal` and `/post`.
- **Purpose**: the canonical thread renderer: header, posts tree, all interactions.
- **Information shown**:
  - Page chrome (Thread.jsx): SNS logo + "`<SNS>` Forum", breadcrumb "Forum › `<topic>` › Thread".
  - Collapsible thread header (▼): title (prop, own title, or "Thread #id"); when collapsed: creator + relative time.
  - Expanded header: thread body (Markdown), thread tips (TipDisplay, verified only), reaction bar for the thread, thread score (`scoreLabel`, compact +1.2K/-3 tokens of VP) with 👍/👎, "Tip", "Edit Thread", thread polls, "📊 Add Poll to Thread", "Created by: <principal> <time>".
  - Proposal link box: "📋 Proposal Discussion — This thread is discussing Proposal #N: <title>" (link to `/proposal`, text only when `hideProposalLink`). Found via `get_thread_proposal_id`, or by regex on title "Proposal #N" / body "Discussion for … Proposal #N".
  - Post-mode banner: "Viewing Post #N in context" + "View Full Thread →".
  - Posts: avatar (BigAvatar, verified badge), name, collapse toggle (▶/▼; negative-score posts collapsed by default), "#id" link → `/post`, post title, "UNREAD" badge (new since last visit, never own), relative time (full on hover), in flat view "Reply to #N: <derived title>" (scrolls to parent or opens it), tip pills, body (Markdown with mention chips), reaction bar, post polls, vote buttons with VP amount and score, optimistic posts shown as "Sending...".
  - "No posts in this thread yet." / "Thread not found or could not be loaded."
- **Actions (view controls)**:
  - "View:" "🌳 Tree" / "📋 Flat" (`localStorage.discussionViewMode`, shared with Discussion).
  - "Sort:" "⭐ Best Score", "📅 Newest First", "📅 Oldest First", "👎 Worst Score", "🔥 Most Active" (`localStorage.threadSortBy`).
  - Collapse/expand individual posts.
- **Read tracking**: `get_last_read_post` / `set_last_read_post` on load (marks highest visible post read; highlight stashed once per visit).
- **Entry points**: Topic list, Feed items/context tags, `/posts`, `/tips` (thread links), mention cards, proposal page "View in Forum", notifications/SMS links (mention SMS links to `/thread` or `/post`).
- **Gating/state**: neurons from WalletContext cache for the thread's SNS; `isAdmin` = Sneed Hub backend admin (`useAdminCheck`), not forum-canister admin.
- **Redesign notes**: ThreadViewer is a 4.6k-line monolith mixing rendering, voting, tipping, polls, messaging, editing; mobile (<768px) moves actions to a "•••" overflow menu and hides labels.

## Posting: comment, reply, edit, delete

- **Routes & params**: inside ThreadViewer (thread, post, proposal pages). Login; edit/delete = author or Sneed Hub admin; thread edit = author or admin.
- **Purpose**: write and maintain posts.
- **Actions**:
  - Top-level comment: "💬 Be the first to comment" / "💬 Add a comment" → form "Start the Discussion"/"Add Your Comment": "Title (optional)", body (MentionTextarea "Share your thoughts...") + emoji/markdown, counters, "Cancel", "✓ Post Comment" → `create_post(thread, [], title, body)`.
  - "Reply" on a post (toggle "Cancel") → ReplyForm "Reply to <name>", "N characters remaining", "Submit Reply" → `create_post(thread, [parent], [], body)`.
  - New post/reply is **auto-upvoted** with the selected neurons (`vote_on_post_with_neurons`) when they have VP.
  - "Edit" → "Edit Post" form ("Post Title (optional)", body, title/body counters with "(n over limit)", "Update Post"/"Cancel") → `update_post`; optimistic edit.
  - "Edit Thread" → "Edit Thread" form ("Thread Title", body, "Update Thread") → `update_thread`.
  - "Delete" → confirm "Are you sure you want to delete this post? This action cannot be undone." → `delete_post` (soft delete).
  - "Message" → MessageDialog to the post author (not own posts).
  - Mentions in create/edit trigger `smsMentioned` (edit pings only newly added mentions).
- **Limits**: post title/body from forum text limits; premium body limit; reply counter uses `max_comment_length` (= post body max).
- **Redesign notes**: no thread delete for users (`deleteThread` imported but unused; only `/admin/forum`); admins see Edit/Delete everywhere based on backend admin role, which may not match forum canister admins.

## Neuron-weighted voting on posts and threads (+ "Voting" neuron panel)

- **Routes & params**: ThreadViewer; Post page shows breakdown. Login + SNS neurons visible to the Sneed Hub principal (hotkey) for the thread's SNS.
- **Purpose**: up/down-vote content with SNS neuron voting power.
- **Information shown**:
  - Per post: 👍 with selected VP, score (e8s→tokens, compact, red if negative), 👎 with VP; tooltip "Vote with X VP" or "Recant upvotes from N neurons: <neuron8…>: X VP"; "You must have neurons with voting power to vote on posts".
  - Thread score: separate from post scores ("The thread's score (its votes, apart from its posts')"), prediction applied instantly.
  - "🗳️ Voting" panel: "Voting Neurons — Select which neurons to use when voting on posts": checkbox per neuron (id prefix, VP), "Total Selected Voting Power", "No neurons available for voting".
- **Actions**:
  - Click 👍/👎 → `vote_on_post_with_neurons(post, up|down, neuronIds)`; clicking the active direction retracts → `retract_vote_with_neurons`. Optimistic score + animation.
  - Thread 👍/👎 → `vote_on_thread_with_neurons` / `retract_thread_vote_with_neurons` (extras IDL; hidden if canister lacks method).
  - Neuron selection persisted per thread: `localStorage.selectedNeurons_<threadId>` (default: all neurons). Same selection drives poll voting.
- **Shared building blocks**: WalletContext neuron cache, `calculateVotingPower`, `forumModel.predictThreadVote`, `scoreLabel`.
- **Redesign notes**: legacy Discussion component uses a different API (`vote_on_post` / `retract_vote`, all neurons, no selection). Votes are per-neuron and public (see Post page breakdown).

## Emoji reactions (posts and threads)

- **Routes & params**: ThreadViewer (thread header + each non-deleted post). Read public; react login.
- **Purpose**: lightweight public emoji reactions.
- **Information shown**: pills "emoji count" in first-given order, mine highlighted; read-only when logged out (tooltip "N reacted with 😀"); picker note "Up to 10 reactions each. Reactions are public, with your principal."
- **Actions**: tap pill to add/take back; "☺+" ("Add a reaction") opens picker: 8 quick emojis, "Search emoji", grid (common + full emoji list). Calls `react(target, emoji, on)` then `get_reactions`. Errors: "At most 10 reactions each; take one back first", "Too many reactions at once; try again in a minute", "Could not save the reaction".
- **Limits**: 10 emojis per user per target; single emoji grapheme ≤64 bytes (`isSingleEmoji`, mirrors canister).
- **Gating/state**: feature auto-hides if the forum build lacks the methods (`isMissingMethod`). Batch loads ≤200 targets.
- **Redesign notes**: reactions don't appear in Feed, /posts, Topic lists or the legacy Discussion component.

## Polls (create, vote, results)

- **Routes & params**: thread polls and post polls inside ThreadViewer; read-only results in Feed/FeedItemCard; creation also in Topic "Start a New Thread". Vote/create login.
- **Purpose**: neuron-weighted polls attached to a thread or a post.
- **Information shown**: "📊 <title>" (+ "ENDED"), description, "Ends: <date> • VP Power: Nx", "⚠️ Vote changes not allowed"; per option: title, description, "N votes", "X VP", "✓ n" (how many of my neurons chose it), percentage bar (winner(s) highlighted green after end); "💡 Voting with N neurons (X total VP)".
- **Actions**:
  - "Vote" per option → `vote_on_poll_with_neurons(poll, option, selectedNeurons)`; blocked (browser `alert`) if no neurons selected, poll ended, or already voted and changes not allowed.
  - Create (thread author: "📊 Add Poll to Thread" if none yet; post author: "Add Poll" on own post if none) → "📊 Create Poll for Thread/Post": title (required), body (required here), options 2–10, End Date/Time (must be future), VP Power (Equal (0 - each vote = 1) / Square Root (0.5) / Linear (1 - default) / Quadratic (2)), "Allow voters to change their votes"; "Create Poll", "Clear", "✕" cancel → `create_poll`.
- **Gating/state**: in Feed polls render with no neurons → "Vote" buttons visible but disabled ("Select neurons to vote").
- **Redesign notes**: uses `window.alert` (rest of app uses appDialog); one poll per thread/post in UI.

## Tipping (posts and threads)

- **Routes & params**: ThreadViewer (post "Tip", thread header "Tip", tip-pill "+" buttons), legacy Discussion "Tip". Login; cannot tip own content.
- **Purpose**: send any wallet token to an author, recorded and verified by the forum.
- **Information shown**:
  - TipDisplay pills per token: logo, total amount, tip count; hover/tap tooltip "`<SYMBOL>` Tips — N tips received", each tip "+amount SYMBOL (≈USD)", from principal, timestamp; total + USD.
  - TipModal "Send Appreciation — Show your support with a thoughtful tip": "Choose Token" dropdown (wallet tokens with balances), "Amount" with "MAX" (balance − fee), "≈ <fiat>" (DenominationContext), balance, fee, decimals, "Recipient — Post Creator", "Tipping for Post/Thread #N" preview (title, 150-char body).
  - Progress: "Sending Tip..." ("Processing your token transfer...") → "Registering Tip..." ("The forum is checking your transfer on the ledger and recording your tip..."), "⚠️ Please don't close your browser during this process"; success "Thank You!" (logo flies into the tip pill with sparkles); error "Something Went Wrong" + reason.
- **Actions**: "Send Tip" → ICRC-1 transfer from user wallet to author's default account (`icrc1Transfer`) → `create_tip(to, post, ledger, amount, block)` or `create_thread_tip(...)` (thread) → re-read tip; unverified tips show "the tip is recorded, but the forum could not confirm it on the ledger yet … shown once verified". Click an existing pill = tip preselecting that token.
- **Limits/validation**: decimals must be readable (never guessed); balance must cover amount + fee ("Insufficient balance. You need X SYMBOL more (including transaction fees)"); clear messages separating "nothing was sent" vs "tokens were sent, do not send again".
- **Gating/state**: only verified tips displayed/counted; particle effects respect `localStorage.particleEffectsEnabled`.
- **Shared building blocks**: TipModal, TipDisplay (both also used by Discussion), `useTokens` (wallet token list), `get_token_conversion_rate`.
- **Redesign notes**: tips are a wallet action living inside the forum; the token list is the user's wallet list only.

## @-mentions

- **Routes & params**: any MentionTextarea (thread create/edit, comment, reply, post edit, legacy Discussion, `/feedback`). Mentions inbox at `/posts?tab=mentions`; SMS opt-in on `/me`.
- **Purpose**: tag members; they get a notification and optionally a Sneed SMS.
- **Information shown**: typing "@" + text opens the PrincipalPicker panel (Private/Public/All tabs, users/canisters toggles), note "Mention someone: they get a notification. Posts show their public name, never your nickname." Rendered mentions show the member's current public name (or viewer's nickname) as a chip linking to `/principal?id=`.
- **Actions**: pick → inserts `[@Name](https://app.sneeddao.com/principal?id=…)`; keyboard ↑/↓/Enter/Tab/Esc; "_"/"-" match spaces.
- **Side effects**: forum records mentions (≤10 per post). After saving, the poster's browser sends a Sneed SMS ("You were mentioned in …", 300-char excerpt + link) as the poster to newly mentioned members who opted in (`get_mention_sms_opted_in`) — best effort, consumes the poster's SMS sending.
- **Settings**: `/me` → Notification Bar Settings → "SMS When Mentioned" (`set_mention_sms`, stored in forum canister).
- **Redesign notes**: SMS composer does not support mentions; mention opt-in lives on `/me`, not on `/notifications` or `/sms`.

## Single post in context + vote breakdown

- **Routes & params**: `/post?postid=<n>&sns=<root>`. Public.
- **Purpose**: permalink to one post with its thread context and full vote audit.
- **Information shown**: SNS forum header; breadcrumb "Forum › `<topic>` › Thread › Post"; ThreadViewer in post mode (focused post highlighted and scrolled to, its ancestors, descendants, plus my own/optimistic sibling replies); "All Votes for This Post": "N upvotes (X VP)", "N downvotes (X VP)", "Net Score: ±X VP"; "Show Details"/"Hide Details": per vote card (Upvote/Downvote, VP, Neuron link, Voter principal, When) sorted by VP (`get_post_votes`).
- **Actions**: everything ThreadViewer offers.
- **Entry points**: "#id" on every post, `/posts`, `/tips`, mention cards, mention SMS links, Feed post items.
- **Redesign notes**: the only place where per-neuron votes (who voted) are visible; not available for thread-level votes.

## Proposal discussion (embedded on `/proposal`)

- **Routes & params**: `/proposal?proposalid=<n>&sns=<root>` → collapsible discussion section with "View in Forum" link (`/thread?threadid=…&sns=…`). Public read; login to post.
- **Purpose**: discuss a governance proposal in its forum thread.
- **Information shown/actions**:
  - If a thread exists: full ThreadViewer titled "Discussion for Proposal #N" (`hideProposalLink`).
  - If none: legacy `Discussion` component: voting status line ("Forum voting enabled • N neurons • X VP" / "No voting power • Add hotkey neurons to vote on posts"), "No comments yet. Be the first to start the discussion!", "Be the first to comment" → Title + MentionTextarea "Share your thoughts on this proposal..." → "Post Comment" creates the thread (`create_proposal_thread_with_auto_setup`, auto-creates forum/Proposals topic) then the post, auto-upvote; logged out: "Please connect your wallet to participate in the discussion."
  - Discussion also renders posts (Flat/Tree, "Low Score", vote arrows, "Retract", "Reply", "Edit", "Delete", "Tip") if a thread appears.
- **Redesign notes**: two parallel discussion renderers; Discussion lacks reactions, polls, emoji/markdown buttons, neuron selection, read tracking, "Message"; uses raw ReactMarkdown (no mention chips).

## Activity feed

- **Routes & params**: `/feed` (`?startFrom=<itemId>`). Public; new-item polling only when logged in.
- **Purpose**: cross-SNS stream of forum activity plus Sneedex auctions.
- **Information shown**:
  - Hero "Sneed's Feed": "Real-time activity from N SNS forums" / "Filtering X of N SNS forums".
  - SNS avatar row (selected first, rest shuffled, max 10 collapsed, "+N"/show all, clear "✕").
  - Items (FeedItemCard): type badge (Forum/Topic/Thread/Post/Auction), SNS logo (→ that forum), creator PrincipalDisplay, SNS name, relative time, title (fallback "<Type> #id"), body excerpt (250 chars Markdown), "Replying to:" parent title + 150-char body, poll (read-only results; "📊 Poll (loading...)"), context tags (topic → `/topic/:id`, thread → `/thread`).
  - Auction items: SOLD/EXPIRED/CANCELLED banner, "Sneedex" badge + asset count, title ("SNS Neuron for Sale", "ICP Staking Bot for Sale", "App for Sale", "Token Lot for Sale", "Bundle of N Assets"), "For Sale:" assets (SNS neuron stake, staking-bot stake, app name, token amount, Sneed Lock, EXT cards, ICRC-7 NFTs), "Buyout:"/"Buyout Only:", "Min Bid:"/"Min Next Bid:", "Ends: <time remaining>", public note.
  - "N new items • Click to view" banner; "Load More Recent"/"Load More Older"; "You've reached the end of the feed"; "No Activity Yet".
- **Actions**:
  - Toggle SNS filter by tapping avatars (persisted `localStorage.feedSnsSelection`; server-side `sns_root_canister_ids`).
  - Auctions toggle (gavel; `localStorage.feedShowAuctions`, default on).
  - Advanced "Filters": "Filter by Type" chips (client-side on returned page), "User" (PrincipalInput → `creator_principals`), "Search Text" ("Search in titles and content..." → `search_text`); "Apply Filters", "Clear Filters".
  - Infinite scroll up/down (`get_feed` with `start_id`), position restored (`sessionStorage.feedScrollPositionId`), click item/title/badge → item page (auction → `/sneedex_offer/:id`).
- **Background**: every 30 s (logged in) `get_current_counter` vs `localStorage.feedLastSeenId`/`feedHighestCheckedId` to count new items (walks filtered pages when SNS filter active).
- **Entry points**: header "Sneed Hub" menu → Feed; header quick-link "Go to Feed"; Hub "Sneed Forum — Latest discussions" widget (5 newest, all SNSes) → "View All".
- **Shared building blocks**: FeedItemCard (Feed + Hub, variant `hub`), Poll, PrincipalInput, Sneedex utils.
- **Redesign notes**: Feed ignores the header SNS selector (has its own multi-select) though it shows the dropdown; type filter is applied per page client-side (pages can come back empty); `Feed.jsx` contains a ~600-line unused `renderFeedItem` duplicate of FeedItemCard; auctions mix marketplace content into the forum feed; Hub widget labelled "Sneed Forum" shows all SNSes.

## My Posts (replies, mentions, own posts, own threads)

- **Routes & params**: `/posts` (`?tab=mentions` honoured; other tab values ignored). Login ("Connect to View Posts").
- **Purpose**: personal forum activity hub.
- **Information shown**: header counts "N replies", "N posts", "N threads"; tabs "Replies to Me", "Mentions", "My Posts", "My Threads" with counts.
  - Reply/post cards: "#id", "Reply from <principal>" (copy + context menu), title, "New" (reply newer than last seen), net score "+X" with "↑up ↓down", body (Markdown), "Reply to #N" link, relative time.
  - Mention cards: "@ <who> mentioned you in <thread title | a post | a thread> · <time>", "new", plain excerpt; "More" pagination (30/page).
  - Thread cards: "Thread #id", title, "N posts", body, time.
  - Empty states per tab ("No Replies Yet", "Nobody has mentioned you yet…", "No Posts Yet", "No Threads Yet").
- **Actions**: switch tab, "Refresh", "Try Again", open post/thread/parent.
- **Side effects**: opening the page marks replies seen (`mark_replies_seen_up_to`) after capturing the old mark; opening Mentions marks mentions seen (`mark_mentions_seen_up_to`) and clears header badge via `mentionsSeen` event.
- **Entry points**: header Sneed Me → Posts; header replies badge (→`/posts`), mentions badge (→`/posts?tab=mentions`); `/me` quick link "My Posts" and stats tiles (Posts → `/posts`, Threads → `/posts?tab=my-threads` — broken deep link).
- **Redesign notes**: overlaps `/principal?id=` "Posts" tab (any user's posts/threads, public); unused `renderPost`/`renderThread` functions; no reactions, tips or edit actions here.

## Tips history

- **Routes & params**: `/tips`. Login ("Connect to View Tips").
- **Purpose**: tips received and given.
- **Information shown**: "N received (≈ fiat total)", "N given (≈ fiat total)", "N unverified tips not shown" (tooltip explains ledger verification); tabs "Tips Received" / "Tips Given" with counts; cards: token logo + counterpart avatar (AvatarPair), amount + symbol, ≈ fiat, "New" with sparkle (received since last seen), "From"/"To" principal (copy, context menu), "Post #N" and "Thread" links, relative time. Empty: "No Tips Received Yet" / "No Tips Given Yet".
- **Actions**: switch tab, "Refresh" (also refreshes header tip count), "Try Again"; navigate to post/thread.
- **Side effects**: marks tips seen (`mark_tips_seen_up_to`) on open; confetti of token logos for new tips (scaled by USD value). Hidden debug: pressing "c"/"v" fires test confetti.
- **Entry points**: header Sneed Me → Tips; header tips badge; `/me` "My Tips".
- **Redesign notes**: lists only post tips (`get_tips_*_by_user_page`); thread tips (canister has `get_thread_tips_received/given_by_user_page`) are never listed here; unused `renderTipRow`/`renderTipCard`.

## Sneed SMS inbox lists (Inbox / Sent / All)

- **Routes & params**: `/sms` (`?view=sent|all|conversations`, default Inbox; also `localStorage.sneed_sms_view`; `?reply=<msgId>` opens a reply; `?recipient=<principal>` opens compose). Login ("Connect to View Messages").
- **Purpose**: direct messages between principals (not encrypted).
- **Information shown**: "Messages — Private messaging on the Internet Computer"; "N received", "N sent" (canister totals); tabs "Inbox", "Sent", "All", "Conversations"; message cards: avatar (sender, or first recipient for sent), subject, "New" badge (newer than last-seen mark), "From: <principal>", "+N recipients", relative time, 120-char Markdown preview; new arrivals slide in. "No Messages Yet — Start a conversation…".
- **Actions**: "Compose", "Refresh" (full reread), "Load older messages" (50/page), card click → message modal, "Thread" (→ `/msg/:id`), "Reply" (received only).
- **Message modal**: subject, time, "Thread", "Message Details" (Subject, From with avatar + copy, To (N) list, Date), Markdown body; "Reply", "Reply All" (everyone but me), "Remove" (if `can_remove_self`; app confirm "Are you sure you want to remove yourself from this message? This action cannot be undone." → `remove_self_from_message`), "Close".
- **Side effects**: opening `/sms` (tab visible) marks received seen (`mark_messages_seen_up_to`); switches to Inbox when unread exist (except Conversations view).
- **Entry points**: header Sneed Me → Messages; header quick-link "Messages"; header messages badge; `/me` "My Messages"; `/msg/:id` "Back to Messages"/"Reply"; notifications "Channels" text link.

## Compose / reply (SMS page)

- **Routes & params**: compose modal on `/sms`. Login.
- **Purpose**: send a message to 1..N principals.
- **Information shown**: "Compose Message — Send a private message" / "Reply to Message — Continue the conversation"; "Recipients (n/max)", "Subject" counter, "Message" counter; "⭐ PREMIUM" tags when premium limits apply.
- **Actions**: recipient PrincipalInput ("Enter principal ID or search by name"), remove (trash), "Add Recipient" (up to max), subject (required), body with EmojiPicker + MarkdownButtons (required), "Cancel", "Send Message" → `send_message({recipients, subject, body, reply_to})`. Errors mapped: "Rate Limited: …", "Invalid Input: …", "Unauthorized: …".
- **Limits**: from `get_config` (fallback subject 200, body 5000, recipients 20) and `get_premium_config` for Premium (higher subject/body, recipients default 50, `premium_rate_limit_minutes` read but never shown).
- **Redesign notes**: three separate composers (this modal, MessageDialog, Conversations composer) with duplicated limit logic.

## Sneed SMS Conversations (chat view)

- **Routes & params**: `/sms?view=conversations`. Login.
- **Purpose**: chat-style grouping of messages by participant set.
- **Information shown**:
  - List pane "Conversations": rows with avatar(s) (pair for groups, note icon for self), title (names / "A, B and N others" / "Notes to self"), relative time, preview ("You: …" or "Name: …" in groups), unread count badge. "Loading conversations…", "No conversations yet. Press New to start one.", "Load older messages".
  - Thread pane: back arrow (mobile), header (1:1 PrincipalDisplay, group "N people:" list), "Load older messages", day dividers (Today/Yesterday/date), sender names in groups, bubbles (mine right) with quoted reply chip (link to `/msg/:id`), subject line when it changes, Markdown body, time, details button (opens the message modal: Reply All, Remove, Thread). Empty: "No messages with them yet. Write the first one below." / "Notes you send only to yourself show here." / "Pick a conversation, or press New to start one."
  - Footer: "Private, not encrypted: Sneed SMS admins can read messages. Details" (→ `/help/sms#privacy`); error "This group has more people than one message can go to (N)."; counter shown past 80% of body limit.
- **Actions**: "New"/"Cancel" → "With whom? Add one person for a chat, more for a group." recipients, "Add person", "Open conversation"; type message ("Message"/"Note to self"), optional "Subject" (defaults to "Re: <last subject>" or first line of body), "Send (Ctrl+Enter)" → `send_message` to all participants with `reply_to` = last message; send failure via appAlert "Message not sent". Per-conversation drafts (memory only).
- **Gating/state**: mobile ≤760 px shows list or open conversation, not both; per-conversation read marks local to the session.
- **Redesign notes**: subject is required by the canister so chat invents subjects; no emoji/markdown buttons here; conversation key = participant set, so the same people across unrelated subjects merge.

## SMS message thread view

- **Routes & params**: `/msg/:id`. Login ("Connect to View Thread").
- **Purpose**: reply-tree view around one message.
- **Information shown**: header = focused subject, "Thread #id", "N messages"; tree with depth-colored nesting; per message: collapse toggle, avatar, subject, "#id" (refocus), time, "Details" (From / To lists), body (Markdown, "Show More"/"Show Less" over 300 chars). Error "Message not found or access denied".
- **Actions**: "Back to Messages", "Expand All"/"Collapse All", "Load Full Context" (ancestors + siblings), expand replies (loads children), "Reply" → `/sms?reply=<id>`.
- **Gating/state**: uses cached inbox for instant first paint; children found by reading the user's entire received+sent history (paged up to 1000 pages) or `get_all_messages`.
- **Redesign notes**: overlaps Conversations view; reply opens compose on another page; heavy client-side scan.

## Quick "Send Message" dialog (from anywhere)

- **Routes & params**: modal; login. Opened from ThreadViewer post "Message" button, `/principal?id=` "Send Message", and the PrincipalDisplay context menu ("Send Message" on right-click/long-press of any principal name across the app).
- **Purpose**: message someone without leaving the page.
- **Information shown/actions**: "Send Message"/"Reply to Message"; "Recipients * (n/max)", PrincipalInput ("Enter principal ID..."), "Remove", "+ Add Recipient (max N)", "Subject *" counter, "Message *" with emoji/markdown + counter, "⭐ PREMIUM" tags; "Cancel", "✉️ Send Message" → `send_message`. Supports prefilled subject/body/replyTo props.
- **Redesign notes**: doesn't refresh the shared SmsContext (sent list catches up on next poll); same limits code duplicated.

## SMS inbox sync engine (background)

- **Routes & params**: global `SmsProvider` (all pages). Login.
- **Purpose**: one shared, cached inbox for header badge and SMS pages.
- **Behaviour**: IndexedDB cache `sneed_sms_cache` per canister+principal (deleted on sign-out); incremental `get_received/sent_messages_page` with `after`, reread newest page every 10 min or when totals/last_updated mismatch; poll every 60 s (30 s while an SMS page is open), only while tab visible, plus on app refresh/visibility; legacy fallback (`get_recent_messages_count`, full lists). Exposes `unreadCount`, `markSeen`, `removeLocal`, `loadOlder`, `arrivals`.
- **Redesign notes**: robust infrastructure that any redesigned messaging surface can reuse; NotificationCard also reads it to show SMS sender avatars.

## Notification inbox (Sneed Notifications)

- **Routes & params**: `/notifications` (default tab Inbox; `?tab=inbox`), `/notifications/:id`, `/notifications/push?t=<created_at>&h=<titleHash>` (push landing, resolved then replaced by `/notifications/<id>`), legacy `/notifications/inbox` → redirect. Login; "The notification service isn't live on this network yet." when canister id missing.
- **Purpose**: phone-style list of everything apps sent via `sneed_notifications`.
- **Information shown**: filter chips "All", "Unread · N", one chip per app; cards: level stripe (Urgent red / Normal amber / Info blue), app icon/name (person avatar for SMS DMs), "Urgent" label, relative time, unread dot ("Not opened on this device"), bold title, 3-line plain preview. Footer "Showing your latest 200." / "That's all. Notifications are kept for 30 days." Empty: "No notifications yet…" + "Choose which apps may notify you"; "You're all caught up."; "Nothing from this app."
  - Single view: enlarged card with full Markdown body; primary button "Open" (internal/external link) or "Top up" for system low-balance; "All notifications"; details rows: App (verified ✓), Topic (title + id; system topics "Channel verification", "Low message balance"), Level, Sent, Delivered (per channel kind/name: delivered/failed/suppressed/pending + reason, or "Only kept here (muted, or no channel for this level)"). Not found: explains 30-day retention / other account.
- **Actions**: "Mark all read", refresh, infinite scroll/"Load more" (25 at a time up to 200, `get_my_history`), open card (marks seen).
- **Gating/state**: read state only in `localStorage.sneedNotificationsSeen:<principal>` (canister keeps none) → per device.
- **Entry points**: header Sneed Me → Notifications; `/me` "Notification inbox" link; push notification taps; low-balance alert.
- **Redesign notes**: no header badge for unread notifications (SMS has one); unread state device-local.

## Notification settings

- **Routes & params**: `/notifications?tab=settings` (also opened by `#message-balance` hash or `?code=<6 digits>`). Login.
- **Purpose**: control which apps may notify you, at what level, through which channels, and pay for paid channels.
- **Information shown/actions**:
  - "Apps asking to notify you": app name (verified ✓), description, topics → "Allow" / "Block" (`allow_app`).
  - Sneed Monitor section (staging only; see below).
  - "How you're notified": level × verified-channel checkbox table (Urgent/Normal/Info with hints), "Always copy to" (channel / Nothing), "Quiet hours": "Hold messages" from/to (device time zone), "Urgent gets through"; "Save" (`set_settings`); "Pause everything:" "1 hour" / "1 day", "Paused until …" + "Resume".
  - "Apps": per allowed/blocked app: status, "Block"/"Allow", expand: "Demote everything one level", "Only to my copy channel", "At most N messages a day", "It may cost me at most X ICP a day", per-topic rule (As declared / Treat as urgent / normal / info / Mute) (`set_permission`).
  - "Other apps": undecided apps → "Allow".
  - "Channels": list (name · kind, "ready"/"waiting for the code", "Remove" except Sneed SMS); verification "Code sent to <name>:" input + "Verify" (adds channel to urgent+normal) + "Send a new code"; device-received code auto-filled (service-worker cache, 15 min). Text: "Sneed SMS is your inbox…" link to `/sms`. Toggle "Show the first 100 characters of new direct messages in notifications…" (`sneed_sms.set_notification_preview`). "Get notifications on this device" (push; iOS only inside installed app, else link to `/help/app`). Discord: "Your Discord user id" + "Add Discord" (Mr Sneed bot DMs a code). "Coming soon: …" for non-live kinds. Addresses encrypted in-browser to the relay (RSA-OAEP + AES-GCM).
  - "Message balance" (`#message-balance`): balance in ICP (≈ fiat), amount + "Top up" (ICRC-2 approve + `top_up`, plus 0.0001 fee), "Withdraw all to my wallet", "Advanced": amount/"All"/"Withdraw" to any principal (PrincipalInput), withdrawn total, reserved amount, last 10 withdrawals (date, amount, to, block); pending withdrawal notice; "Warn me when my message balance falls below X ICP" + "Save" (`set_low_balance_alert`).
  - Bottom status toast for each action ("tap to close").
- **Redesign notes**: mixes three concerns (permissions/routing, channel onboarding, prepaid ICP wallet) plus Sneed Monitor; the header notification bar toggles live separately on `/me`.

## Sneed Monitor (proposal watching) — inside Notification settings

- **Routes & params**: section on `/notifications?tab=settings` (`#sneed-monitor`). Login; renders nothing where `sneed_mon` has no canister id (production for now → staging only).
- **Purpose**: get notified of votable proposals while Sneed Hub is closed.
- **Actions (labels)**: "Tell me about proposals I can vote on" (watch all SNSes with vote neurons; "Turn off"), "Watch them too", "Choose an SNS…", "NNS (Internet Computer)", "NNS neurons", topics ("Save topics"), "Sneed Monitor balance", "Top up from my wallet", deposit address / "Account id (for exchanges)", "I've sent it: check deposit", "Auto top-up" (ICRC-2 allowance, 1–3650 days), "Withdraw", charges list ("Newer"/"Older"), "Allow" (lets Sneed Monitor notify you). Calls `watch`, `unwatch`, `update_watch`, `credit_deposit`, `set_auto_top_up`, `withdraw_balance`, `set_nns_neurons`, `refresh_my_neurons`, etc.
- **Redesign notes**: a governance feature buried in notification settings; separate balance from the notification message balance; overlaps header "votable proposals" badge and `/active_proposals`.

## Push notifications (installed app / service worker)

- **Routes & params**: `public/sw.js` (global), `PushOpenHandler` (global component).
- **Purpose**: make Sneed Hub installable and show pushes from the Sneed Notifications relay.
- **Behaviour**: `push` → shows notification `{title, body, link, tag}` (icon `/icon-192.png`); caches 6-digit "Your code: NNNNNN" verification codes for the settings page; `notificationclick` → navigates/opens a window at the link (default `/notifications`), else posts `sneed-open` to the page; PushOpenHandler navigates in-app or refreshes data if already on that URL. No offline caching.
- **Redesign notes**: links land on `/notifications/push?...` (resolved to `/notifications/:id`) or the app's own deep links.

## Header notification bar (in-app alert hooks)

- **Routes & params**: every page (Header), when logged in and "Show Header Notifications" is on (and header not collapsed). No toasts; only a strip of clickable count chips under the header with a bell label.
- **Purpose**: surface new social/financial events.
- **Chips (source → click)**:
  - Update available ("Update", optional auto-refresh countdown) → reload (independent of login).
  - Replies (`useReplyNotifications`: forum `get_recent_replies_count`, every 2 min) → `/posts`.
  - Mentions (`useMentionNotifications`: `get_recent_mentions_count`, every 2 min + `mentionsSeen` event; shown under the Replies setting) → `/posts?tab=mentions`.
  - Messages (`useSmsNotifications`: SmsContext `unreadCount`) → `/sms`.
  - Votable proposals (`useVotableProposalsNotifications`: neuron cache → `fetchVotableProposalsCount`, every 5 min + `votableProposalsRefresh` event) → `/active_proposals`.
  - Tips (`useTipNotifications`: `get_recent_tips_count`, every 2 min) → `/tips`.
  - Collectibles (`useCollectiblesNotifications`: Sneed RLL voting rewards for hotkey neurons, ICPSwap LP fees, SNS neuron maturity, each ≥ fee and ≥ USD threshold (`localStorage.collectiblesThreshold`, default $1); every 5 min; tooltip "N items to collect worth $X (fees, rewards, maturity)", privacy-masked) → ConsolidateModal (collect all).
  - Low cycles → top-up dialog; outdated bots → upgrade dialog; bot health (chores/log errors) → dialog (outside this slice).
- **Settings**: `/me` → "Notification Bar Settings": "Show Header Notifications" (master), "SMS When Mentioned", "Reply Notifications", "Tip Notifications", "Message Notifications", "Collectibles Notifications", "Collectibles Threshold", "Votable Proposals", "Outdated Bots", "Low Cycles" (+ bot ones); stored in localStorage and backend user settings, synced across tabs via storage/custom events.
- **Gating/state**: counts are "sticky" per principal across page remounts (`useStickyCount`); each hook creates its own forum actor (`createNotificationForumActor`).
- **Redesign notes**: two unrelated notification systems: this client-polled bar (forum/SMS/wallet events) vs. the Sneed Notifications inbox (push/Discord/SMS delivery) which has no badge; settings for the bar are on `/me`, for the inbox on `/notifications`.

## Shared composer and rendering tools

- **EmojiPicker**: favourites row from usage stats (`localStorage.sneed_emoji_usage_v1`), "Open emoji palette" with categories + "Search emojis…", custom emoji list ("Add custom emoji (paste here)…", `localStorage.sneed_custom_emojis_v1`, synced across tabs); inserts at caret. Used by Topic, ThreadViewer forms, SMS compose, MessageDialog (not Conversations, not Discussion).
- **MarkdownButtons**: "Bold", "Italic", "Insert link", "Inline code" (Markdown). Same users as EmojiPicker.
- **MarkdownBody**: safe Markdown (blocks javascript:/data:/vbscript: URLs, links open in new tab), single newlines kept, @-mention links rendered as name chips → `/principal`. Used across forum, feed, posts, SMS, notifications, Drive, Principal.
- **MentionTextarea**: see @-mentions.
- **PrincipalDisplay / PrincipalInput**: names, nicknames, verified badges; context menu (copy, "Send Message", set nickname, copy ICRC-1 account); picker for recipients/creators.
- **useTextLimits**: forum limits + premium overrides.

---

## Slice observations

**(a) Entities, with their views and actions**

- **Forum** (one per SNS): view `/forum`; create ("Create Forum"); admin edit/delete in `/admin/forum`. Feed item type "Forum".
- **Topic / subtopic**: views Forum cards, `/topic/:id`; user create only special topics (General, Governance, Preproposals; Proposals auto); admin CRUD elsewhere. Stats: thread count, unread count.
- **Thread**: views Topic list, `/thread`, proposal page, Feed, `/posts` My Threads, `/principal` Posts tab; actions create (+poll), edit (author/admin), vote (neurons), react, tip, add poll; read marks; proposal link. No user delete.
- **Post** (comment/reply): views ThreadViewer, `/post`, Feed, `/posts` (mine, replies to me), `/principal`; actions create, reply, edit, delete, vote/retract, react, tip, message author, add poll; per-vote audit on `/post`.
- **Vote** (neuron-weighted, posts and threads): cast/retract in ThreadViewer (selected neurons per thread) and legacy Discussion (all neurons); breakdown only for posts on `/post`.
- **Reaction** (emoji, ≤10/user/target): ThreadViewer only.
- **Tip** (post tip, thread tip; ICRC-1 transfer + forum record + ledger verification): send via TipModal; view TipDisplay pills, `/tips` (post tips only), header tips badge.
- **Poll** (thread or post; options, VP power, end time, vote-change rule): create in Topic form or ThreadViewer; vote in ThreadViewer; read-only in Feed.
- **Mention**: write via MentionTextarea; view `/posts?tab=mentions`, header chip; optional SMS ping (opt-in on `/me`).
- **Read mark / seen marks**: thread read (unread badges/highlights), replies/tips/mentions/messages seen (header counts), feed last-seen (localStorage), notification seen (localStorage).
- **Feed item** (forum/topic/thread/post + Sneedex auction): `/feed`, Hub widget.
- **Message** (Sneed SMS; subject+body, recipients, reply_to): views Inbox/Sent/All list + modal, Conversations bubbles, `/msg/:id` tree; actions compose, reply, reply all, remove self, load older, refresh.
- **Conversation** (derived: participant set): Conversations view; actions start new, send.
- **Notification** (sneed_notifications history item: app, topic, level, title, body, link, deliveries): inbox list, single view; mark read (device-local).
- **Notification app / permission / topic rule**: settings Apps sections; allow/block, demote, copy-only, caps, per-topic level/mute.
- **Channel** (Sneed SMS, push device, Discord; telegram/email coming soon): add, verify code, resend, remove; level routing, always-copy, quiet hours, pause.
- **Message balance** (prepaid ICP for paid deliveries): top up, withdraw, history, low-balance alert.
- **Sneed Monitor watch** (staging): watch/unwatch SNS/NNS, topics, balance, auto top-up.
- **Header alert counts** (replies, mentions, messages, tips, votable proposals, collectibles, bots, cycles, update).

**(b) Overlaps with other areas**

- Proposal pages (`/proposal`) embed ThreadViewer or the legacy Discussion; the Topic "Proposals" thread list links back to proposals. Governance alerts appear three ways: header votable chip → `/active_proposals`, Sneed Monitor (notifications settings), Proposal discussion threads.
- Wallet: tipping uses the wallet token list and balances; notification top-up/withdraw and Sneed Monitor are ICP payments; collectibles chip opens the wallet's ConsolidateModal.
- Principal/user pages: `/principal?id=` has "Send Message" and a public Posts tab (duplicate of `/posts` for others); PrincipalDisplay context menu offers "Send Message" everywhere.
- `/me`: holds the header notification settings, "SMS When Mentioned", links to `/posts`, `/tips`, `/sms`, notification inbox/settings, and forum stats tiles.
- Sneedex: auctions injected into `/feed`; Hub shows Feed widget.
- Premium: raises forum post/thread body limits and SMS subject/body/recipient limits (and SMS rate limit, not surfaced).
- Neurons: forum and poll voting depend on the wallet's SNS neuron cache (hotkey setup lives in neuron pages/help).
- Feedback (`/feedback`) creates public forum threads and SMS pings; admin pages `/admin/forum`, `/admin/sms`, `/admin/feedback` are the moderation side (topic CRUD, delete/undelete, forum admins, premium configs, SMS authorized senders).
- Help: `/help/forum`, `/help/sms`, `/help/notifications`.

**(c) Dead, hidden, half-built or confusing**

- Two discussion engines: `ThreadViewer` (current) and `Discussion` (legacy, used only before a proposal thread exists) with different voting APIs, no reactions/polls/emoji tools.
- Dead code: `Feed.jsx` `renderFeedItem` (~600 lines, superseded by FeedItemCard), `filterFeedItemsByType`, `getTypeColor`; `Posts.jsx` `renderPost`/`renderThread`; `Tips.jsx` `renderTipRow`/`renderTipCard`; `ThreadViewer` imports `deleteThread`/`updateThread` but has no thread delete.
- `/tips` ignores thread tips (canister has thread-tip-by-user pages); header tip count source unclear for thread tips.
- `/me` links `/posts?tab=my-threads`, but `/posts` only honours `tab=mentions` (opens Replies to Me).
- Logged-out users see "Create Forum", special-topic prompts and the "Start a New Thread" form whose buttons silently do nothing.
- Topic/forum links drop `?sns=`; subtopic pills do full page reloads; Feed shows the global SNS dropdown but uses its own SNS filter; Hub's "Sneed Forum" widget shows all SNSes.
- Admin edit/delete visibility uses the Sneed Hub backend admin, not the forum canister's admin list.
- Poll creation duplicated (Topic vs Poll.jsx) with inconsistent validation; Poll uses `window.alert`.
- SMS: three composers (compose modal, MessageDialog, Conversations), `premium_rate_limit_minutes` fetched but never shown, MessageDialog doesn't refresh the shared inbox; `/msg/:id` scans the whole mailbox to find replies; `?recipient=` deep link exists but no in-app link uses it (only external/agent use).
- Notifications: two parallel systems (client-polled header bar vs. Sneed Notifications inbox with no badge); inbox read state is per device; settings text refers to "the bell in the header" which actually shows SMS counts; Sneed Monitor staging-only and hidden inside notification settings; push/Discord channels depend on relays ("Coming soon" otherwise).
- Hidden: `/tips` keyboard "c"/"v" test confetti; neuron selection persisted per thread in localStorage; negative-score posts auto-collapsed; new posts auto-upvoted with your neurons; mention SMS sent from the poster's own principal.
- Legacy/redirect routes: `/notifications/inbox` → `/notifications?tab=inbox`; `/notifications/push?t=&h=` → `/notifications/:id`.
