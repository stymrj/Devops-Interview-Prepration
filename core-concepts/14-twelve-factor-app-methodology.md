# 12-Factor App Methodology Interview Preparation Guide

*How to Answer 12-Factor App Methodology Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [What Is the 12-Factor App](#what-is-the-12-factor-app)
2. [Codebase Dependencies and Config](#codebase-dependencies-and-config)
3. [Backing Services and Build Release Run](#backing-services-and-build-release-run)
4. [Processes State and Concurrency](#processes-state-and-concurrency)
5. [Port Binding Disposability and Dev Prod Parity](#port-binding-disposability-and-dev-prod-parity)
6. [Interview Traps and Real World Calls](#interview-traps-and-real-world-calls)

---

## What Is the 12-Factor App

### Q1: What is the 12-factor app methodology, and why did it emerge?

**How to Answer:**

"The 12-factor app is a set of principles from Heroku, published in 2011, for building software-as-a-service apps that deploy cleanly and scale predictably. It came out of the pain of that era — snowflake servers, config baked into code, deployments that worked on one machine and nowhere else.

The twelve factors cover the whole lifecycle: how you version code, declare dependencies, handle config, treat backing services, run processes, and ship to production. It's basically a checklist for 'would this app survive on someone else's platform?'

I don't treat it as scripture — some factors have aged — but in interviews it's the shared language for talking about cloud-native design. If I say an app 'violates factor three,' every DevOps engineer in the room knows exactly what I mean."

**Key Point:** "12-factor is Heroku's 2011 playbook for building SaaS apps that deploy cleanly and scale predictably — not scripture, but the shared language DevOps engineers use to talk about cloud-native design."

---

### Q2: How would you summarize all twelve factors in under a minute?

**How to Answer:**

"Here's my sixty-second version: one codebase in version control with many deploys. Dependencies declared explicitly, never assumed. Config in the environment, never in code. Backing services treated as attached resources you can swap. Strict separation of build, release, and run stages. Processes stateless and share-nothing. Self-contained apps that bind their own ports. Concurrency via the process model — scale out, not up. Fast startup and graceful shutdown. Dev and prod kept as similar as possible. Logs as event streams, never files. Admin tasks run as one-off processes.

That summary is what interviewers actually want. They rarely ask you to recite the list — they ask how one factor changes a design decision. Knowing the map cold lets you navigate to the right factor fast."

**Key Point:** "Know the twelve-factor map cold — code, dependencies, config, services, build-release-run, stateless processes, port binding, concurrency, disposability, parity, logs, admin tasks — so you can navigate to the right factor under pressure."

---

## Codebase Dependencies and Config

### Q3: Factor I says one codebase, many deploys. What does that actually forbid?

**How to Answer:**

"It forbids having separate codebases per environment — no 'the prod branch is basically a different app' situations. One repo, tracked in version control, and every deploy is that same codebase plus different config.

It doesn't forbid monorepos. One repo containing many apps is fine as long as each app is deployable independently from the same codebase conceptually. What it kills is drift — the classic trap where staging runs code that's three weeks behind prod and testing means nothing.

In practice I enforce this with trunk-based development and deploy-from-main pipelines. If your staging environment needs its own branch to exist, you've already violated the factor."

**Key Point:** "One codebase means one repo, version-controlled, with every environment running the same code plus different config — separate branches per environment is the smell it forbids."

---

### Q4: Why must dependencies be declared explicitly and isolated? Isn't that obvious now?

**How to Answer:**

"Factor two says: never rely on implicit system tools or packages. Declare everything in a manifest — package.json, requirements.txt, go.mod — and isolate the app from the system with virtualenvs or containers. It seems obvious now because the factor won.

The pain it solved was 'works on my machine' — an app that depended on a system-wide libcurl version nobody pinned, or a Python package installed by hand on the server once in 2019. Declared and isolated means the deploy is reproducible from scratch.

Where I still see violations is containers built FROM latest with unpinned apt packages, or scripts that call system tools like jq without installing them. If the base image changes and your app breaks, your dependencies weren't really declared."

```dockerfile
# pin everything — the factor in one Dockerfile habit
FROM python:3.12-slim-bookworm
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

**Key Point:** "Declare every dependency in a manifest and isolate from the system — unpinned base images and hand-installed packages are the modern way this factor still gets violated."

---

### Q5: Why is storing config in the environment such a big deal?

**How to Answer:**

"Because config varies between deploys and code doesn't. Database URLs, API keys, feature flags — these change between dev, staging, and prod, while the codebase stays identical. If config lives in code, you can't promote the same artifact through environments.

Environment variables give you language-agnostic config without touching code or rebuilding the image. One container image runs everywhere; only the environment changes. That's the whole deployment model of Kubernetes, serverless, and PaaS.

The hard line: anything that varies between deploys is config, everything else is code. A common interview follow-up is 'is the app's timeout config or code?' — if you'd change it between staging and prod, it's config. Get it out of the repo."

```bash
# config varies per deploy, code doesn't — same image, different env
docker run -e DATABASE_URL="$PROD_DB" -e LOG_LEVEL=warn myapp:1.4.2
```

**Key Point:** "Config is anything that varies between deploys — it goes in the environment so one immutable artifact runs in every environment without code changes."

---

## Backing Services and Build Release Run

### Q6: What is a backing service, and why treat it as an attached resource?

**How to Answer:**

"A backing service is anything your app consumes over the network — databases, queues, caches, SMTP, object storage. The factor says treat them as attached resources: your app connects via a URL or credentials stored in config, with zero distinction between a local service and a third-party one.

The payoff is swap-ability. If your Postgres is just a DATABASE_URL, you can point it at a local container in dev, RDS in staging, and Aurora in prod — or switch vendors entirely — without changing code. The app never knows and never cares.

The trap I see in interviews is people hardcoding service discovery or baking in vendor SDKs everywhere. If swapping your queue from RabbitMQ to SQS requires a code change, you've coupled to a backing service. The URL-in-config rule is the test."

**Key Point:** "A backing service is any network resource your app consumes — treat it as a swappable URL in config, so switching databases or queues never requires a code change."

---

### Q7: What is the strict separation of build, release, and run stages?

**How to Answer:**

"Build turns your codebase into an executable bundle — compiling, installing dependencies, producing an image. Release combines that build with config to create a deployable unit. Run executes the release in an environment. The factor demands these three stages stay strictly separated.

The critical rule: builds happen once, releases are cheap, and you never modify a release to deploy it. If staging needs a fix, you build again from code — you don't SSH into the container and edit a file. The release gets a unique ID, like a git SHA or timestamp, so every deploy is traceable.

This is where CI/CD pipelines come from directly. Your pipeline's build job, the config injection at deploy time, and the rollout are this factor in action. Interviewers love asking 'why not build in production?' — because then builds aren't reproducible and your artifact depends on the machine it was built on."

**Key Point:** "Build once from code, combine with config to make a uniquely-identified release, run the release unchanged — never patch a running deployment, always rebuild from source."

---

### Q8: Why does it matter that a release can't be mutated?

**How to Answer:**

"Because a mutable release destroys the one guarantee deployments need: the thing you tested is the thing you shipped. If anyone can change a running release, staging results mean nothing — you tested artifact A and you're running artifact A-plus-someone's-hotfix.

Immutability also makes rollbacks trivial. Release 42 is broken? Run release 41 again. No archaeology, no 'wait, what changed since Tuesday.' That only works if releases are atomic, identified, and never edited in place.

Kubernetes nails this: a Deployment's pod template hash identifies the release, and rolling back is just pointing at the previous ReplicaSet. The factor is eleven years older than Kubernetes, but the mechanism is exactly what it describes."

**Key Point:** "Immutable releases mean what you tested is what you shipped — and rollbacks become trivial because the previous release still exists, unchanged, ready to run again."
