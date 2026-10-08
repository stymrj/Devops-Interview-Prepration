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

