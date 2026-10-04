# DevOps Interview Preparation

*Deep-dive Q&A guides for DevOps & SRE interviews — written exactly how you should answer in the room.*

Every guide follows one format: **question → how to answer it out loud → code/commands → key point**. Practice reading the answers aloud until they sound natural.

## Subjects

| # | Subject | Guides |
|---|---------|--------|
| 1 | [Linux & System Administration](linux-system-admin/) | 6/6 |
| 2 | [Git & Version Control](version-control/) | 4/4 |
| 3 | [Core Concepts (DevOps/SRE)](core-concepts/) | 4/4 |
| 4 | [Containers (Docker)](containers/) | 5/5 |
| 5 | [Kubernetes](kubernetes/) | 3/8 |
| 6 | [CI/CD](ci-cd/) | ⬜ |
| 7 | [Infrastructure as Code](infrastructure-as-code/) | ⬜ |
| 8 | [Cloud (AWS)](cloud-aws/) | ⬜ |
| 9 | [Monitoring & Logging](monitoring-logging/) | ⬜ |
| 10 | [Networking & Security](networking-security/) | ⬜ |
| 11 | [Best Practices](best-practices/) | ⬜ |
| 12 | [Mock Interviews](mock-interviews/) | ⬜ |

## Publishing log

A new guide lands here every day. Progress so far:

- [x] 01 — [Linux Fundamentals for DevOps](linux-system-admin/01-linux-fundamentals-for-devops.md)
- [x] 02 — [Filesystem Hierarchy & Permissions](linux-system-admin/02-filesystem-hierarchy-permissions.md) (+ [hands-on lab](linux-system-admin/labs/02-filesystem-permissions-lab.md), [cheat sheet](linux-system-admin/cheat-sheets/02-filesystem-permissions-cheatsheet.md))
- [x] 03 — [Process Management & systemd](linux-system-admin/03-process-management-systemd.md) (+ [hands-on lab](linux-system-admin/labs/03-process-management-systemd-lab.md), [cheat sheet](linux-system-admin/cheat-sheets/03-process-management-systemd-cheatsheet.md))
- [x] 04 — [Networking on Linux](linux-system-admin/04-networking-on-linux.md) (+ [hands-on lab](linux-system-admin/labs/04-networking-on-linux-lab.md), [cheat sheet](linux-system-admin/cheat-sheets/04-networking-on-linux-cheatsheet.md))
- [x] 05 — [Bash Scripting for DevOps](linux-system-admin/05-bash-scripting-for-devops.md) (+ [hands-on lab](linux-system-admin/labs/05-bash-scripting-for-devops-lab.md), [cheat sheet](linux-system-admin/cheat-sheets/05-bash-scripting-for-devops-cheatsheet.md))
- [x] 06 — [Text Processing & Log Analysis](linux-system-admin/06-text-processing-log-analysis.md) (+ [hands-on lab](linux-system-admin/labs/06-text-processing-log-analysis-lab.md), [cheat sheet](linux-system-admin/cheat-sheets/06-text-processing-log-analysis-cheatsheet.md))
- [x] 07 — [Git Fundamentals & Daily Workflow](version-control/07-git-fundamentals-daily-workflow.md) (+ [hands-on lab](version-control/labs/07-git-fundamentals-lab.md), [cheat sheet](version-control/cheat-sheets/07-git-fundamentals-cheatsheet.md))
- [x] 08 — [Branching Strategies](version-control/08-git-branching-strategies.md) (+ [hands-on lab](version-control/labs/08-git-branching-strategies-lab.md), [cheat sheet](version-control/cheat-sheets/08-git-branching-strategies-cheatsheet.md))
- [x] 09 — [Git Internals](version-control/09-git-internals.md) (+ [hands-on lab](version-control/labs/09-git-internals-lab.md), [cheat sheet](version-control/cheat-sheets/09-git-internals-cheatsheet.md))
- [x] 10 — [Advanced Git](version-control/10-advanced-git.md) (+ [hands-on lab](version-control/labs/10-advanced-git-lab.md), [cheat sheet](version-control/cheat-sheets/10-advanced-git-cheatsheet.md))
- [x] 11 — [DevOps & SRE Fundamentals](core-concepts/11-devops-sre-fundamentals.md) (+ [hands-on lab](core-concepts/labs/11-devops-sre-fundamentals-lab.md), [cheat sheet](core-concepts/cheat-sheets/11-devops-sre-fundamentals-cheatsheet.md))
- [x] 12 — [SLI, SLO, SLA & Error Budgets](core-concepts/12-sli-slo-sla-error-budgets.md) (+ [hands-on lab](core-concepts/labs/12-sli-slo-sla-error-budgets-lab.md), [cheat sheet](core-concepts/cheat-sheets/12-sli-slo-sla-error-budgets-cheatsheet.md))
- [x] 13 — [DORA Metrics & Platform Engineering](core-concepts/13-dora-metrics-and-platform-engineering.md) (+ [hands-on lab](core-concepts/labs/13-dora-metrics-and-platform-engineering-lab.md), [cheat sheet](core-concepts/cheat-sheets/13-dora-metrics-and-platform-engineering-cheatsheet.md))
- [x] 14 — [12-Factor App Methodology](core-concepts/14-twelve-factor-app-methodology.md) (+ [hands-on lab](core-concepts/labs/14-twelve-factor-app-methodology-lab.md), [cheat sheet](core-concepts/cheat-sheets/14-twelve-factor-app-methodology-cheatsheet.md))
- [x] 15 — [Docker Deep Dive](containers/15-docker-deep-dive.md) (+ [hands-on lab](containers/labs/15-docker-deep-dive-lab.md), [cheat sheet](containers/cheat-sheets/15-docker-deep-dive-cheatsheet.md))
- [x] 16 — [Dockerfile Best Practices & Multi-Stage Builds](containers/16-dockerfile-best-practices.md) (+ [hands-on lab](containers/labs/16-dockerfile-best-practices-lab.md), [cheat sheet](containers/cheat-sheets/16-dockerfile-best-practices-cheatsheet.md))
- [x] 17 — [Docker Networking](containers/17-docker-networking.md) (+ [hands-on lab](containers/labs/17-docker-networking-lab.md), [cheat sheet](containers/cheat-sheets/17-docker-networking-cheatsheet.md))
- [x] 18 — [Docker Storage & Volumes](containers/18-docker-storage-volumes.md) (+ [hands-on lab](containers/labs/18-docker-storage-volumes-lab.md), [cheat sheet](containers/cheat-sheets/18-docker-storage-volumes-cheatsheet.md))
- [x] 19 — [Docker Compose](containers/19-docker-compose.md) (+ [hands-on lab](containers/labs/19-docker-compose-lab.md), [cheat sheet](containers/cheat-sheets/19-docker-compose-cheatsheet.md))
- [x] 20 — [Kubernetes Architecture](kubernetes/20-kubernetes-architecture.md) (+ [hands-on lab](kubernetes/labs/20-kubernetes-architecture-lab.md), [cheat sheet](kubernetes/cheat-sheets/20-kubernetes-architecture-cheatsheet.md))
- [x] 21 — [Pods, ReplicaSets & Workload Controllers](kubernetes/21-pods-replicasets-workload-controllers.md) (+ [hands-on lab](kubernetes/labs/21-pods-replicasets-workload-controllers-lab.md), [cheat sheet](kubernetes/cheat-sheets/21-pods-replicasets-workload-controllers-cheatsheet.md))
- [x] 22 — [Deployments and Rollout Strategies](kubernetes/22-deployments-rollout-strategies.md) (+ [hands-on lab](kubernetes/labs/22-deployments-rollout-strategies-lab.md), [cheat sheet](kubernetes/cheat-sheets/22-deployments-rollout-strategies-cheatsheet.md))

---

*Built for interview prep, one deep-dive at a time.*
