---
title: "How Ads Really Get Chosen: Signals, Auctions, Pacing, and Measurement"
date: 2026-07-16
permalink: /signals-to-ad-auctions/
author_profile: true
read_time: true
excerpt: "A practical mental model for how signals become targeting decisions, auction scores, budget allocation, and conversion measurement."
tags:
  - ads
  - machine-learning
  - ranking
  - auctions
  - recommender-systems
---

**Table of Contents**
* [Signals Are Observations, Not Objectives](#signals-are-observations-not-objectives)
* [Delivery Is Broader Than Ranking](#delivery-is-broader-than-ranking)
* [The Conversion Is Whatever the Advertiser Optimizes](#the-conversion-is-whatever-the-advertiser-optimizes)
* [The Ad Score: What the Auction Really Ranks](#the-ad-score-what-the-auction-really-ranks)
* [Pacing and Ranking Are Not the Platform Competing With Itself](#pacing-and-ranking-are-not-the-platform-competing-with-itself)
* [Score, Payment, and Business Value Are Not the Same](#score-payment-and-business-value-are-not-the-same)
* [What About Cars and Real Estate?](#what-about-cars-and-real-estate)
* [The Whole Loop, One Impression at a Time](#the-whole-loop-one-impression-at-a-time)
* [Reference](#reference)

---

From the outside, online advertising looks simple: an advertiser bids, a platform picks an ad, a user sees it. Look inside and the vocabulary blurs together — *signals*, *targeting*, *delivery*, *ranking*, *pacing*, *conversion*, *value*, and *payment* all name different layers of the same decision.

This post builds a mental model by following one advertising opportunity from beginning to end, using public abstractions rather than any proprietary implementation. The story in one line:

> Signals describe what happened. Targeting defines the opportunity. Pacing manages scarce budget across time. Ranking picks the winner now. Measurement closes the loop.

## Signals Are Observations, Not Objectives

A signal is an observed piece of information the system can use to make or evaluate a decision:

- A user viewed a product page, watched a video, or submitted a form.
- An ad belongs to a category and is eligible for a placement.
- An advertiser reported a purchase, lead, subscription, or store visit.
- A user hid, skipped, or reported an ad.

The same signal plays several roles at once. A purchase is a **measurement event** for reporting, a **training label** for a conversion model, and an input for finding people who resemble existing customers.

![Signals: one data layer, three jobs](/images/blog/ads-signals-three-jobs.png)

Meta describes three broad uses of advertising data — targeting, delivery, and measurement[1] — best read as three questions:

- **Targeting: who and what is eligible?** Audience constraints, the campaign objective, and the optimized action — a click, lead, purchase, or conversion value.
- **Delivery: what should actually be shown?** Candidate retrieval, eligibility, budget controls, auction-time prediction, ranking, and placement.
- **Measurement: what happened afterward?** Connecting impressions and clicks to outcomes, reporting performance, and producing new labels.

Together they form a loop:

$$
\text{Signals}
\rightarrow
\text{Targeting and Delivery}
\rightarrow
\text{User Action}
\rightarrow
\text{Measurement}
\rightarrow
\text{New Signals}
$$

Signals connect the whole system, not just one model.

## Delivery Is Broader Than Ranking

*Delivery* and *ranking* are often used interchangeably, but they are not the same. A simplified pipeline:

$$
\text{All Ads}
\rightarrow
\text{Eligibility and Targeting}
\rightarrow
\text{Candidate Retrieval}
\rightarrow
\text{Pacing and Ranking}
\rightarrow
\text{Final Delivery}
$$

**Delivery** is the whole process that turns a campaign into impressions, respecting targeting, budget, schedule, policy, placement, and frequency. **Ranking** is the narrower auction-time question:

> For this user, at this moment, in this placement, which eligible ad creates the highest value?

Ranking is one stage of delivery, which also decides whether an ad reaches ranking at all and whether it can keep spending over time.

## The Conversion Is Whatever the Advertiser Optimizes

**Conversion** is the most misread word in advertising. It does not necessarily mean an online purchase — it is whatever the advertiser wants you to do after seeing the ad: click a link, view a landing page, install an app, submit a lead, subscribe, or buy a car.

When an advertiser picks a performance goal, it tells delivery which result to pursue.[2] At auction time a model estimates the chance of that result:

$$
eCVR
=
P(\text{optimized event}
\mid
\text{impression, user, ad, context})
$$

This conditional probability — often written $$P(\text{Conv} \mid \text{Imp})$$ — is exactly what the ads ranking model is trained to predict. And the same user-ad pair can have very different probabilities for different goals:

$$
P(\text{Click}\mid\text{Impression})
\neq
P(\text{Purchase}\mid\text{Impression})
$$

A user who clicks everything but rarely buys is attractive to a click campaign and useless to a purchase campaign. Hence the rule:

> You get what you optimize for.

If the system only ever sees *lead submitted*, it learns who submits forms — not who becomes a good customer.

## The Ad Score: What the Auction Really Ranks

Public descriptions of Meta's auction combine three factors: the advertiser's bid, the estimated action rate, and ad quality.[1] The auction ranks by a single number, the **ad score**, with two parts:

$$
\text{Ad Score}
=
\text{Ad Value}
+
\text{User Value}
$$

**Ad Value** is what the ad is worth to the advertiser: across every optimized event the ad is expected to produce, its predicted rate times what the advertiser says each event is worth. It is the value the platform delivers to advertisers — and what keeps them spending.

**User Value** is how much the user is likely to value the experience; it shifts with time and placement. It lifts a welcome, relevant ad and penalizes a misleading, clickbait, or offensive one — the kind you immediately cross out — even when the bid is high.

Concretely, the auction evaluates a single **total bid**:

$$
\text{Ad Score}
=
\underbrace{\text{Paced Bid} \times eCVR \times \text{Subsidy}}_{\text{Ad Value}}
+
\underbrace{\text{Quality Bid}}_{\text{User Value}}
$$

- **Paced Bid** — what the advertiser is effectively willing to pay for this impression, after pacing adjusts the raw bid.
- **eCVR** — the $$P(\text{Conv} \mid \text{Imp})$$ from above; the product $$\text{Paced Bid} \times eCVR$$ is the **advertiser bid**, the expected pay for the impression.
- **Subsidy** — a multiplier that promotes chosen ads. To nudge advertisers toward a format, the platform might give every Shop ad a subsidy of $$1.1$$, lifting its ad value by 10%.
- **Quality Bid** — the User Value term as a bid, so a high monetary bid cannot buy its way past a bad experience.

![How one impression becomes an auction decision](/images/blog/ads-auction-decision.png)

A quick example:

| Candidate | Paced bid per conversion | eCVR | Ad value | User-value adjustment | Ad score |
|---|---:|---:|---:|---:|---:|
| Ad A | $50 | 3% | 1.50 | +0.10 | 1.60 |
| Ad B | $30 | 6% | 1.80 | +0.05 | 1.85 |

Ad A bids more, but Ad B has the higher ad score, so Ad B wins. "Highest bidder wins" is incomplete; the better statement is that among eligible ads, the highest auction-time ad score wins. We return to what the winner actually *pays* [below](#score-payment-and-business-value-are-not-the-same).

## Pacing and Ranking Are Not the Platform Competing With Itself

The architecture can look circular: pacing decides how aggressively a campaign bids, then ranking picks the highest-value ad. Is the platform bidding against itself? No — the two solve different problems.

![Pacing and ranking solve different problems](/images/blog/ads-pacing-vs-ranking.png)

**Pacing: one campaign across time.** A campaign has a fixed budget and a stream of future opportunities, and must decide how to spread its budget across many auctions:

$$
\max_{\pi}
\mathbb{E}\left[ \sum_{t=1}^{T} r_t \right]
\quad \text{subject to} \quad
\mathbb{E}\left[ \sum_{t=1}^{T} \text{Cost}_t \right] \leq B
$$

where $$r_t$$ is the advertiser's objective for opportunity $$t$$ and $$B$$ is the budget. Meta research on multiplicative pacing[3] reads this as a **shadow price of budget**: scarce budget makes a campaign selective, underspending makes it aggressive, and better traffic expected later raises the opportunity cost of spending now. So pacing is not "pick the highest pCVR" — a likely conversion can still be a poor use of budget if the opportunity is expensive.

**Ranking: many campaigns in one auction.** Ranking compares candidates horizontally at a single moment; pacing compares one campaign's opportunities vertically across time.

> Pacing decides how much this campaign wants to win now. Ranking decides which campaign wins now.

They interact constantly, but they are not duplicates.

## Score, Payment, and Business Value Are Not the Same

Three quantities get collapsed constantly:

$$
\text{Ranking Score}
\neq
\text{Payment}
\neq
\text{Advertiser's Business Value}
$$

**Ranking score** (the ad score above) decides *who wins*. **Payment** decides *what the winner is charged* — second-price-style auctions charge a critical clearing price, not the full bid. Google, for example, describes actual CPC as the minimum needed to beat the ad immediately below, not the maximum bid.[6] Modern auctions are more complex, so don't read the payment rule off a ranking equation.

**Advertiser business value** is the expected economic outcome:

$$
\text{Expected Business Value}
=
P(\text{Conversion}\mid\text{Impression})
\times
\text{Value per Conversion}
$$

The platform's revenue tracks the *payment*, not the advertiser's full business value — so calling the auction score or clearing price the "real" value of an ad is a mistake.

## What About Cars and Real Estate?

Some purchases never happen online. A user sees a car ad today, talks to a salesperson next week, and buys months later in person. The fix is to optimize a *measurable* point in the funnel — for a car:

$$
\text{Impression}
\rightarrow
\text{Lead}
\rightarrow
\text{Qualified Lead}
\rightarrow
\text{Test Drive}
\rightarrow
\text{Purchase}
$$

and for real estate:

$$
\text{Impression}
\rightarrow
\text{Inquiry}
\rightarrow
\text{Qualified Buyer}
\rightarrow
\text{Property Tour}
\rightarrow
\text{Offer}
\rightarrow
\text{Closing}
$$

![Long-funnel advertising and offline conversions](/images/blog/ads-offline-funnel.png)

The model can optimize any event that is defined and reliably reported — $$P(\text{Lead}\mid\text{Imp})$$, $$P(\text{Qualified Lead}\mid\text{Imp})$$, or $$P(\text{Offline Purchase}\mid\text{Imp})$$. Deeper events sit closer to real business value but are sparser and slower.

Why depth matters:

| Candidate | Lead probability | Probability a lead is qualified |
|---|---:|---:|
| Ad A | 1.0% | 5% |
| Ad B | 0.5% | 40% |

Optimize raw leads and Ad A wins. But the qualified-lead probability tells another story:

$$
P(Qualified)_A = 1.0\% \times 5\% = 0.05\%
\qquad
P(Qualified)_B = 0.5\% \times 40\% = 0.20\%
$$

Ad B produces fewer forms but four times as many qualified leads. This is the familiar complaint — "lots of cheap leads, all low quality." If the platform only sees form submissions, it is doing exactly what it was asked.

Meta's Conversions API can return CRM and offline events, including lower-funnel stages and store visits,[4][5] moving the learning target closer to qualified leads or real sales. Long-funnel models still fight hard problems — sparse labels, delayed feedback, identity matching, attribution uncertainty, and selection bias — but the principle is simple:

> Optimization is only as good as the depth, reliability, and timeliness of the feedback signal.

## The Whole Loop, One Impression at a Time

Now trace one opportunity end to end. The advertiser defines a goal, and signals from past impressions, actions, and reported outcomes train the models. Targeting builds the candidate set — only campaigns that satisfy audience, placement, policy, schedule, and budget can compete. Pacing sets each campaign's aggressiveness, ranking scores the survivors as ad value plus user value, and the auction hands the impression to the highest score. Measurement then observes the outcome — a click, lead, purchase, or offline event — ties it back to the campaign, and turns it into a new signal for the next round.

The complexity is an illusion of viewing the parts separately. Seen as a loop, the deepest distinction isn't "model vs. auction" — it's the **time horizon**. A campaign spends across millions of future auctions; the marketplace allocates one impression now; measurement later decides what can be observed and attributed. Modern ad delivery is the machinery that stitches those horizons into one learning loop.

---

## Reference

1. [Meta: About Ad Auctions](https://www.facebook.com/business/help/430291176997542)
2. [Meta: About Performance Goals](https://www.facebook.com/business/help/355670007911605)
3. [Meta Research: Pacing Equilibrium in First Price Auction Markets](https://research.facebook.com/publications/pacing-equilibrium-in-first-price-auction-markets/)
4. [Meta for Developers: Conversions API for CRM Integration](https://developers.facebook.com/documentation/ads-commerce/conversions-api/conversion-leads-integration)
5. [Meta for Developers: Conversions API Best Practices](https://developers.facebook.com/documentation/ads-commerce/conversions-api/best-practices)
6. [Google Ads: Actual Cost-per-click Definition](https://support.google.com/google-ads/answer/6297?hl=en)

---

*Last updated: July 16, 2026*
