# 17. Privacy and security

Retarget Customer handles what shoppers do and how to reach them. The safeguards below are part of the product, not an add-on. Each one that can be tested is tested (`19-acceptance-tests.md` and `20-build-order-and-reporting.md`).

The consent wording and retention periods in these files are starting points. The owner should have them reviewed for the countries the store sells in, such as India's DPDP Act and the EU's GDPR, before real shoppers are messaged.

## Consent

Each kind of consent is separate. Having one never implies another.

| Consent to | Given by | Withdrawn by |
|---|---|---|
| Tracking | Accepting the store's banner or the tracker's built-in notice | The store calling `consent.revoke()`; clearing cookies |
| WhatsApp messages | Ticking the box beside the number in the capture prompt, or the store's own checkout box | Replying with an opt-out word; the preferences page |
| Email | Ticking the box beside the address | The unsubscribe link; the preferences page |
| Browser notifications | Pressing "Notify me in this browser" and allowing it in the browser | The "Turn off" action; the browser's own settings; the preferences page |
| On-site messages | Covered by consent to tracking | "Don't show again"; the preferences page |
| A phone call | Requesting a call with the box ticked. Valid for one call-back, for `call_request_valid_hours` | Declining on the call; the request expiring |

- Consent boxes are never pre-ticked.
- Every grant and every withdrawal is recorded in `consents`: the shopper, the channel, the version of the text they saw, the source, the page, and the time. Records are kept after an opt-out, as proof of what was agreed and when it ended.
- A shopper who chooses "Stop all messages" is opted out of every channel at once.
- Phone numbers and email addresses are not verified in v1. A mistyped number can reach the wrong person, who can opt out with one reply. Double opt-in is a later option.

## Personal data

Personal data here means phone numbers, email addresses, first names and push subscriptions.

- **Encrypted at rest**, with the key from `ENCRYPTION_KEY`. Lookups by address use a keyed hash stored beside the encrypted value.
- **Masked in the console**: `+91 98XXX XXX10`, `pr***@example.com`. The one exception is "Reveal number" on the Calls page, which writes to the audit log.
- **Never in logs**, never in `agent_tasks`, never in error reports.
- **Never sent to a language model.** Agents work with the shopper ID. Real values are filled into a message by code, at the moment of sending.
- **IP addresses** are used for rate limiting and are not stored with events.
- Test fixtures use invented names, numbers and addresses.

## What each model call may contain

| Agent | Receives | Never receives |
|---|---|---|
| Grouping, Classifier | Page structure and labels from snapshots, which hold no input values | Any shopper data |
| Profile (summary) | Profile fields and the product's name | Name, phone number, email address, IDs |
| Copy | Purpose, reason, shape, format, three profile fields, language, voice, the product's name and category | Name, phone number, email address, IDs, events |
| Humanizer | Draft text, limits, tone, language, the list of placeholders | Everything else |
| Reply | The shopper's message as written, the purpose of our last message, the product's data, the store's facts | Name, phone number, email address, IDs |

Every model output is validated against its schema before it is used.

## Untrusted text

Three kinds of text come from outside: text on the store's pages, product names and categories, and shoppers' messages. All of it is data for the agents, never instructions.

- It cannot change settings, tiers, channels or Guard's checks, because none of those is decided by a model.
- Prompts mark it as content and say that anything in it that reads like an instruction is to be treated as content.
- Values filled into messages are stripped of line breaks and cut to length (`13-message-pipeline.md`).

## Isolation between stores

- Every table carries `site_id`, and every query is scoped by it.
- Every console request is checked against the sites the signed-in merchant owns.
- A test proves that one merchant cannot read or change another's sites, shoppers, messages, templates or settings, by ID guessing or otherwise.

## The open endpoints

The tracker's endpoints accept requests from anyone's browser, so they trust nothing:

- The request's origin must be the site's verified domain.
- Bodies are validated against a schema; unknown fields are dropped; sizes are capped.
- Requests are rate-limited by site and by IP address.
- The never-captured list is enforced again on the server (`06-events-and-event-map.md`).
- An opt-in is accepted only with its consent text version and, for a store checkout opt-in, its consent flag.

## Setup mode

- The setup token is random, single-use, and must be used within 5 minutes. It travels in the address fragment, which browsers do not send to servers, and the tracker removes it from the address bar at once.
- It is exchanged only from the site's own origin.
- The setup credential lasts 2 hours, is sent in a header and never as a cookie, works for one site, and can only post snapshots, edit the draft map and register a test device.
- `rc-setup.js` never loads without a successful exchange.

## Webhooks

- Meta's webhooks are accepted only with a valid signature, computed over the raw body with the store's app secret.
- The verification handshake answers only to the store's own verify token.
- Each event is processed once.

## Links and tokens

- The click redirect sends people only to the store's own verified domain. It cannot be used to redirect elsewhere.
- Unsubscribe and preferences tokens are random and stored hashed. A `GET` never changes anything; only a `POST` does.

## Outbound connections

- The crawler loads only the verified domain, never submits forms, and in production refuses addresses that resolve to private or loopback ranges.
- SMTP connections are allowed only on standard mail ports, and in production not to private or loopback addresses.

## Secrets

- Platform secrets come from the environment and are never committed.
- Per-store secrets are stored encrypted and are write-only in the console: the Meta access token and app secret, the SMTP password, and the push signing key.

## The console

- Passwords are hashed with Argon2id. Sign-in is rate-limited.
- The session cookie is httpOnly, Secure and SameSite. Requests that change something need a CSRF token.
- The console loads no third-party scripts and sends a Content-Security-Policy header that says so.

## Retention and erasure

- Raw events and page snapshots are deleted after `retention_days`. Daily analytics totals, which identify no one, are kept.
- **Erasure.** "Delete this shopper's data" in the console, and "Delete my data" on the preferences page, remove the shopper's profile, events, sessions, messages, conversations, consents and contact points. What remains is a keyed hash of each address on a do-not-contact list, so that the person is not contacted again unless they opt in afresh.
- Removing a store deletes all of its data.

## Audit log

The audit log records, with the merchant and the time: number reveals, event map publishes and rollbacks, settings changes, channel connections and disconnections, template submissions, manual replies, and data deletions.
