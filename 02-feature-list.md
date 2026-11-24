# 02. Feature list

Every feature of v1 in one line, with the file that specifies it. Use this as a checklist: when the build is done, each line should be true.

## Connect the store

| Feature | What it does | Specified in |
|---|---|---|
| Merchant accounts | Sign up, sign in, sign out and password reset; one account can hold several stores | `07` |
| Site connection | Add a domain and get a site ID and one script tag to paste into the store | `08` |
| Two-check verification | A store is connected only when the tag is found on its live page and the tracker's ping arrives from that domain | `08` |
| Recent events view | Shows the latest events received, so the merchant can see the tracker working | `07` |

## Track shoppers

| Feature | What it does | Specified in |
|---|---|---|
| First-party tracker | Our own script records shopper actions; no third-party analytics, pixels or session replay | `05` |
| Tracking consent | Nothing is stored or sent until the shopper accepts, through the store's banner or our built-in notice | `05` |
| Anonymous identity | A random visitor ID and session ID in the shopper's own browser; no fingerprinting | `05` |
| Element-level capture | Records clicks, submits and changes on mapped triggers, reading only the fields the map lists | `05`, `06` |
| Section-view timing | Measures how long price, reviews, size guide, delivery and returns stay on screen | `05` |
| Never-captured list | Refuses passwords, card data, payment iframes and query strings, even if a map entry asks for them | `05` |
| Single-page store support | Works out the page type again when the store navigates without a reload | `05` |
| Tag health | Counts mapped triggers that are missing from live pages and reports them | `05`, `08` |

## Scan and tag the site

| Feature | What it does | Specified in |
|---|---|---|
| Automatic crawl | A headless browser samples a few pages of each URL pattern from the sitemap and home page | `08` |
| Setup mode | The merchant opens their own store with a one-time token to walk through cart and checkout, and to review tags | `08` |
| Page-type grouping | Groups sampled pages into home, category, product, search, cart, checkout steps and order confirmation | `08` |
| Trigger tagging | Lists every button, link, form, field and content section with a stable selector | `08` |
| Action classification | Gives each trigger an action class, the data to capture and a confidence score | `08` |
| Visual tag review | Outlines tags on the live store so the merchant can confirm, change or ignore each one | `08` |
| Versioned event map | Publish, keep history and roll back | `06`, `08` |
| Rescan suggestions | Suggests a rescan when tags stop matching live pages | `08`, `15` |

## Capture contact and consent

| Feature | What it does | Specified in |
|---|---|---|
| Capture prompt | At a high-intent moment, asks how the shopper wants to hear from the store: WhatsApp, email or browser notifications, each with its own unticked consent | `05` |
| Prompt modes | Plain, incentive (a promised code shown on screen at once and sent as a welcome message) or off | `05`, `11` |
| Store checkout opt-ins | The store's own phone and email fields and consent boxes can be mapped as opt-ins | `05`, `06` |
| Call-back request | The shopper asks for a phone call about their order | `05`, `14` |
| Consent records | Stores what was agreed, when and on which page, per channel | `09`, `17` |
| Opt-out on every channel | A reply word on WhatsApp, an unsubscribe link in email, a turn-off action on notifications, a don't-show-again control on the store | `14` |
| Preferences page | A shopper can switch channels off, stop everything or ask for deletion | `09` |
| Identity linking | A contact detail links a shopper's browsers into one record; duplicates merge | `09` |

## Understand behaviour

| Feature | What it does | Specified in |
|---|---|---|
| Sessions and abandonment | A session ends after 30 idle minutes; one that ends without a purchase is scored | `09` |
| Intent score | 0 to 100, from weighted signals such as repeat views, add to cart and checkout steps | `11` |
| Shopper profile (ICP) | Stage, price sensitivity, interests, concerns, top product, language, device, active hours, reachable channels, tone and a summary | `12` |
| Store profile (ICP) | The aggregate picture of the store's buyers, rebuilt daily | `12` |
| Exit diagnosis | Names one of eight exit reasons, with the events that prove it | `11` |
| Analytics | Funnel, drop-off points by page type and trigger, exit reasons, revenue at stake and reach by channel | `09` |

## Decide the recovery

| Feature | What it does | Specified in |
|---|---|---|
| Single-task agents over A2A | Eighteen agents, one job each, exchanging work as A2A messages with Agent Cards | `10` |
| Coordinator and chain config | A thin coordinator runs the chain for each recovery; chains live in config so agents and plays can be added | `10` |
| Trail | Every agent task is logged, so the merchant can see why a shopper got a message | `10` |
| Strategy | Maps the exit reason to one of eight plays | `11` |
| Recommendation | Pitches the same product, or the closest in-stock alternative | `11` |
| Offer tiers | None, free shipping, then percentage tiers; the smallest incentive that fits the shopper | `11` |
| Discount guardrails | Cap, cooldown, excluded products, welcome-code reuse and step-down when a pool is empty | `11` |
| Coupon pools | Merchant-uploaded codes per tier with real expiry dates; reserved, confirmed and released | `09` |
| Channel choice | Picks one channel per message by consent, connection, past response and the merchant's order | `11` |
| Guard | Checks consent, opt-out, no-message group, purchase, frequency, incentive, text, quiet hours and channel state | `11` |
| Second message | After 24 hours without a response: one tier up, on a different channel where possible | `11` |
| Approval queue and autopilot | The merchant approves, edits or rejects each message, or lets approved messages go out automatically | `07`, `11` |

## Write the message

| Feature | What it does | Specified in |
|---|---|---|
| Five-part message | Hook, reason line, offer, deadline with link, opt-out | `13` |
| Placeholders | Models write placeholders, never real names, prices, codes or links; code fills them in | `13` |
| Copy agent | Drafts the message in the format of the chosen channel | `13` |
| Isolated Humanizer | Rewrites a draft in a natural voice, with no tools, data or memory | `13` |
| Text rules in code | Placeholders intact, no invented numbers or links, length limits, no pressure phrases | `13` |
| Message template library | Stored wording per channel, purpose, offer shape, language and tone, with a seed set | `13` |
| WhatsApp template approval | Variants are generated, checked, submitted to Meta and tracked through approval | `13`, `14` |
| Template selection | Picks the best approved template, with fallbacks | `11` |

## Deliver

| Feature | What it does | Specified in |
|---|---|---|
| WhatsApp through Meta | Sends from the store's own WhatsApp Business number through the Cloud API, with approved templates and free-form replies inside the 24-hour window | `14` |
| Meta webhooks | Receives delivery and read receipts, shopper messages and template status changes, each signature-checked | `14` |
| Email | Text-first email with one button, an unsubscribe link and one-click unsubscribe headers, sent through the store's SMTP server | `14` |
| Web push | Browser notifications, which also reach visitors who gave no phone number or email | `14` |
| On-site message | A welcome-back card shown on the shopper's next visit | `14` |
| Call-back tasks | A call list for store staff with a brief, a logged number reveal and outcomes | `14` |
| Failure handling and fallback | Retries, marks dead addresses, and falls back to the next channel | `11`, `14` |
| Caps and hours | Daily caps per channel, quiet hours and call hours | `11`, `16` |
| Tracked resume link | A link that records the click and returns the shopper to their cart or product | `09` |

## Converse

| Feature | What it does | Specified in |
|---|---|---|
| Reply agent | Handles opt-outs in code, labels replies, answers from store facts, makes up to two counter-offers, and says plainly that it is automated | `15` |
| Conversations inbox | WhatsApp threads with needs-a-person flags, manual replies inside the window, and messages from unknown contacts | `07`, `15` |

## Learn

| Feature | What it does | Specified in |
|---|---|---|
| Attribution | Counts a purchase as recovered when the code was used or the link clicked within the window | `15` |
| No-message group | Holds out a share of shoppers to measure true lift | `15` |
| Results breakdown | By channel, reason, play and tier, with discount cost and message cost | `15` |
| Suggestions | Proposed changes that the merchant applies with one click | `15` |

## Console

| Feature | What it does | Specified in |
|---|---|---|
| Web console | Pages for Sites, Scan, Tags, Channels, Message templates, Coupons, Messages, Conversations, Calls, Shoppers, Analytics, Suggestions and Settings | `07` |
| Shopper view | A list and a detail page with profile, timeline and messages | `07` |

## Privacy and safety

| Feature | What it does | Specified in |
|---|---|---|
| Protected personal data | Phone numbers, email addresses and names are encrypted at rest, masked on screen, and never in logs or model calls | `17` |
| Store isolation | One merchant can never read another's data | `17` |
| Untrusted text | Text from store pages and from shoppers cannot change rules, settings or checks | `17` |
| Retention and erasure | Raw events expire; a shopper's data can be deleted, keeping only a hashed do-not-contact entry | `17` |
| Audit log | Records sensitive merchant actions, such as revealing a phone number | `17` |

## Build support

| Feature | What it does | Specified in |
|---|---|---|
| Demo store and test doubles | A local store, a fake Meta server, mock email and push services, a fake model and a controllable clock | `04`, `19` |
| Acceptance tests | Eight end-to-end tests with fixed data and expected results | `19` |
