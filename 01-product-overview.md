# 01. Product overview

## The problem

tores lose a large share of high-intent shoppers at checkout and cannot say why. Standard analytics records page-level events, not the specific field, step or price change that made someone leave. Follow-up is generic and late. Analytics, messaging and discounting sit in separate tools, so nothing learns from what worked.

## What Retarget Customer does

1. **Tracks precisely.** Our own tracker script on the store records what each shopper does, element by element: which product they opened, which section they lingered on, which checkout step they reached, which error they hit.
2. **Understands.** A set of single-task agents scores how close each shopper was to buying, builds a profile of them, and names the reason they left.
3. **Reaches out.** The shopper gets one personal message on a channel they agreed to: WhatsApp, email, a browser notification, a message on the store when they come back, or a phone call they asked for. The message speaks to their reason for leaving and carries an offer only when the reason calls for one.
4. **Learns.** Clicks, replies, purchases and opt-outs feed back, so that tagging, scoring, offers and channel choice improve and discounts are not given away needlessly.

In one line: we find out why shoppers leave and win them back, automatically.

## The people

| Person | What they do |
|---|---|
| Merchant | The store's admin. Uses the web console to connect the store, review tags, connect channels, set limits, and approve or automate messages |
| Shopper | Visits the store. Never installs anything. Is tracked only after accepting, and contacted only on channels they opted in to |
| Store staff | Phone shoppers who asked for a call, and answer conversations the agents hand over. They use the same console |

## The parts

| Part | What it is |
|---|---|
| Tracker | Our script on the store. Records behaviour, asks for consent, shows the capture prompt and on-site messages |
| Console | The merchant's web app |
| Backend | Receives events, runs the agents, stores everything, talks to the channels |
| Agents | Eighteen single-task workers that pass work to each other over the A2A protocol |
| Channels | WhatsApp through Meta's Cloud API, email, web push, on-site message, call-back task |

## Terms

| Term | Meaning |
|---|---|
| Site | One store, identified by a site ID |
| Tracker | Our script running on the store |
| Console | The merchant's web app |
| Setup mode | A special mode of the tracker, switched on only for a signed-in merchant, used to walk through the store during a scan and to review tags |
| Page type | A kind of page: product, cart, a checkout step |
| Trigger | An element or section on a page type that we watch: a button, a field, the price block |
| Action class | The meaning given to a trigger, such as `add_to_cart` or `section_view` |
| Event map | The published list of triggers per page type, with their action classes and the data to capture. Versioned |
| Visitor | One browser, known by a random visitor ID |
| Shopper | One person. A shopper can have several visitors, linked by a contact detail they gave |
| Contact point | A way to reach a shopper on one channel: a WhatsApp number, an email address, a browser push subscription |
| Channel | A way of delivering a message: `whatsapp`, `email`, `web_push`, `on_site`, `call` |
| Recovery | One attempt to win back one shopper after one abandonment |
| Play | The approach chosen for a recovery, such as `discount` or `reminder` |
| Tier | The incentive level: `none`, `free_shipping`, `pct_5`, `pct_10` |
| Shape | What an offer needs the message to say: `none`, `free_shipping` or `percent` |
| Message template | Stored wording for one channel, purpose, shape, language and tone, with placeholders for the values |
| Customer service window | On WhatsApp, the 24 hours after a shopper's last message, during which the store may reply in its own words |
| Trail | The logged sequence of agent tasks behind a scan or a message |

"Template" on its own always means a message template in these files. Kinds of page are called page types.

## End to end

1. The merchant signs up in the console, adds the store's domain, and gets a site ID and a script tag.
2. The merchant pastes the tag into the store. The console verifies it.
3. The merchant starts a scan. The crawler samples the store's public pages. The merchant then opens the store in setup mode and walks through cart and checkout once. The scan agents produce a draft event map.
4. The merchant reviews the tags on the live store and publishes the map.
5. The merchant connects the channels they want, uploads coupon codes and sets limits.
6. The tracker loads the published map and sends events for every shopper who has accepted tracking.
7. At a high-intent moment the tracker's capture prompt asks the shopper how they would like to hear from the store. Each channel has its own consent.
8. When a session ends without a purchase, the agents score it, update the shopper's profile and diagnose the exit. If the shopper can be reached, the agents choose a play, an offer, a channel and the words.
9. The message waits for the merchant's approval, or goes straight out if autopilot is on.
10. The channel delivers it. Replies on WhatsApp go to the Reply agent, or to a person when the agent cannot help.
11. Clicks, purchases and opt-outs are recorded. Learning measures what worked and writes suggestions for the merchant.

## One shopper, start to finish

Priya opens the Aero Run shoes on Monday and leaves. On Tuesday she opens them again, picks size 8, spends 45 seconds on the price, and leaves without adding to cart. She had opted in to WhatsApp on an earlier visit.

Every value the system uses falls into one of four groups:

```json
{
  "captured_by_tracker": {
    "product_id": "AR-001",
    "product_name": "Aero Run shoes",
    "price": 3000,
    "product_visits": 2,
    "variant_selected": "8",
    "price_section_seconds": 45,
    "added_to_cart": false
  },
  "known_about_shopper": {
    "first_name": "Priya",
    "whatsapp_opt_in": true,
    "language": "en-IN"
  },
  "decided_by_rules": {
    "intent_score": 50,
    "stage": "considering",
    "price_sensitivity": "high",
    "exit_reason": "price_hesitation",
    "play": "discount",
    "tier": "pct_10",
    "offer_price": 2700,
    "coupon_code": "SAVE10-7Q2K",
    "expires_at": "2026-11-30",
    "channel": "whatsapp"
  },
  "set_by_merchant": {
    "intent_threshold": 40,
    "abandon_after_minutes": 30,
    "max_discount_pct": 10,
    "discount_cooldown_days": 30,
    "quiet_hours": "21:00-09:00",
    "channel_priority": ["whatsapp", "email", "web_push", "on_site"]
  }
}
```

Thirty minutes after her last action, the store's WhatsApp Business number sends her this, using a template Meta has approved:

```
Hi Priya, it's Demo Store. Still thinking about the Aero Run shoes? You can have 10% off, so ₹2,700 instead of ₹3,000. Use code SAVE10-7Q2K by 30 Nov.
Reply STOP to opt out.

[ View my offer ]
```

The button opens a tracked link back to the product. If she replies "can you do better?", the Reply agent answers in the store's own words, because her message opened the 24-hour window.

The same decision reaches other shoppers differently:

- A shopper who opted in to **email** only gets an email whose subject and body were written for this message and rewritten by the Humanizer, with one button and an unsubscribe link.
- A shopper who only allowed **browser notifications** gets a short notification with the offer and the code, without their name.
- A shopper who gave **no contact detail** sees a welcome-back card on the store the next time they visit, if the merchant has switched on-site messages on.
- A shopper who pressed **Request a call back** appears on the staff's call list with the reason they left and the offer to quote.

A different reason gives a different message. A shopper who left when the shipping cost appeared is offered free delivery, not a percentage. A shopper whose payment failed gets a link back to payment and no offer at all.
