# Jenkins Interview Preparation Guide

*How to Answer Jenkins Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Pipeline Basics](#pipeline-basics)
2. [Agents and Executors](#agents-and-executors)
3. [Credentials and Security](#credentials-and-security)
4. [Shared Libraries](#shared-libraries)
5. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

## Pipeline Basics

### Q1: What's the difference between a declarative and a scripted pipeline?

**How to Answer:**

"Declarative is the opinionated, structured syntax — `pipeline`, `agent`, `stages`, `steps`. It validates the syntax upfront and gives you better error messages when you mess up. Scripted is raw Groovy — way more flexible, but also way easier to write something nobody can read. I'd start every new pipeline in declarative and only drop to scripted blocks for something declarative can't express. In interviews they usually want to hear that declarative is the default choice now."

```groovy
pipeline {
    agent any
    stages {
        stage('Build') { steps { sh 'make build' } }
    }
}
```

**Key Point:** "Declarative is structured and validated; scripted is raw Groovy power you use sparingly."

---

### Q2: What is a Jenkinsfile, and how does a multibranch pipeline use it?

**How to Answer:**

"A Jenkinsfile is just pipeline-as-code — it lives in the repo root next to the code it builds. Any change to the build process goes through a normal pull request, reviewed like any other code. A multibranch pipeline scans the repo and creates a job per branch that has a Jenkinsfile, so feature branches get tested automatically. When the branch is deleted, the job gets cleaned up too. It basically killed the old 'one job per branch, configured by clicking' workflow."

**Key Point:** "Jenkinsfile = pipeline as code in the repo; multibranch auto-creates jobs per branch."

---

### Q3: How do stages, steps, and post sections work together?

**How to Answer:**

"A pipeline is made of stages — logical chunks like Build, Test, Deploy. Each stage contains steps, which are the actual commands: `sh`, `git`, `echo`. The `post` block runs after the pipeline or a stage finishes, based on conditions like `always`, `success`, or `failure`. I use `post` for cleanup and notifications — always archive artifacts, notify on failure. One gotcha: `post` failure handlers only see the first error, so if two stages fail you debug the first one."

**Key Point:** "Stages group steps; post handles cleanup and notifications based on the build outcome."

---

## Agents and Executors

### Q4: How do agents, nodes, and executors fit together?

**How to Answer:**

"The controller is the brain — it schedules work but ideally never builds anything itself. Agents are the workers that actually run builds, each with one or more executors, and each executor runs one build at a time. You assign work with labels — `agent { label 'docker-linux' }` — and Jenkins picks an agent carrying that label. The big rule: never run builds on the controller. If a build eats all the memory on the controller, the whole Jenkins goes down."

**Key Point:** "Controller schedules, agents build, labels route work — keep builds off the controller."

---

### Q5: When would you use ephemeral agents instead of static ones?

**How to Answer:**

"Static agents are permanent machines — simple, but you end up maintaining pets that drift out of sync and collect junk. Ephemeral agents spin up per build — Docker containers or Kubernetes pods — and die when the build finishes. Every build starts from a clean, known state, which kills the 'it worked yesterday' mystery. The tradeoff is spin-up time and the infrastructure to host them. For most teams today, Kubernetes pods via the Kubernetes plugin are the default — scalable and clean."

```groovy
agent {
    kubernetes {
        yaml '''
        spec:
          containers:
          - name: maven
            image: maven:3.9
            command: ['cat']
        '''
    }
}
```

**Key Point:** "Ephemeral agents give every build a clean slate; static agents become pets that drift."

---

### Q6: What's the difference between `agent any`, `agent none`, and `agent { label ... }`?

**How to Answer:**

"`agent any` means Jenkins picks any available executor — fine for simple pipelines, but you have no control over the environment. `agent none` at the top means no agent is allocated until a stage declares one — useful when stages need different environments, like Linux for build and a Mac for iOS signing. `agent { label 'gpu' }` pins the pipeline or stage to agents with that label. I'd say `agent none` plus per-stage agents is the cleanest pattern for multi-environment pipelines."

**Key Point:** "any = whatever's free, none = decide per stage, label = pin to the right environment."

---

## Credentials and Security

### Q7: How do you handle secrets in a Jenkins pipeline?

**How to Answer:**

"You store them in Jenkins' credentials store — never hardcoded in the Jenkinsfile, never in a shell variable that gets echoed. Then you bind them with `withCredentials` so they only exist inside that block. Jenkins automatically masks them in console output, so even if a command prints the secret it shows as `****`. Different types exist for different jobs — username/password, secret text, SSH keys, and files. The rule is simple: if it's a secret, it comes from the store, not from the repo."

```groovy
withCredentials([string(credentialsId: 'prod-api-key', variable: 'API_KEY')]) {
    sh 'curl -H "Authorization: Bearer $API_KEY" https://api.example.com/deploy'
}
```

**Key Point:** "Store secrets in Jenkins credentials, bind them with withCredentials, never hardcode them."

---

### Q8: What's the biggest Jenkins security mistake you've seen?

**How to Answer:**

"Running builds on the controller with full admin access, honestly. Any job running on the controller can read every credential in the store — it's a one-liner in Groovy to dump them. Second biggest: overly broad credentials scope — one global Docker Hub token shared across fifty pipelines instead of per-team scoped credentials. And folder-scoped credentials help a lot here: a team's credentials live in their folder, not visible to everyone. Least privilege applies to Jenkins exactly like it applies to IAM."

**Key Point:** "Builds on the controller can read all credentials; scope credentials tightly with folders."

---

## Shared Libraries

### Q9: What is a Jenkins shared library and why would you use one?

**How to Answer:**

"A shared library is reusable pipeline code pulled from a separate Git repo — common steps like 'build a Docker image and push it' that fifty pipelines all need. You write a custom step once in `vars/`, then every pipeline calls `dockerBuildPush()` instead of copy-pasting thirty lines of shell. The `vars/` directory holds the custom steps anyone can call; `src/` holds plain Groovy classes for the actual logic. Without one, every pipeline team reinvents the same helpers and they all drift apart over time."

**Key Point:** "Shared libraries turn copy-pasted pipeline steps into one reusable, versioned function."

---

### Q10: How do you version and safely roll out shared library changes?

**How to Answer:**

"You pin the library version in the pipeline — `@Library('my-lib@1.4.2')` — so a bad library change doesn't break every pipeline at once. New versions roll out by bumping the pin pipeline by pipeline, same as any dependency upgrade. For testing, most teams load the library from a branch in a test pipeline before merging to main. There's also the replay feature, which lets you rerun a build with modified pipeline code — handy for quickly testing a library fix without committing. The trap is pointing everything at `master` — then one bad merge breaks fifty builds simultaneously."

**Key Point:** "Pin library versions per pipeline and roll out changes like dependency upgrades."

---

## Troubleshooting and Best Practices

### Q11: A build is stuck in the queue forever. How do you debug it?

**How to Answer:**

"First I check the build's console and the queue page — Jenkins usually tells you the reason, like 'waiting for next available executor on label X'. Nine times out of ten it's a label mismatch: the pipeline asks for `linux-arm` and no agent has that label. Then I check if agents are actually connected — an agent that went offline silently leaves builds hanging. If executors are all busy, either the pool is too small or a stuck build is holding them — the build queue widget and the executor list on each node show you exactly that. Last resort, I check the controller logs for scheduling errors."

**Key Point:** "Stuck queues are usually label mismatches, offline agents, or executors held by hung builds."

---

### Q12: How do you make pipelines fast and reliable?

**How to Answer:**

"Fail fast — run the quick checks first so a broken commit dies in two minutes, not forty. Cache aggressively: dependency caches, Docker layer caches, anything that doesn't change between builds. Parallelize independent stages — unit tests and linting don't need to wait for each other. And make every stage rerunnable independently so a flaky test failure doesn't force a full rebuild. I also archive artifacts and test reports in `post` so failures are diagnosable without rerunning. Speed matters because a slow pipeline is one developers stop trusting and start bypassing."

```groovy
stage('Checks') {
    parallel {
        stage('Unit Tests') { steps { sh 'make test' } }
        stage('Lint')      { steps { sh 'make lint' } }
    }
}
```

**Key Point:** "Fail fast, cache everything, parallelize independent stages, keep stages independently rerunnable."

---

### Q13: What's the difference between `input`, `timeout`, and `retry` in a pipeline?

**How to Answer:**

"`input` pauses the pipeline and waits for a human to approve — that's your manual gate before production. `timeout` kills a stage or the whole build if it runs too long, which saves you from hung builds eating executors forever. `retry` reruns a flaky step a few times before giving up — useful for network calls, but dangerous if you use it to paper over real bugs. I wrap `input` in a `timeout` too, otherwise an approval that nobody clicks blocks the executor indefinitely. The combination is: retry the flaky bits, timeout everything, and gate prod with input."

**Key Point:** "input gates on humans, timeout kills hangs, retry handles flakes — wrap input in timeout."
