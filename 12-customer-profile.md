# 12. Customer profile (ICP)

"ICP" covers two things here. Both are built only from our own tracker's data and from what the shopper told us. Nothing is bought, looked up or inferred from outside sources.

- **Shopper profile.** One per shopper, kept current by the Profile agent whenever a session ends.
- **Store profile.** The aggregate picture of the store's buyers, rebuilt every night by Learning. This is the store's ideal customer profile in the usual marketing sense.

## Shopper profile

```json
{
  "shopper_id": "shp_01",
  "stage": "considering",
  "price_sensitivity": "high",
  "interests": ["Running shoes"],
  "concerns": ["price"],
  "top_product": "AR-001",
  "language": "en-IN",
  "device": "mobile",
  "active_hours": "13:00-15:00",
  "sessions_30d": 2,
  "orders": 0,
  "reachable_on": ["whatsapp"],
  "responds_on": [],
  "tone": { "formality": "casual", "length": "short", "emoji": "none" },
  "summary": "Looked at the Aero Run shoes on two days and spent most of that time on the price. No order yet."
}
```

### How each field is worked out

| Field | Rule |
|---|---|
| `stage` | The furthest reached in the last 7 days: `customer` (any purchase), `checkout`, `cart`, `considering` (a product viewed twice, or any product section in view for 20 seconds or more), `browsing` |
| `price_sensitivity` | Last 7 days. `high`: the price section in view for 40 seconds or more in total, or any coupon attempt, or the same product viewed three times. `medium`: the price section in view for 20 seconds or more, or the same product viewed twice. Otherwise `low` |
| `interests` | The top three categories by product views in the last 30 days |
| `concerns` | Each section with 20 seconds or more in view: `price`, `fit` (size guide), `delivery`, `returns`, `quality` (reviews) |
| `top_product` | The product with the most views and seconds in view in the last 7 days |
| `language` | The browser language at opt-in, replaced by the language of the shopper's own replies once they reply |
| `device` | The one used most: `mobile` or `desktop` |
| `active_hours` | The two-hour window with the most sessions |
| `sessions_30d`, `orders` | Counts |
| `reachable_on` | The channels on which the shopper has an active opt-in and is not marked unreachable |
| `responds_on` | The channels on which the shopper replied or clicked in the last 30 days, most recent first |
| `tone` | `formality` from the store's `brand_voice` setting; `length` is `short`; `emoji` is `none` unless the merchant allows emoji |
| `summary` | One or two sentences written by a model from the fields above, for the merchant to read |

### The summary

- It is written only when a recovery passes the eligibility check, in the chain step "Profile (summary)" (`10-agents-and-a2a.md`).
- The model receives the profile fields and the product's name. It receives no name, phone number, email address or ID.
- It must describe what the shopper did, in plain words, and must not guess at who they are. "Spent most of the visit on the price" is right. "A budget-conscious young professional" is not.
- If the model fails, the profile is stored without a summary.

### Where the profile is used

| Field | Used by | For |
|---|---|---|
| `price_sensitivity` | Offer | The starting tier of the `discount` play |
| `stage`, `concerns`, `price_sensitivity` | Copy | Choosing what the message speaks to |
| `language`, `tone` | Selector, Copy, Humanizer | Language and voice |
| `reachable_on`, `responds_on` | Channel | Which channel to use |
| `active_hours` | The Shoppers page and the store profile | Shown to the merchant. v1 does not time messages by it |
| `summary` | The Messages and Calls pages | Context for the person approving or calling |

## Store profile

Rebuilt nightly from the last 90 days of shoppers who purchased, with all abandoners alongside for comparison.

```json
{
  "site_id": "site_01",
  "period_days": 90,
  "buyers": 412,
  "top_categories": [
    { "category": "Running shoes", "share": 0.46 },
    { "category": "Trail shoes", "share": 0.21 }
  ],
  "typical_order_value": { "median": 2849, "p25": 2400, "p75": 3600 },
  "price_sensitivity_mix": { "high": 0.38, "medium": 0.33, "low": 0.29 },
  "common_concerns": ["price", "fit", "delivery"],
  "devices": { "mobile": 0.81, "desktop": 0.19 },
  "active_hours": ["13:00-15:00", "20:00-22:00"],
  "sessions_before_purchase": { "median": 2 },
  "reach": { "whatsapp": 0.31, "email": 0.22, "web_push": 0.12, "none": 0.47 },
  "best_recovery": { "reason": "shipping_cost", "play": "free_shipping", "conversion": 0.19 }
}
```

The Analytics page shows it in plain words, for example: "Most buyers are on mobile, look at running shoes, visit twice before ordering, and check the price and the size guide."

The store profile contains no individual shopper's data. A category or segment with fewer than 10 shoppers is left out, so that no one can be picked out of it.
