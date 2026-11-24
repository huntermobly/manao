# 11. Decision rules

These are the exact rules for the agents that decide. All of them are code, not a model (`03-decisions-and-scope.md`, decision 8). The acceptance tests in `19-acceptance-tests.md` depend on every number here.

## Intent: score the shopper's buying intent

Add up the points below. Each row counts at most once, however many times it happens. The score covers the shopper's events from the last 7 days that are not already counted in a closed recovery, and is capped at 100.

| Signal | Points |
|---|---|
| A product viewed for the first time | 10 |
| The same product viewed again, in this or a later session | 15 |
| Price section in view for 20 seconds or more | 15 |
| Reviews, size guide, delivery or returns in view for 20 seconds or more | 10 |
| A variant selected | 10 |
| An item added to cart | 30 |
| Checkout step `contact` reached | 10 |
| Checkout step `shipping` reached | 5 |
| Checkout step `payment` reached | 10 |
| A coupon attempted | 5 |

Seconds in view are totals for a section across the scored events, not the length of one stay.

Whether a view is a first view or a repeat is judged against the shopper's whole history, not only against the events being scored. A product opened again after an earlier recovery has closed is a repeat view.

The points are the defaults of the `intent_weights` setting, so that a suggestion from Learning can change one.

## Diagnosis: name the exit reason

Check the rules in order. The first match wins. Return the events that matched as evidence.

| Order | Reason | Rule |
|---|---|---|
| 1 | `payment_failed` | A `payment_failed` event after the `payment` step was reached |
| 2 | `coupon_rejected` | The last `coupon_attempt` was rejected |
| 3 | `shipping_cost` | The last step reached is `shipping`, a shipping cost above zero was shown, and `payment` was not reached |
| 4 | `form_friction` | Two or more `form_error` events on the last step reached |
| 5 | `out_of_stock` | A `variant_select` with `in_stock: false` on the last product viewed, and no `add_to_cart` after it |
| 6 | `price_hesitation` | No checkout step reached, and either the price section was in view for 20 seconds or more or the same product was viewed at least twice |
| 7 | `needs_reassurance` | No checkout step reached, and reviews, size guide or returns were in view for 20 seconds or more |
| 8 | `no_clear_reason` | None of the above |

## Strategy: choose the play

| Reason | Play | What the message does |
|---|---|---|
| `payment_failed` | `resume_payment` | Links back to payment and mentions other ways to pay |
| `coupon_rejected` | `working_coupon` | Gives a code that works |
| `shipping_cost` | `free_shipping` | Leads with free delivery |
| `form_friction` | `resume_checkout` | Links back to the step and offers help |
| `out_of_stock` | `alternative` | Suggests the closest in-stock option |
| `price_hesitation` | `discount` | Offers a discount sized to the shopper |
| `needs_reassurance` | `reassure` | Invites the question behind the doubt: fit, delivery, returns |
| `no_clear_reason` | `reminder` | Plain reminder |

## Recommendation: choose the product to pitch

- Return the product of the shopper's last product-related event (a view, a section view, a variant selection or an add to cart), unless the play is `alternative`.
- For `alternative`, return the in-stock product in the same category that is closest in price to that product. If there is none, the recovery closes as `no_action`.
- Only products the tracker has seen are available. A product counts as in stock unless the tracker's latest data for it says otherwise.

## Offer: choose the tier and reserve the code

Tiers run in this order: `none`, `free_shipping`, then each percentage tier that the merchant has a coupon pool for, up to `max_discount_pct`.

| Play | Starting tier |
|---|---|
| `discount` | Set by `price_sensitivity`: `high` takes the highest percentage tier within the cap, `medium` the lowest percentage tier, `low` takes `free_shipping` |
| `free_shipping` | `free_shipping` |
| `working_coupon` | The lowest percentage tier |
| All others | `none` |

Then apply these, in order:

1. **Welcome code first.** If the starting tier is above `none` and the shopper holds an unused, unexpired code from a welcome offer, offer that same code at its tier. Reserve nothing new, and skip rules 2, 3 and 5.
2. **Excluded products.** If the pitched product, or its category, is in `no_discount_products`, lower any percentage tier to `free_shipping`.
3. **Cooldown.** If the shopper received a percentage tier in an earlier recovery, or as a welcome offer, within the last `discount_cooldown_days`, lower any percentage tier to `free_shipping`. Tiers given earlier in the same recovery do not count.
4. **Cap.** Never go above `max_discount_pct`.
5. **Stock of codes.** If the tier's pool has no unused, unexpired code, step down to the next tier that has one, and record why in `notes`.
6. **Moving up.** A second message, or a counter-offer requested by Reply, moves up one tier from the last one, within the same limits. If the tier cannot move up, keep it and reuse the same code.

Outputs:

- `shape`: `none` for the tier `none`, `free_shipping` for `free_shipping`, `percent` for any percentage tier.
- `offer_price`: the price, or the cart's item total when there is a cart, after the discount, rounded to the currency's usual smallest unit (whole rupees for INR). Shipping is not included.
- `expires_at`: the coupon's real expiry and nothing else. A message never states a deadline that is not the coupon's.

### Welcome offer

When the capture prompt is in `incentive` mode, a shopper who opts in has been promised a tier, and the promise is kept at once:

1. The opt-in request calls Offer for the promised tier, never above the cap. Offer reserves a code.
2. The response to the tracker carries the code and its expiry, and the widget shows them on screen.
3. The coordinator sends the same code as a welcome message on the channel the shopper chose: `Offer > Selector > Guard (final) > Delivery`, with the purpose `welcome_offer`.

A welcome message answers the shopper's own action. It is not held back by the no-message group, quiet hours or the frequency check, and it does not count towards them.

## Channel: choose the channel for this message

Input: the shopper's contact points with their consent state and last response, the store's channels with their status, `channel_priority`, the channels already tried for this message, the channels already used in this recovery, and whether the shopper has an open call request.

1. If the shopper has an open call request and the call-back channel is on, return `call`.
2. Candidates are the channels that the store has enabled, on which the shopper has an active opt-in and is not marked unreachable, and that have not already failed for this message. For `on_site`, consent to tracking counts as the opt-in, unless the shopper chose "Don't show again".
3. Prefer candidates whose channel is connected. If none is connected, keep them all; the message will be deferred.
4. For a second message, drop the channels already used in this recovery, as long as at least one other candidate remains.
5. Order the candidates: first the channels on which the shopper replied or clicked in the last 30 days, most recent first; then the rest in the order of `channel_priority`.
6. Return the first candidate and its mode.

| Channel | Mode |
|---|---|
| `whatsapp` | `free_text` if the shopper's customer service window is open, otherwise `approved_text` |
| `email`, `web_push`, `on_site` | `free_text` |
| `call` | `task` |

## Selector: choose a stored message template

Input: the channel, the purpose (the play, or `follow_up` for a second message, or `welcome_offer`), the shape, the language, and the tone's formality.

On a channel in `approved_text` mode, only templates with the status `approved` are candidates. Otherwise templates with the status `ready` or `approved` are.

The first match in this order wins:

1. The purpose, shape, language and tone
2. The purpose, shape and language, in any tone
3. The purpose `generic`, with the shape, language and tone
4. The purpose `generic`, with the shape and language, in any tone
5. Steps 1 to 4 again with the language `en`

Return the template and the list of variables it needs. If nothing matches, return `none`; the coordinator then tries another channel (`10-agents-and-a2a.md`, "Fallbacks"). Record in `notes` which step matched.

## Guard: approve, defer, send back or veto

Guard is called twice in a recovery. The **eligibility** call, before Strategy, runs checks 1 to 5. A recovery that fails any of them stops there: no code is reserved and no model is called. The **final** call, just before Delivery, runs all ten in order.

| # | Check | Fails when | Result |
|---|---|---|---|
| 1 | Consent | The shopper has never opted in to any channel the store has enabled | Veto `no_consent` |
| 2 | Opt-out | The shopper did opt in, but every such opt-in has been withdrawn, or they chose "stop all messages" | Veto `opted_out` |
| 3 | No-message group | The shopper is in the no-message group | Veto `holdout` |
| 4 | Purchase | The shopper has bought since the abandonment | Veto `purchased` |
| 5 | Frequency | `max_messages_per_recovery` is reached, or the last message we started to this shopper was less than `min_hours_between_messages` ago | Veto `frequency_cap` |
| 6 | Channel consent | The shopper has no active opt-in on the chosen channel | Send back to Channel: `channel_not_consented` |
| 7 | Incentive | The tier is above the cap; the cooldown or a product exclusion was ignored; or the coupon is missing, expired or reserved for someone else | Send back to Offer: `over_cap`, `cooldown`, `excluded_product`, `coupon_invalid` |
| 8 | Text | A text rule in `13-message-pipeline.md` is broken; or, in `approved_text` mode, the template is no longer approved or a variable has no value | Send back to the Humanizer, Copy or Selector, with the rule's name |
| 9 | Hours | The time is inside quiet hours in the store's time zone; for a call, outside call hours | Defer to the end of quiet hours, or the start of call hours: `quiet_hours`, `call_hours` |
| 10 | Channel state | The channel is not connected, or its daily cap is reached | Defer: `channel_disconnected`, `daily_cap` |

Notes:

- A veto stops the message. If nothing has been delivered in the recovery, the recovery closes as `blocked` with the reason. Otherwise it stays open until its window ends.
- For checks 1, 2 and 6, consent to tracking counts as the opt-in for `on_site`, unless the shopper chose "Don't show again". So when the on-site channel is on, a shopper who gave no contact detail still passes check 1.
- A deferred message keeps its reserved code. A deferred message, and a message the merchant has edited, is checked again from check 1 before it is sent.
- Quiet hours do not apply to on-site messages, which are shown only when the shopper is on the store.
- The daily cap counts messages sent on that channel since midnight in the store's time zone.

Three kinds of message answer something the shopper did, and are treated differently:

| Message | Checks skipped |
|---|---|
| A reply to a shopper's own WhatsApp message, sent within 24 hours of it | 3, 5 and 9 |
| A welcome message after an incentive opt-in | 3, 5 and 9 |
| A call the shopper requested. Check 1 passes when the request is no older than `call_request_valid_hours` | 3 and 5 |

## Delivery: schedule and send

- Send when Guard has approved and either autopilot is on or the merchant has approved the message.
- Confirm the reserved code at the moment of sending.
- Fill the placeholders with real values only now (`13-message-pipeline.md`), and hand the message to the channel adapter (`14-delivery-channels.md`).
- Record the status as it changes: `queued`, `awaiting_approval`, `deferred`, `sent`, `delivered`, `read`, `clicked`, `replied`, `failed`, `cancelled`, `expired`.

What "delivered" means depends on the channel:

| Channel | Delivered when |
|---|---|
| WhatsApp | Meta reports `delivered`. `read` follows only if the shopper has read receipts on |
| Email | The SMTP server accepts the message. The console labels this "sent", because plain SMTP cannot confirm that a message arrived |
| Web push | The push service accepts it; the service worker's display receipt confirms it |
| On-site message | The card is shown. Until then the message is `queued`; if the shopper does not return within `attribution_window_days`, it becomes `expired` and its code is released |
| Call | The staff member marks the call `reached` |

### When a send fails

The channel adapter reports one of these reasons. Some arrive at once; on WhatsApp most arrive later, as a failed receipt.

| Reason | Meaning | What happens |
|---|---|---|
| `temporary` | A network error, a server error or a rate limit | Retry on the same channel with backoff, up to three times |
| `undeliverable` | This address cannot receive: not on WhatsApp, mailbox rejected, push subscription gone | Mark the contact point `unreachable`. Fall back to another channel. If there is none, the recovery closes as `blocked` with `unreachable` |
| `limit` | The provider declined to deliver to this person for now, such as Meta's limit on marketing messages per person | Fall back to another channel. If there is none, defer for 24 hours and try once more |
| `auth` | The channel's credentials were rejected | Set the channel's status to `error` and pause its queue. Fall back to another channel. If there is none, defer with `channel_disconnected` |
| `template` | The approved text is missing, or the provider refused it | Flag the template. Selector chooses the next match. If there is none, fall back to another channel |
| `window_closed` | A free-form WhatsApp message was attempted outside the 24-hour window | Do not send. If it was a reply, flag the conversation for a person |

**Falling back** means running the chain again from Channel with the failed channel excluded, when the `fallback_on_failure` setting is on. The fallback message carries the same offer and the same reserved code. A failed attempt does not count towards the frequency check. At most two fallbacks per message.

### Second message

If `second_message_after_hours` is above zero, and that long has passed since the first message was sent with no reply, purchase or opt-out, the coordinator runs the second-message chain: `Offer > Channel`, then the chain for the mode, with `message_number: 2` and the purpose `follow_up`.

- Offer moves up one tier (rule 6).
- Channel prefers a channel not yet used in this recovery (rule 4).
- Guard's frequency check still applies, so the second message is the last one the store starts in that recovery under the default settings.
