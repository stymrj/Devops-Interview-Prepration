# Jenkins Cheat Sheet

*One-page dense reference — declarative syntax, agents, credentials, shared libraries.*

## Minimal Declarative Pipeline

```groovy
pipeline {
    agent any
    options { timeout(time: 30, unit: 'MINUTES') }
    environment { APP = 'myapp' }
    stages {
        stage('Build') { steps { sh 'make build' } }
        stage('Test')  { steps { sh 'make test' } }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'make deploy' }
        }
    }
    post {
        always  { archiveArtifacts artifacts: 'build/**' }
        failure { mail to: 'team@x.com', subject: "FAILED: ${env.JOB_NAME}" }
    }
}
```

## Agents

| Syntax | Meaning |
|---|---|
| `agent any` | Any available executor |
| `agent none` | No agent until a stage declares one |
| `agent { label 'docker' }` | Agents with label `docker` |
| `agent { docker { image 'node:20' } }` | Ephemeral Docker container |
| `agent { kubernetes { yaml '...' } }` | Ephemeral K8s pod |

`$NODE_NAME` in `sh` = which agent ran the step. Never build on the controller.

## Common Steps

| Step | Use |
|---|---|
| `sh 'cmd'` / `sh(script: 'cmd', returnStdout: true)` | Shell, optionally capture output |
| `echo "msg"` | Console log line |
| `git url: '...', branch: 'main'` | Checkout |
| `checkout scm` | Checkout the Jenkinsfile's own repo |
| `withEnv(['K=v']) { }` | Scoped env vars |
| `withCredentials([...]) { }` | Scoped secrets |
| `input message: 'Go?', ok: 'Yes'` | Manual approval gate |
| `timeout(time: 10, unit: 'MINUTES') { }` | Kill hangs |
| `retry(3) { }` | Retry flaky steps |
| `parallel a: { }, b: { }` | Run branches concurrently (or parallel stages) |
| `error 'msg'` | Fail the build with a message |

## Credentials Binding

```groovy
withCredentials([
  string(credentialsId: 'api-key', variable: 'KEY'),
  usernamePassword(credentialsId: 'docker', usernameVariable: 'U', passwordVariable: 'P'),
  file(credentialsId: 'kubeconfig', variable: 'KCFG'),
  sshUserPrivateKey(credentialsId: 'ssh', keyFileVariable: 'SSHKEY')
]) {
  sh 'echo $KEY'   // prints **** — always masked
}
```

Store in Manage Jenkins → Credentials (global / folder-scoped). Folder scope = per-team least privilege.

## Shared Libraries

```groovy
@Library('my-lib@1.4.2') _   // pin version — never float on main for prod pipelines
// then call vars/myStep.groovy as a step:
myStep('arg1')
```

Repo layout: `vars/*.groovy` = callable steps (`def call(...)`), `src/` = plain Groovy classes. Configure at Manage Jenkins → System → Global Pipeline Libraries. Test changes from a branch before merging to main.

## Multibranch

New Item → Multibranch Pipeline → branch source (GitHub). Scans repo, builds every branch with a Jenkinsfile. `when { branch 'main' }` / `when { changeRequest() }` gate stages.

## Debugging Quick Hits

- Stuck in queue → label mismatch, offline agent, or all executors busy (check node executor list)
- `MissingMethodException` → scripted-syntax typo
- Declarative validation error → misspelled section, fails before running
- Secrets in console → only safe via `withCredentials`; masked as `****`
- Replay → rerun a build with edited pipeline code (great for testing library fixes)
- Blue Ocean / stage view → find which stage or parallel branch failed

## CLI (jenkins-cli.jar)

```bash
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth user:token help
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth user:token build my-job -s -v
```
