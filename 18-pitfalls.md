# 18. Things that will trip you up

Read this before writing code. Each item is a place where this build is likely to go wrong, with what to do about it.

## WhatsApp and Meta

1. **A conversation can only be started with an approved template.** Free text outside the 24-hour window is refused. Build the window check into the channel adapter, and never try to get around approval with a template that is mostly one variable.
2. **Approval takes time and can fail.** A real template can wait hours or days at Meta and can be rejected. Nothing in the product may assume an instant approval. Until the three `generic` templates are approved, WhatsApp cannot send, and the console must say so plainly.
3. **A real Meta connection needs the owner's account.** Build against `fake-meta`. Leave connecting a real number, getting a real template approved and sending one message as a manual check (`20-build-order-and-reporting.md`).
4. **Webhook signatures are computed over the raw body.** If the framework parses the JSON first and you serialise it again, the signature will not match. Capture the raw bytes.
5. **Webhooks repeat and arrive out of order.** A `delivered` receipt can arrive before `sent`, and any webhook can arrive twice. Deduplicate, and never move a status backwards.
6. **Failures are often asynchronous.** A send can be accepted and then fail minutes later through a receipt. Fallback to another channel has to work from the receipt, not only from the send call.
7. **Template variables have rules.** No line breaks, no tabs, no long runs of spaces. Clean every value before filling it in.
8. **Meta's API changes.** The version, field names, error codes and prices in these files were written from memory of the documentation. Read the current documentation before coding, and say in your report where it differed.
9. **Anyone can message the store's number.** Messages from unknown numbers must reach the Conversations page and must not start an agent.

## Other channels

10. **Web push needs a service worker on the store's own origin.** It cannot be served from our domain. The merchant has to place one file, and the console has to check it. Register it under a narrow scope so that it does not replace a service worker the store already has.
11. **Web push barely works on iPhones**, and every browser needs a click before it will show the permission prompt. Do not present web push as reaching everyone.
12. **Email needs more than SMTP credentials.** Without SPF, DKIM and DMARC on the sender's domain, mail lands in spam. The console can check DNS and warn; it cannot fix the merchant's DNS.
13. **Plain SMTP gives no bounce feedback.** A message the server accepts may still bounce later. v1 does not see that. Say so in the console; do not show "delivered" for email, show "sent".
14. **A GET must never unsubscribe.** Mail scanners open every link. Only a POST may change anything.

## Tracker and setup mode

15. **The tracker must stay small and harmless.** Keep the core under its size budget by loading widgets and the setup module only when needed. Catch every error. The store's page must never break because of us.
16. **`sendBeacon` with a JSON content type can be blocked** by cross-origin rules. Send the body as plain text and parse it on the server.
17. **Safari limits script-set cookies and storage** to about a week. Treat recognising a returning visitor as best effort, and link the visitor again when the shopper opts in again.
18. **A store's Content-Security-Policy can block the tracker.** Verification should detect this and say so, not report a vague failure.
19. **The merchant's own visits must not become shopper data.** In setup mode the tracker captures nothing. Check this with a test.
20. **The setup token is a credential.** Keep it in the fragment, remove it from the address bar at once, and never log it.

## Scan

21. **The crawler can be blocked.** Bot protection may refuse a headless browser. The scan must still finish from the walk-through alone.
22. **Selectors break.** Stores use generated class names and change their layouts. Check every selector against all samples, and build the `map_miss` counter early.
23. **Payment forms usually sit in iframes from another origin**, and some stores hand the whole checkout to a third party. The tracker cannot see inside either. The last step it can see is where tracking stops.
24. **Model output varies from run to run.** Keep decisions in code, validate every model output against its schema, make tests assert structure and rules instead of exact wording, and run tests against the fake model.

## Decisions and data

25. **Coupons must exist in the store.** Never generate a code the store does not know.
26. **A deadline in a message must be the coupon's real expiry.** Nothing else may be presented as a deadline.
27. **An incentive promised in the capture prompt must be delivered on screen**, at once. A shopper who buys in the same visit would otherwise never receive it.
28. **Reserving a coupon must be atomic**, and every retry must be idempotent. Otherwise two shoppers get the same code, or one shopper gets two messages.
29. **Merging shoppers is easy to get wrong.** When a contact detail links two records, everything attached to both must end up on one, and there must be one open recovery at most.
30. **Time zones.** Store every time in UTC. Apply quiet hours, call hours and daily caps in the store's time zone. Use the controllable clock everywhere, including in the job queue.
31. **Personal data leaks through side doors**: logs, error messages, agent task records, analytics exports. Add a test that searches them for the fixture's phone numbers and email addresses.

## Protocols and tools

32. **A2A is young and changes.** Check the current specification and the package's version before writing against it. If the official SDK does not fit, implement the small subset named in `04-architecture-and-stack.md` and say so.
33. **A stub is not done.** If a part is stubbed or could not be verified, say so in the report. The real Meta connection will be one such part.
