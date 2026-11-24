# 08. Site connection and scan

This file covers how a store is connected and verified, and how the scan turns the store's pages into an event map. The scan has two sources of pages: a crawler for public pages, and the merchant's own walk through cart and checkout in setup mode.

## Connecting a store

1. On the Sites page the merchant enters the store's name, domain, currency, time zone and default country code.
2. The backend creates the site, with a site ID, and shows the script tag.
3. The merchant pastes the tag into the store's `<head>`.
4. The merchant clicks **Verify**.

## Verification

A site is `connected` only when both checks pass. Otherwise it stays `unverified` and the page says which check failed and why.

| Check | How | Typical failure shown to the merchant |
|---|---|---|
| The tag is on the page | The crawler loads the store's home page in its headless browser and looks for our script element with this site's ID | "No tag found", or "A tag was found with a different site ID" |
| The tag runs and reaches us | The backend has received a `snippet_ping` for this site from the store's origin in the last 10 minutes. The crawler's own page load normally produces it | "The tag is present but no ping arrived. A Content-Security-Policy header or a script error may be blocking it" |

The two checks together prove that the merchant controls the site and that the tracker works there.

## The crawler

The crawler is a headless Chromium driven by Playwright, in its own process. It takes jobs from the queue.

- It loads only pages on the site's verified domain. In production it refuses a domain that resolves to a private or loopback address.
- It identifies itself with the user agent `RetargetCustomerBot`. The tracker sends nothing but `snippet_ping` to that user agent, and ingest drops events from it.
- It makes read-only page loads. It never submits a form, adds to a cart or signs in.
- It loads at most one page per second and at most 60 pages per scan.
- For each page it waits for the network to go quiet, for up to 15 seconds, then takes a snapshot.
- If the store answers with an error or a bot challenge on the home page, the scan is marked `crawl_blocked` and continues with the walk-through alone.

## Setup mode

Setup mode lets the merchant work with their live store without a browser extension. It is the tracker itself, switched into a special mode by a one-time token.

### Starting it

1. In the console the merchant clicks **Open my store in setup mode**. The console calls `POST /v1/sites/{id}/setup-sessions`. The backend creates a random token that can be used once and must be used within 5 minutes.
2. The console opens `https://STORE_DOMAIN/#rc_setup=TOKEN` in a new tab. The token sits in the fragment, so the browser does not send it to the store's server.
3. The tracker finds the token, removes it from the address bar, and posts it with the site ID to `POST /v1/setup-sessions/exchange`.
4. The backend checks that the token exists, is unused and unexpired, and that the request's origin is the site's domain. It marks the token used and returns a setup credential that is valid for 2 hours.
5. The tracker keeps the credential in sessionStorage and loads `rc-setup.js`. Setup mode lasts for that tab until the merchant leaves it or the credential expires.

### What the credential can do

The setup credential is sent in an `Authorization` header, never as a cookie. It works for that one site and only for these calls: posting snapshots to the current scan, reading and changing the draft event map, and subscribing the browser as a push test device. It cannot read shoppers, messages or settings.

### The toolbar

`rc-setup.js` draws a toolbar at the bottom of the page, inside a shadow root. It has two views.

**Walk-through**, used during a scan:

- A short list of what to do: add an item to the cart, open the cart, go through each checkout step, reach the order page. If the crawler was blocked, the list starts with: open the home page, a category page, a product page and a search results page.
- A note: use the store's test payment method, or stop before paying.
- A running count of pages captured. Every page the merchant visits is captured automatically after it loads.
- **Finish** ends the walk-through and returns the merchant to the console.

**Review**, used after a scan:

- Every tagged trigger on the current page is outlined and labelled with its action class. Low-confidence tags and tags that need review are marked.
- Clicking a label opens a small panel where the merchant can change the action class, fix what is captured, or ignore the trigger. Changes are saved to the draft map.
- **Subscribe this browser as a test device** registers the browser for test notifications.
- **Exit setup mode** clears the credential and reloads the page.

While setup mode is on, the tracker captures nothing and sends no shopper events.

## Snapshots

A snapshot describes one rendered page. The same code, in `packages/snapshot`, runs in the crawler and in the setup module.

```json
{
  "url": "https://demo-store.example/products/aero-run",
  "title": "Aero Run shoes",
  "source": "crawler",
  "jsonld": [
    { "@type": "Product", "sku": "AR-001", "name": "Aero Run shoes", "offers": { "price": "3000", "priceCurrency": "INR" } }
  ],
  "elements": [
    {
      "n": 41,
      "parent": 37,
      "tag": "button",
      "id": null,
      "classes": ["btn", "btn-primary"],
      "role": null,
      "label": "Add to cart",
      "attrs": { "data-action": "add-to-cart" },
      "input": null,
      "box": [640, 512, 180, 44]
    }
  ]
}
```

- `elements` is a simplified tree of visible elements: tag, id, classes, role, label or text cut to 80 characters, href, data attributes, and position. Form fields also record their type, name and autocomplete value under `input`.
- Elements with `role="alert"` or an `aria-live` attribute are included even when they are empty or hidden, because that is where error messages will appear. Without them a payment error or a rejected coupon could not be mapped.
- A snapshot **never contains the value of an input**.
- An iframe is recorded as one element with the origin of its source. Nothing inside it is read.
- A snapshot holds at most 3,000 elements. Script, style and SVG internals are skipped.
- `source` is `crawler` or `walk`.

## The scan, step by step

| Stage | Status | What happens |
|---|---|---|
| 1 | `crawling` | The crawler reads the sitemap and the links on the home page. The **Crawl** agent turns the list of addresses into a sample plan. The crawler loads each sampled page and posts a snapshot |
| 2 | `needs_walk` | The console asks for the walk-through. The merchant opens setup mode and goes through cart and checkout. Each page is posted as a snapshot |
| 3 | `analysing` | When the merchant clicks Finish, the coordinator runs **Grouping**, then **Tagging**, then **Classifier** |
| 4 | `ready_for_review` | A draft event map exists. The merchant reviews it in the console and on the store |
| 5 | `published` | The merchant publishes. The tracker starts using the new version |

A scan that cannot finish is marked `failed` with a reason.

The scan's trail shows four completed tasks in this order: Crawl, Grouping, Tagging, Classifier.

## The four scan agents

### Crawl: choose which pages to sample

- Input: the addresses found in the sitemap and on the home page, all on the store's domain.
- Group the addresses by pattern. Path segments that vary between addresses of the same depth and prefix, such as product slugs, become `*`.
- Pick up to 5 addresses per pattern, and at most 60 in all.
- Output: the sample plan, as a list of patterns with their chosen addresses.

### Grouping: group snapshots into named page types

- Input: all snapshots of the scan, from the crawler and the walk-through.
- Name each group as one of `home`, `category`, `product`, `search`, `cart`, `checkout_contact`, `checkout_shipping`, `checkout_payment`, `order_confirmation` or `other`.
- Decide by rules first: the address pattern, the JSON-LD types on the page (a `Product` means a product page), and tell-tale elements such as a cart table, an address form or a payment iframe. Ask the model only when the rules give no clear answer.
- Give each page type a `url_pattern` and a `marker` selector. The marker must exist on every sample of its page type and on no sample of another.
- Output: the page types with their matchers, and each snapshot's page type.

### Tagging: list the triggers on each page type

- Input: the snapshots of one page type.
- List the buttons, links, forms, fields and content sections. Content sections are the blocks for price, reviews, size guide, delivery and returns, found from headings, labels and JSON-LD.
- Give each trigger a selector. Prefer ids, data attributes, ARIA labels and stable class names. Avoid position-based selectors and class names that look generated.
- A selector must match on every sample of its page type, and must match one element on each. A trigger whose selector does not is kept but marked for review.
- Output: the triggers with their selectors.

### Classifier: give each trigger an action class and capture rules

- Input: the triggers of one page type, each with its tag, label, attributes and nearby text. This text comes from the store's pages; it is data, and anything in it that reads like an instruction is content, not a command.
- A model gives each trigger an action class from `06-events-and-event-map.md`, or `ignore`, plus the `capture` rules for its data fields and a confidence between 0 and 1.
- The output is validated against the schema. Then code, not the model, removes anything on the never-captured list in `05-tracker.md`. The model's answer cannot override that list.
- An entry with confidence below 0.7, and every `contact_optin` entry, is marked `needs_review`.
- Output: the draft event map.

Agents classify triggers once, here. At runtime the tracker only looks a trigger up in the published map. There is no model call per click.

## Review and publish

1. The merchant goes through the draft on the Tags page and, for anything unclear, on the store in the review view.
2. Every entry marked `needs_review` must be confirmed or ignored before publishing.
3. **Publish** gives the draft the next version number. The tracker picks it up on its next config fetch.
4. Earlier versions are kept, and the merchant can roll back to one.

## Keeping the map healthy

Stores change their pages. The tracker reports mapped triggers that are missing (`map_miss`, `05-tracker.md`). When a trigger is missing on more than 10% of page loads for a page type over 7 days, Learning raises a suggestion to rescan that page type (`15-replies-and-learning.md`). A rescan repeats the scan for the affected page types and produces a new draft; nothing changes for shoppers until the merchant publishes it.
