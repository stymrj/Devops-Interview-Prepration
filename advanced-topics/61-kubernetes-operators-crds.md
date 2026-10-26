# Kubernetes Operators & CRDs Interview Preparation Guide

*How to Answer Kubernetes Operators & CRDs Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Operators Exist](#why-operators-exist)
2. [Custom Resources & CRDs](#custom-resources--crds)
3. [The Controller & Reconciliation Loop](#the-controller--reconciliation-loop)
4. [Building Operators](#building-operators)
5. [Production Details & Debugging](#production-details--debugging)

---

## Why Operators Exist

### Q1: What is a Kubernetes Operator, in one simple answer?

**How to Answer:**

"I think of an Operator as a human SRE's runbook turned into code that never sleeps. A Deployment knows how to keep pods running, but it doesn't know how to back up a database, fail over a primary, or upgrade a cluster safely. An Operator extends the Kubernetes API with a custom resource — say, a PostgresCluster — and a controller that watches it and drives the real world toward what you declared. You stop running the database by hand and start declaring what you want."

**Key Point:** "An Operator is operational knowledge in code: a custom resource to declare intent, plus a controller that keeps reality matching it."

---

### Q2: When does an app actually need an Operator, and when is it overkill?

**How to Answer:**

"I reach for an Operator when the app has state and lifecycle steps a Deployment can't express — backups, failover, sharding, upgrades that need ordering. A stateless web API almost never needs one; a Helm chart plus a Deployment is enough. The test I use is simple: if the runbook has steps that must happen in order and someone has to stay awake for them, that's an Operator. If it's just 'run three replicas,' it's overkill."

**Key Point:** "Stateful lifecycle steps in a strict order = Operator. Stateless replicas = Deployment and Helm, don't over-engineer it."

---

### Q3: Operator vs Helm chart vs plain controller — how do you tell them apart?

**How to Answer:**

"I see Helm as a templating tool: it renders YAML and installs it, then mostly steps back. A controller is the generic pattern — a loop that watches state and reconciles it. An Operator is a controller with domain knowledge, built around a custom resource for one specific application. So Helm gets you installed, but the Operator stays around managing day-2 operations like backups and failover. In practice I often use both: Helm to install the Operator, the Operator to run the app."

**Key Point:** "Helm installs, controllers reconcile, Operators reconcile with application-specific brains — they complement each other, they don't compete."

---

## Custom Resources & CRDs

### Q4: What is a CRD, and what does it give you that a ConfigMap can't?

**How to Answer:**

"A CRD, a CustomResourceDefinition, teaches the API server a brand-new resource type — its name, its schema, its versions. Once it's installed, I can kubectl get postgresclusters just like pods, with validation, RBAC, and watch events for free. A ConfigMap is just a bag of strings; nothing validates it, nothing watches it with intent, and no controller is obliged to act on it. A custom resource is a first-class API object with a contract. That's the whole difference."

**Key Point:** "A CRD makes your app a native Kubernetes citizen: validated, RBAC-controlled, watchable — a ConfigMap is just untyped data."

---

### Q5: Walk me through creating a CRD and using it.

**How to Answer:**

"I write a CRD YAML with the group, the kind, and a schema under versions — say group db.example.com, kind PostgresCluster, with spec fields like replicas and storage size. I apply it, and the API server starts serving that resource. Then I create an instance of it, a custom resource, declaring spec.replicas: 3. From that point kubectl works on it, and my controller watches it. The schema validation means a typo like replicas: 'three' gets rejected at the API server before my code ever sees it."

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: postgresclusters.db.example.com }
spec:
  group: db.example.com
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas: { type: integer, minimum: 1 }
```

**Key Point:** "CRD = group, kind, versioned schema. After that, your custom resource behaves like any built-in object — validated at the API server."

---

### Q6: What's the difference between spec and status, and why does it matter?

**How to Answer:**

"I treat spec as the user's wish and status as the controller's report card — and I never let them mix. The user writes spec.replicas: 3, and my controller writes status.phase: Ready and status.currentReplicas: 3. Keeping them separate means a user edit never fights a controller update, and kubectl shows real state instead of guesses. Interviewers love this because it reveals whether you understand the API contract. If a controller writes into spec, something is wrong."

**Key Point:** "Spec is desired state, status is observed state — the controller writes status only, so users and controllers never fight over the same fields."

---

## The Controller & Reconciliation Loop

### Q7: Explain the reconciliation loop like you'd explain it to a teammate.

**How to Answer:**

"The loop is dead simple and that's the point. Something changes — a PostgresCluster is created or a pod dies — and the API server sends my controller an event with the object's key. My controller reads the current state, compares it to the desired state in spec, and takes the smallest action that closes the gap. Then it requeues and checks again. It never assumes its last action worked; every pass re-reads reality. That's why controllers survive crashes: they just pick up where the world actually is."

**Key Point:** "Watch, compare desired vs actual, act, requeue — and never trust your last action, always re-read reality."

---

### Q8: What does 'level-based, not edge-based' actually mean in practice?

**How to Answer:**

"It means my controller reacts to the current state of the world, not to the event that happened. If the controller crashes mid-upgrade and restarts, it doesn't need to remember what happened — it looks at the cluster now and figures out the next step. An edge-based script that says 'on create, do X' breaks the moment it misses an event. Level-based design is why Kubernetes is self-healing, and it's the answer interviewers want when they ask why controllers are robust."

**Key Point:** "Level-based = act on current state, not on missed events. That's what makes a controller survive restarts without losing track."

---

### Q9: What are finalizers, and when do you need one?

**How to Answer:**

"I add a finalizer when deleting the custom resource must trigger cleanup outside Kubernetes — dropping a cloud database, deleting a backup bucket, deregistering a DNS record. The finalizer blocks deletion: the object sits in Terminating until my controller finishes cleanup and removes the finalizer. The trap is forgetting to remove it, which leaves the object stuck forever. So my rule is: finalizers only for external cleanup, and the cleanup code path gets tested like it's production code, because it is."

**Key Point:** "Finalizers gate deletion until external cleanup finishes — use them only for real outside-the-cluster work, and always test the removal path."

---

## Building Operators

### Q10: Kubebuilder vs Operator SDK vs writing a controller from scratch?

**How to Answer:**

"I start with Kubebuilder for most cases — it scaffolds the project, generates the CRD and RBAC manifests, and uses controller-runtime, which is the standard library everyone shares. Operator SDK is built on the same runtime and adds extras like Helm or Ansible-based operators, which I pick when the team isn't writing Go. Writing from scratch with raw client-go is a learning exercise, not a production choice. In an interview, saying Kubebuilder plus controller-runtime shows you know the ecosystem's default."

**Key Point:** "Kubebuilder is the default, Operator SDK adds Helm/Ansible flavours on the same runtime — raw client-go is for learning, not shipping."

---

### Q11: How do you handle versioning when your CRD schema has to change?

**How to Answer:**

"I version the API like any other API: v1alpha1, then v1beta1, then v1, with one storage version that etcd actually stores. When the shape changes — say a field gets renamed — I write a conversion webhook so old versions still work. The trap is changing the stored version casually, because every existing object has to convert. My rule is: additive changes are cheap, renames and removals need a conversion plan and a migration window. And I never serve two versions without deciding which one is storage."

**Key Point:** "One storage version, additive changes are free, renames need a conversion webhook — never change the stored shape without a migration plan."

---

## Production Details & Debugging

### Q12: My Operator's custom resource is stuck, or the app isn't converging — how do you debug it?

**How to Answer:**

"I start with the object itself: kubectl describe shows conditions and events, and status tells me what the controller last believed. Then I check the controller logs for the reconcile errors and whether it's requeueing with backoff. After that I look at RBAC — a missing permission on the child resource is the most common silent failure. If the object is stuck Terminating, it's almost always a finalizer that never got removed. I fix the controller, not the object: deleting things by hand just hides the bug."

```bash
kubectl describe postgrescluster mydb -n prod
kubectl logs -n operators deploy/postgres-operator | tail -50
kubectl get postgrescluster mydb -n prod -o yaml | grep -A5 finalizers
```

**Key Point:** "Describe the object, read controller logs, check RBAC, suspect the finalizer on stuck deletes — fix the controller, not the symptom."

---
