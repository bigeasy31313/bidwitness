# BidWitness pipeline

How a stranger becomes a paid subscriber, and how each brief gets made and sent.
Verified against the live systems on 2026-11-13.

Status at that check: site live on GitHub Pages, four n8n workflows active, one Stripe
webhook endpoint live, subscriber table empty, zero subscriptions.

## Systems

| Piece | What it does |
| --- | --- |
| GitHub Pages (`gh-pages` branch) | Serves bidwitness.com. `main` is docs/legacy and is not deployed. |
| Make.com (Free) | Website intake scenario: Jira ticket, confirmation email, subscriber sync. |
| n8n Cloud (bidwitness.app.n8n.cloud) | Signup intake, Monday brief, Stripe billing, unsubscribe. |
| n8n Data Table "BidWitness Subscribers" | Single source of truth for who gets a brief. |
| Brevo | All outbound mail, sent over the HTTP API (`/v3/smtp/email`) from hello@bidwitness.com. |
| Stripe (Theta Work Labs LLC) | $299/month subscription, 14-day trial, two free briefs. |
| Jira, project Bid (SCRUM) | Ticket per signup, work tracking. |
| Permit data | Socrata / ArcGIS open data, CORS-open, queried from the browser and from n8n. |

## 1. Site

- `index.html` — metro/trade picker and the brief request form. The form posts to the
  FormSubmit relay for hello@bidwitness.com, which forwards to the Make webhook. The
  confirmation card shows the Jira ticket number Make returns.
- `live/brief.html?city=&trade=` — client-side brief generator; queries the permit feed
  and does the trade matching in the browser, no backend.
- `live/stats.json` — permit counts on the hero, blanked when the file goes stale.
- `thanks.html`, `unsubscribe.html`, `sample-brief.html`, four policy pages.
- Checkout — one Stripe payment link ($299/month, 14-day trial), linked from the hero CTA.

## 2. Intake (Make)

Custom webhook receives the FormSubmit forward, then:

1. `jira:createAnIssue` — opens the ticket.
2. `datastore:AddRecord` — records the signup in the Make data store.
3. `http:ActionSendData` → Brevo — sends the confirmation email.
4. `http:ActionSendData` → n8n signup webhook — syncs the subscriber into the n8n table.
5. `gateway:WebhookRespond` — returns `{ok, ticket}` so the page can show the ticket.

The Brevo send must stay an HTTP module. Converting it back to Make's Email/SMTP module
fails, because Make cannot resolve an SMTP connection created outside its own editor.

## 3. Subscriber record (n8n Data Table)

Columns: `email`, `city`, `trade`, `trade_raw`, `status`, `signup_date`, `last_brief_at`,
`last_brief_count`, `source`, `stage`, `subscription_id`, `customer_id`, `briefs_sent`,
`past_due_invoice`, `grace_invoice`.

- `status` is coverage capability: `active`, `manual` (no machine-readable feed),
  `needs_review` (city not recognised).
- `stage` is the money state: `trial` on signup → `active` → `past_due` → `canceled`, or
  `unsubscribed`.
- Upsert key is `email` + `city` at intake, and `email` alone from billing.

## 4. Weekly brief (n8n "Monday brief")

Cron: Mondays 06:00 America/New_York. Twelve nodes.

1. Get Subscribers — rows where `stage` is `trial`, `active` or `past_due`.
2. Plan Queries — one permit query per subscriber, across Chicago, New York, Seattle,
   Austin, New Orleans, Cincinnati and Washington DC.
3. Fetch Permit Data — HTTP, text response, parsed in code so index order survives.
4. Tag and Parse — trade canonicalisation and open-slot matching.
5. Build Briefs — HTML, with the unsubscribe link and postal address.
6. Has Brief to Send? → Prepare Brevo → Send Brief (Brevo) → Mark Brief Sent.
7. Apply Brief Cap — decides who actually gets a brief this week.

Who gets a brief:

- `unsubscribed` or `canceled` — never.
- `past_due` — exactly one grace brief, keyed to the invoice that bounced, so Stripe's
  retries do not keep sending.
- `active`, or anything with a `subscription_id` — weekly, uncapped.
- Cardless trial — two briefs, then stop.

A city feed that fails is labelled "brief delayed" rather than "0 open slots", so a 503
never becomes a false claim.

## 5. Billing (n8n "Stripe billing")

Stripe posts four live events to the n8n webhook:
`checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`,
`customer.subscription.deleted`.

| Event | Stage written | Notes |
| --- | --- | --- |
| `checkout.session.completed` | `trial` | Writes `city`/`trade` from the Payment Link custom fields, plus customer and subscription ids. |
| `invoice.paid` | `active` | `billing_reason = subscription_create` is the $0 trial invoice, so it stays `trial`. |
| `invoice.payment_failed` | `past_due` | Stores the bounced invoice id for the grace rule. |
| `customer.subscription.deleted` | `canceled` | Matched on `subscription_id`, because the event carries no email. |

Status paths update the stage without wiping metro/trade; the cancellation path works off
the subscription id.

## 6. Unsubscribe (n8n "Unsubscribe")

Every brief carries a signed-token link. The webhook verifies the token, marks the row
`unsubscribed`, and replies. An invalid or stripped link returns a manual-removal message
instead of failing silently.

## 7. Ops

- Deploy: commit to `gh-pages`, push, Pages rebuilds in about a minute.
- Delivery evidence: Brevo event log per send; n8n execution history for failures.
- The Stripe account descriptor shows THETA WORK LABS, so the checkout charge does not
  read as BidWitness.

## Known gaps (2026-11-13)

1. No customer has run the full path yet: table empty, zero subscriptions, every test event
   injected by hand.
2. The Stripe webhook does not verify signatures, so anyone holding the URL could post a
   fabricated stage change. Add signature checking before paid traffic.
3. Status-only billing upserts still map `city`/`trade` from the event; confirm a status
   event without those fields cannot blank an existing row's city and trade.
4. Two records of the same subscriber exist (Make data store and n8n table). Only the n8n
   table drives briefs and billing; the Make copy is historical.
5. Policy pages are not lawyer-reviewed and the governing state in the terms is unset.
6. Manual markets (Mississippi and similar) are hand-built, two business days, nothing
   automated.
