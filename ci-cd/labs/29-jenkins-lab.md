# Jenkins Hands-On Lab

*8 exercises to make Jenkins muscle memory — run them against a local Jenkins or a throwaway Docker container.*

## Setup

```bash
# Spin up a throwaway Jenkins (admin/admin, port 8080)
docker run -d --name jenkins-lab -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
docker logs jenkins-lab 2>&1 | grep -A2 "initialAdminPassword" || \
  docker exec jenkins-lab cat /var/jenkins_home/secrets/initialAdminPassword
```

---

## Exercise 1: Your first declarative pipeline

**Goal:** Create a freestyle-free pipeline job from a Jenkinsfile.

1. In Jenkins: New Item → Pipeline → name it `hello-pipeline`.
2. Under Pipeline, choose "Pipeline script" and paste:

```groovy
pipeline {
    agent any
    stages {
        stage('Greet') {
            steps {
                echo "Hello from ${env.BUILD_NUMBER}"
                sh 'whoami && pwd'
            }
        }
    }
}
```

3. Build Now. Open the console output.

**Expected output:** `Hello from 1` plus the agent's user and workspace path. Blue/green stage view shows one green stage.

**Why it matters:** This is the smallest possible pipeline — it proves your controller and at least one executor work before you add complexity.

---

## Exercise 2: Scripted vs declarative — feel the difference

**Goal:** Compare error messages between the two syntaxes.

1. Duplicate `hello-pipeline` as `hello-scripted`, switch to a scripted pipeline:

```groovy
node {
    stage('Greet') {
        echo "Hello from scripted"
        sh 'whoami'
    }
}
```

2. Build it — works.
3. Now break both: in the declarative job, misspell `stages` as `stagez`. In the scripted job, misspell `node` as `nod`.
4. Build both.

**Expected output:** Declarative fails fast with a clear validation error naming the bad section. Scripted fails with a Groovy `MissingMethodException` stack trace.

**Why it matters:** This is the interview answer for "declarative vs scripted" — declarative validates upfront, scripted errors are Groovy stack traces.

---

## Exercise 3: Multibranch pipeline from a repo

**Goal:** See Jenkins auto-create jobs per branch.

1. Create a Git repo (local or GitHub) with a `Jenkinsfile` at the root containing a 2-stage pipeline.
2. Push a second branch with a slightly different Jenkinsfile (add an `echo` in one stage).
3. In Jenkins: New Item → Multibranch Pipeline → add your repo as the branch source. Save.

**Expected output:** Jenkins scans, finds both branches, and builds each with its own Jenkinsfile. Deleting the branch later removes its job on the next scan.

**Why it matters:** Multibranch replaced "one manually-clicked job per branch" — this is how modern teams give every PR a pipeline for free.

---

## Exercise 4: Pin work to a labeled agent

**Goal:** Route a stage to a specific agent.

1. Go to Manage Jenkins → Nodes. Note the built-in node labels (or add a label like `lab` to it — for real setups you'd add separate agents).
2. Create a pipeline with:

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'built-in' }
            steps { sh 'echo "building on $NODE_NAME"' }
        }
    }
}
```

3. Build, then change the label to `does-not-exist` and build again.

**Expected output:** First build runs and prints the node name. Second build sits in the queue saying "waiting for next available executor on 'does-not-exist'".

**Why it matters:** Label mismatch is the #1 cause of stuck queues — now you've seen it deliberately and know exactly what the queue message means.

---

## Exercise 5: Store and use a credential safely

**Goal:** Bind a secret without leaking it.

1. Manage Jenkins → Credentials → (global) → Add Credentials: Kind = Secret text, ID = `lab-api-key`, Secret = `super-secret-123`.
2. New pipeline job `secret-test`:

```groovy
pipeline {
    agent any
    stages {
        stage('Use secret') {
            steps {
                withCredentials([string(credentialsId: 'lab-api-key', variable: 'API_KEY')]) {
                    sh 'echo "key length: ${#API_KEY}"'
                    sh 'echo $API_KEY'
                }
            }
        }
    }
}
```

3. Build and read the console output.

**Expected output:** The first `sh` prints `key length: 15`. The second prints `****` — Jenkins masks the secret even when you echo it directly.

**Why it matters:** Proves the masking behavior and the `withCredentials` pattern — the core of the "how do you handle secrets" interview answer.

---

## Exercise 6: Build a shared library step

**Goal:** Write a reusable `vars/` step and call it.

1. Create a repo `jenkins-shared-lab` with this structure:

```
vars/
  sayHi.groovy
```

`sayHi.groovy`:
```groovy
def call(String name) {
    echo "Hi ${name}, from the shared library!"
}
```

2. Manage Jenkins → System → Global Pipeline Libraries → add it (name `lab-lib`, default version = your branch, retrieval = Modern SCM → your repo).
3. Pipeline job:

```groovy
@Library('lab-lib') _
pipeline {
    agent any
    stages {
        stage('Lib') { steps { sayHi('Satyam') } }
    }
}
```

**Expected output:** Console shows `Hi Satyam, from the shared library!`.

**Why it matters:** This is the exact mechanism behind "stop copy-pasting pipeline code" — one repo, one step, every pipeline calls it.

---

## Exercise 7: Pin a library version and test a change

**Goal:** See why `@Library('lib@version')` matters.

1. In your shared-lib repo, create a branch `v2` that changes the greeting text.
2. Create two pipeline jobs: one with `@Library('lab-lib') _`, one with `@Library('lab-lib@v2') _`.
3. Merge the `v2` change to main and rebuild both.

**Expected output:** The unpinned job picks up the new greeting; the `@v2` job already had it; a job pinned to the old default (if you'd pinned `@main`-before-merge... try pinning a tag) stays stable.

**Why it matters:** Version pinning is how you roll out library changes safely instead of breaking fifty pipelines with one merge.

---

## Exercise 8: Timeout, retry, and input in one pipeline

**Goal:** Combine the three flow-control primitives.

```groovy
pipeline {
    agent any
    stages {
        stage('Flaky') {
            steps {
                retry(3) {
                    sh 'curl -sf https://httpbin.org/status/200'
                }
            }
        }
        stage('Approve') {
            steps {
                timeout(time: 60, unit: 'SECONDS') {
                    input message: 'Deploy to prod?', ok: 'Ship it'
                }
                echo 'Deploying...'
            }
        }
    }
    post {
        always { echo "Build ${currentBuild.result ?: 'SUCCESS'} finished" }
    }
}
```

**Expected output:** The flaky stage retries on network failure. The pipeline pauses at "Deploy to prod?" — approve it within 60 seconds or the build aborts. `post` echoes the final result either way.

**Why it matters:** This is the production-deploy pattern: retry flakes, gate with human input, timeout the wait so executors don't hang forever.

---

## Exercise 9: Parallel stages and failure diagnosis

**Goal:** Speed up a pipeline and read parallel failures.

```groovy
pipeline {
    agent any
    stages {
        stage('Checks') {
            parallel {
                stage('Tests') { steps { sh 'sleep 5 && echo "tests green"' } }
                stage('Lint')  { steps { sh 'sleep 5 && echo "lint green"' } }
                stage('Broken'){ steps { sh 'exit 1' } }
            }
        }
    }
}
```

**Expected output:** All three run at once (total ~5s, not 15s). The build fails, and the stage view highlights exactly which parallel branch went red.

**Why it matters:** Parallelization is the cheapest pipeline speedup, and the stage view makes parallel failures trivial to locate — both are interview talking points.

---

## Exercise 10: Cleanup

**Goal:** Leave no trace.

```bash
docker stop jenkins-lab && docker rm jenkins-lab
docker volume rm jenkins_home
```

**Expected output:** Container and volume gone; `docker ps -a` and `docker volume ls` no longer list them.

**Why it matters:** Lab hygiene — throwaway Jenkins instances with default creds should never linger on your machine.
