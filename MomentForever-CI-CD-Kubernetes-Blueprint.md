# MomentForever Enterprise CI/CD and Kubernetes Deployment Blueprint

**Document type:** Engineering deployment standard\
**Status:** Working baseline for local implementation, designed to
evolve toward AWS\
**Scope:** `platform`, `booking`, `payment`, `gateway`, shared
infrastructure, CI, CD/GitOps, configuration, secrets, operations\
**Primary tools:** GitHub Actions, Docker, Kubernetes, Helm or
Kustomize, Argo CD\
**Last reviewed:** 2026-10-01

> This is the single working blueprint for building and validating the
> MomentForever deployment platform locally before moving to AWS. Values
> such as registry names, domains, namespaces, database names, topic
> names, ports, and cloud resources must be verified against the actual
> repositories and deployment environment. Examples are templates, not
> claims about resources that already exist.

------------------------------------------------------------------------

## 1. Objectives and non-goals

### Objectives

1.  Keep application source/CI and deployment configuration/CD in
    separate Git repositories.
2.  Build, test, scan, version, and publish immutable container images
    for each service.
3.  Make the CD repository the source of truth for Kubernetes desired
    state.
4.  Use Argo CD to reconcile the cluster with the CD repository.
5.  Provide service-specific configuration and database credentials
    without embedding environment-specific values in images.
6.  Ensure required ConfigMaps and Secrets exist before containers that
    require them start.
7.  Handle the distinction between configuration availability and
    dependency readiness.
8.  Make deployments observable, repeatable, auditable, recoverable, and
    least-privilege.
9.  Replicate the workflow locally first, then move to AWS with minimal
    application changes.

### Non-goals for the initial local milestone

-   Production-grade high availability on a single laptop.
-   Treating a laptop as a secure production server.
-   Exposing databases, brokers, or administrative interfaces publicly.
-   Automatically deploying every commit to production.
-   Assuming Kubernetes can guarantee that every external dependency is
    healthy merely because its configuration exists.

------------------------------------------------------------------------

## 2. Target architecture

``` text
Developer
   |
   | push / pull request
   v
Application repository (one per service)
   |
   | GitHub Actions: test, quality checks, build image, scan
   v
Container registry (GHCR / Docker Hub / approved registry)
   |
   | immutable image digest or unique commit tag
   v
CI updates image reference in deployment repository
   |
   v
CD / infrastructure repository (desired state)
   |
   | Argo CD watches Git and reconciles
   v
Kubernetes cluster
   |
   +-- platform Deployment + Service
   +-- booking Deployment + Service
   +-- payment Deployment + Service
   +-- gateway Deployment + Service
   |
   +-- ConfigMaps (non-sensitive runtime settings)
   +-- Secret references (sensitive runtime settings)
   +-- probes, resource requests/limits, security context
   |
   +-- Kafka
   +-- Redis
   +-- MongoDB
   +-- SQL database per microservice
   +-- Consul (only where still required)
```

### Responsibilities

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Application repository              Java source, Maven build, tests,
                                      Dockerfile, service CI workflow

  GitHub Actions CI                   Validate code, build and scan
                                      image, publish image,
                                      propose/update image version in CD
                                      repo

  Container registry                  Store versioned images; Kubernetes
                                      pulls images from it

  CD repository                       Desired Kubernetes state,
                                      Helm/Kustomize configuration,
                                      environment overlays, Argo CD
                                      definitions and Secret references

  Argo CD                             Compare Git desired state with
                                      cluster state and reconcile
                                      according to sync policy

  Kubernetes                          Schedule Pods, resolve
                                      ConfigMap/Secret references,
                                      restart containers, provide
                                      Service/DNS and rollout mechanisms

  External secret controller or       Deliver secret values from an
  secret provisioning                 approved secret store to workloads

  Terraform (later AWS phase)         Provision cloud infrastructure such
                                      as networking, EKS, registry, IAM,
                                      and managed services

  Operators / platform owner          Approve production changes, manage
                                      access, incidents, backups,
                                      upgrades and policy
  -----------------------------------------------------------------------

**Important:** Argo CD applies desired Kubernetes resources; it is not
the container registry and does not normally pull application images
itself. The Kubernetes node/container runtime pulls images.

------------------------------------------------------------------------

## 3. Repository strategy

### 3.1 Application repositories

Keep a separate repository for each independently released microservice,
for example:

``` text
momentforever-platform/
momentforever-booking/
momentforever-payment/
momentforever-gateway/
```

Expected contents:

``` text
service-repository/
├── src/
├── pom.xml
├── Dockerfile
├── .dockerignore
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release-image.yml   # optional separate workflow
├── README.md
└── docs/
```

Application repositories should not contain production credentials or
production cluster manifests. Keep runtime configuration externalized
and use the same image across environments where practical.

### 3.2 CD / infrastructure repository

Example layout:

``` text
momentforever-infra/
├── README.md
├── argocd/
│   ├── root-app.yaml
│   └── applications/
├── charts/                     # if using Helm
│   └── momentforever-service/
├── apps/
│   ├── platform/
│   ├── booking/
│   ├── payment/
│   └── gateway/
├── environments/
│   ├── local/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── infrastructure/
│   ├── kafka/
│   ├── redis/
│   ├── mongodb/
│   ├── databases/
│   └── consul/
├── policies/
└── runbooks/
```

Choose **one primary packaging approach** for workloads: - **Helm:**
useful for reusable service templates and environment-specific values. -
**Kustomize:** useful for a common base plus environment overlays. -
**Plain YAML:** fine for learning, but repetition and drift become
harder to manage at scale.

Do not mix multiple packaging methods for the same resources without a
clear ownership rule.

### 3.3 Environment separation

Use separate overlays/values and preferably separate namespaces and
access boundaries for `local`, `dev`, `staging`, and `prod`. For
stronger production isolation, use separate clusters/accounts for
higher-risk environments.

Never put real secrets in any Git branch, including private
repositories. Private Git is not a secret manager.

------------------------------------------------------------------------

## 4. Service and infrastructure inventory

Maintain this inventory and verify it against actual application
configuration before deployment.

  -------------------------------------------------------------------------------------
  Workload          Own SQL database            Possible other      Notes
                                                dependencies        
  ----------------- --------------------------- ------------------- -------------------
  `platform`        Platform database           Kafka, Redis,       Verify actual topic
                                                MongoDB, Consul if  names and cache
                                                configured          usage

  `booking`         Booking database            Kafka, Redis,       Verify booking
                                                MongoDB, Consul if  event topics and
                                                configured          snapshot/cache
                                                                    requirements

  `payment`         Payment database            Kafka,              Payment provider
                                                Stripe/webhook      secrets must be
                                                configuration,      sensitive
                                                Consul if           
                                                configured          

  `gateway`         Usually none                Downstream          Verify current
                                                services, JWT       gateway
                                                configuration,      discovery/routing
                                                Consul if           model
                                                configured          

  Kafka             Its own storage/cluster     Producer/consumer   Topics, partitions,
                    configuration               clients             retention,
                                                                    replication and
                                                                    ACLs are
                                                                    operational
                                                                    concerns

  Redis             Its own                     Application caches  Define eviction,
                    persistence/configuration                       persistence,
                                                                    authentication and
                                                                    resource limits

  MongoDB           Its own storage/auth        Applications using  Define auth,
                    configuration               document/GridFS     persistence,
                                                storage             backups and access

  SQL databases     One database/schema per     JDBC clients        Use separate
                    service as designed                             service
                                                                    credentials; no
                                                                    shared superuser
                                                                    credentials

  Consul            Its own                     Service             Kubernetes DNS may
                    configuration/storage if    discovery/config    cover basic service
                    used                        integration         discovery; retain
                                                                    Consul only where
                                                                    needed
  -------------------------------------------------------------------------------------

A single local PostgreSQL container can host separate databases for
development, but each service should use its own database and
least-privilege user. Production isolation may require separate managed
instances, clusters, or accounts according to availability, security,
recovery, and cost requirements.

------------------------------------------------------------------------

## 5. Configuration and secret management

### 5.1 Configuration categories

  -----------------------------------------------------------------------
  Value                   Classification          Delivery method
  ----------------------- ----------------------- -----------------------
  Kafka bootstrap address Non-secret              ConfigMap

  Kafka topic names       Non-secret              ConfigMap

  Redis service DNS/port  Non-secret              ConfigMap

  MongoDB host without    Non-secret              ConfigMap
  credentials                                     

  JDBC URL without        Usually non-secret,     ConfigMap
  credentials             environment-dependent   

  Database username       Sensitive identity      Secret / external
                                                  secret store

  Database password       Secret                  External secret manager
                                                  preferred; Kubernetes
                                                  Secret as runtime
                                                  target

  MongoDB URI containing  Secret                  Secret / external
  credentials                                     secret store

  Stripe secret API key / Secret                  Secret / external
  webhook signing secret                          secret store

  JWT signing key         Secret                  Secret / external
                                                  secret store

  TLS private keys        Secret                  Secret manager /
                                                  approved certificate
                                                  workflow

  Public service route    Non-secret              ConfigMap or chart
  and application name                            values
  -----------------------------------------------------------------------

### 5.2 Spring Boot property mapping

Spring Boot should read environment variables using externalized
configuration. Keep placeholders in `application.yml` and do not commit
real environment credentials.

Example only; align property names with each service's real
`application.yml`:

``` yaml
spring:
  application:
    name: moment-forever-platform
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}

platform:
  kafka:
    booking-requested-topic: ${BOOKING_REQUESTED_TOPIC}
```

If a service uses custom property names, bind those names rather than
changing application code to match this example blindly. Avoid logging
secrets or dumping the full environment.

### 5.3 ConfigMap example

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-config
  namespace: momentforever
data:
  DB_URL: "jdbc:postgresql://platform-db:5432/platform_db"
  KAFKA_BOOTSTRAP_SERVERS: "kafka:9092"
  BOOKING_REQUESTED_TOPIC: "booking-requested"
```

All addresses and names are examples. In Kubernetes, `platform-db` and
`kafka` must resolve to actual Services or valid endpoints from the
Pod's namespace/network.

### 5.4 Secret reference in a Deployment

``` yaml
envFrom:
  - configMapRef:
      name: platform-config
      optional: false
  - secretRef:
      name: platform-runtime-secret
      optional: false
```

The referenced Secret must contain the keys expected by the application,
for example `DB_USERNAME` and `DB_PASSWORD`.

Kubernetes checks required ConfigMap/Secret references before starting
the container. If a required referenced object is missing, the container
cannot start normally and the Pod reports a configuration-related error.
This does **not** prove the values are correct or that a database/broker
is reachable.

### 5.5 Secret strategy

**Local learning phase** - Use untracked local files or securely
provision Kubernetes Secrets from local environment variables. - Add
secret files to `.gitignore` and `.dockerignore`. - Use fake credentials
only. - Never commit real credentials or merely base64-encode them into
Git. - Restrict access to the local Kubernetes cluster and workstation.

**AWS phase** - Prefer AWS Secrets Manager or Systems Manager Parameter
Store for appropriate values. - Use an approved integration such as
External Secrets Operator or the Secrets Store CSI Driver, depending on
requirements. - Authenticate through workload identity/IAM roles rather
than long-lived AWS keys in manifests. - Encrypt secrets at rest,
restrict IAM and Kubernetes RBAC, audit access, rotate credentials, and
test rotation. - If a controller synchronizes an AWS value into a
Kubernetes Secret, account for initial synchronization timing and
failure handling. - Do not assume that a `Secret` object alone is
encrypted at rest or that every cluster has encryption/RBAC configured
correctly.

### 5.6 Configuration loading versus dependency readiness

These are different guarantees:

1.  Kubernetes can make required ConfigMap/Secret values available
    before starting a container.
2.  Kubernetes cannot guarantee that PostgreSQL, Kafka, Redis, MongoDB
    or Consul is healthy merely because the configuration exists.
3.  Applications should use bounded connection timeouts, sensible
    retries/backoff, and fail with actionable logs when required
    dependencies remain unavailable.
4.  Startup/readiness/liveness probes should reflect application
    behavior, not just whether the process exists.
5.  Readiness should prevent traffic from reaching a service that cannot
    currently serve requests.
6.  Avoid making liveness depend on every downstream system; otherwise a
    shared dependency outage can cause restart storms.
7.  If a secret or ConfigMap changes and is injected as environment
    variables, existing processes do not automatically receive the new
    environment. Trigger a controlled rollout. A mounted ConfigMap
    volume can update eventually, but the application must support
    reloading and the update is not instantaneous.

------------------------------------------------------------------------

## 6. Kubernetes workload baseline

Each service should have: - Deployment - ClusterIP Service unless a
different exposure is needed - ConfigMap or chart values for
non-sensitive settings - Secret reference - startup, readiness and
liveness probes where supported - resource requests and limits based on
measured usage - a non-root security context where the image supports
it - a service account with only required permissions - appropriate
rollout strategy and termination behavior - logs to stdout/stderr and
metrics/health endpoints as needed

### 6.1 Deployment template fragment

This is a pattern to adapt; confirm the correct application port, health
paths, image, resource values and property names for each service.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform
  namespace: momentforever
spec:
  replicas: 2
  selector:
    matchLabels:
      app: platform
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: platform
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: platform
          image: ghcr.io/ORG/platform-microservice:COMMIT_SHA
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: platform-config
                optional: false
            - secretRef:
                name: platform-runtime-secret
                optional: false
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 20
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
```

**Do not copy probe paths or resource numbers blindly.** Configure
Spring Boot Actuator health groups and verify the exact endpoints/port
in the running service. A non-root setting only works if the image can
run as a non-root user. Add `readOnlyRootFilesystem: true` only after
confirming the application has writable temp/log paths handled
correctly.

For private registries, configure image-pull authentication through an
approved Kubernetes pull secret or workload identity mechanism.

### 6.2 Service template

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: platform
  namespace: momentforever
spec:
  type: ClusterIP
  selector:
    app: platform
  ports:
    - name: http
      port: 8080
      targetPort: http
```

Other services in the same namespace can use the Kubernetes Service DNS
name. Use an Ingress or Gateway API implementation only when external
HTTP access is required.

### 6.3 Namespace and network isolation

-   Use namespaces for environment/application boundaries, not as the
    only security boundary.
-   Apply default-deny NetworkPolicies and explicitly allow required
    traffic where the CNI supports enforcement.
-   Permit only intended access to databases and brokers.
-   Keep database, broker, Redis, MongoDB and Consul administrative
    ports private.
-   Use TLS for sensitive traffic and appropriate authentication between
    services and infrastructure.
-   Define PodDisruptionBudgets only where availability and replica
    count make them meaningful.

------------------------------------------------------------------------

## 7. Infrastructure deployment strategy

### 7.1 Local phase

Choose one initial path and document it: - Run infrastructure
dependencies in Docker Compose while learning application
containerization, then move dependencies into Kubernetes; or - Run
dependencies in Kubernetes from the beginning to exercise service DNS,
persistent volumes, probes and lifecycle behavior.

For a faithful Kubernetes deployment rehearsal, run the dependencies
inside the cluster or use a clearly documented external endpoint. Do not
mix `localhost` assumptions: inside a Pod, `localhost` means that Pod,
not the developer laptop or another Pod.

### 7.2 Stateful workloads

Kafka, Redis, MongoDB and SQL databases are stateful. For a local lab,
Helm charts or basic manifests can be acceptable, but production
requires deliberate decisions about: - persistent volumes and storage
classes - backups and restore testing - replication and high
availability - upgrade order and compatibility - credentials and
encryption - resource limits and disk alerts - recovery objectives and
retention - maintenance windows and failure testing

For production AWS, evaluate managed services (for example, managed
PostgreSQL, Kafka-compatible messaging, Redis-compatible caching, and
managed MongoDB-compatible options) against cost, feature compatibility,
operational effort and requirements. Do not assume every local container
should become a production Pod.

### 7.3 Startup ordering

Kubernetes does not provide a global guarantee that all Deployments
start in a particular order. Argo CD sync waves and health assessment
can help order Kubernetes resource application, but they do not by
themselves prove that a database is fully ready to accept application
traffic.

Use: - readiness/health checks on infrastructure - application
retry/backoff and bounded timeouts - migration jobs or controlled schema
migration strategy - explicit dependency checks only when they are
reliable and bounded - Argo CD sync waves/health gates where appropriate

Avoid a permanent dependency on init containers that wait forever
without timeout, diagnostics or a recovery plan.

------------------------------------------------------------------------

## 8. CI pipeline standard

### 8.1 Triggers

Recommended starting policy: - Pull requests to protected branches:
compile, test, static checks, dependency/security checks. - Push to
`main` or the chosen integration branch: repeat required checks and
publish an immutable image after the required policy gates. -
Release/tag: optional controlled release promotion. - Manual dispatch:
controlled re-run or approved release task.

Adjust branch names to the real repository. Avoid publishing production
tags from untrusted pull-request code.

### 8.2 Required CI stages

1.  Checkout the exact commit.
2.  Set up the required JDK (Java 17 for the current Spring Boot 3.1.5
    baseline unless the application has been intentionally upgraded).
3.  Configure Maven dependency caching.
4.  Run formatting/static analysis and unit tests according to project
    standards.
5.  Run `mvn --batch-mode --no-transfer-progress clean verify`.
6.  Run integration tests with required dependencies supplied by
    Testcontainers or dedicated test services, where appropriate.
7.  Build the Docker image.
8.  Scan source/dependencies/image and generate an SBOM according to the
    chosen tools and policy.
9.  Publish only from trusted branches/tags after required checks pass.
10. Record image digest and commit SHA; update the CD repository through
    a narrowly scoped, auditable mechanism.
11. Retain logs and test reports; fail the job on test, build, or policy
    failure.

### 8.3 GitHub Actions security requirements

-   Set the minimum `permissions` required for each job.
-   Pin third-party Actions to reviewed full commit SHAs for
    high-assurance production workflows; manage updates with a
    dependency update process.
-   Use GitHub environments and approvals for production
    publication/promotion.
-   Prefer short-lived OIDC federation for cloud credentials instead of
    long-lived access keys.
-   Use narrowly scoped registry credentials or `GITHUB_TOKEN` where
    supported.
-   Never print secrets, turn on shell tracing around secrets, or expose
    secrets to untrusted PR workflows.
-   Protect `main` and release branches; require passing checks and
    reviews.
-   Use concurrency controls to avoid stale deployments.
-   Do not grant CI broad cluster-admin access.
-   Treat a commit to the CD repository as a deployment change and
    protect it with reviews/policy appropriate to the environment.

### 8.4 Example minimal Maven CI workflow

Adapt the JDK, action versions, branch names, Maven commands and tests
to the real repository. For a high-assurance pipeline, pin actions to
reviewed full commit SHAs rather than relying only on mutable major
tags.

``` yaml
name: Platform CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: platform-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
          cache: maven

      - name: Verify
        run: mvn --batch-mode --no-transfer-progress clean verify
```

This is a baseline example, not a complete enterprise pipeline: action
versions must be reviewed and pinned according to policy; image build,
scanning, publishing, provenance/SBOM, and CD update stages are added as
separately permissioned jobs.

### 8.5 Image tagging policy

-   Never rely on `latest` as the sole deployment reference.
-   Use unique tags such as the full or sufficiently unique Git commit
    SHA.
-   Prefer deploying by image digest for immutable promotion.
-   Do not rebuild a different image for staging and production; promote
    the same artifact where possible.
-   Record the source commit, build run, image digest and scan/SBOM
    results.

------------------------------------------------------------------------

## 9. CD and GitOps standard

### 9.1 Desired-state workflow

1.  CI publishes a new image.
2.  CI creates a commit or pull request updating the image reference in
    the CD repository.
3.  The change passes the CD repository's review and policy.
4.  Argo CD detects the commit and compares desired state with cluster
    state.
5.  Argo CD applies changes according to sync policy.
6.  Kubernetes schedules the new Pods, pulls the image and injects
    referenced configuration.
7.  Probes determine whether new Pods are ready.
8.  Rollout status and application telemetry confirm the result.
9.  On failure, halt promotion and use the documented rollback/recovery
    process.

### 9.2 Argo CD principles

-   Bootstrap Argo CD once through a documented administrative process.
-   Store the root Application/App-of-Apps configuration in the CD
    repository.
-   Scope Argo CD Projects to approved repositories, destinations,
    namespaces and resource kinds.
-   Use automated sync for low-risk environments only after validation.
-   Enable pruning/self-healing only with explicit ownership and
    deletion safeguards; review destructive changes.
-   Use manual approval or promotion controls for production.
-   Do not give application CI credentials unrestricted access to the
    cluster.
-   Monitor Argo CD health, sync state and reconciliation errors.
-   Ensure the cluster can authenticate to private Git repositories and
    image registries using least privilege.

### 9.3 Local bootstrap

The first Argo CD installation and root Application are bootstrap
operations. They cannot be managed by Argo CD before Argo CD itself
exists. Document and secure this one-time process, then let GitOps
manage the resources that follow.

### 9.4 Image update approach

Preferred options: - CI opens a pull request in the CD repository
changing the image digest/tag; or - CI commits an image-reference update
using a narrowly scoped bot identity when policy permits.

A pull request provides an audit trail and approval point. Never allow
an arbitrary untrusted PR to modify production image references or
receive production credentials.

------------------------------------------------------------------------

## 10. Database migration and schema ownership

-   Each microservice owns its data model and database changes.
-   Use the service's established migration tool (for example, Flyway or
    Liquibase) consistently.
-   Do not let multiple services casually modify the same schema.
-   Design schema changes to be backward-compatible during rolling
    deployments: expand, deploy compatible code, migrate data if
    required, then contract later.
-   Run migrations as an explicit, observable deployment step or
    controlled job with suitable permissions; avoid every replica racing
    to run destructive migrations.
-   Back up before risky migrations and test restore/recovery.
-   Keep migration credentials separate from normal application
    credentials when feasible.
-   Do not roll back application code blindly if the database migration
    is not backward-compatible.

------------------------------------------------------------------------

## 11. Observability and operational readiness

### Logs

-   Write structured logs to stdout/stderr.
-   Include service name, environment, version/commit, correlation/trace
    identifiers where available.
-   Never log passwords, tokens, full credential-bearing URIs, payment
    secrets or sensitive personal data.

### Health and metrics

-   Enable and secure Spring Boot Actuator endpoints intentionally.
-   Separate liveness (process can continue), readiness (can accept
    traffic) and startup checks.
-   Do not expose management endpoints publicly without
    authentication/network controls.
-   Add metrics, tracing and central log collection as the platform
    matures.

### Alerts

At minimum, define signals for: - repeated Pod restarts and
CrashLoopBackOff - failed rollout or unavailable replicas - failed
readiness probes - elevated HTTP error rate and latency - database
connection exhaustion - Kafka consumer lag and repeated processing
failures - dead-letter queue growth - Redis memory/eviction pressure -
storage capacity and failed backups - Argo CD OutOfSync/Degraded
states - certificate or credential expiry where applicable

### Rollback and recovery

-   Roll back to a known-good image digest and Git revision.
-   Keep a record of the change that caused the incident.
-   Check database migration compatibility before rolling back code.
-   Restore data only through a tested recovery procedure.
-   Define who can approve and execute production rollback.

------------------------------------------------------------------------

## 12. Security baseline

-   Protect branches and require reviews for production-affecting
    changes.
-   Use least-privilege GitHub workflow permissions, registry scopes,
    Kubernetes RBAC and cloud IAM.
-   Use OIDC federation for CI-to-cloud access where supported.
-   Keep secrets out of Git, container images, build args, logs, test
    reports and artifacts.
-   Scan dependencies and images; define severity thresholds and
    exception expiry.
-   Generate and retain SBOM/provenance where tooling supports it.
-   Run containers as non-root where possible, drop unnecessary
    capabilities, and avoid privileged containers.
-   Set resource requests/limits and use appropriate Pod Security
    standards.
-   Restrict network access with NetworkPolicies where supported.
-   Encrypt traffic and sensitive data at rest according to the
    environment's requirements.
-   Pin base images and dependencies; patch them through a managed
    process.
-   Audit changes to production configuration and secret access.
-   Keep local development credentials distinct from staging/production
    credentials.

------------------------------------------------------------------------

## 13. Local implementation plan

Build the lab in these milestones. Do not proceed to the next milestone
until its verification criteria pass.

### Milestone 0 --- Inventory and prerequisites

-   [ ] Confirm OS and container runtime.
-   [ ] Confirm Java/Maven versions and that each service builds
    locally.
-   [ ] Confirm each service's port, context path, health endpoints, DB,
    Kafka topics and other actual dependencies.
-   [ ] Confirm repository names, default branches and registry choice.
-   [ ] Decide whether local infrastructure starts in Docker Compose or
    Kubernetes.
-   [ ] Ensure secret files are ignored by Git.

**Exit criteria:** a checked-in inventory exists, all service builds are
understood, and no real credentials are committed.

### Milestone 1 --- Local Kubernetes

-   [ ] Install Docker Desktop (or supported container runtime),
    `kubectl`, local cluster tool and Helm.
-   [ ] Create a cluster with sufficient CPU/RAM/disk for the chosen
    workload.
-   [ ] Verify `kubectl get nodes` and `kubectl get pods -A`.
-   [ ] Set a dedicated context and namespace.

**Exit criteria:** cluster is Ready and a simple test Pod can start.

### Milestone 2 --- Infrastructure

-   [ ] Deploy or connect Kafka, Redis, MongoDB, per-service SQL
    databases and Consul if required.
-   [ ] Configure persistent storage for stateful local workloads where
    needed.
-   [ ] Create separate database/user credentials for each service.
-   [ ] Verify DNS and connectivity from a test Pod.
-   [ ] Verify Kafka topic provisioning and authentication assumptions.
-   [ ] Document ports and internal service DNS names.

**Exit criteria:** infrastructure endpoints are reachable from the same
network context as the applications, and persistence behavior is
understood.

### Milestone 3 --- Containerize services

-   [ ] Build an image for each service.
-   [ ] Use multi-stage builds and a minimal supported runtime image
    where practical.
-   [ ] Run as non-root where compatible.
-   [ ] Keep configuration external to the image.
-   [ ] Verify container ports, context paths, graceful shutdown and
    health endpoints.
-   [ ] Tag images with unique versions/commit identifiers.

**Exit criteria:** each service starts with injected local configuration
and logs no secrets.

### Milestone 4 --- Kubernetes configuration

-   [ ] Create namespaces, ConfigMaps, Secret references, Deployments
    and Services.
-   [ ] Set `optional: false` for required ConfigMap/Secret references
    or use the default required behavior.
-   [ ] Configure probes only after verifying endpoint semantics.
-   [ ] Set realistic initial resource requests/limits.
-   [ ] Verify missing required Secrets prevent the application
    container from starting.
-   [ ] Verify a wrong DB password or unavailable Kafka produces
    diagnosable failures and bounded recovery behavior.
-   [ ] Verify readiness prevents traffic reaching an unready Pod.

**Exit criteria:** configuration injection and dependency-failure
behavior are tested.

### Milestone 5 --- CI

-   [ ] Add CI workflow to each service repository.
-   [ ] Require build/test checks on pull requests.
-   [ ] Add image build and security checks.
-   [ ] Publish images only from trusted branches/releases.
-   [ ] Record image digest and source commit.
-   [ ] Test registry authentication from the local cluster.

**Exit criteria:** a source change produces a tested, identifiable
image.

### Milestone 6 --- CD and Argo CD

-   [ ] Add Kubernetes manifests/Helm/Kustomize to the CD repository.
-   [ ] Install Argo CD and bootstrap the root Application.
-   [ ] Configure repository credentials securely.
-   [ ] Apply namespace, project, destination and sync policies.
-   [ ] Test an image-reference change through Git.
-   [ ] Test a failed rollout and recovery.
-   [ ] Decide which environments allow automated sync and which require
    approval.

**Exit criteria:** a reviewed CD repository change causes Argo CD to
reconcile the cluster without a manual `kubectl apply` for routine
deployments.

### Milestone 7 --- End-to-end release

-   [ ] Push a harmless platform change.
-   [ ] Observe CI checks and image publication.
-   [ ] Observe the CD image-reference update.
-   [ ] Observe Argo CD sync.
-   [ ] Verify the new image digest, Pod readiness and service behavior.
-   [ ] Verify rollback to a known-good revision.
-   [ ] Capture the steps in a runbook.

**Exit criteria:** the full source-to-running-workload path is
repeatable and documented.

------------------------------------------------------------------------

## 14. Verification commands

Examples below assume the namespace is `momentforever`; change it if the
actual namespace differs.

``` bash
# Cluster and nodes
kubectl cluster-info
kubectl get nodes -o wide

# Namespace and workloads
kubectl get namespaces
kubectl get pods -n momentforever
kubectl get deployments,services -n momentforever

# Diagnose a failing Pod
kubectl describe pod <pod-name> -n momentforever
kubectl logs <pod-name> -n momentforever --all-containers=true
kubectl get events -n momentforever --sort-by=.lastTimestamp

# Check required configuration objects exist (do not print secret values)
kubectl get configmap platform-config -n momentforever
kubectl get secret platform-runtime-secret -n momentforever

# Rollout status
kubectl rollout status deployment/platform -n momentforever
kubectl rollout history deployment/platform -n momentforever

# Argo CD, if installed in its standard namespace
kubectl get applications -n argocd
```

Do not paste decoded Secret values, credential-bearing environment
output, or unredacted logs into tickets or chat.

------------------------------------------------------------------------

## 15. Troubleshooting guide

  ----------------------------------------------------------------------------------
  Symptom                        Likely causes           First checks
  ------------------------------ ----------------------- ---------------------------
  `CreateContainerConfigError`   Required                `kubectl describe pod`;
                                 ConfigMap/Secret        verify object name and
                                 missing or invalid      namespace
                                 reference               

  `ImagePullBackOff`             Wrong image/tag,        Pod events, image
                                 registry auth or        reference,
                                 network issue           pull-secret/workload
                                                         identity setup

  `CrashLoopBackOff`             Application startup     Container logs, previous
                                 exception, invalid      logs, events, connection
                                 config, unavailable     settings
                                 dependency              

  Readiness probe failing        Wrong port/path,        Verify actual port/context
                                 Actuator group          path/health group and logs
                                 disabled, app not ready 

  DB connection failure          Wrong                   Test DNS and port from a
                                 DNS/port/credentials,   debug Pod; verify
                                 SQL not ready, network  service/user/database
                                 policy                  

  Kafka connection failure       Incorrect bootstrap     Inspect broker listener
                                 address, broker         configuration and test from
                                 listener advertised     the Pod network
                                 incorrectly, auth/topic 
                                 ACLs                    

  Config changed but behavior    Environment variables   Confirm rollout restarted
  did not                        are fixed for the       Pods and inspect non-secret
                                 existing process        configuration version

  Argo CD OutOfSync              Git desired state       Application status, repo
                                 differs from cluster or access, manifest rendering
                                 reconciliation failed   and events

  Rollout stuck                  New Pods unready,       `kubectl rollout status`,
                                 insufficient resources, describe Pod, events and
                                 image pull or probe     logs
                                 failures                

  Local app works but Pod fails  `localhost`/host port   Test from inside the
                                 assumptions, missing    cluster and use Service DNS
                                 env vars, different     
                                 DNS/network             
  ----------------------------------------------------------------------------------

------------------------------------------------------------------------

## 16. AWS migration plan

After the local end-to-end flow is reliable:

1.  Provision AWS networking, IAM and EKS using Terraform or an approved
    IaC process.
2.  Choose the production registry and authentication model.
3.  Choose managed or self-managed Kafka, Redis, MongoDB and SQL options
    based on compatibility, resilience, operations and cost.
4.  Establish secret storage, encryption, workload identity and audit
    policies.
5.  Install Argo CD through a controlled bootstrap process.
6.  Configure environment-specific values and secret references in the
    CD repository.
7.  Configure ingress, TLS, DNS, network policy and monitoring.
8.  Test backup/restore, scaling, node failure, dependency failure and
    rollback.
9.  Promote the same immutable image digest through environments.
10. Document ownership, on-call responsibilities, change approvals,
    recovery objectives and costs.

Do not copy local endpoints such as `localhost`, local registry
addresses, or laptop-only credentials into AWS manifests.

------------------------------------------------------------------------

## 17. Definition of done

The baseline platform is ready for a controlled first deployment when:

-   [ ] All four service repositories build and test independently.
-   [ ] Images are tagged immutably and can be pulled by the cluster.
-   [ ] The CD repository is the authoritative desired state.
-   [ ] Argo CD can reconcile approved changes from Git.
-   [ ] ConfigMaps and required Secrets are present before the
    application container starts.
-   [ ] No production secrets are stored in Git or images.
-   [ ] Each service has its own database identity and least-privilege
    access.
-   [ ] Infrastructure connectivity and failure modes have been tested.
-   [ ] Probes correctly distinguish startup, readiness and liveness.
-   [ ] Rollout, rollback, migration and troubleshooting procedures are
    documented.
-   [ ] CI permissions, CD access and production approvals are
    appropriately restricted.
-   [ ] Logs and metrics support diagnosing deployment failures.
-   [ ] Recovery and backups have been tested for any stateful
    production data.

------------------------------------------------------------------------

## 18. Decisions to record before implementation

Complete this section in the CD repository README or this document
before treating the configuration as production-ready.

  -----------------------------------------------------------------------
  Decision                            Selected value
  ----------------------------------- -----------------------------------
  Local operating system/runtime      TODO

  Local Kubernetes tool (`kind`,      TODO
  Minikube, or Docker Desktop         
  Kubernetes)                         

  Container registry                  TODO

  Manifest packaging (Helm or         TODO
  Kustomize)                          

  Local infrastructure strategy       TODO
  (Compose or Kubernetes)             

  Environment/namespace naming        TODO

  Default branch names per repository TODO

  Exact service ports and context     TODO
  paths                               

  Actual per-service DB names and     TODO
  non-secret JDBC URLs                

  Actual Kafka bootstrap address and  TODO
  topic inventory                     

  Redis/MongoDB/Consul usage per      TODO
  service                             

  Secret manager and Kubernetes       TODO
  integration                         

  Image tag/digest promotion policy   TODO

  CI action pinning and scanning      TODO
  policy                              

  Production approval and rollback    TODO
  owner                               

  Backup and recovery objectives      TODO
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 19. Operating principles

1.  **Git describes desired state; the cluster is reconciled to it.**
2.  **Images contain application code, not environment credentials.**
3.  **Configuration availability is not dependency health.**
4.  **Each service owns its database and schema lifecycle.**
5.  **Immutable artifacts move between environments; do not rebuild them
    for each environment.**
6.  **Production changes are reviewable, auditable and recoverable.**
7.  **Secrets are provisioned from a secure source and never committed
    to Git.**
8.  **Every deployment failure should have enough logs, events and
    metrics to diagnose without exposing sensitive data.**
9.  **Local success is a prerequisite for cloud rollout, not proof of
    production readiness.**
10. **Validate all example values against the real MomentForever
    repositories before applying manifests.**
