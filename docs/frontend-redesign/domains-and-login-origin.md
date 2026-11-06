# Redesign: domain and login origin

Two decisions made on 2026-11-06, before the redesign starts. They affect login and the deploy setup,
so they need to be in place before the first real user logs in.

1. The site lives at **`sneedhub.app`**, on a single origin. Sections stay path-prefixed, as in the
   [sitemap](sitemap.md) (`sneedhub.app/wallet/…`, `sneedhub.app/govern/…`).
2. The II **derivation origin is the canister-id URL of this fork's new frontend canister**, not
   the domain.

Background: this repo is a fork. The old Sneed Hub (asset canister `pxtkg-giaaa-aaaal-ajjzq-cai`,
served at `app.sneeddao.com`) stays live as it is. The fork gets a **new** frontend canister.
[spec/derivation_origin.md](../../spec/derivation_origin.md) explains why principals must not
depend on a domain. Its research (§1) applies here unchanged.

## 1. `sneedhub.app`

### Registration

- **`.app` is HTTPS-only.** The whole TLD is on the browsers' HSTS preload list. The IC boundary
  nodes issue the TLS certificate, so this is fine. It only means nothing can ever be served over
  plain `http` on this domain. Local development on `localhost` is not affected.
- **What the DNS provider needs:** CNAME flattening or ALIAS/ANAME at the apex (see below). For
  safety it also needs what [spec/derivation_origin.md §2.2](../../spec/derivation_origin.md) lists:
  organisation accounts with several admins, registry lock, DNSSEC, CAA and multi-year auto-renew.
  Once principals come from the canister (§2 below), losing the domain costs availability, not
  funds. It is still the address users know.

### IC custom domain

From the [custom domains guide](https://docs.internetcomputer.org/guides/frontends/custom-domains.md)
(read 2026-11-06):

- **DNS records:**
  - the domain itself → `sneedhub.app.icp1.io`. **The apex cannot have a CNAME**, so this needs the
    provider's ALIAS/ANAME record or CNAME flattening (Cloudflare does this). On Cloudflare, use
    "DNS only" (not proxied) and turn off Universal SSL.
  - `TXT _canister-id.sneedhub.app = <new canister id>` (exactly one)
  - `CNAME _acme-challenge.sneedhub.app → _acme-challenge.sneedhub.app.icp2.io`
- **`/.well-known/ic-domains`** on the new canister lists `sneedhub.app`. Today the file holds
  `app.sneeddao.com` (`src/app_sneeddao_frontend/public/.well-known/ic-domains`). The fork must
  replace it, because the old canister keeps serving `app.sneeddao.com`.
- **Registration API:** `POST https://icp.net/custom-domains/v1/sneedhub.app`, then `GET` the same
  URL for the status (`registering` → `registered`). `GET …/validate` checks the DNS first. `PATCH`
  repoints the domain after a DNS change, and `DELETE` removes it. An `expired` registration needs a
  new `POST`.
- Agents are already safe. The app sets the agent `host` to `https://icp0.io` explicitly (for
  example in `SendTokenModal.jsx` and `PendingStakesNotice.jsx`), so it does not fall back to the
  page origin on a custom domain. Keep it that way.

### The canister URL works too

The app must also work completely at **`https://<new canister id>.icp0.io`**, the address nobody can
take away. Both URLs serve the same app with the same paths, and both log in with the same principal
(§2).

## 2. Derivation origin: the new canister's URL

**Decision.** Every II login (1.0 and 2.0) passes
`derivationOrigin: 'https://<new frontend canister id>.icp0.io'`. Principals then depend on a
canister, not on `sneedhub.app`. The domain can be lost, moved or changed without changing who
owns what. This is option A of
[spec/derivation_origin.md §2.1](../../spec/derivation_origin.md), applied to the fork's new
canister instead of `pxtkg`.

New frontend canister id: **`TBD`** (fill in when it is created; add it to `canister_ids.json`).

### What to build

- **Login.** Add a per-network `DERIVATION_ORIGIN` constant in `src/AuthContext.jsx`, and pass it
  in both `loginWithII1` and `loginWithII2` (`authClient.login({ identityProvider,
  derivationOrigin, … })`). Staging uses its own canister's URL. Local uses the local canister URL
  or none.
- **`/.well-known/ii-alternative-origins`** on the new canister:
  `{"alternativeOrigins": ["https://sneedhub.app"]}`. The canister URL itself needs no entry
  (spec §1, item 6). II fetches the file from
  `https://<cid>.icp0.io/.well-known/ii-alternative-origins` on every login, so it must be:
  - certified (the asset canister certifies it),
  - served without redirects,
  - sent with the headers `Content-Type: application/json`, `Access-Control-Allow-Origin: *` and
    `Cache-Control: no-store` (add a rule to `.ic-assets.json5`; `.well-known` is already
    un-ignored there).
- **An origin that is not in the list cannot log in.** II refuses it as "not listed in the list of
  allowed alternative origins". Any domain added later needs an entry here too.
- **`/.well-known/ii-derivation-origin`** (written by `scripts/build-api-docs.mjs`) publishes the
  canister URL.
- Write the URL with `icp0.io`. `ic0.app` gives identical principals, but `<cid>.raw.icp0.io` does
  not (spec §1, item 5).

### Rules that follow from it

- **Set it before the first real login on the new site.** Changing the derivation origin later means
  a full principal migration (spec §3–4). It is a one-way door.
- **This canister must live forever.** If it is deleted, or freezes and is lost, nobody can log in
  with these principals again, because canister ids are never reused. It needs a large freezing
  threshold, a sneed_mon watch with auto top-up, and no "deploy to a fresh canister". Future
  redesigns upgrade this canister in place. A later canister can join only as an alternative origin
  listed here.
- **Changing the list after DAO handover.** Once the canister is under Sneed DAO
  (`commit_proposed_batch` proposals), adding or removing an origin takes a proposal: about 4 days,
  or less when a majority votes yes early. For emergencies, see the option A notes in spec §2.1
  and §6.

### Consequence to plan for: new principals

A user logging in on the new site gets a principal derived from the new canister. It differs from
their principal on `app.sneeddao.com`, which is derived from that domain. If the new site uses the
same backend canisters (ledgers, SNS neurons, locks, bots, names, forum, SMS), a returning user sees
an empty wallet and none of their neurons, locks, bots, name or posts until they move them. The only
way to keep the old principals would be to derive from `https://app.sneeddao.com`, which is exactly
the domain risk this decision removes. So the move wizard designed in spec §3–4 belongs in the new
site. The old site's principal is "old", the new site's is "new", and the two-login wizard runs on
the new site.

## Open questions

1. Who holds `sneedhub.app`, at which registrar, and with which DNS provider (it must support
   apex flattening)?
2. Does the move wizard (spec §4) ship with the new site's first release, or does the new site start
   with only a "your old account is at app.sneeddao.com" notice?
