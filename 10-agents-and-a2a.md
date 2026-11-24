# 10. Agents and A2A

## The idea

Each agent does exactly one job. Agents pass work to each other only as Agent2Agent (A2A) protocol messages. This keeps every decision small, testable and visible, and it lets a new agent be added later by registering it, not by rewiring the others.

Twelve of the eighteen agents are plain code. Six call a language model (Grouping, Classifier, Profile, Copy, Humanizer and Reply), and none of those decides money, consent or channel (`03-decisions-and-scope.md`, decision 8).

## The eighteen agents

| # | Agent | Its one task | Built with | Returns |
|---|---|---|---|---|
| 1 | Crawl | Choose which pages to sample | Code | Sample plan |
| 2 | Grouping | Group snapshots into named page types | Code, model for unclear cases | Page types with matchers |
| 3 | Tagging | List the triggers on each page type | Code | Triggers with selectors |
| 4 | Classifier | Give each trigger an action class and capture rules | Model | Draft event map with confidence |
| 5 | Intent | Score the shopper's buying intent | Code | Score from 0 to 100 |
| 6 | Profile | Keep the shopper's profile current | Code, model for the summary | Profile |
| 7 | Diagnosis | Name the exit reason | Code | Reason and evidence |
| 8 | Strategy | Choose the play | Code | Play |
| 9 | Recommendation | Choose the product to pitch | Code | Product |
| 10 | Offer | Choose the tier and reserve the code | Code | Tier, code, prices, expiry |
| 11 | Channel | Choose the channel for this message | Code | Channel and mode |
| 12 | Selector | Choose a stored message template | Code | Template and its variables |
| 13 | Copy | Draft a message | Model | Draft fields with placeholders |
| 14 | Humanizer | Rewrite a draft in a natural voice | Model, isolated | Fields with the same placeholders |
| 15 | Guard | Approve, defer, send back or veto | Code | Verdict and reason |
| 16 | Delivery | Schedule and send through the channel | Code | Send status |
| 17 | Reply | Handle a shopper's reply | Code for opt-outs, model for the rest | Reply action |
| 18 | Learning | Measure results and suggest changes | Code | Report and suggestions |

Where each is specified:

| Agents | File |
|---|---|
| Crawl, Grouping, Tagging, Classifier | `08-site-connection-and-scan.md` |
| Intent, Diagnosis, Strategy, Recommendation, Offer, Channel, Selector, Guard, Delivery | `11-decision-rules.md` |
| Profile | `12-customer-profile.md` |
| Copy, Humanizer | `13-message-pipeline.md` |
| Reply, Learning | `15-replies-and-learning.md` |

## How they work together

- Every agent publishes an **Agent Card**: its name, a one-sentence description, its single skill, its input and output schemas, and a version.
- Work is sent as an A2A message whose data part matches the agent's input schema. The result comes back as the task's artifact. Use the protocol's own task states.
- In v1 all eighteen run inside the backend, each behind its own A2A endpoint. Call them through the A2A client even though they are local, so that any of them can move to its own service later.
- A thin **coordinator** passes each piece of work through the agents in order and writes every task to `agent_tasks`. It is not an agent and holds no business logic. It follows the chains below and acts on Guard's verdicts.
- Every message carries `trace_id` (the recovery, scan or template ID) and `site_id`.
- Every task has an idempotency key made of the trace ID, the step and the attempt number, so that a retry never reserves a second coupon or sends a second message.
- An agent that calls a language model gets a 20-second timeout and one retry. If it still fails, the coordinator follows the fallback given for that agent.

## The chains

A chain is the order in which the coordinator calls agents for one kind of work.

```
Every session end      Intent > Profile

Recovery, start        Diagnosis > Guard (eligibility) > Profile (summary)
                       > Strategy > Recommendation > Offer > Channel

Then, by the mode Channel returned:
  approved_text        Selector > Guard (final) > Delivery
  free_text            Copy > Humanizer > Guard (final) > Delivery
  task                 Guard (final) > Delivery

Second message         Offer > Channel > the chain for the mode
Counter-offer          Offer > Copy > Humanizer > Guard (final) > Delivery
Welcome offer          Offer > Selector > Guard (final) > Delivery

Scan                   Crawl > Grouping > Tagging > Classifier
Template authoring     Copy > Humanizer > Guard (text rules only)
Shopper reply          Reply > Guard (final) > Delivery
Nightly                Learning
```

The three modes:

| Mode | Used when | What supplies the words |
|---|---|---|
| `approved_text` | The channel only allows text approved in advance: a WhatsApp message outside the 24-hour window | A stored template that the provider has approved, chosen by Selector |
| `free_text` | The channel allows any text: email, web push, on-site message, WhatsApp inside the window | Copy and the Humanizer, for this one message |
| `task` | The "message" is work for a person: a call-back | No text is sent. Delivery creates a call task with a brief |

"Profile (summary)" is a second call to the Profile agent, asking only for the one- or two-sentence summary. It comes after the eligibility check so that no model is called for a shopper who cannot be contacted.

### Guard's verdicts

| Verdict | What the coordinator does |
|---|---|
| `approve` | Carries on to the next agent |
| `veto` | Stops this message. If nothing has been delivered in the recovery, the recovery closes as `blocked` with the reason; otherwise it stays open until its window ends |
| `send_back` | Runs the chain again from the agent Guard names, passing Guard's reason. At most twice per message |
| `defer` | Holds the message until the time or condition Guard names, then runs Guard (final) again from the top |

### Fallbacks

| Failure | What the coordinator does |
|---|---|
| Copy fails twice | Switches that message to stored text: Selector picks the fixed template for the channel, then Guard (final), then Delivery |
| The Humanizer fails or is rejected twice | Uses Copy's draft if it passes Guard, otherwise stored text (`13-message-pipeline.md`) |
| Selector finds no usable template | Runs Channel again without that channel. If no other channel is available, the message is deferred with `no_approved_text` |
| Delivery reports a failure | Follows the failure table in `11-decision-rules.md` |
| The profile summary fails | Carries on without a summary |
| Reply's model fails | Sends the hand-off text and flags the conversation for a person |

## Chains live in config

Keep the chains and the plays in a `plays` config, not in code:

```json
{
  "chains": {
    "session_end": ["intent", "profile"],
    "recovery_start": ["diagnosis", "guard:eligibility", "profile:summary", "strategy", "recommendation", "offer", "channel"],
    "approved_text": ["selector", "guard:final", "delivery"],
    "free_text": ["copy", "humanizer", "guard:final", "delivery"],
    "task": ["guard:final", "delivery"],
    "second_message": ["offer", "channel"],
    "counter_offer": ["offer", "copy", "humanizer", "guard:final", "delivery"],
    "welcome_offer": ["offer", "selector", "guard:final", "delivery"],
    "scan": ["crawl", "grouping", "tagging", "classifier"],
    "template_authoring": ["copy", "humanizer", "guard:text"]
  },
  "plays": {
    "resume_payment": { "start_tier": "none" },
    "working_coupon": { "start_tier": "lowest_percentage" },
    "free_shipping": { "start_tier": "free_shipping" },
    "resume_checkout": { "start_tier": "none" },
    "alternative": { "start_tier": "none" },
    "discount": { "start_tier": "by_price_sensitivity" },
    "reassure": { "start_tier": "none" },
    "reminder": { "start_tier": "none" }
  }
}
```

In v1 every play uses the same chains. A later play, such as a back-in-stock alert or a review request after purchase, can add or skip agents by adding an entry here. Adding an agent means registering its Agent Card and naming it in a chain.

## An Agent Card

Follow the field names of the A2A version you install. The content is this:

```json
{
  "name": "Offer",
  "description": "Chooses the incentive tier for one message and reserves a coupon code.",
  "version": "1.0.0",
  "url": "http://localhost:3000/a2a/offer",
  "skills": [
    {
      "id": "choose_offer",
      "name": "Choose offer",
      "description": "Given the play, the shopper's price sensitivity and the merchant's limits, returns the tier, a reserved code, prices and expiry."
    }
  ]
}
```

The input and output schemas for every agent live in `packages/shared` and are referenced from the cards.

## An example exchange

Coordinator to Offer:

```json
{
  "trace_id": "rec_7f2c",
  "site_id": "site_01",
  "input": {
    "play": "discount",
    "price_sensitivity": "high",
    "product_id": "AR-001",
    "price": 3000,
    "currency": "INR",
    "max_discount_pct": 10,
    "cooldown_active": false,
    "product_excluded": false,
    "welcome_code": null,
    "message_number": 1,
    "last_tier": null
  }
}
```

Offer's artifact:

```json
{
  "tier": "pct_10",
  "shape": "percent",
  "discount_pct": 10,
  "offer_price": 2700,
  "coupon_id": "cpn_0193",
  "coupon_code": "SAVE10-7Q2K",
  "expires_at": "2026-11-30T23:59:00+05:30",
  "notes": []
}
```

## The trail

`agent_tasks` records, for every task: the trace ID, the agent, the step, the state, the input, the output, the times, and whether a model was called. The Messages page shows this as the trail behind each message, so the merchant can see why a shopper got that offer on that channel.

Inputs and outputs are stored without personal data. Agents work with the shopper ID; phone numbers, email addresses and names are added only by code at the moment a message is filled in and sent (`13-message-pipeline.md`).

## Limits on loops

| Loop | Cap |
|---|---|
| Guard sends work back | Twice per message |
| Reply asks Offer for a counter-offer | Twice per recovery |
| Channel fallback after a failed delivery | Twice per message |
| Automated replies in one conversation | Ten in 24 hours, then the conversation is flagged for a person |
