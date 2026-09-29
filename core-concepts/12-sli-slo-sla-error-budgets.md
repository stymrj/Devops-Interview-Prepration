# SLI, SLO, SLA and Error Budgets Interview Preparation Guide

*How to Answer SLI, SLO, SLA and Error Budgets Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [SLIs What You Actually Measure](#slis-what-you-actually-measure)
2. [SLOs Setting the Target](#slos-setting-the-target)
3. [SLAs Contracts With Teeth](#slas-contracts-with-teeth)
4. [Error Budgets The Reliability Currency](#error-budgets-the-reliability-currency)
5. [Burn Rates Catching Trouble Early](#burn-rates-catching-trouble-early)
6. [Traps and Judgment Calls](#traps-and-judgment-calls)

---

## SLIs What You Actually Measure

### Q1: What is an SLI? Give me a real example.

**How to Answer:**

"An SLI — Service Level Indicator — is the actual measurement of how your service behaves, the raw number. An SLO is the target you set on it, and an SLA is the contract. The SLI is the bottom of that stack: just the metric, no promises attached.

Real example from my world: the ratio of HTTP responses that come back 2xx within 300ms, divided by all responses. Over the last 30 days that number might be 99.94%. That's the SLI — the fact.

The rookie mistake is confusing the indicator with the objective. Saying 'our SLO is 99.9% uptime' without being able to say what you measure, over what window, for which requests — that's not an SLO, that's a wish."

**Key Point:** "An SLI is the raw measurement of service behavior — requests succeeded over total requests, latency under a threshold — with no target or promise attached yet."

---

### Q2: What makes a good SLI? What's a bad one?

**How to Answer:**

"A good SLI measures what the user experiences, not what the machine feels. Users feel request success rate and latency. They don't feel your CPU utilization or pod restart count — those are signals for me, not SLIs.

The classic bad SLI is 'server uptime.' A box can be up and healthy while the app on it returns 500s to every user. Uptime of the VM tells you nothing about the service. Measure at the boundary the user touches: the load balancer, the API gateway, the checkout endpoint.

My rule: if the SLI is green but users are angry, it's a bad SLI. Valid, meaningful, and user-facing — it has to be measurable, it has to reflect real experience, and it has to be the thing you'd actually page on."

```promql
# availability SLI: successful requests over total
sum(rate(http_requests_total{status=~"2.."}[5m]))
/ sum(rate(http_requests_total[5m]))
```

**Key Point:** "Good SLIs measure user experience at the service boundary — success rate and latency — not internal signals like CPU or uptime that can look green while users suffer."

---

## SLOs Setting the Target

### Q3: What is an SLO? How do you pick the number?

**How to Answer:**

"An SLO — Service Level Objective — is the target you set on an SLI over a time window. Something like: 99.9% of requests succeed within 300ms, measured over 30 days. The SLI is the measurement, the SLO is the promise you make to yourselves about it.

How do you pick the number? You look at what the business actually needs and what you've historically achieved. If the product team needs 99.9% and you've been running at 99.95%, that's a comfortable target. Picking 99.99% because it sounds impressive is how you end up freezing all deploys for a month.

The number has to be defensible to both sides. Too loose and it's meaningless — you hit it while users churn. Too tight and the error budget is zero, so engineering grinds to a halt. The right SLO is one where missing it occasionally is affordable and hitting it takes real work."

**Key Point:** "An SLO is your internal target on an SLI over a window — pick it from business need plus historical reality, tight enough to matter, loose enough to leave an error budget."

---

### Q4: Why not just target 100%? What's wrong with more nines?

**How to Answer:**

"Because 100% is infinitely expensive. Every extra nine — 99.9 to 99.99 to 99.999 — costs roughly ten times the engineering effort for reliability users can't even perceive. 99.9% means 43 minutes of downtime a month; 99.99% means about 4 minutes. Can your users tell the difference? Almost never.

The Google SRE framing I use: reliability is a feature, and like any feature it has a cost. Past a certain point you're spending engineering years to buy minutes nobody notices. That effort should go into product.

There's also a hidden cost: chasing perfection makes teams afraid to change anything. If the target is unattainable, every deploy feels dangerous, velocity dies, and ironically the system gets more fragile because nobody touches it. Sensible nines keep shipping safe."

**Key Point:** "100% reliability is infinitely expensive and freezes change — each extra nine costs ~10x the effort for downtime users can't feel, so target the nines the business actually needs."

---

## SLAs Contracts With Teeth

### Q5: What's the difference between an SLO and an SLA?

**How to Answer:**

"The SLO is your internal target; the SLA is the external promise with consequences. If you miss your SLO, you freeze deploys and fix reliability. If you miss your SLA, you pay the customer money. Same measurement, completely different stakes.

In practice the SLA is always looser than the SLO. You might run an internal SLO of 99.9% but only promise customers 99.5% in the contract. That gap is your safety margin — the SLO fires early so you fix things before the SLA is ever threatened.

The interview trap is saying you have an SLA when you mean an SLO. If nobody owes anyone money or credits, it's an SLO. Real talk: most companies I interview at have SLOs and call them SLAs. The distinction that matters is whether a breach has contractual consequences."

**Key Point:** "SLO is the internal target, SLA is the external contract with penalties — keep the SLA looser than the SLO so you get an early warning before money is on the line."

---

### Q6: What happens when you breach an SLA?

**How to Answer:**

"You pay, literally. SLAs are backed by service credits — miss the target and the customer gets a percentage of their bill back, usually tiered: 10% credit under 99.9%, 25% under 99%, that kind of structure. It's not a lawsuit, it's contractual compensation.

But the real cost isn't the credits, it's trust. A cloud provider that keeps breaching SLA gets churn, not just credit payouts. That's why the SLO-to-SLA gap matters so much — the SLO is the tripwire, the SLA is the cliff.

One more thing interviewers like: SLAs almost always have exclusions — scheduled maintenance windows, force majeure, customer-caused issues. If a candidate quotes an SLA number without asking 'excluding what?' they're reciting, not understanding. The fine print is where SLAs live."

**Key Point:** "Breaching an SLA means service credits to the customer — but the real damage is trust, which is why your SLO should trip long before the SLA is at risk."
