# 03. Decisions and scope

## Decisions already made

These are fixed. If one of them blocks you or looks wrong, stop and ask. Do not work around it.

1. **First-party tracking only.** We write our own tracker and our own ingest pipeline. No Google Analytics, tag managers, advertising pixels, session-replay tools or analytics SDKs, on the store or in the console. The tracker knows a browser only by a random ID stored in that browser; it does no fingerprinting. Shopper data leaves our backend in two ways only, and both are part of the product. The first is the messages themselves, sent through the channels the merchant has connected: Meta for WhatsApp, the merchant's SMTP server for email, the browser's push service for notifications. The second is language-model calls, which must never contain a phone number, an email address, a name or a visitor ID (`13-message-pipeline.md` shows how).

2. **The merchant's console is a web app. There is no browser extension.** The console connects a site, runs the scan, shows tags for review, connects channels, and displays analytics, messages and conversations. Work that has to happen on the store's own pages, such as taking page snapshots during a scan and outlining tags for review, is done by the tracker's setup mode. Setup mode switches on only for a signed-in merchant holding a one-time token (`08-site-connection-and-scan.md`).

3. **Single-task agents that talk over A2A.** Each agent does exactly one job. Agents exchange work only as Agent2Agent (A2A) protocol messages. No agent imports another agent's code or reads its memory.

4. **WhatsApp goes through Meta's official WhatsApp Business Cloud API**, from the store's own WhatsApp Business number, using the merchant's Meta credentials. Do not use unofficial WhatsApp clients or WhatsApp Web automation (Baileys and similar). There is no QR-code linking.

5. **Approved text where the channel requires it.** Meta lets a business start a WhatsApp conversation only with a message template that Meta has approved. So the first WhatsApp message of a recovery always uses an approved template, and only the variable values change per shopper. Text written for one message is used on WhatsApp only inside the 24-hour customer service window, and on email, web push and on-site messages. Never work around approval, for example with a template whose body is one large variable.

6. **Several channels behind one interface, one channel per message.** v1 delivers through WhatsApp, email, web push, on-site message and call-back task. All of them sit behind one `DeliveryChannel` interface, so that more can be added later without touching the agents.

7. **Consent first, per channel.** Consent to tracking and consent to each channel are separate, and each is recorded. A shopper is contacted on a channel only if they opted in to that channel on that store. Every message carries a way to opt out that fits its channel. An opt-out holds until the shopper opts in again.

8. **Rules decide money, consent and channel.** Whether to contact a shopper, which incentive to give, which channel to use and whether a message may go out are decided by deterministic code, using thresholds from the merchant's settings. Language models are used only to classify pages and page elements, summarise a profile, write and humanize text, and read shopper replies.

9. **The Humanizer is isolated.** It receives draft text and style hints and returns text. It has no tools, no data access and no memory (`13-message-pipeline.md`).

## Scope

### In v1

- The tracker: core script, on-page widgets, setup module and service worker (`05-tracker.md`)
- A demo store to test against
- The web console, with every page listed in `07-web-console.md`
- The backend: ingest, sessions, shoppers, coordinator, agents, coupon pools, links, webhooks, analytics
- The crawler that samples a store's public pages
- Eighteen agents (`10-agents-and-a2a.md`)
- Five delivery channels: WhatsApp through Meta's Cloud API, email through SMTP, web push, on-site message, call-back task (`14-delivery-channels.md`)
- The message template library with its seed set, plus WhatsApp template submission and approval tracking
- The eight acceptance tests (`19-acceptance-tests.md`)

### Not in v1, but leave room for them

- Store-platform plugins (Shopify, WooCommerce) and their coupon and checkout integrations
- More channels behind the same interface: SMS, RCS, automated voice calls, Instagram and Messenger messages
- Meta's Embedded Signup, which would let a merchant connect WhatsApp without pasting credentials
- Processing email bounces and complaints through a provider's webhooks, and reading email replies
- Double opt-in for phone numbers and email addresses
- New plays: upsell after a purchase, back-in-stock alert, price-drop alert, review request
- Applying Learning's suggestions automatically; v1 shows them and the merchant applies them
- Billing, team roles, several admins per store
- Reading inside third-party checkouts or payment iframes

## Questions for the owner

Ask these at the start. If there is no answer, use the default and carry on.

| Question | Default |
|---|---|
| Which language model provider and model? | Claude through the Anthropic API, with the model name read from an environment variable |
| Which languages should messages support? | English, with the language passed through every agent and stored on every message template so that others can be added |
| How do stores connect WhatsApp? | Each store enters its own Meta credentials in the console (`14-delivery-channels.md`). Embedded Signup comes later |
| How is email sent? | Through SMTP details the merchant enters for their store. Account emails such as password reset go through one platform SMTP setting |
| Which store platform comes first after the demo store? | None; stay platform-neutral |
| Where will it be hosted? | Local only for v1 |
| What address is `OUR_DOMAIN`? | Read from the environment; `http://localhost:3000` locally |
