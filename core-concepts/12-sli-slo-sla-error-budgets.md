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

---

## Error Budgets The Reliability Currency

### Q7: What is an error budget? How do you calculate it?

**How to Answer:**

"The error budget is the amount of failure your SLO allows — literally 100% minus the SLO, over the window. SLO of 99.9% over 30 days means a 0.1% error budget, which is about 43 minutes of downtime a month. That's the budget the team gets to spend on deploys, experiments, and incidents.

Think of it as a currency. Budget remaining means ship fast — take risks, deploy often, run experiments. Budget burned means stop shipping features and invest in reliability until it recovers. It turns the dev-versus-ops argument into a number both sides can see.

The trap is treating the budget as a target to spend. You don't aim to use all 43 minutes — it's a ceiling, not a goal. Teams that brag about 'using the whole budget' are one incident away from freezing everything next month."

**Key Point:** "An error budget is 100% minus your SLO — the allowed unreliability per window; spend it on shipping when it's healthy, freeze features and harden when it's gone."

---

### Q8: What actually changes when the error budget is exhausted?

**How to Answer:**

"Feature work stops. That's the whole point of the mechanism. When the budget hits zero, the team shifts from shipping features to reliability work — bug fixes, hardening, reducing toil, paying down the debt that burned the budget. No exceptions for 'just this one small feature.'

In practice it means deploys get gated. At Google, the SRE team can literally veto launches for a service that's out of budget. In smaller teams it's a team agreement: the dashboard goes red, and the sprint plan gets reprioritized to reliability tickets.

The power move in an interview is noting that this has to be agreed with product beforehand. If you invent the policy during the incident, product sees it as ops blocking progress. If it's agreed when the SLO is set, it's just the rule everyone signed. Error budgets only work as a pre-committed contract, not a surprise."

**Key Point:** "Zero budget means feature freezes and reliability-only work until it recovers — and this only works if product agreed to the rule when the SLO was set, not mid-incident."

---

## Burn Rates Catching Trouble Early

### Q9: What is burn rate? Why not just alert on the SLO directly?

**How to Answer:**

"Burn rate is how fast you're consuming the error budget relative to the window. If your 30-day budget is 43 minutes and you burn 20 minutes in one hour, you're burning way faster than sustainable — that's a fast burn, and it deserves a page.

You can't just alert 'SLO breached' because by the time the 30-day number moves, the incident is ancient history. A 30-day SLO barely twitches during a two-hour outage. Burn rate translates the long-window math into something actionable right now.

The standard setup is two alerts: fast burn — like 14x the sustainable rate over an hour — pages someone immediately, and slow burn — 2x over a day or two — opens a ticket. Fast burn means wake up; slow burn means investigate this week. That split is what keeps you from paging at 3am for a problem that isn't urgent."

**Key Point:** "Burn rate is budget consumption speed versus the window — fast burn pages, slow burn tickets, because the 30-day SLO number itself moves too slowly to alert on."

---

### Q10: How do you set burn-rate alerts without causing alert fatigue?

**How to Answer:**

"You pair a short window with a long window before paging — that's the multiwindow trick from Google's SRE workbook. A 1-hour fast burn alone can fire on a blip; requiring the 5-minute window to also be burning confirms it's real. Two conditions, one page, far fewer false alarms.

The numbers: page when you're burning at 14x over the last hour AND the last 5 minutes, ticket when you're at 6x over 6 hours AND the last 30 minutes. These aren't magic — you tune them against your actual incident history.

And the fatigue killer nobody mentions: the alert message has to say what to do. 'Burn rate 14x' is noise. 'Burn rate 14x on checkout success rate — runbook link, likely deploy, last good deploy 22:14' is actionable. Every alert that fires without a next step trains the team to ignore alerts."

```promql
# fast-burn page: 14x budget consumption, confirmed on short window
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  / sum(rate(http_requests_total[1h]))
  > 14 * 0.001
) and (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
  > 14 * 0.001
)
```

**Key Point:** "Use multiwindow burn-rate alerts — long window plus short window confirmation — and make every alert message say what to do next, or it just trains people to ignore pages."

---

## Traps and Judgment Calls

### Q11: Should every service have the same SLO?

**How to Answer:**

"Absolutely not, and this is a common interview trap. A payment service processing money gets 99.99%; an internal admin dashboard that three people use gets 99%. Same SLO for everything means you're over-engineering the unimportant and probably under-delivering on the critical.

The SLO should follow the blast radius and the business cost of failure. I ask: who notices when this is down, how fast, and what does a minute of downtime cost? The answers set the nines. A batch job that retries harmlessly doesn't need the same budget as checkout.

There's a real-world angle too: uniform SLOs make error budgets meaningless. If everything is 99.9%, nothing is prioritized, and the team can't tell which budget burn actually matters. Differentiated SLOs are how you focus reliability effort where it earns its keep."

**Key Point:** "SLOs follow business cost of failure, not uniformity — payments get four nines, internal tools get two, and that differentiation is what makes error budgets meaningful."

---

### Q12: Your SLO says 99.9% but users are complaining. What do you do?

**How to Answer:**

"First I trust the users, not the dashboard — if the SLI is green and users are unhappy, the SLI is measuring the wrong thing. This is the single most common SLO failure mode: measuring what we can, not what users feel.

I'd dig into what the complaints actually describe. Usually it's one of three things: the SLI averages away the pain (p50 latency is fine but p99 is terrible), it misses a whole population (mobile users on slow networks aren't in the measured path), or it measures the wrong event (API returns 200 but with empty data — technically successful, actually broken).

The fix is to change the SLI, and that's allowed — SLOs are hypotheses, not laws. I'd add the missing dimension, maybe a p99 latency SLI or a per-platform breakdown, and re-baseline the target. The embarrassing version of this is defending the dashboard while users leave. Never be that team."

**Key Point:** "Green SLO plus angry users means the SLI measures the wrong thing — trust the complaints, find the blind spot (p99, a platform, silent failures), and fix the SLI. SLOs are hypotheses."
