# 07. Web console

The console is the merchant's web app. It replaces the browser extension of earlier drafts. Store staff use it too.

## General

- A single-page app served by the backend at `/app`. Everything except sign-in, sign-up and password reset needs a signed-in merchant.
- A left-hand navigation lists the pages below. A site switcher at the top chooses which store the pages show; a merchant can have several.
- The session is an httpOnly cookie. Requests that change something carry a CSRF token.
- Phone numbers and email addresses are always shown masked, such as `+91 98XXX XXX10` and `pr***@example.com`. The one exception is the logged "Reveal number" action on the Calls page.
- Every list has an empty state that says what to do next.
- The console loads no third-party scripts, fonts or analytics. Charts are drawn by code bundled with the app.
- All data comes from the backend API (`09-backend-api-and-data.md`). The console stores nothing in the browser except interface preferences.

## Pages

### Account

| Screen | What it does |
|---|---|
| Sign up | Name, email, password of at least 10 characters. Creates the merchant and signs them in |
| Sign in | Email and password. Rate-limited |
| Forgot password | Emails a single-use link that is valid for one hour |
| Sign out | Ends the session |

### Home

A getting-started checklist for the selected store, with each step's state:

1. Connect the store
2. Scan it and publish the tags
3. Connect at least one channel
4. Upload coupon codes
5. Review the limits in Settings
6. Switch on the capture prompt

Once messages are going out, the page also shows four numbers for the last 7 days: shoppers who abandoned, how many of them could be reached, messages sent, and recovered revenue.

### Sites

- **Add a store**: store name, domain, currency, time zone, default country code.
- **Script tag**: shown with a copy button, and a one-line instruction to paste it into the store's `<head>`.
- **Verify**: runs the two checks in `08-site-connection-and-scan.md` and shows each as passed or failed, with the reason for a failure. The store's status is `unverified` or `connected`.
- **Recent events**: the last 20 events received (time, page type, action), so the merchant can see the tracker working. No visitor details.
- **Remove store**: deletes the store and its data after confirmation.

### Scan

- **Start scan** begins a scan (`08-site-connection-and-scan.md`).
- The page shows the scan's stages as they happen: finding pages, sampling pages (with a count), waiting for the walk-through, analysing, ready for review.
- When the crawler has finished, the page asks for the walk-through and shows **Open my store in setup mode**, which opens the store in a new tab.
- If the crawler was blocked, the page says so and asks the merchant to visit one page of each kind during the walk-through.
- When the scan is ready, the page summarises it: page types found, triggers tagged, how many need review. A link leads to Tags.
- Earlier scans are listed below.

### Tags

- A table of the draft event map, grouped by page type. Columns: the trigger (its label and selector), how it fires, its action class, what it captures, its confidence, and its state (`ok`, `needs review`, `ignored`).
- **Edit** opens a dialog to change the action class, fix a capture rule or ignore the trigger.
- **Review on my store** opens the store in setup mode with the review overlay.
- **Publish** creates the next version. It is disabled until every entry marked `needs review` has been confirmed or ignored.
- **Versions**: the list of published versions with a rollback button.
- **Tag health**: for each page type, the share of page loads in the last 7 days on which a mapped trigger was missing.

### Channels

One card per channel, each with its status, an on/off switch, a daily cap and a cost per message (used in the recovery report). `14-delivery-channels.md` specifies what each connection does.

| Card | Fields and actions |
|---|---|
| WhatsApp | WhatsApp Business Account ID, phone number ID, access token, app secret. After saving, the card shows the display number and verified name returned by Meta, the callback address and verify token to paste into the Meta app, whether Meta's webhook check has arrived, and a count of message templates by status. **Send test** sends an approved template to a number the merchant types. **Disconnect** deletes the stored credentials. The token and secret are write-only: after saving, only their last four characters are shown |
| Email | SMTP host, port, username, password, encryption; From name, From address, Reply-to; postal address for the footer. Saving opens a connection to check the details. A DNS check shows whether the From domain has SPF and DMARC records; it informs and does not block. **Send test** emails the merchant |
| Web push | Step 1: download `rc-sw.js` and place it at the store's root. Step 2: **Check** confirms the file is reachable. The switch can be turned on only after the check passes. **Send test notification** sends to browsers the merchant subscribed as test devices in setup mode |
| On-site message | On/off switch, corner position, and a preview of the card |
| Call-back | On/off switch, call hours, how long a request stays valid, and the number of attempts |

Below the cards, **Channel order** is a list the merchant drags into their preferred order (`channel_priority`).

### Message templates

- A list of message templates, filtered by channel, purpose, shape, language, tone and status.
- A template's page shows its text with placeholders highlighted, a preview with sample values, and its history. For WhatsApp it also shows the name, category and status at Meta, and the reason if Meta rejected it.
- **New variant**: choose channel, purpose, shape, language and tone. The backend runs Copy, the Humanizer and Guard's text rules, and shows the result with its trail. The merchant saves it as `ready`.
- **Edit** creates a new version, which is checked by Guard's text rules on save.
- **Submit to Meta** (WhatsApp only), for one template or for all that are `ready`. **Refresh statuses** reads the current statuses from Meta.
- **Retire** takes a template out of use.
- When WhatsApp is connected but no template is approved, a banner says that WhatsApp cannot send until Meta approves templates, with a button to submit the seed set.

### Coupons

- One pool per tier: `free_shipping`, `pct_5`, `pct_10`. The merchant can add other whole-percentage tiers.
- **Upload**: paste codes or upload a CSV, one code per line; set the `valid_until` date; tick "These codes exist in my store, each works once, and they expire on this date".
- Each pool shows counts of unused, reserved, issued, used and expired codes, with a warning when fewer than 10 are unused.
- Unused codes can be removed.

### Messages

- The **autopilot** switch sits at the top, with one line explaining it: when on, messages that pass every check go out without waiting.
- Four tabs: **Needs approval**, **Scheduled** (deferred, with the reason and the time), **Sent**, and **Failed or cancelled**.
- Each message shows its channel, the masked address, the shopper's profile summary, the exit reason with its evidence, the play, tier and product, the text as the shopper will see it, and the trail of agent tasks behind it.
- On a message that needs approval the merchant can **Approve**, **Reject**, or change the words. On channels with free text they edit the text. On a WhatsApp template message they cannot edit the words, because Meta approved them; they can choose another approved template. Any change goes through Guard again, and a failure names the rule.
- Several messages can be approved at once.

### Conversations

- A list of WhatsApp threads: the contact (masked number and first name, or "Unknown contact"), the last message, the time, a **Needs a person** flag, and whether the reply window is open and for how long.
- A thread shows every message in order, with its direction and delivery state. Automated replies are labelled "Assistant" and link to their trail.
- The reply box is enabled while the 24-hour window is open. A reply typed here is sent as written. When the window has closed, the box is disabled and says why.
- **Pause assistant** stops automated replies in that thread for 24 hours so that a person can take over. **Mark resolved** clears the flag.

### Calls

- Three tabs: **To call**, **Scheduled** and **Done**.
- Each task shows the first name if given, the masked number with **Reveal number** (logged in the audit log) and a call link, when the call was requested and for when, the number of attempts, and a brief: why the shopper left, the product and cart value, the offer to quote with its code and expiry, and the profile summary.
- Outcome buttons: **Reached**, **No answer**, **Wrong number**, **Declined**, with a notes field.

### Shoppers

- A list with: first name or "Visitor", stage, price sensitivity, last seen, the channels they can be reached on, and markers for opted out and no-message group.
- Search by exact phone number or email address.
- A shopper's page shows the profile and its summary, a timeline of events grouped by session, their recoveries and outcomes, their messages, and their consent history.
- **Delete this shopper's data** erases them (`17-privacy-and-security.md`).

### Analytics

A date range applies to the whole page.

| Section | Shows |
|---|---|
| Funnel | Shoppers reaching product view, add to cart, each checkout step and purchase, by day |
| Drop-off points | The last action before exit, by page type and trigger, with counts and shares |
| Exit reasons | Recoveries by reason, with the revenue at stake for each |
| Reach | Abandoners who could be reached on WhatsApp, on email, by web push, on-site only, or not at all |
| Recovery | Messages sent, delivered, read, clicked and replied, by channel; recovered orders and revenue; discount cost; message cost; lift over the no-message group; and a breakdown by reason, play and tier |
| Store profile | The aggregate picture of the store's buyers (`12-customer-profile.md`) |

### Suggestions

Learning's suggestions (`15-replies-and-learning.md`), each with the numbers behind it and the change it proposes. **Apply** makes the change; **Dismiss** hides it. Applied suggestions are listed below.

### Settings

The settings in `16-settings.md`, grouped as they are there. An **Activity log** tab lists sensitive actions: number reveals, map publishes, settings changes, channel connections and data deletions.

## The setup-mode toolbar

One part of the merchant's experience is not in the console: the toolbar that the tracker draws on the store's own pages in setup mode. It is specified in `08-site-connection-and-scan.md`.
