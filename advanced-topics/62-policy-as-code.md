# Policy as Code (OPA & Kyverno) Interview Preparation Guide

*How to Answer Policy as Code Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Policy as Code](#why-policy-as-code)
2. [OPA & Rego](#opa--rego)
3. [Kyverno in Kubernetes](#kyverno-in-kubernetes)
4. [Enforcement, Exceptions & Debugging](#enforcement-exceptions--debugging)

---

## Why Policy as Code

### Q1: What is policy as code, in one simple answer?

**How to Answer:**

"I think of policy as code as turning 'you must not do X' from a wiki page into a rule that a machine actually enforces. Instead of trusting every developer to remember that images must come from our registry, I write that rule once, store it in Git, and let an admission controller reject anything that breaks it. The policy gets reviewed, versioned, and tested like any other code. So compliance stops being a hope and becomes a gate that runs on every deploy."

**Key Point:** "Policy as code = rules written in code, stored in Git, enforced automatically on every deploy — not a wiki page people forget."

---

### Q2: Why isn't RBAC and code review enough — why do we need a policy engine at all?

**How to Answer:**

"RBAC answers 'who can do what,' not 'is this workload safe to run.' A developer with full namespace access can still deploy an image with root privileges or the latest tag, and RBAC won't blink. Code review catches some of it, but reviewers get tired and YAML diffs are easy to skim past. A policy engine checks the actual object at admission time, every single time, with no fatigue. I use RBAC for access, and policy for the shape of what's allowed through that access."

**Key Point:** "RBAC controls who can deploy; policy controls what they're allowed to deploy — you need both, they answer different questions."

---

### Q3: OPA vs Kyverno — how do you decide which one to use?

**How to Answer:**

"I pick Kyverno when the team lives in YAML and wants policies fast — its rules are Kubernetes manifests, so anyone who writes a Deployment can write a policy. I pick OPA when I need one engine across everything, not just Kubernetes — Terraform plans, CI pipelines, API gateways, the lot. OPA's Rego is more powerful but it's a new language to learn. In practice I often see both: Kyverno for cluster admission, OPA for the wider platform. The interview answer is: Kyverno for Kubernetes-only speed, OPA for cross-platform depth."

**Key Point:** "Kyverno = YAML-native, Kubernetes-only, fastest to adopt. OPA = Rego, one engine for every system, steeper learning curve."

---

## OPA & Rego

### Q4: How does OPA actually work — what is it checking, and when?

**How to Answer:**

"OPA is a small decision engine: you feed it some input data and a policy, and it answers allow or deny. In Kubernetes it usually runs as Gatekeeper, which is an admission webhook — the API server sends every create or update request to it before the object is saved. OPA evaluates the Rego policy against that request and returns a decision. If it denies, the API server rejects the request and the user sees the violation message. Nothing is stored that broke the rule, because the object never got created."

**Key Point:** "OPA decides allow or deny on input data; in Kubernetes, Gatekeeper runs it as an admission webhook before anything is persisted."

---

### Q5: Walk me through a real Rego policy — say, only allowing images from our registry.

**How to Answer:**

"I write a rule that says: deny any container whose image doesn't start with our registry prefix. Rego reads like a query — I loop over the containers in the incoming Pod spec and check each image string. If one fails the prefix test, the deny rule produces a message naming that container. I keep the message human-readable, because the developer who sees it at 2 AM needs to know exactly what to fix. The whole policy is about ten lines, and I test it with opa test before it ever touches a cluster."

```rego
package kubernetes.admission

deny[msg] {
  container := input.request.object.spec.containers[_]
  not startswith(container.image, "registry.example.com/")
  msg := sprintf("image %v must come from registry.example.com/", [container.image])
}
```

**Key Point:** "A Rego deny rule loops over containers, checks the image prefix, and returns a message the developer can act on — test it with opa test first."

---

### Q6: What's the difference between Gatekeeper's ConstraintTemplates and Constraints?

**How to Answer:**

"I think of a ConstraintTemplate as the reusable rule and a Constraint as one instance of it with real parameters. The template holds the Rego and says 'this policy takes a list of allowed registries.' The Constraint then says 'for this cluster, the allowed list is registry.example.com.' That split matters because one template serves many teams — each team writes its own Constraint with its own values, without touching Rego. It also means updating the logic in one place rolls out to every Constraint built on it."

**Key Point:** "Template = the reusable Rego logic. Constraint = that logic plus real parameters for one scope. One template, many Constraints."

---

## Kyverno in Kubernetes

### Q7: What is Kyverno, and why do teams pick it over OPA for Kubernetes?

**How to Answer:**

"Kyverno is a Kubernetes-native policy engine where policies are just YAML manifests — ClusterPolicy objects you kubectl apply like anything else. That lowers the barrier a lot: the team doesn't learn Rego, they write rules in the same YAML they already know. It also has things OPA doesn't do out of the box, like mutating defaults — automatically adding a label or a resource limit when a Pod is created. The tradeoff is scope: Kyverno only speaks Kubernetes, while OPA works everywhere. For a team that only needs cluster guardrails, Kyverno gets them there faster."

**Key Point:** "Kyverno policies are plain YAML ClusterPolicies — no new language — plus mutation and generation that OPA doesn't ship natively."

---

### Q8: Walk me through a Kyverno policy — block the latest tag and require a team label.

**How to Answer:**

"I write a ClusterPolicy with two validate rules: one that rejects any container image ending in :latest, and one that requires the label team on every Pod. The rule matches on Pods, and the validation uses a JMESPath expression to check the fields. When a developer applies a Pod without the label, the API server rejects it with the message I wrote in the policy — something like 'the team label is required.' I keep messages specific, because a vague denial just creates a support ticket. The whole thing is one YAML file in Git, reviewed like any other change."

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-team-label }
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-team-label
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "Every Pod needs the team label so we know who owns it."
        pattern:
          metadata:
            labels:
              team: "?*"
```

**Key Point:** "A Kyverno validate rule matches a kind, checks a pattern, and returns your message — one YAML file, reviewed in Git like any change."

---

### Q9: Kyverno has validate, mutate, and generate rules — when do you use each?

**How to Answer:**

"I use validate to block bad things — no latest tags, no missing labels, no hostPath. I use mutate to fix things automatically instead of blocking — add a default resource limit, inject a sidecar, set a label the team forgot. And I use generate to create supporting objects when something is created — like auto-creating a NetworkPolicy whenever a new namespace appears. The mental model is simple: validate says no, mutate says 'let me fix that for you,' generate says 'and here's the thing that should come with it.' Most real policies are validate; mutate and generate are for toil you want to disappear."

**Key Point:** "Validate blocks, mutate fixes defaults in place, generate creates companion objects — validate is the workhorse, the other two remove toil."

---

## Enforcement, Exceptions & Debugging

### Q10: How do you roll out a new policy without breaking every deploy on day one?

**How to Answer:**

"I never start in Enforce mode. I start in Audit — or Kyverno's Audit mode, or Gatekeeper's dryrun — which logs violations without blocking anything. I watch the audit logs for a week, find out which teams would break, and fix the workloads or the policy before anyone gets blocked. Then I flip to Enforce, usually per-namespace, starting with non-production. The interview trap here is saying you just turn policies on — that's how you block a Friday deploy and lose trust in the whole program. Audit first, enforce gradually, and the rollout is boring, which is the point."

**Key Point:** "Audit first, watch violations for days, fix workloads, then flip to Enforce namespace by namespace — never enforce a new policy on day one."

---

### Q11: How do you handle exceptions — a team that genuinely needs to break a rule?

**How to Answer:**

"I build exceptions into the policy itself, not as a side channel. In Kyverno I use an exclude block for a namespace or a workload, with a comment and an expiry mindset — exceptions get reviewed on a schedule, not granted forever. In Gatekeeper I use the Constraint's exemptions or a namespaceSelector that skips a labelled namespace. What I never do is tell a team to just delete the policy or give them cluster-admin to bypass it. A visible, time-boxed exception keeps the guardrail honest; a secret bypass kills it."

**Key Point:** "Exceptions live inside the policy as excludes or exemptions, reviewed on a schedule — never a secret bypass or a deleted policy."

---

### Q12: A deployment just got rejected with a policy error — how do you debug it?

**How to Answer:**

"I start with the error message itself, which names the policy and the rule that fired — that's why good messages matter. Then I check which policy it is: kubectl get clusterpolicy for Kyverno, or the Constraint for Gatekeeper, and read the rule to see what it actually checks. If the workload is genuinely wrong, I fix the YAML. If the policy is wrong — say it blocks a legitimate image — I fix the policy in Git and it rolls out like any change. The thing I never do is bypass the webhook by hand-editing, because then the next deploy fails again and nobody learned anything."

```bash
kubectl get clusterpolicy                          # list Kyverno policies
kubectl describe clusterpolicy require-team-label  # read the rule that fired
kubectl get events --field-selector reason=PolicyViolation
```

**Key Point:** "Read the denial message, read the policy that fired, fix the workload or fix the policy in Git — never bypass the webhook by hand."

---

