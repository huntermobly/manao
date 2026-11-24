# 06. Events and the event map

## Event batches

The tracker sends events in batches to `POST /v1/events`:

```json
{
  "site_id": "site_01",
  "visitor_id": "v_81f3",
  "session_id": "s_20c4",
  "map_version": 3,
  "click_id": null,
  "context": { "device": "mobile", "language": "en-IN" },
  "events": [
    {
      "ts": "2026-11-24T14:03:10+05:30",
      "page_type": "product",
      "path": "/products/aero-run",
      "action": "section_view",
      "data": { "section": "price", "seconds_in_view": 45, "product_id": "AR-001" }
    }
  ]
}
```

`context` is sent on the first batch of a session. `click_id` is set on the first batch after the shopper arrives from one of our links. The backend stores each event as its own row, with the batch's site, visitor, session and map version copied onto it.

## Action classes

| Action class | Fires when the shopper | `data` fields |
|---|---|---|
| `product_view` | opens a product page | `product_id`, `name`, `price`, `currency`, `category`, `in_stock` |
| `section_view` | keeps a section on screen | `section` (`price`, `reviews`, `size_guide`, `delivery`, `returns`), `seconds_in_view`, `product_id` |
| `variant_select` | picks a size or colour | `product_id`, `variant`, `in_stock` |
| `add_to_cart` | adds an item | `product_id`, `variant`, `quantity`, `cart_value` |
| `remove_from_cart` | removes an item | `product_id`, `cart_value` |
| `coupon_attempt` | applies a code | `code`, `accepted` |
| `checkout_step` | reaches a step | `step` (`contact`, `shipping`, `payment`), `shipping_cost`, `order_total` |
| `form_error` | sees a field error | `step`, `field`, `message` |
| `payment_failed` | has a payment declined | `method`, `message` |
| `purchase` | lands on the order page | `order_id`, `order_value`, `coupon_code` |
| `session_exit` | leaves or hides the tab | `last_action` |

Money values are numbers in the store's currency, such as `3000`. The currency itself is a site setting.

Two more names appear in the stream but are not shopper actions:

| Name | Purpose |
|---|---|
| `snippet_ping` | Proves the tag is installed. Carries the site ID only |
| `map_miss` | Reports mapped triggers that were missing from a page: `{ "page_type": "product", "missing": ["entry_12"] }` |

One action class never produces an event:

| Action class | Purpose |
|---|---|
| `contact_optin` | Maps the store's own contact form and consent box. When the box is ticked on submit, the tracker posts an opt-in to `POST /v1/optin` (`05-tracker.md`) |

## The event map

The event map is the published result of a scan. It has two lists: page types and entries.

### Page types

```json
{ "name": "product", "url_pattern": "/products/*", "marker": "[data-product-id]" }
```

`name` is one of `home`, `category`, `product`, `search`, `cart`, `checkout_contact`, `checkout_shipping`, `checkout_payment`, `order_confirmation`, `other`. `url_pattern` matches the path, with `*` standing for one or more path segments. `marker` is a selector that must exist on the page.

### Entries

```json
{
  "id": "entry_07",
  "page_type": "product",
  "trigger": { "selector": "button[data-action='add-to-cart']", "on": "click" },
  "action": "add_to_cart",
  "capture": {
    "product_id": { "from": "jsonld", "path": "sku" },
    "variant": { "from": "selector", "selector": "select[name='size']", "read": "value" },
    "quantity": { "from": "selector", "selector": "input[name='qty']", "read": "value", "as": "number" },
    "cart_value": { "from": "selector", "selector": "[data-cart-total]", "read": "text", "as": "money" }
  },
  "confidence": 0.93,
  "needs_review": false
}
```

`on` is one of `load`, `click`, `submit`, `change`, `view`, `appear` (`05-tracker.md`, "Capture").

### Where a captured value comes from

| `from` | Reads | Extra keys |
|---|---|---|
| `jsonld` | The schema.org JSON-LD on the page | `path`, such as `sku` or `offers.price` |
| `selector` | Another element on the page | `selector`, `read` |
| `trigger` | The trigger element itself | `read` |
| `url` | A part of the path | `pattern`, such as `/products/{value}` |
| `constant` | A fixed value written in the entry | `value` |

`read` is one of `text`, `value`, `checked`, or `attr:NAME` for a named attribute.

`as` converts the value: `string` (the default), `number`, `money` (strips currency symbols and separators, giving a number) or `boolean`.

### Examples of the less obvious entries

A coupon result appears after the store has checked the code, so it is mapped with `appear`, once for each outcome:

```json
{
  "id": "entry_21",
  "page_type": "cart",
  "trigger": { "selector": ".coupon-error", "on": "appear" },
  "action": "coupon_attempt",
  "capture": {
    "code": { "from": "selector", "selector": "input[name='coupon']", "read": "value" },
    "accepted": { "from": "constant", "value": false }
  },
  "confidence": 0.88,
  "needs_review": false
}
```

A checkout step fires on load:

```json
{
  "id": "entry_30",
  "page_type": "checkout_shipping",
  "trigger": { "selector": "form#shipping", "on": "load" },
  "action": "checkout_step",
  "capture": {
    "step": { "from": "constant", "value": "shipping" },
    "shipping_cost": { "from": "selector", "selector": "[data-shipping-cost]", "read": "text", "as": "money" },
    "order_total": { "from": "selector", "selector": "[data-order-total]", "read": "text", "as": "money" }
  },
  "confidence": 0.9,
  "needs_review": false
}
```

A content section is mapped with `view`:

```json
{
  "id": "entry_11",
  "page_type": "product",
  "trigger": { "selector": "[data-section='price']", "on": "view" },
  "action": "section_view",
  "capture": {
    "section": { "from": "constant", "value": "price" },
    "product_id": { "from": "jsonld", "path": "sku" }
  },
  "confidence": 0.95,
  "needs_review": false
}
```

The store's own contact form with its consent box:

```json
{
  "id": "entry_33",
  "page_type": "checkout_contact",
  "trigger": { "selector": "form#contact", "on": "submit" },
  "action": "contact_optin",
  "capture": {
    "channel": { "from": "constant", "value": "whatsapp" },
    "opted_in": { "from": "selector", "selector": "input[name='wa_optin']", "read": "checked" },
    "address": { "from": "selector", "selector": "input[name='phone']", "read": "value" },
    "first_name": { "from": "selector", "selector": "input[name='first_name']", "read": "value" }
  },
  "confidence": 0.85,
  "needs_review": true
}
```

The tracker reads `opted_in` first and reads the other fields only when it is true. A `contact_optin` entry is always marked `needs_review`, so that the merchant confirms the box really is a marketing consent box.

## Versions

- A scan produces a **draft** map. The merchant edits it during review.
- Publishing gives the draft the next version number and makes it the map the tracker loads. The previous version is kept.
- The merchant can roll back to any earlier version. A rollback publishes that version's content under a new version number, so version numbers only ever go up.
- Every event carries the `map_version` it was captured under.

## What ingest accepts

`POST /v1/events` and `POST /v1/optin` are open endpoints, so they check everything:

- The request's `Origin` must be the site's verified domain.
- The body must match the schema. Unknown fields are dropped, not stored.
- A batch may hold at most 50 events and 64 KB.
- An event's `ts` may be at most 24 hours in the past and 5 minutes in the future.
- Requests are rate-limited by site and by IP address.
- Events from our crawler's user agent are dropped.
- The never-captured list (`05-tracker.md`) is enforced again here: a `data` field that looks like a card number, or a `contact_optin` payload without its consent flag, is rejected.
