# 14. Delivery channels

v1 delivers through five channels. One message goes out on one channel, chosen by the Channel agent (`11-decision-rules.md`).

| Channel | Reaches | Needs from the shopper | Needs from the merchant | Mode | What comes back |
|---|---|---|---|---|---|
| `whatsapp` | A phone | A number and a WhatsApp opt-in | Meta credentials and approved templates | `approved_text`; `free_text` inside the window | Sent, delivered, read, failed; replies |
| `email` | An inbox | An address and an email opt-in | SMTP details and a sender domain set up for email | `free_text` | Accepted or refused; clicks |
| `web_push` | A browser | Notification permission | One file on the store | `free_text` | Accepted, displayed; clicks |
| `on_site` | The store itself, on the next visit | Consent to tracking | A switch | `free_text` | Shown, clicked, dismissed |
| `call` | A phone, through a person | A call request | Staff and call hours | `task` | The call's outcome |

## One interface

Every channel implements the same interface, so that the agents never know which provider is behind it.

```ts
type ChannelName = 'whatsapp' | 'email' | 'web_push' | 'on_site' | 'call';
type Mode = 'approved_text' | 'free_text' | 'task';
type ChannelStatus = 'off' | 'not_connected' | 'connected' | 'error';
type Failure = 'temporary' | 'undeliverable' | 'limit' | 'auth' | 'template' | 'window_closed';

interface DeliveryChannel {
  name: ChannelName;
  status(siteId: string): Promise<ChannelStatus>;
  modeFor(siteId: string, contactPointId: string): Promise<Mode>;
  send(message: OutboundMessage): Promise<SendResult>;
  test(siteId: string, target: string): Promise<SendResult>;
}

interface SendResult {
  ok: boolean;
  providerMessageId?: string;
  failure?: Failure;
  detail?: string; // the provider's error text, with no personal data
}
```

What comes back later reaches the backend by different routes (Meta's webhooks, the service worker, the tracker, the console). Each adapter turns it into the same events:

```ts
type ChannelEvent =
  | { kind: 'receipt'; providerMessageId: string; status: 'sent' | 'delivered' | 'read' | 'failed'; failure?: Failure; at: string }
  | { kind: 'inbound'; siteId: string; from: string; text: string; providerMessageId: string; at: string }
  | { kind: 'template_status'; siteId: string; providerTemplateId: string; status: 'approved' | 'pending' | 'rejected' | 'paused'; reason?: string };
```

`11-decision-rules.md` says what the coordinator does with each failure.

---

## WhatsApp through Meta's Cloud API

### Before you write code

Meta's API version, template rules, error codes and prices change. Read Meta's current documentation for the WhatsApp Business Platform (the Cloud API, message templates and webhooks) before writing against it. This section describes the behaviour the product needs. Where it names a field or an error code, that was correct when this was written; confirm it.

### What the merchant brings

A Meta app with WhatsApp added, a WhatsApp Business Account, and a phone number registered for the Cloud API. From these, four values:

| Value | Used for |
|---|---|
| WhatsApp Business Account ID | Managing message templates |
| Phone number ID | Sending messages |
| Access token | Calling the API. It should be a token that does not expire (a system user token) with permission to send messages and to manage templates |
| App secret | Checking that webhooks really come from Meta |

### Connecting

1. The merchant enters the four values on the Channels page.
2. The backend reads the phone number's details from the Graph API with the token. If that works, it stores the display number and verified name and sets the status to `connected`. If Meta refuses, the card shows Meta's error and nothing is stored. On connecting, the backend also reads the account's existing templates and matches them to the library by name, so that templates approved earlier are usable at once.
3. The credentials are stored encrypted. The API never returns them; the console shows only their last four characters.
4. The card shows a callback address, `https://OUR_DOMAIN/webhooks/meta/{webhook_key}`, and a verify token. The merchant pastes both into the Meta app's webhook settings and subscribes the app to message events and template status updates.
5. Meta calls the address to verify it. The card then shows the webhook as verified.

Sending works as soon as step 2 succeeds. Receipts, replies and template status changes arrive only after step 5, and the card warns until then.

### Webhooks

- **Verification.** Meta sends a `GET` with `hub.mode=subscribe`, `hub.verify_token` and `hub.challenge`. If the token matches the site's, answer 200 with the challenge as plain text. Otherwise answer 403.
- **Signature.** Every `POST` carries `X-Hub-Signature-256`: an HMAC-SHA256 of the raw request body, keyed with the app secret. Compute it over the body exactly as received, before any parsing, and reject a mismatch with 401. A rejected webhook changes nothing.
- **Routing.** The `webhook_key` in the path names the site. Also check that the phone number ID inside the payload is that site's.
- **Speed.** Answer 200 at once and do the work in a job.
- **Repeats.** Meta retries. Record each event's key in `webhook_events` and ignore one already seen.

A webhook carries one of three things: a receipt for a message we sent, a message from a person, or a change to a template's status.

### Sending an approved template

```json
{
  "messaging_product": "whatsapp",
  "to": "91XXXXXXXXXX",
  "type": "template",
  "template": {
    "name": "rc_discount_percent_casual_en_v1",
    "language": { "code": "en" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Priya" },
          { "type": "text", "text": "Demo Store" },
          { "type": "text", "text": "Aero Run shoes" },
          { "type": "text", "text": "10%" },
          { "type": "text", "text": "₹2,700" },
          { "type": "text", "text": "₹3,000" },
          { "type": "text", "text": "SAVE10-7Q2K" },
          { "type": "text", "text": "30 Nov" }
        ]
      },
      {
        "type": "button",
        "sub_type": "url",
        "index": "0",
        "parameters": [{ "type": "text", "text": "k3f9" }]
      }
    ]
  }
}
```

This is posted to the phone number's `messages` endpoint with the token as a bearer token. The body parameters are in the order the template's variables were numbered at submission. The button parameter is the click ID, which completes the address `https://OUR_DOMAIN/c/{{1}}`. The response gives Meta's message ID, which is stored as `provider_message_id`.

### Sending free text

Allowed only while the shopper's customer service window is open.

```json
{
  "messaging_product": "whatsapp",
  "to": "91XXXXXXXXXX",
  "type": "text",
  "text": { "body": "We can do a bit better on the Aero Run shoes. ..." }
}
```

### The customer service window

- The window opens when a shopper sends the store a message, and lasts 24 hours from their latest message. Store its end on the contact point.
- Inside the window the store may send free text. Outside it, only an approved template.
- Check the window before sending free text. Never attempt a free-text send outside it; report `window_closed` instead.

### Receipts

Meta reports `sent`, `delivered`, `read` and `failed` for each message, matched by Meta's message ID. `read` arrives only when the shopper has read receipts on. A `failed` receipt carries an error, mapped as below.

### Errors

| Meta reports | Code when written | Our reason |
|---|---|---|
| The message cannot be delivered to this number | 131026 | `undeliverable` |
| More than 24 hours have passed since the shopper's last message | 131047 | `window_closed` |
| Meta chose not to deliver a marketing message to this person now | 131049 | `limit` |
| A rate limit was hit | 130429, 131048 | `temporary` |
| The template does not exist, is not approved, or its parameters do not match | 132001, 132000 | `template` |
| The access token is invalid or has expired | 190 | `auth` |
| Anything else | | `temporary` once, then the message is `failed` |

### Message templates at Meta

A template from our library (`13-message-pipeline.md`) is submitted like this:

1. `{opt_out_line}` becomes the literal sentence in the template's language. It is fixed text, not a variable.
2. Every other placeholder becomes a numbered variable, `{{1}}`, `{{2}}` and so on, in order of first appearance. The mapping is stored with the template.
3. Every variable gets an example value from a fixed sample set, never from a real shopper: "Asha", the store's name, "Classic Runner", "₹3,000", "₹2,700", "10%", "SAVE10-EXAMPLE", "30 Nov".
4. The template gets one button that opens a web address. Its label is the template's `button_label`. Its address is `https://OUR_DOMAIN/c/{{1}}`.
5. The category is marketing. Meta decides the category in the end and may change it.
6. The name is `rc_{purpose}_{shape}_{tone}_{language}_v{version}`, in lower case.

```json
{
  "name": "rc_discount_percent_casual_en_v1",
  "language": "en",
  "category": "MARKETING",
  "components": [
    {
      "type": "BODY",
      "text": "Hi {{1}}, it's {{2}}. Still thinking about the {{3}}? You can have {{4}} off, so {{5}} instead of {{6}}. Use code {{7}} by {{8}}.\nReply STOP to opt out.",
      "example": { "body_text": [["Asha", "Demo Store", "Classic Runner", "10%", "₹2,700", "₹3,000", "SAVE10-EXAMPLE", "30 Nov"]] }
    },
    {
      "type": "BUTTONS",
      "buttons": [
        { "type": "URL", "text": "View my offer", "url": "https://OUR_DOMAIN/c/{{1}}", "example": ["https://OUR_DOMAIN/c/example"] }
      ]
    }
  ]
}
```

Meta's statuses map to ours: approved to `approved`, pending or in review to `pending`, rejected to `rejected`, paused or disabled to `paused`. Changes arrive by webhook. **Refresh statuses** on the Message templates page reads them directly, for when a webhook was missed.

Approval can take from minutes to days, and Meta can reject a template. Until at least the three `generic` templates are approved, WhatsApp cannot send, and the console says so.

### Opting in and out

- Meta requires that a person has opted in before a business messages them. Our opt-in record is that proof (`17-privacy-and-security.md`).
- A shopper opts out by replying with an opt-out word. The Reply agent handles it in code (`15-replies-and-learning.md`).

### People who write first

The store's number can be messaged by anyone, not only by shoppers we know. A message from a number that is not a known shopper is stored in Conversations as an **unknown contact** and flagged for a person. No agent runs on it and nothing is sent automatically. The merchant can answer by hand while the window is open. These messages must not be dropped: with the Cloud API there is no phone app in which the merchant would otherwise see them.

### Cost and limits

- Meta charges for template messages, by category and country. When this was written, replies inside the window were not charged. The merchant enters their rate as the channel's cost per message, and the recovery report uses it.
- Meta limits how many people a number may start conversations with each day, and raises the limit as the number earns trust. The channel's `daily_cap` should be at or below Meta's limit.

### `fake-meta`

The tests and local runs use `apps/fake-meta` in place of Meta. It must:

- Answer the phone number read used when connecting, and reject an unknown token
- Accept a message send, record it, and return a message ID
- Accept a template submission, record it, and list templates with their statuses
- On command from a test: post a signed receipt, a signed inbound message or a signed template status change; fail the next send with a chosen error; start rejecting the token; run the webhook verification handshake with a right or wrong token; post a webhook with a bad signature

---

## Email

### Connecting

The merchant enters SMTP details for their own mail provider: host, port, username, password and encryption. They also enter the From name and address, an optional Reply-to, and a postal address for the footer. On save, the backend opens a connection and signs in, to check the details. The password is stored encrypted.

The Channels page also looks up the From domain's SPF and DMARC records and shows whether they exist. Large mailbox providers expect a sending domain to be set up this way; without it, mail goes to spam. The check informs. It does not block.

### What is sent

- A subject, and a body in both plain text and simple one-column HTML carrying the same words.
- One button, linking to the tracked link.
- A footer, added by code: why the shopper is getting this ("You're getting this because you asked {store_name} for offers by email"), the store's name and postal address, an unsubscribe link and a preferences link.
- The headers `List-Unsubscribe`, with the unsubscribe address, and `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, so that mail apps can show their own unsubscribe button.
- No images and no open-tracking pixel. Opens are not measured; clicks are, through the tracked link.

### What comes back

- If the SMTP server accepts the message, it is `sent`, which counts as delivered for email.
- If the server refuses the address permanently, the reason is `undeliverable`. A temporary refusal or a connection error is `temporary`. A failed sign-in is `auth`.
- Bounces that arrive later, and spam complaints, are not processed in v1. Replies go to the store's Reply-to inbox and are not read by the agents.

Local runs send to the mail catcher in Docker Compose. Tests use `MockEmailProvider`.

---

## Web push

### Connecting

1. When the channel is first enabled, the backend creates a signing key pair for the site (VAPID keys). The private key is stored encrypted. The public key goes to the tracker in the config.
2. The merchant downloads `rc-sw.js` from the Channels page and places it at the store's root (`05-tracker.md`, "Web push").
3. **Check** fetches the file from the store and confirms its content. The channel can be switched on only when the check passes.

### What is sent

A push carries a small JSON payload: the title, the body, the tracked link, a message ID and a receipt token. It is sent with the `web-push` package, with a time-to-live of 24 hours, so that a notification which could not be delivered within a day is dropped, not shown late.

The service worker shows the notification, with a "Turn off" action where the browser supports actions.

### What comes back

- The push service accepts the message: `sent`.
- The service worker reports that it displayed the notification, using the receipt token: `delivered`.
- The push service answers that the subscription no longer exists: `undeliverable`. Delete the subscription.
- A click opens the tracked link, which records it.

### Limits to know

- On iPhones and iPads, web push works only for sites the person has added to their home screen. Most iPhone visitors cannot subscribe.
- Browsers show their permission prompt only in response to a click, and some hide it for sites that ask too eagerly. The capture prompt asks first, in our own widget.
- Subscriptions expire and are replaced. The service worker re-subscribes when the browser tells it to.

Tests use `MockPushService`.

---

## On-site message

An on-site message is not sent anywhere. It waits in the backend for the shopper's next visit, and the tracker shows it as a card (`05-tracker.md`, "On-site messages").

- **Who gets one.** A shopper with consent to tracking who has not chosen "Don't show again". No phone number or email address is needed, so this channel reaches shoppers that no other can.
- **Lifecycle.** The message is `queued` when created. When the tracker asks for it at the start of a later session, Guard checks it again; if it still passes, the tracker shows it and it becomes `delivered`, and its code is confirmed. If the shopper does not return within `attribution_window_days`, it becomes `expired` and its code is released.
- **Content.** A title, a short body and one button, written by Copy and the Humanizer. No name.
- **What comes back.** Shown, clicked, dismissed, and "Don't show again", which is an opt-out for this channel.
- Quiet hours and daily caps do not apply.

---

## Call-back

A call-back is a phone call made by a person on the store's staff, to a shopper who asked for one. Nothing is sent to the shopper automatically.

### The request

The shopper presses "Request a call" in the capture prompt, gives a number, chooses when, and ticks "Call me about my order". That request is the consent for this call. It is valid for `call_request_valid_hours`.

### The task

The request opens a recovery at once (`09-backend-api-and-data.md`). The chain runs in `task` mode, and Delivery creates a call task instead of sending a message. The task is due:

| The shopper chose | Due |
|---|---|
| As soon as possible | Now, or at the start of the next call hours |
| Later today | Three hours from now, within call hours; otherwise the start of the next call hours |
| Tomorrow | The start of call hours tomorrow |

The task carries a brief, built by code from the decision data: why the shopper left, the product and cart value, the shipping cost if one was shown, the offer to quote with its code and expiry, and the profile summary.

### Making the call

On the Calls page a staff member presses **Reveal number**, which is written to the audit log, calls, and records the outcome.

| Outcome | Effect |
|---|---|
| `reached` | The task closes. The code is confirmed as issued. For attribution, the call counts as delivered |
| `no_answer` | The attempt is counted. The task is due again in two hours, within call hours. After `max_call_attempts` it closes as `expired` and the code is released |
| `wrong_number` | The task closes. The contact point is marked `unreachable`. The code is released |
| `declined` | The task closes. The shopper's call request is ended. The code is released |

If the shopper purchases before the call, the task closes on its own.

---

## Adding a channel later

A new channel needs:

1. An implementation of `DeliveryChannel`
2. A format in `13-message-pipeline.md`: its fields, limits and allowed placeholders
3. A consent text and an option in the capture prompt
4. A way to opt out
5. A place in `channel_priority`

The agents do not change. Likely additions are SMS, RCS, automated voice calls, and Instagram and Messenger messages. Some of these, like WhatsApp, allow only text approved in advance, and would use `approved_text` mode.
