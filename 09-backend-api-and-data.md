# 09. Backend: API and data

## Endpoints

Names are indicative. Keep them consistent if you change them.

### Called by the tracker

No login. Each request is checked by site ID and by the request's origin (`06-events-and-event-map.md`, "What ingest accepts").

| Endpoint | Purpose |
|---|---|
| `GET /v1/sites/{site_id}/config` | Everything the tracker needs: event map, consent mode, prompt rules, channel flags |
| `POST /v1/events` | A batch of events |
| `POST /v1/optin` | Consent to tracking, an opt-in to a channel, or a call-back request |
| `POST /v1/optout` | A withdrawal made on the store: tracking revoked, notifications turned off, on-site messages dismissed for good |
| `POST /v1/onsite/next` | The on-site message waiting for this visitor, if any |
| `POST /v1/onsite/events` | An on-site message was shown, clicked or dismissed |
| `POST /v1/push/receipts` | The service worker displayed a notification |
| `POST /v1/setup-sessions/exchange` | Swaps a one-time setup token for a setup credential |

### Called from the store in setup mode

These need the setup credential in an `Authorization` header (`08-site-connection-and-scan.md`).

| Endpoint | Purpose |
|---|---|
| `POST /v1/scans/{id}/snapshots` | A page snapshot from the walk-through |
| `GET`, `PUT /v1/sites/{id}/event-map/draft` | Read and change the draft map during review |
| `POST /v1/sites/{id}/push-test-devices` | Register this browser for test notifications |

### Public files and links

| Endpoint | Purpose |
|---|---|
| `GET /rc.js`, `/rc-ui.js`, `/rc-setup.js`, `/sw/v1.js` | The tracker's files |
| `GET /c/{click_id}` | Records a click and redirects to the store |
| `GET`, `POST /u/{token}` | Email unsubscribe |
| `GET`, `POST /p/{token}` | The shopper's preferences page |

### Webhooks

| Endpoint | Purpose |
|---|---|
| `GET /webhooks/meta/{webhook_key}` | Meta's verification handshake |
| `POST /webhooks/meta/{webhook_key}` | Receipts, shopper messages and template status changes from Meta |

### Called by the console

These need the merchant's session. Every one is scoped to sites the merchant owns.

| Group | Endpoints |
|---|---|
| Auth | `POST /v1/auth/signup`, `/login`, `/logout`, `/password-reset`, `/password-reset/confirm`; `GET /v1/me` |
| Sites | `POST /v1/sites`, `GET /v1/sites`, `GET /v1/sites/{id}`, `POST /v1/sites/{id}/verify`, `DELETE /v1/sites/{id}`, `GET /v1/sites/{id}/recent-events` |
| Setup mode | `POST /v1/sites/{id}/setup-sessions` |
| Scans | `POST /v1/sites/{id}/scans`, `GET /v1/sites/{id}/scans`, `GET /v1/scans/{id}`, `POST /v1/scans/{id}/finish` |
| Event map | `GET`, `PUT /v1/sites/{id}/event-map/draft`; `POST /v1/sites/{id}/event-map/publish`; `GET /v1/sites/{id}/event-map/versions`; `POST /v1/sites/{id}/event-map/rollback` |
| Channels | `GET /v1/sites/{id}/channels`; `PUT /v1/sites/{id}/channels/{channel}`; `POST /v1/sites/{id}/channels/{channel}/test`; `DELETE /v1/sites/{id}/channels/{channel}` |
| Message templates | `GET /v1/sites/{id}/message-templates`; `POST /v1/sites/{id}/message-templates/generate`; `PUT /v1/message-templates/{id}`; `POST /v1/message-templates/{id}/submit`; `POST /v1/sites/{id}/message-templates/sync`; `POST /v1/message-templates/{id}/retire` |
| Coupons | `GET /v1/sites/{id}/coupon-pools`; `POST /v1/sites/{id}/coupon-pools`; `DELETE /v1/coupon-pools/{id}/unused` |
| Messages | `GET /v1/sites/{id}/messages`; `GET /v1/messages/{id}`; `POST /v1/messages/{id}/approve`, `/edit`, `/reject`; `GET /v1/messages/{id}/trail` |
| Conversations | `GET /v1/sites/{id}/conversations`; `GET /v1/conversations/{id}`; `POST /v1/conversations/{id}/reply`, `/pause-assistant`, `/resolve` |
| Call tasks | `GET /v1/sites/{id}/call-tasks`; `POST /v1/call-tasks/{id}/reveal`; `POST /v1/call-tasks/{id}/outcome` |
| Shoppers | `GET /v1/sites/{id}/shoppers`; `GET /v1/shoppers/{id}`; `DELETE /v1/shoppers/{id}` |
| Settings | `GET`, `PUT /v1/sites/{id}/settings`; `GET /v1/sites/{id}/audit-log` |
| Analytics | `GET /v1/sites/{id}/analytics/funnel`, `/exits`, `/reasons`, `/reach`, `/recovery`, `/profile` |
| Suggestions | `GET /v1/sites/{id}/suggestions`; `POST /v1/suggestions/{id}/apply`, `/dismiss` |

### Agents

Each agent has an A2A endpoint and an Agent Card (`10-agents-and-a2a.md`). These endpoints are for the coordinator only and must not be reachable from the internet.

## Visitors and shoppers

- A **visitor** row is created the first time a browser sends an event. A **shopper** row is created with it. At first one shopper has one visitor.
- When an opt-in arrives with a phone number or email address, normalise it (phone numbers to E.164, email addresses to lower case with spaces trimmed) and look for the same address on another shopper of the same store, using a keyed hash of the address.
  - If there is none, add the contact point to this visitor's shopper.
  - If there is one, **merge** the two shoppers. The older shopper record survives. Its visitors, contact points, consents, events, sessions, messages and recoveries are joined with the newer one's, and the profile is rebuilt. If both have an open recovery, keep the one opened last and close the other as `no_action`.
- The no-message group is decided once, when the shopper is created: store a bucket from 0 to 99 taken from a stable hash of the shopper ID. A shopper is in the group when their bucket is below `no_message_group_pct`. A merge keeps the surviving shopper's bucket.

## Sessions

- A session is the events of one visitor under one `session_id`.
- A job runs every minute and ends every session whose last event is at least `abandon_after_minutes` old. Tests run this job on demand after moving the clock.
- When a session ends, the coordinator runs Intent and Profile for that shopper.

## Recoveries

- If the score is at or above `intent_threshold`, and the shopper has not purchased since the scored events, the coordinator opens a recovery and runs the chain in `10-agents-and-a2a.md`.
- A shopper has one open recovery at a time. A later session updates the open recovery's score and evidence. It does not open another and does not start another first message.
- Events counted in a closed recovery are not scored again.
- A call-back request opens a recovery at once, without waiting for the session to end and without the intent threshold, because the shopper asked to be contacted.
- A purchase closes the open recovery, cancels anything still scheduled for that shopper, and releases any code that was reserved but never sent.

A recovery closes in one of these ways:

| Status | Meaning |
|---|---|
| `recovered` | The shopper purchased and the purchase is attributed to a message (`15-replies-and-learning.md`) |
| `purchased` | The shopper purchased, but before any message was delivered or without using the code or the link |
| `not_recovered` | No purchase within `attribution_window_days` of the last delivered message, or of opening if nothing was delivered |
| `no_action` | Nothing suitable could be offered, for example no in-stock alternative |
| `blocked` | It failed Guard's eligibility checks or the shopper could not be reached. The reason is kept |
| `opted_out` | The shopper opted out during the recovery |

Every recovery, including blocked ones, records whether the shopper purchased within `attribution_window_days` of its opening. Learning needs this for the no-message group.

## Coupons

Codes must exist in the store, so the backend never invents them.

```ts
interface CouponProvider {
  reserve(siteId: string, tier: Tier, shopperId: string): Promise<Coupon | null>;
  confirm(couponId: string, messageId: string): Promise<void>;
  release(couponId: string): Promise<void>;
  markUsed(siteId: string, code: string, orderId: string): Promise<void>;
}
```

v1 has one provider: pools of codes that the merchant uploads per tier, each pool with a `valid_until` date that the merchant confirms matches the store. Providers for store platforms come later behind the same interface.

| State | Meaning |
|---|---|
| `unused` | In the pool |
| `reserved` | Offer has set it aside for one shopper's message |
| `issued` | The message carrying it was sent, shown or read out |
| `used` | A `purchase` event arrived with this code |
| `expired` | Its pool's `valid_until` has passed |

- Offer reserves a code. Delivery confirms it at the moment the message goes out. A cancelled, rejected or failed message releases it back to `unused`.
- Reserving is atomic, so two shoppers can never be given the same code.
- The demo store checks codes through a small test endpoint, so tests can check single use and expiry.

## Links

### Resume link

`{checkout_link}` is `https://OUR_DOMAIN/c/{click_id}`.

- It redirects to the cart address when the shopper had a cart holding the pitched product, and to the pitched product's address otherwise. For a welcome message, which is not about a product, it redirects to the page the shopper opted in on.
- It adds `?rc_click={click_id}`, so the tracker can tie the return visit to the message.
- A cart cannot be rebuilt on another device without the store's help. The optional `cart_link_template` setting covers stores that support cart links.
- It records the first click time and a count.
- It redirects only to the site's own verified domain. An unknown ID redirects to nowhere and shows a plain "link not found" page.

### Unsubscribe link

`https://OUR_DOMAIN/u/{token}` is in every email.

- `GET` shows a page with one button, "Unsubscribe from {store_name} emails", and a link to the preferences page. `GET` alone never unsubscribes, because mail scanners open links.
- `POST` unsubscribes. Mail apps send it for one-click unsubscribe; the page's button sends it too.

### Preferences page

`https://OUR_DOMAIN/p/{token}` lets a shopper manage how the store may contact them, without signing in.

- A switch for each channel they have opted in to
- "Stop all messages from {store_name}"
- "Delete my data", which erases them after a confirmation (`17-privacy-and-security.md`)

Tokens for both pages are random, stored hashed, and tied to one shopper.

## Analytics

| Endpoint | Returns |
|---|---|
| `funnel` | Shoppers reaching product view, add to cart, each checkout step and purchase, by day |
| `exits` | The last action before exit, by page type and trigger, with counts and shares |
| `reasons` | Recoveries by exit reason, with the revenue at stake for each |
| `reach` | Abandoners who could be reached on WhatsApp, on email, by web push, on-site only, and not at all |
| `recovery` | Per channel: messages sent, delivered, read, clicked, replied. Overall and by reason, play and tier: recovered orders, recovered revenue, discount cost, message cost, and lift over the no-message group |
| `profile` | The store profile (`12-customer-profile.md`) |

Definitions:

- **Revenue at stake**: for each recovery in the range that has not ended in a purchase, the cart value at exit, or the price of the pitched product when there was no cart.
- **Recovered revenue**: the order value of purchases attributed to a message.
- **Discount cost**: for a percentage code, the amount taken off; for a free-shipping code, the shipping cost the shopper was shown.
- **Message cost**: the number of charged messages on each channel times that channel's cost per message. On WhatsApp, template messages are charged and replies inside the window are not.
- **Lift**: the purchase rate of messaged shoppers minus the purchase rate of the no-message group, both within the attribution window.

## Tables

Every table carries `site_id`, and every query is scoped by it.

| Table | Holds |
|---|---|
| `merchants` | Email, password hash, name |
| `sites` | Merchant, domain, store name, currency, time zone, status, webhook key |
| `settings` | One validated JSON document per site (`16-settings.md`) |
| `channels` | Per site and channel: enabled, status, encrypted config, daily cap, cost per message, last error |
| `event_maps` | Version, state (`draft`, `published`, `retired`), page types, entries |
| `scans` | Status, sample plan, crawl-blocked flag, times |
| `snapshots` | Scan, address, source, page type, snapshot data |
| `setup_sessions` | Hashed token, hashed credential, used and expiry times |
| `visitors` | Visitor ID, shopper, first and last seen, device, language, tracking consent |
| `shoppers` | Encrypted first name, no-message bucket, stop-all flag, merged-into |
| `contact_points` | Shopper, channel, encrypted address or push subscription, address hash, state (`active`, `opted_out`, `unreachable`), last response time, WhatsApp window end |
| `consents` | Shopper, visitor, channel (including `tracking`), granted or withdrawn, text version, source, page, time |
| `events` | Visitor, shopper, session, time, map version, page type, path, action, data |
| `sessions` | Visitor, shopper, start, last event, end, purchased flag |
| `products` | Product ID, name, category, price, address, in-stock flag, last seen. Built from tracked data |
| `profiles` | Shopper, profile fields, summary, updated time |
| `recoveries` | Shopper, status, score, reason, evidence, play, product, tier, opened and closed times, purchase-within-window flag |
| `messages` | Recovery, shopper, contact point, channel, direction, mode, purpose, message number, template, text with placeholders, encrypted rendered text, status, defer and failure reasons, provider message ID, coupon, click ID, times |
| `message_templates` | Channel, purpose, shape, language, tone, text fields, version, origin, status, provider name and ID, rejection reason |
| `conversations` | Shopper or unknown contact, needs-a-person flag, assistant-paused-until, resolved time |
| `call_tasks` | Shopper, recovery, requested time, preferred time, due time, status, attempts, brief, notes |
| `coupon_pools` | Tier, `valid_until`, confirmed time |
| `coupons` | Pool, code, state, shopper, message |
| `clicks` | Click ID, message, target address, first click time, count |
| `link_tokens` | Hashed token, shopper, kind (`unsubscribe`, `preferences`) |
| `agent_tasks` | Trace ID, agent, step, state, input, output, times, whether a model was called |
| `suggestions` | Kind, evidence, proposed change, state |
| `audit_log` | Merchant, action, subject, time |
| `webhook_events` | Provider, event key, received time. The event key is unique, so a repeated webhook is ignored |
