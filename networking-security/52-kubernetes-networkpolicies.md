# Kubernetes NetworkPolicies Interview Preparation Guide

*How to Answer Kubernetes NetworkPolicies Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why NetworkPolicies Matter](#why-networkpolicies-matter)
2. [Policy Anatomy and Selectors](#policy-anatomy-and-selectors)
3. [Ingress Rules in Practice](#ingress-rules-in-practice)
4. [Egress Rules and DNS](#egress-rules-and-dns)
5. [Enforcement Reality and Interview Gotchas](#enforcement-reality-and-interview-gotchas)

---

## Why NetworkPolicies Matter

### Q1: What is a NetworkPolicy and why should I care about it?

**How to Answer:**

"A NetworkPolicy is a Kubernetes resource that controls which pods can talk to which other pods, and on which ports. By default, Kubernetes is a flat network — every pod can reach every other pod."

"That's convenient until it's terrifying. If one pod gets compromised, the attacker can scan the whole cluster. I've seen a rogue pod in a dev namespace probe prod Redis because nothing stopped it."

"NetworkPolicies give you micro-segmentation without sidecars or service meshes. It's the cheapest zero-trust control in the cluster."

**Key Point:** "Default K8s networking is flat — NetworkPolicies add the segmentation that keeps one compromised pod from reaching everything."

---

### Q2: How is a NetworkPolicy different from a Service?

**How to Answer:**

"They solve opposite problems. A Service gives pods a stable address to *receive* traffic — it says nothing about who is allowed to call."

"A NetworkPolicy is a firewall rule attached to pods. It decides who can send traffic in and where traffic can go out, on which ports."

"The confusion comes because both involve labels and ports. I remember it as: Services route traffic, NetworkPolicies restrict it."

**Key Point:** "Services route traffic; NetworkPolicies restrict it. One gives an address, the other enforces who may use it."

---

### Q3: What happens when a namespace has no NetworkPolicies at all?

**How to Answer:**

"Everything talks to everything. That's the Kubernetes default — the CNI sets up a flat L3 network with no isolation, and no policy object means no filtering anywhere."

"A lot of teams assume namespaces give isolation. They don't — namespaces organize objects, but traffic still flows freely between them unless a policy says otherwise."

"I'd flag this in a cluster review immediately. A compromised frontend pod phoning every database in the cluster is a normal-day scenario without policies."

**Key Point:** "No policies = no filtering. Namespaces are organization, not isolation."

---

## Policy Anatomy and Selectors

### Q4: Walk me through the structure of a basic ingress NetworkPolicy.

**How to Answer:**

"A NetworkPolicy has four moving parts: podSelector picks which pods the rule applies to, policyTypes declares ingress or egress, ingress lists the allowed sources and ports, and egress lists allowed destinations."

"The selector is the anchor — traffic is either allowed by a matching rule or dropped. There's no 'deny all then allow' boilerplate because isolation is automatic once a policy selects the pod."

"Here's the minimal shape I'd draw on a whiteboard:"

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: frontend } }
      ports:
        - port: 8080
```

**Key Point:** "podSelector + policyTypes + from/to rules. Match it and you're allowed; miss it and you're isolated."

---

### Q5: How do podSelector, namespaceSelector, and ipBlock differ in rules?

**How to Answer:**

"They're three ways to name who's talking. podSelector matches pods in the *same* namespace as the policy — that's the common case for app-to-app traffic."

"namespaceSelector matches pods across namespaces, usually combined with a podSelector so it's 'these pods in those namespaces'. And ipBlock matches raw CIDRs for things outside the cluster, like a corporate office IP."

"I use podSelector for app tiers, namespaceSelector for cross-team traffic like monitoring scraping metrics, and ipBlock only for external sources — like letting the office VPN hit a staging admin port."

**Key Point:** "podSelector for same-namespace, namespaceSelector for cross-namespace, ipBlock for traffic from outside the cluster."

---

### Q6: What does it mean when podSelector is empty?

**How to Answer:**

"An empty podSelector — just `podSelector: {}` — selects *every* pod in the namespace. It's how you write a default-deny-all policy."

"That's the standard first policy people apply: isolate the namespace, then open holes deliberately. Without it, each policy only isolates the pods it selects, and everything else stays open."

"I'd phrase it in the interview like: empty selector is the big red 'do not talk' button for the whole namespace."

```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

**Key Point:** "Empty podSelector = all pods in the namespace. That's your default-deny."

---

### Q7: Are NetworkPolicies additive or does one override another?

**How to Answer:**

"They're additive — union semantics. If any policy selects a pod and allows the traffic, the traffic flows. You can't write a deny rule that overrides an allow."

"This trips people up because firewalls usually have ordered deny rules. NetworkPolicies don't: once a pod is selected by a policy, it's isolated to exactly the union of all allowed rules from every policy selecting it."

"The practical effect: write policies per concern — one for app ingress, one for monitoring, one for egress — and they'll stack correctly. Just don't expect to carve out exceptions with a deny."

**Key Point:** "Union semantics — every selecting policy adds allowed traffic. There's no deny override."

---

*(Part 2 continues: ingress rules in practice, egress and DNS, enforcement reality.)*
