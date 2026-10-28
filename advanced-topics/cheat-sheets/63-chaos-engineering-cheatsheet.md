# Chaos Engineering in Practice — Cheat Sheet

One page. Read it in the cab on the way to the interview.

## Core picture

| Piece | Job |
|-------|-----|
| Chaos engineering | Controlled, deliberate failure to prove recovery — on your schedule |
| Steady state | User-facing metric (checkout success, p99, error rate) watched live |
| Hypothesis | "If X fails, steady state holds because Y" — stated before breaking anything |
| Blast radius | How much you break: one pod, one namespace, one zone — never everything first |
| Game day | Scheduled team rehearsal of a realistic failure; debrief is the deliverable |

## Experiment design (every time)

1. Define steady state in one sentence
2. Write the hypothesis in one sentence
3. Pick the smallest blast radius that tests it
4. Set an auto-abort threshold on the steady-state metric
5. Announce it, run it, compare before vs after
6. End with confidence or a ticket — never neither

## Tools at a glance

- **Chaos Mesh:** K8s-native, YAML CRDs (PodChaos, NetworkChaos, StressChaos), GitOps-friendly
- **LitmusChaos:** Big pre-built experiment hub + UI, good for teams starting out
- **Gremlin:** Managed, works on VMs and bare metal too, paid
- **AWS FIS:** Fault injection at the AWS layer (EC2, API throttling)

## Blast-radius controls

- Scope: one pod → one namespace → one zone → one region (in that order)
- Time: minutes, fixed duration, never open-ended
- Auto-abort: metric threshold stops the experiment without a human noticing
- Kill switch: a human who can halt it instantly, and a channel where it was announced

## What latency experiments expose

- Retry storms: retries amplify slowness into a full slowdown
- Fix is usually a tighter timeout or a circuit breaker, not more retries
- Watch p99, not averages — averages hide the users who suffered

## Measuring the program

- Weaknesses found and fixed (tickets closed)
- Time to detect and recover in game days, trending down
- Real incidents survived because that exact scenario was tested
- Never: number of experiments run (vanity metric)

## Traps interviewers check

- No hypothesis = vandalism, not chaos engineering
- Staging proves the experiment is safe; only prod proves the system
- Surprise prod experiments destroy trust and get the program banned
- Unfixed findings are worse than no findings — they become embarrassing real outages
- Starting too big (whole region day one) instead of one pod
