# 20. Build order, definition of done, and reporting

## Build order

Build in this order. Each milestone ends with its tests passing. Do not start a milestone on top of a failing one.

| # | Milestone | What it includes | Ends with |
|---|---|---|---|
| 1 | Foundations | Repository and shared schemas. Backend skeleton, database, job queue and the controllable clock. The demo store. The tracker's core: config, consent to tracking, identity, page types, capture, section views, transport. Ingest | Events from the demo store appear in the database, and none appear before tracking is accepted |
| 2 | Connect and scan | Console shell and accounts. Site connection and verification. The crawler. Setup mode and its toolbar. The A2A plumbing and the four scan agents. The Tags page, review and publish | Test 1 passes |
| 3 | Understand | Visitors, shoppers and merging. The capture prompt and consent records. Sessions. Intent, Profile and Diagnosis. The analytics endpoints. The Analytics and Shoppers pages | Unit tests for the scoring, profile and diagnosis rules pass |
| 4 | Decide and send on WhatsApp | The coordinator and the `plays` config. Strategy, Recommendation, Offer and coupon pools. Channel and Selector. The template library with its seed set. Guard and Delivery. The WhatsApp adapter sending approved templates through `fake-meta`, and reading template statuses. The welcome offer. The Messages, Coupons and Settings pages | Tests 2, 3 and 4 pass |
| 5 | Write | Copy, the Humanizer and Guard's text rules. Writing template variants. Submitting templates to Meta and receiving status webhooks. The Message templates page | Test 5 passes |
| 6 | Converse and learn | Meta webhooks for receipts and shopper messages. The customer service window. Reply and counter-offers. The Conversations and Channels pages. The click redirect. Attribution, the Learning report and suggestions | Test 6 passes |
| 7 | More channels | Email. Web push, with the service worker. On-site messages. Call-back and the Calls page. Fallback between channels. The second message across channels. The unsubscribe and preferences pages | Tests 7 and 8 pass |
| 8 | Hardening | The three checks below. The README. The manual checklist | Everything in "Done means" is true |

### Hardening checks

| Check | What it proves |
|---|---|
| Store isolation | A second merchant cannot read or change the first merchant's sites, shoppers, messages, templates or settings, including by guessing IDs |
| No personal data in side doors | After the acceptance tests have run, the fixture's phone numbers, email addresses and first names appear nowhere in the logs or in `agent_tasks` |
| No third-party hosts | No build output, no console page and no store page load with the tracker contacts any host other than ours and the store's |

## Done means

- One command runs all eight acceptance tests and the three hardening checks, and they pass.
- One command starts the whole system locally.
- The README covers setup, the environment variables, connecting the demo store, and the manual checklist below.
- Every feature in `02-feature-list.md` is true.

### Manual checklist

These need the owner's real accounts, so no test covers them. List them in the README and say in your report that they have not been run.

1. Connect a real WhatsApp Business number with the owner's Meta credentials. Paste the callback address and verify token into the Meta app, and see the webhook verified.
2. Submit one template, see it approved, and send it to your own number.
3. Reply from your phone, and see the message in Conversations and an answer arrive.
4. Send a test email through a real SMTP server, and check that it does not land in spam.
5. Subscribe a real browser to notifications on a store that hosts `rc-sw.js`, and receive a test notification.

## Reporting back

At the end of each milestone, report:

- What you built and which tests pass
- What you could not verify. The real Meta connection will be one such item
- Every place where you departed from these files, and why
- Anything in these files that turned out to be wrong, out of date or contradictory. Meta's API and the A2A protocol are the likeliest places

Do not call something done on the strength of a stub. If a part is stubbed, say so.

## Questions for the owner

The questions to ask at the start, with their defaults, are in `03-decisions-and-scope.md`. If one of the fixed decisions there blocks you, stop and ask. Do not work around it.
