# VProfile Java Web Application

A Java-based web application built with Spring MVC, backed by MySQL, Memcached, RabbitMQ, and Elasticsearch. The project includes a full CI/CD pipeline using GitHub Actions, SonarQube, Amazon ECR, and Helm GitOps with a manual approval gate before deploying to Kubernetes.

---

## Tech Stack

- **Java 8** — Spring MVC, Spring Security, Spring Data JPA, Hibernate
- **MySQL** — primary database
- **Memcached** — caching layer
- **RabbitMQ** — message broker
- **Elasticsearch** — search engine
- **Maven** — build and dependency management
- **Docker** — containerization (multistage build)
- **Amazon ECR** — container image registry
- **SonarQube** — code quality and static analysis
- **Helm** — Kubernetes deployment via GitOps

---

## Project Structure

```
├── .github/workflows/ci.yml       # GitHub Actions CI/CD pipeline
├── Docker-files/
│   ├── app/multistage/Dockerfile  # Multistage Docker build for the app
│   ├── db/Dockerfile              # MySQL container
│   └── web/Dockerfile             # Nginx reverse proxy
├── src/
│   ├── main/java                  # Application source code
│   ├── main/resources             # Config files
│   └── test/java                  # Unit tests
├── pom.xml                        # Maven build config
└── sonar-project.properties       # SonarQube project config
```

---

## CI/CD Pipeline

The pipeline is defined in [`.github/workflows/ci.yml`](.github/workflows/ci.yml) and has 4 jobs:

### 1. `build` — triggered on PRs to `main`
- Checks out source code
- Sets up Java 21 with Maven and SonarQube cache
- Runs `mvn clean verify checkstyle:checkstyle` — compiles, runs unit tests, packages the WAR, and generates a Checkstyle report at `target/checkstyle-result.xml`
- Runs SonarQube scan — picks up test coverage (JaCoCo), Checkstyle violations, and code smells
- Evaluates the SonarQube quality gate — fails the pipeline if quality standards are not met

### 2. `docker-build-push` — triggered on push to `main` only
- Authenticates to AWS via OIDC (no long-lived credentials)
- Logs in to Amazon ECR
- Builds the Docker image using the multistage Dockerfile
- Scans the image with Trivy for CRITICAL and HIGH vulnerabilities (non-blocking, `exit-code: 0`)
- Pushes the image to ECR with two tags: `<commit-sha>` and `latest`
- Uploads the Trivy scan report as a pipeline artifact (retained for 14 days)

### 3. `update-helm` — triggered on push to `main` only (manual approval gate)
This job does **not** directly update the Helm `values.yaml` on `main`. Instead it introduces a manual approval step via a Pull Request:

1. Checks out the Helm GitOps repository on `main`
2. Creates or resets a branch called `helm-approval` from `main`
3. Updates `helm/vprofile/values.yaml` with the new image registry and tag using `yq`
4. Commits and pushes the change to the `helm-approval` branch
5. Opens a Pull Request from `helm-approval` → `main` in the Helm repo (skips if one already exists)
6. Sends a Slack notification with the PR status

A team member must **review and merge the PR** in the Helm repository before the new image is applied to the Kubernetes cluster. This gives you a manual gate between CI and deployment.

### 4. `notify` — always runs
- Sends a Slack notification with the overall pipeline result (success or failure)
- Skips silently if `SLACK_WEBHOOK` is not configured

---

## Pipeline Flow

```
PR to main          →  build job (test, checkstyle, sonarqube)
                              ↓
Merge to main       →  docker-build-push (build, scan, push to ECR)
                              ↓
                       update-helm (create helm-approval branch + open PR)
                              ↓
                       Manual review & merge PR in Helm repo
                              ↓
                       ArgoCD/Flux picks up values.yaml change → deploys
```

---

## Prerequisites

### GitHub Secrets

| Secret | Description |
|---|---|
| `SONAR_TOKEN` | SonarQube authentication token |
| `CI_AWS_ROLE_ARN` | AWS IAM role ARN for OIDC authentication |
| `GITOPS_PAT` | GitHub Personal Access Token for pushing to the Helm repo and creating PRs |
| `SLACK_WEBHOOK` | Slack incoming webhook URL (optional) |

### GitHub Variables

| Variable | Description |
|---|---|
| `SONAR_HOST_URL` | SonarQube server URL |
| `AWS_REGION` | AWS region (e.g. `us-east-1`) |
| `ECR_REPOSITORY` | ECR repository name |
| `HELM_REPO_NAME` | Name of the Helm GitOps repository |

### AWS Setup
- An IAM role with ECR push permissions, configured for GitHub Actions OIDC federation
- An ECR repository created in your AWS account

### SonarQube Setup
- A running SonarQube instance
- Project key: `vprofile-java99` (as defined in `sonar-project.properties`)

### Helm Repo Setup
- The `GITOPS_PAT` token must have `repo` scope to create branches and open PRs in the Helm repository

---

## Running Locally

### Build and test
```bash
mvn clean verify
```

### Build with Checkstyle report
```bash
mvn -B -U clean verify checkstyle:checkstyle
```

### Run with Jetty (local dev server)
```bash
mvn jetty:run
```
App will be available at `http://localhost:8080`

---

## Branch Strategy

| Branch | Pipeline Behaviour |
|---|---|
| `feature/*` | No pipeline triggered |
| PR → `main` | Runs `build` job only (test, checkstyle, sonarqube) |
| push to `main` | Runs docker build/push → creates Helm approval PR → notify |
