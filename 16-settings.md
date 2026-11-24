# 16. Merchant settings

Every setting belongs to one store. The Settings page groups them as below. A change is written to the audit log and applies to decisions made after it.

## General

| Setting | Default | Meaning |
|---|---|---|
| `autopilot` | off | Send messages that pass every check without waiting for the merchant |
| `timezone` | Asia/Kolkata | The store's time zone. Quiet hours, call hours and daily caps use it |
| `default_country_code` | +91 | Filled into the phone field of the capture prompt, and used to read numbers typed without a country code |
| `brand_voice` | casual | `casual`, `neutral` or `formal`. Sets the tone of messages and which templates are preferred |
| `allow_emoji` | off | Whether written messages may contain emoji |
| `store_facts` | empty | Delivery, returns and payment text that the Reply agent may quote |
| `privacy_policy_url` | empty | The store's privacy policy. The capture prompt cannot be switched on without it |
| `retention_days` | 90 | How long raw events and page snapshots are kept |

## Scoring

| Setting | Default | Meaning |
|---|---|---|
| `intent_threshold` | 40 | The minimum score for a recovery |
| `abandon_after_minutes` | 30 | Idle time that ends a session |
| `intent_weights` | The table in `11-decision-rules.md` | Points per signal |

## Offers

| Setting | Default | Meaning |
|---|---|---|
| `max_discount_pct` | 10 | The cap on any percentage tier |
| `discount_cooldown_days` | 30 | The gap before the same shopper can get a percentage tier in a new recovery |
| `no_discount_products` | empty | Product IDs or categories that never get a percentage discount |

Coupon codes are managed on the Coupons page, not here.

## Messaging

| Setting | Default | Meaning |
|---|---|---|
| `max_messages_per_recovery` | 2 | Messages we start per recovery, across all channels |
| `second_message_after_hours` | 24 | The delay before the second message; 0 turns it off |
| `min_hours_between_messages` | 24 | The gap between two messages we start to one shopper |
| `quiet_hours` | 21:00 to 09:00 | No message is started in this window |
| `channel_priority` | `whatsapp`, `email`, `web_push`, `on_site` | The order in which channels are preferred |
| `fallback_on_failure` | on | Try the next channel when a send fails |
| `pressure_phrases` | empty | Phrases the merchant adds to the pressure list |
| `opt_out_words` | empty | Words the merchant adds to the opt-out words |

## Capture and consent

| Setting | Default | Meaning |
|---|---|---|
| `consent_mode` | `required` | Whether the tracker waits for consent to tracking |
| `consent_notice` | off | Show the tracker's built-in consent notice, for stores without their own banner |
| `capture_prompt_mode` | `off` | `plain`, `incentive` or `off` |
| `capture_prompt_tier` | none | The tier promised in `incentive` mode. It can never be above `max_discount_pct` |
| `capture_prompt_channels` | `whatsapp`, `email`, `web_push` | The channels the prompt offers, in order. Only enabled channels are shown |
| `capture_prompt_cooldown_days` | 7 | How long the prompt stays away after a shopper closes it |

## Measurement

| Setting | Default | Meaning |
|---|---|---|
| `no_message_group_pct` | 10 | The share of shoppers held out for measurement |
| `attribution_window_days` | 7 | The window for counting a purchase as recovered |

## Links

| Setting | Default | Meaning |
|---|---|---|
| `cart_link_template` | empty | An optional address pattern that rebuilds a cart, for stores that support cart links |

## Per channel

These live on the Channels page, with each channel's connection details (`14-delivery-channels.md`).

| Channel | Setting | Default | Meaning |
|---|---|---|---|
| All | `enabled` | off | A channel is off until the merchant connects it and switches it on |
| `whatsapp` | `daily_cap` | 100 | Messages started per day |
| `email` | `daily_cap` | 500 | Emails per day |
| `web_push` | `daily_cap` | 500 | Notifications per day |
| `whatsapp`, `email`, `web_push` | `cost_per_message` | 0 | What one charged message costs the merchant. Used in the recovery report |
| `on_site` | `position` | bottom right | Which corner the card appears in |
| `call` | `call_hours` | 10:00 to 19:00 | When staff may call |
| `call` | `call_request_valid_hours` | 48 | How long a call request stays valid |
| `call` | `max_call_attempts` | 2 | Attempts before a call task expires |

## Limits on values

| Setting | Allowed |
|---|---|
| `max_discount_pct` | A whole number from 0 to 50 |
| `no_message_group_pct` | A whole number from 0 to 50 |
| `intent_threshold` | 0 to 100 |
| `max_messages_per_recovery` | 1 to 3 |
| `quiet_hours`, `call_hours` | A start and an end time; the window may cross midnight |

The backend validates the whole settings document against a schema on every save and rejects a document that breaks a limit.
