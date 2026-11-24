# 05. Tracker

The tracker is our own script. It runs on the store, in the shopper's browser, and it is the only source of behaviour data in the product. Nothing else may be added to the store to collect data (`03-decisions-and-scope.md`, decision 1).

## Install

The merchant adds one tag to the store's `<head>`:

```html
<script async src="https://OUR_DOMAIN/rc.js" data-site="SITE_ID"></script>
```

Web push needs one more step, a small file at the store's root (see "Web push" below).

If the store sends a Content-Security-Policy header, it must allow `OUR_DOMAIN` in `script-src` and `connect-src`. Verification reports this when it is the cause of a failure (`08-site-connection-and-scan.md`).

## Files

| File | Loaded | Contains |
|---|---|---|
| `rc.js` | On every page | Config, consent, identity, page types, capture, section views, transport, prompt rules. Under 15 KB gzipped, no runtime dependencies |
| `rc-ui.js` | Only when something must be shown | The consent notice, the capture prompt, the on-site message card, the push subscription step |
| `rc-setup.js` | Only in setup mode | Page snapshots, the guided walk toolbar, the tag review overlay |
| `sw/v1.js` | Imported by the store's `rc-sw.js` | Shows push notifications and handles their clicks |

Everything the tracker draws sits inside a shadow root, so the store's styles and ours cannot affect each other. If the tracker finds itself inside an iframe, it does nothing.

## What happens on a page load, in order

1. Read `data-site`.
2. If the page was opened in setup mode, enter setup mode and stop (see "Setup mode").
3. If the browser's user agent is our crawler's, send `snippet_ping` and stop.
4. Fetch the config.
5. Send `snippet_ping`.
6. Check consent to tracking. Without it, stop here, apart from showing the built-in notice if it is switched on.
7. Set up identity, work out the page type, attach the capture listeners and section observers.
8. If the address carries `rc_click`, attach it to the first batch of events.
9. Ask the backend for a pending on-site message, if that channel is on.
10. Start evaluating the capture prompt rules.

## Config

`GET /v1/sites/{site_id}/config` returns everything the tracker needs. Cache it for the browser session.

```json
{
  "site_id": "site_01",
  "store_name": "Demo Store",
  "map_version": 3,
  "page_types": [
    { "name": "product", "url_pattern": "/products/*", "marker": "[data-product-id]" }
  ],
  "entries": [],
  "consent": { "mode": "required", "builtin_notice": false },
  "capture_prompt": {
    "mode": "plain",
    "promised_tier": null,
    "channels": ["whatsapp", "email", "web_push"],
    "rules": [
      { "when": "product_viewed_twice" },
      { "when": "price_in_view", "seconds": 20 },
      { "when": "leaving", "page_types": ["cart", "checkout_contact", "checkout_shipping", "checkout_payment"] }
    ],
    "cooldown_days": 7,
    "default_country_code": "+91",
    "privacy_policy_url": "https://demo-store.example/privacy",
    "consent_text_versions": { "whatsapp": "wa-en-1", "email": "em-en-1", "web_push": "wp-en-1", "call": "ca-en-1" }
  },
  "web_push": { "enabled": true, "vapid_public_key": "BASE64URL", "sw_path": "/rc-sw.js", "sw_scope": "/rc-push/" },
  "on_site": { "enabled": false },
  "call_back": { "enabled": false }
}
```

`entries` holds the published event map (`06-events-and-event-map.md`). The backend lists only channels that the merchant has enabled. It serves `mode: "incentive"` only while the promised tier's coupon pool still has unused, unexpired codes; otherwise it serves `plain`.

`snippet_ping` carries the site ID and nothing else: no visitor ID, no cookie. It is sent once per browser session, whether or not the shopper has accepted tracking, so that the backend can verify the install.

## Consent to tracking

The merchant chooses a mode in settings.

| Mode | Behaviour |
|---|---|
| `required` (default) | Before the shopper accepts, the tracker sets no cookie, writes nothing to storage and sends no events. Only the config request and `snippet_ping` leave the page |
| `not_required` | The tracker starts at once. The merchant chooses this knowingly, for places where consent is not needed |

Acceptance reaches the tracker in one of two ways:

- **The store's own banner** calls `window.retargetCustomer.consent.grant()`. Because the tracker loads asynchronously, the store may call before it has loaded by pushing to a queue, which the tracker drains on load:

  ```html
  <script>
    window.rcq = window.rcq || [];
    window.rcq.push(['consent', 'grant']);
  </script>
  ```

- **The built-in notice**, if the merchant has switched it on: one line of text, a link to the store's privacy policy, and two buttons, "Accept" and "No thanks". A "No thanks" is remembered for the rest of the browser session.

On acceptance the tracker creates the IDs, records the consent through `POST /v1/optin` with `channel: "tracking"`, and starts capturing. `window.retargetCustomer.consent.revoke()` stops tracking, deletes the cookie and stored IDs, and records the withdrawal through `POST /v1/optout`.

## Identity

- `visitor_id`: a random 128-bit value with the prefix `v_`, kept in a first-party cookie named `rc_vid` (12 months, `SameSite=Lax`, `Secure`) and copied to localStorage. If one copy is missing, restore it from the other.
- `session_id`: a random value with the prefix `s_`, kept in a cookie named `rc_sid` that expires after 30 minutes without events.
- The first batch of a session also carries `device` (`mobile` or `desktop`) and `language` (the browser's language).
- No fingerprinting: no canvas, font, audio or hardware probing, and no identifier other than the two above.

## Page types

Work out the current page type from the matchers in the config. A page matches a type when its path matches `url_pattern` and an element matching `marker` exists. If several match, the first in the list wins. If none match, the page type is `other`.

Stores built as single-page apps change pages without a reload. Wrap `history.pushState` and `history.replaceState` and listen for `popstate`. On each change, close the open section timers, work out the page type again, and fire that page type's `load` triggers.

## Capture

The event map gives each trigger a selector and one of six ways of firing:

| `on` | Fires when |
|---|---|
| `load` | The page type has been worked out. Once per page load |
| `click` | The shopper clicks the element or anything inside it |
| `submit` | A form is submitted |
| `change` | A field's value changes |
| `view` | A content section stays on screen (see "Section views") |
| `appear` | An element matching the selector is added to the page or becomes visible, such as an error message. Watch with a MutationObserver |

For `click`, `submit` and `change`, attach one delegated listener per event type on `document`, so that elements added later are covered. When an event arrives, find the closest ancestor that matches a trigger for the current page type.

When a trigger fires, read only the fields listed in that entry's `capture`, convert them as the entry says, and queue the event. Never read a field that is not listed. If a listed field cannot be read, send the rest and mark the event `partial: true`.

## Section views

Use IntersectionObserver on each mapped section.

- A section counts as in view while at least half of it is on screen, or, for a section taller than the screen, while it covers at least half of the screen.
- A stay counts once it has lasted 3 seconds without a break. Time while the tab is hidden does not count.
- When the stay ends (the section leaves the screen, the page is hidden, or the page type changes), emit one `section_view` event with `seconds_in_view` for that stay, in whole seconds.

The backend adds up the stays. The tracker also keeps the running total locally, because the capture prompt rules need it.

## Exit

On `pagehide`, and when the tab becomes hidden, close the open section timers and flush the queue with `navigator.sendBeacon`, adding a `session_exit` event that names the last action. This can happen more than once in a session. The backend treats `session_exit` as a marker; the backend, not the tracker, decides when a session has ended.

## Never captured

The tracker refuses these even if a map entry asks for them, and the Classifier must not map them:

- Password fields
- Card numbers, expiry dates, CVV, UPI PINs, and any field whose `autocomplete` value starts with `cc-`
- Anything inside an iframe
- Query strings: send the path only, plus our own `rc_click` parameter
- Phone numbers and email addresses, except through a `contact_optin` entry whose consent box is ticked (see "Store checkout opt-ins")
- The typed contents of free-text fields such as order notes and search boxes

## Capture prompt

The capture prompt is how an anonymous visitor becomes someone the store may contact. It is a small widget drawn by `rc-ui.js`.

### When it appears

It appears when any rule in the config matches, and only after the shopper has accepted tracking.

| Rule | Matches when |
|---|---|
| `product_viewed_twice` | The shopper opens a product they have opened before, in this or an earlier session |
| `price_in_view` | The price section of a product has been in view for the given number of seconds in total |
| `leaving` | On the listed page types: on a desktop, the pointer leaves through the top of the window; on a touch device, 30 seconds pass without any interaction |

It is shown at most once per session. After the shopper closes it, it stays away for `cooldown_days`. It is never shown on the order confirmation page. It is never shown by a rule to a shopper who already has an active opt-in on one of the channels it offers: once the store can reach someone, it does not ask again. The store's own button can still open it (see "Public API").

### What it shows

The merchant chooses which channels to offer and in what order (`capture_prompt.channels`). The first is shown open; the others are one tap away under "Other ways to hear from us".

| Channel | Fields | Consent | Button |
|---|---|---|---|
| WhatsApp | Phone number with the country code filled in from `default_country_code`; optional first name | Unticked box: "Send me offers and updates from {store_name} on WhatsApp" | "Send it to my WhatsApp" |
| Email | Email address; optional first name | Unticked box: "Email me offers and updates from {store_name}" | "Email it to me" |
| Browser notifications | None | Pressing the button is the consent; the browser then shows its own permission prompt | "Notify me in this browser" |
| Call-back, if enabled, on cart and checkout pages only | Phone number; optional first name; when to call: as soon as possible, later today, or tomorrow | Unticked box: "Call me about my order" | "Request a call" |

Every form's submit button stays disabled until its box is ticked and the field is valid. The widget always shows a link to the store's privacy policy, the line "You can opt out at any time", and a close button. The capture prompt cannot be switched on until the merchant has entered a privacy policy address.

The browser notifications option is hidden when the browser has no push support or the shopper has already blocked notifications for the store.

### Modes

| Mode | Heading | After the shopper submits |
|---|---|---|
| `plain` | "Want us to keep this for you?" | "Thanks. You're on the list." Nothing is sent until there is a reason to |
| `incentive` | Names one tier, for example "Get 10% off from {store_name}" | The widget shows the code and its expiry on screen at once, with a copy button. The same code is also sent as a welcome message on the channel the shopper chose (`11-decision-rules.md`, "Welcome offer") |
| `off` | The prompt never appears | |

The incentive must be delivered on screen and not only by message, because a shopper who goes on to buy in the same visit would otherwise never receive what was promised.

### What it sends

`POST /v1/optin`:

```json
{
  "site_id": "site_01",
  "visitor_id": "v_81f3",
  "channel": "whatsapp",
  "address": "+91XXXXXXXXXX",
  "first_name": "Priya",
  "source": "prompt",
  "consent_text_version": "wa-en-1",
  "promised_tier": null,
  "path": "/products/aero-run",
  "ts": "2026-11-24T14:03:10+05:30"
}
```

- For `web_push`, send `subscription` (the browser's push subscription object) in place of `address`.
- For `call`, add `when`: `now`, `later_today` or `tomorrow`.
- `source` is `prompt`, `checkout` or `store_button`.
- In incentive mode the response carries `coupon_code` and `expires_at` for the widget to show.

## Store checkout opt-ins

Many stores already ask for a phone number or email at checkout, with their own marketing checkbox. The event map can map such a form with the action class `contact_optin` (`06-events-and-event-map.md`). When the form is submitted, the tracker first reads the checkbox. Only if it is ticked does it read the address field and post the same opt-in payload with `source: "checkout"`. If the box is not ticked, the tracker reads nothing and sends nothing.

## Web push

Browser notifications reach a shopper who gave no phone number or email. They need a service worker on the store's own origin, which is why the merchant places one file.

1. **The merchant's file.** `rc-sw.js`, at the store's root, containing one line:

   ```js
   importScripts('https://OUR_DOMAIN/sw/v1.js');
   ```

   The console offers this file for download and checks that it is reachable.

2. **Subscribing.** When the shopper presses "Notify me in this browser", the tracker calls `Notification.requestPermission()` inside that click. If permission is granted, it registers `/rc-sw.js` with the scope `/rc-push/`, so that it does not replace a service worker the store already has, and subscribes with `userVisibleOnly: true` and the site's public key. It then posts the subscription to `POST /v1/optin`.

3. **Receiving.** On a `push` event, `sw/v1.js` shows the notification with the title, body, and a "Turn off" action where the browser supports actions, and reports to `POST /v1/push/receipts` that it was displayed.

4. **Clicking.** A click on the notification opens its link, which is our tracked link. A click on "Turn off" unsubscribes and posts to `POST /v1/optout`.

5. **Renewal.** On `pushsubscriptionchange`, subscribe again and post the new subscription.

Never ask for notification permission on page load or without a click from the shopper.

## On-site messages

If the on-site channel is on, the tracker asks `POST /v1/onsite/next` once at the start of each session, sending the site ID and visitor ID. If a message is waiting, `rc-ui.js` shows it as a small card in a bottom corner: a title, a line or two of text, one button, a close button and a "Don't show again" link.

- The card is not shown on checkout pages or the order confirmation page. It waits for the next page that is neither.
- The tracker reports `shown`, `clicked` and `dismissed` to `POST /v1/onsite/events`.
- "Don't show again" posts an opt-out for the `on_site` channel.
- At most one card per session.

## Return from a message

Links in our messages bring the shopper back with `?rc_click={click_id}` on the address. The tracker sends that value once, as the `click_id` of the first batch of that page load, so that the visit can be tied to the message. It then removes the parameter from the address bar with `history.replaceState`.

## Setup mode

Setup mode is how the merchant's console works with the live store without a browser extension. `08-site-connection-and-scan.md` describes it fully. The tracker's part is this:

1. On load, look for `rc_setup` in the address fragment (`#rc_setup=TOKEN`).
2. If it is there, remove it from the address bar at once, and exchange the token at `POST /v1/setup-sessions/exchange`.
3. If the exchange succeeds, keep the returned credential in sessionStorage, so that setup mode continues as the merchant moves between pages in that tab, and load `rc-setup.js`.
4. While in setup mode, capture nothing and send no shopper events. The merchant's own clicks must not appear in the store's analytics.
5. If the exchange fails, carry on as a normal page load.

`rc-setup.js` must never load without a successful exchange.

## Transport

- Queue events and post them to `POST /v1/events` every 5 seconds, when 20 are waiting, and on exit.
- Send the body as JSON with the content type `text/plain`, so that the browser does not need a preflight request and `sendBeacon` is not blocked.
- Retry a failed post three times with backoff, then drop the batch quietly. The tracker must never break the store's page.
- The tracker contacts no host other than `OUR_DOMAIN`.

## Tag health

If a page type is matched but a trigger that the map expects there is missing from the page, count it. One session in ten reports the counts in a `map_miss` event. The backend uses them to suggest a rescan (`08-site-connection-and-scan.md`).

## Public API

| Call | Effect |
|---|---|
| `window.retargetCustomer.consent.grant()` | Records acceptance of tracking and starts capture |
| `window.retargetCustomer.consent.revoke()` | Stops tracking and deletes the IDs |
| `window.retargetCustomer.prompt.open()` | Opens the capture prompt from the store's own button, for example a "Get updates on WhatsApp" link. The opt-in is sent with `source: "store_button"`. The same consent boxes apply |
