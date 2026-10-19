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
## Ingress Rules in Practice

### Q8: How would you lock down an API so only the frontend can reach it?

**How to Answer:**

"I'd select the API pods and allow ingress only from pods labeled frontend, on the API's port. Everything else — including other namespaces — gets dropped automatically."

"The shape is dead simple, and it's the pattern I reach for on every new service:"

```yaml
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: frontend } }
      ports: [{ port: 8080 }]
```

"One detail people miss: once any policy selects the API pods, ingress from *everyone else* is denied — including kube-system probes unless you add them. Health checks breaking after a policy deploy is the classic symptom."

**Key Point:** "Allow the known callers and the isolation does the rest — but remember to allow your own health checks."

---

### Q9: How do you allow traffic from another namespace, like monitoring?

**How to Answer:**

"I'd combine a namespaceSelector with a podSelector inside the same rule — 'pods labeled prometheus in the namespace labeled monitoring'. The selectors AND together, so both have to match."

"This is the safe way to give the monitoring team scraping access without opening the app to the whole monitoring namespace. I've also used it for a logging daemonset scraping every namespace."

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels: { team: monitoring }
        podSelector:
          matchLabels: { app: prometheus }
    ports: [{ port: 9090 }]
```

"The gotcha: namespaceSelector matches on the *namespace's* labels, so the namespace itself must be labeled — I label every namespace with `team` or `env` at creation for exactly this reason."

**Key Point:** "namespaceSelector AND podSelector pin down exact sources. Label your namespaces at creation time."

---

### Q10: Can you do allow-listing by port ranges or protocols?

**How to Answer:**

"Ports are explicit — you list numbers, optionally with a protocol. There's no port-range syntax in the standard API, which is honestly fine because listing the three ports your app uses is more readable anyway."

"For protocols, `protocol: TCP` is the default; you'd set UDP for things like DNS or a QUIC service. EndPort exists for ranges but needs protocol set and is rarely needed."

"I'd tell the interviewer my rule of thumb: one rule entry per caller-per-port-group. It's verbose but it reads like documentation of your architecture."

**Key Point:** "List exact ports, set protocol explicitly for UDP. Verbose rules read like architecture docs."

---

## Egress Rules and DNS

### Q11: Why do egress policies break DNS if you're not careful?

**How to Answer:**

"This is the most famous NetworkPolicy footgun. You lock down egress to 'only the database', and suddenly your app can't resolve *any* hostname because CoreDNS lives in kube-system on port 53."

"CoreDNS is a normal pod behind a service. Once egress is restricted, the DNS lookup never leaves the app's network namespace. The app fails with name-resolution errors while the database rule looks perfectly fine."

"The fix is a two-rule egress: allow UDP+TCP port 53 to the kube-dns service, then allow your actual destinations. I keep this pattern as a template because I will forget it otherwise:"

```yaml
egress:
  - to:
      - namespaceSelector: { matchLabels: { name: kube-system } }
        podSelector: { matchLabels: { k8s-app: kube-dns } }
    ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP
  - to:
      - podSelector: { matchLabels: { app: database } }
    ports: [{ port: 5432 }]
```

**Key Point:** "Egress policies isolate DNS too — always allow port 53 to CoreDNS or everything breaks."

---

### Q12: How would you prevent pods from exfiltrating data to the internet?

**How to Answer:**

"I'd write an egress policy that allows only the cluster-internal destinations the app needs — database, cache, service endpoints — and nothing else. No ipBlock, no 0.0.0.0/0, so external traffic simply has no allowed path."

"This is the practical data-exfiltration control: a compromised app can't phone a random external IP because the CNI drops the packets. Combine it with an image-policy so attackers can't pull tooling in either."

"The nuance: this stops L3 exfiltration only. If the app legitimately needs outbound HTTPS — for an API or S3 — you have to allow those CIDRs or use a proper egress proxy with FQDN filtering, which plain NetworkPolicies can't express."

**Key Point:** "Allow-list egress to internal destinations only. For real FQDN-level control, layer an egress proxy or service mesh."

---

## Enforcement Reality and Interview Gotchas

### Q13: Does every Kubernetes cluster actually enforce NetworkPolicies?

**How to Answer:**

"No — and this is a favorite interview trap. NetworkPolicy is just an API object. Enforcement happens in the CNI plugin, and several popular CNIs don't implement it."

"Calico, Cilium, and Weave Net enforce them. Flannel, the default in a lot of tutorials and older kubeadm setups, does not. You can apply a hundred policies on Flannel and change nothing."

"Before claiming a cluster is segmented, I'd run a test: spin up two pods in different namespaces, apply a deny-all, and try to ping. The first time I did this on a Flannel cluster, the ping went straight through — very educational."

```bash
kubectl run a --image=busybox -- sleep 3600
kubectl run b --image=busybox -- sleep 3600
kubectl apply -f deny-all.yaml
kubectl exec a -- ping -c 2 $(kubectl get pod b -o jsonpath='{.status.podIP}')
```

**Key Point:** "Policies without an enforcing CNI are decorations. Calico and Cilium enforce; Flannel ignores them."

---

### Q14: NetworkPolicy applied but traffic still flows — how do you debug it?

**How to Answer:**

"I'd check three things in order. First, is the selector actually matching my pods — a typo in a label means the policy selects nothing and isolation never kicks in."

"Second, does the CNI enforce policies at all — Flannel, or a cloud CNI with policy enforcement disabled. Third, am I testing the right direction — I once debugged 'broken ingress' for twenty minutes before realizing I'd applied an egress policy."

"My fastest verification is a curl pod in the source namespace hitting the target's ClusterIP. Also `kubectl describe networkpolicy` shows which pods it selected, which usually reveals the label mismatch immediately."

**Key Point:** "Debug order: label match → CNI enforcement → direction (ingress vs egress). `describe` shows what got selected."

---

### Q15: When would you choose a service mesh or cloud security groups instead?

**How to Answer:**

"NetworkPolicies are L3/L4 only — IPs and ports. If I need identity-based rules, like 'only the payments service's TLS identity can call the ledger API', or L7 path filtering, that's service mesh territory."

"Similarly, traffic between the cluster and the outside world — office VPNs, RDS instances — is often better controlled at the cloud layer with security groups or VPC NACLs, because the IP is stable and the control is closer to the perimeter."

"My default stack: NetworkPolicies for pod-to-pod segmentation inside the cluster, mesh mTLS for service identity and L7 policy, cloud security groups for the cluster boundary. Each tool owns the layer it's good at."

**Key Point:** "NetworkPolicies own L3/L4 inside the cluster. Mesh owns identity and L7; cloud groups own the perimeter."
