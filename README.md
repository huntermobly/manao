# Retarget Customer: build documents (v1)

These files are the complete brief for building the first version of Retarget Customer. They are written for Claude to build from and assume the reader has no other context.

Retarget Customer finds out why shoppers leave an online store without buying, and wins them back with a personal message on WhatsApp, by email, as a browser notification, as a message on the store itself, or with a phone call the shopper asked for.

## How to use these files

1. Read files `01` to `04` first. They explain the product, the decisions that are fixed, and how the pieces fit.
2. Read `18-pitfalls.md` before writing code.
3. Build in the order given in `20-build-order-and-reporting.md`. Each milestone names the acceptance tests it must pass.
4. The other files are the reference for each part. Read a file in full before building the part it covers.

If two files disagree, stop and ask. Do not pick one.

## Files

| File | What it covers |
|---|---|
| `01-product-overview.md` | The problem, what the product does, the people involved, terms, the end-to-end flow, and one shopper worked through |
| `02-feature-list.md` | Every feature in one line, with the file that specifies it. Use it as a checklist |
| `03-decisions-and-scope.md` | Decisions that are fixed, what is in and out of v1, and questions for the owner |
| `04-architecture-and-stack.md` | The pieces, the stack, the repository layout, environment variables and how to run it |
| `05-tracker.md` | The tracker script that runs on the store |
| `06-events-and-event-map.md` | Event format, action classes and the event map |
| `07-web-console.md` | Every page of the merchant's web console |
| `08-site-connection-and-scan.md` | Connecting and verifying a store, the crawler, setup mode, and the scan that produces the event map |
| `09-backend-api-and-data.md` | Endpoints, sessions, shoppers, recoveries, coupons, links, analytics and tables |
| `10-agents-and-a2a.md` | The eighteen agents, how they exchange work over A2A, the coordinator and the chains |
| `11-decision-rules.md` | The exact rules for scoring, diagnosis, plays, offers, channel choice, checks and sending |
| `12-customer-profile.md` | The shopper profile and the store profile (ICP) |
| `13-message-pipeline.md` | Message shape, placeholders, Copy, the isolated Humanizer, text rules and the message template library |
| `14-delivery-channels.md` | WhatsApp through Meta, email, web push, on-site messages and call-back |
| `15-replies-and-learning.md` | Handling shopper replies, the conversations inbox, and measuring results |
| `16-settings.md` | Every merchant setting with its default |
| `17-privacy-and-security.md` | Consent, personal data, isolation between stores and other safeguards |
| `18-pitfalls.md` | The places where this build is most likely to go wrong |
| `19-acceptance-tests.md` | The shared fixture and eight acceptance tests |
| `20-build-order-and-reporting.md` | Milestones, the definition of done and what to report back |

## Conventions

- `OUR_DOMAIN` stands for the public address the backend is served from. It comes from the environment.
- Names in `code font` are identifiers. Use them as written.
- The **merchant** is the store's admin. The **shopper** is a visitor to the store.
- Times in examples are India Standard Time.
- Section and file references use the file name, for example "see `11-decision-rules.md`".

## What these files replace

Earlier drafts called the product "cart recovery", gave the merchant a Chrome extension, and sent WhatsApp messages through an unofficial library. All three are gone:

- The product is called **Retarget Customer**.
- The merchant's console is a **web app**. There is no browser extension.
- WhatsApp goes through **Meta's official WhatsApp Business Cloud API**.

These files replace every earlier draft.
