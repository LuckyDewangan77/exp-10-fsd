# Student Lab — full-stack CI/CD experiment

Spring Boot REST API + vanilla JavaScript/Vite frontend + PostgreSQL + Kafka, tested and deployed with GitHub Actions, GHCR, Docker Compose, and an Nginx canary router.

**Start locally. Do not configure remote deployment until `bash scripts/verify-local.sh` passes on your Docker-enabled machine.** This is an educational, single-VM deployment architecture, not a claim of production-grade high availability.

## 1. What is implemented

| Requirement | Implementation |
|---|---|
| CRUD API and database | Spring Boot 3.4.4, Java 21, Spring Data JPA, PostgreSQL 16, Flyway |
| Frontend | Vite 6.4.5, browser JavaScript, same-origin API client, create/edit/delete/list UI |
| Unit tests | JUnit 5 + Mockito; service success, missing records, invalid DTOs, duplicate email and Kafka failure; frontend Node tests |
| Integration tests | Spring Boot Test + MockMvc; actual PostgreSQL and Kafka; CRUD, statuses, validation, persistence |
| Testcontainers | JUnit-managed PostgreSQL and Kafka containers, automatic cleanup, real event round-trip |
| Contracts | Pact JS consumer tests exercise the real frontend client; Pact JVM verifies the backend |
| Containers | Multi-stage backend/frontend Dockerfiles and four-service local Compose stack |
| CI/CD | Tests → builds → immutable GHCR image digests → staging → production canary |
| Canary | Old/new full-stack releases side by side; 10% → 50% → 100%; health + per-slot 5xx monitoring; automatic rollback |

### Architecture

```text
LOCAL: browser :8088 → frontend Nginx → Spring Boot → PostgreSQL
                                              └→ Kafka student-events

CI: Node consumer → Pact mock → generated contract
    Pact JVM provider verifier → real Spring Boot → Testcontainers PostgreSQL + Kafka

DEPLOYMENT VM:
Internet → gateway Nginx :80 → blue frontend  → blue backend  ─┐
                          └→ green frontend → green backend ─┼→ shared production DB + Kafka
Internal-only staging frontend → staging backend ─────────────┘  (separate staging DB/topic)
```

Frontend and backend are routed **together as a release**, rather than independently weighting each service. Requests are weighted, not users: this example has no sticky sessions or login sessions.

## 2. Prerequisites and installation

For local development: Git, Java **21**, Maven **3.9+**, Node.js **22**, npm, Python 3, curl, Docker Engine/Desktop with Compose **v2.24+**. Allocate about 6–8 GB memory to Docker. Docker must be running for integration/provider tests.

Example Ubuntu 24.04 commands (also suitable inside WSL2 with an appropriate Docker installation):

```bash
sudo apt-get update
sudo apt-get install -y git curl ca-certificates gnupg openjdk-21-jdk maven python3
# Node via nvm; inspect third-party install scripts according to your security policy.
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh -o /tmp/nvm-install.sh
bash /tmp/nvm-install.sh
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"
nvm install 22
nvm use 22
```

Docker Engine + Compose on Ubuntu (skip if Docker Desktop is already configured):

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $VERSION_CODENAME stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"
# Log out and in again to activate Docker group membership.
```

Docker group membership grants effectively root-level host access. Use only a dedicated trusted lab VM/account. On macOS/Windows install Java 21, Maven, Node 22 and Docker Desktop; run shell scripts in Bash/WSL, not PowerShell.

Verify tools:

```bash
java -version
mvn -version
node --version
docker version
docker compose version
docker info
```

## 3. Local execution — no GHCR, VM or GitHub secrets required

Extract this project and enter its root:

```bash
cd student-cicd
cp .env.example .env
npm ci --prefix frontend
# Entire stack; first build/download may take several minutes.
docker compose up -d --build
docker compose logs -f backend
```

Open **http://localhost:8088**. When the backend reports it has started:

```bash
curl -f http://localhost:8088/health
python3 scripts/smoke.py http://localhost:8088
```

The smoke script creates one uniquely named test student, checks CRUD and validation, and removes that record. Database data persists in a named volume. Kafka is ephemeral in the local stack; the deployment stack persists its data.

```bash
docker compose down       # retain PostgreSQL data
# Destructive reset of the local experiment database:
docker compose down -v
```

### Run backend/frontend without rebuilding containers

```bash
# Start just the dependencies:
docker compose up -d db kafka
# In terminal 1 (do not have the backend container bound to :8080):
DB_URL=jdbc:postgresql://localhost:5432/students \
DB_USER=students DB_PASSWORD=students KAFKA_BOOTSTRAP=localhost:29092 \
mvn -f backend/pom.xml spring-boot:run
# In terminal 2:
npm ci --prefix frontend
npm run dev --prefix frontend
```

Open http://localhost:5173. Vite proxies `/api` to port 8080. Container-to-container Kafka uses `kafka:9092`; host-run Java uses `localhost:29092`. If you changed `.env`, pass matching DB credentials to the host-run backend. `.env` is consumed by Compose, not automatically by Maven.

### API examples

```bash
curl -i -X POST http://localhost:8088/api/students \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com"}'
curl http://localhost:8088/api/students
# Substitute the ID returned by POST if it is not 1.
curl http://localhost:8088/api/students/1
curl -i -X PUT http://localhost:8088/api/students/1 \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada Lovelace","email":"ada@example.com"}'
curl -i -X DELETE http://localhost:8088/api/students/1
# Expected 400 and {"error":"Validation failed"}:
curl -i -X POST http://localhost:8088/api/students \
  -H 'Content-Type: application/json' -d '{"name":"","email":"bad"}'
```

| Endpoint | Success | Important errors |
|---|---|---|
| `GET /api/students` | 200 array | — |
| `GET /api/students/{id}` | 200 student | 404 |
| `POST /api/students` | 201 student + Location | 400 invalid, 409 duplicate email, 503 event failure |
| `PUT /api/students/{id}` | 200 student | 400, 404, 409, 503 |
| `DELETE /api/students/{id}` | 204 empty | 404, 503 |
| `GET /api/version` | 200 release SHA/local | — |
| `GET /health` (frontend) | 200 when ready | 503 if DB/Kafka unhealthy |

A student has `id`, `name` (nonblank, max 100), and `email` (valid, nonblank, max 200, unique). Error responses contain a stable `error` field. A database constraint prevents duplicate emails even under concurrent requests.

## 4. Local tests — run these before enabling deployments

One-command verification (leaves the stack running for inspection):

```bash
bash scripts/verify-local.sh
```

Individual commands from the repository root:

```bash
# Unit tests: do not need Docker.
mvn -B -f backend/pom.xml clean test
npm ci --prefix frontend
npm test --prefix frontend

# Integration: Docker required; Testcontainers picks its own free mapped ports.
mvn -B -f backend/pom.xml verify -DskipUnitTests=true

# Contracts: generate fresh frontend requirements, then verify the real backend.
npm run test:contract --prefix frontend
(cd backend && mvn -B verify -Pcontracts -DskipUnitTests=true)

# Build and offline routing checks:
npm run build --prefix frontend
mvn -B -f backend/pom.xml package -DskipTests -Djacoco.skip=true
python3 scripts/test-routing.py
```

Run the unit phase first if using `-DskipUnitTests=true`. Maven Surefire runs `*Test`, Failsafe runs `StudentApiIT` and `KafkaIT`; profile `contracts` replaces its includes with `StudentContractIT`. **`mvn test` alone does not run integration/contract verification.** Provider verification intentionally fails when no generated Pact is available; do not use an option that permits missing pacts.

- `StudentServiceTest`: all CRUD paths, missing entity, constraints, database conflict, event failure. Mockito-injected objects are not Spring proxies, so the invalid-input unit test explicitly invokes the Jakarta Validator; HTTP integration tests verify that Spring actually enforces validation.
- `StudentApiIT`: real HTTP dispatch through MockMvc and real PostgreSQL/Kafka. Verifies persisted state after create, update and delete.
- `KafkaIT`: creates a student through the service, consumes the matching event with a unique Kafka consumer group, polls to a bounded deadline and closes the consumer.
- `ContainerTestBase`: `@Container` + `@Testcontainers` lifecycle and `@DynamicPropertySource`; no reuse flag and no fixed host ports. Each test class dirties its Spring context before its containers disappear, preventing reuse of a stale datasource.
- Pact covers every CRUD operation plus 400 and 404 responses. Integer/string matchers check response types; missing required fields or incompatible status codes fail verification. The backend is not replaced with mocks during verification.
- JaCoCo enforces at least **90% service-class line coverage** (and an 80% branch rule where branches exist). Service control flow currently has no instrumented branches, so the branch rule is not evidence of branch coverage. Other classes have reported coverage but no artificial whole-project threshold.

Reports: `backend/target/surefire-reports`, `backend/target/failsafe-reports`, `backend/target/site/jacoco/index.html`. Generated contracts: `frontend/pacts/`. The workflow uploads them even when tests fail.

## 5. GitHub repository and CI execution

After local verification passes, create an empty GitHub repository and push:

```bash
git init
git branch -M main
git add .
git commit -m "Add tested student application and canary pipeline"
git remote add origin git@github.com:YOUR_OWNER/YOUR_REPO.git
git push -u origin main
```

The workflow `.github/workflows/ci-cd.yml` runs for pull requests, pushes to `main`, and manual dispatch. GitHub-hosted Ubuntu runners already provide Docker, so Testcontainers works without separate PostgreSQL/Kafka Actions services. Do **not** run untrusted pull requests using privileged self-hosted production runners.

1. Checkout; install Java 21/Node 22; restore dependency caches.
2. Run backend/frontend unit tests and JaCoCo gate.
3. Run REST/database/Kafka integration tests.
4. Generate Pact consumer contracts and verify the provider using fresh containers.
5. Build frontend/backend artifacts and Docker images (also on pull requests), then run the complete Compose stack and a CRUD smoke test.
6. Only trusted `main` runs publish to GHCR using `GITHUB_TOKEN` with `packages:write`.
7. Deploy the exact resulting image digests to staging; check readiness, API and release version.
8. Deploy those **same digests** to production with canary monitoring and rollback.

A failed job blocks downstream jobs. Pull requests never receive deployment secrets and never publish. Main deployments are serialized, not cancelled midway through a traffic change. The VM also takes an exclusive file lock.

### GHCR configuration

Images are named `ghcr.io/<lower-case-owner>/<lower-case-repo>-backend:<commit-SHA>` and `...-frontend:<commit-SHA>`. No separate registry password is needed in Actions: the workflow uses the built-in short-lived `GITHUB_TOKEN`. Ensure repository Actions policy permits the workflow's package write permission. For an existing package, grant this repository access in the package's Actions settings if needed.

The VM must be able to pull these packages. Either make packages public in GitHub's package settings or provision a dedicated account's **classic PAT with only `read:packages`** and appropriate package access/organization SSO authorization. On the VM, as the deployment user:

```bash
read -rsp 'GHCR read-only PAT: ' GHCR_TOKEN; echo
printf '%s' "$GHCR_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_ACCOUNT --password-stdin
unset GHCR_TOKEN
```

That credential is stored in the VM user's Docker configuration; use a credential helper where possible. Never commit it or put it in the deployment archive. Actions' token is not a permanent VM login token. Builds use tags for discovery, but deployments use `@sha256:...` digests to avoid mutable-tag races. The supplied builds target x86-64 Linux; use an x86-64 VM or add a deliberate multi-platform build.

## 6. External deployment infrastructure — provision once

Use a dedicated **Ubuntu 24.04 x86-64 VM**, recommended 4 vCPU, 8 GB RAM, sufficient disk for two app releases plus PostgreSQL/Kafka. It needs outbound access to GHCR/Docker registries. No cloud-specific provider or Kubernetes is required.

Install Docker/Compose using section 2 and ensure `python3`, `curl`, `flock` (util-linux), OpenSSH server and systemd are present. Java/Node/Maven are not needed on the deployment VM. The gateway uses host port 80, which must be free.

Create a dedicated deployment account and SSH key:

```bash
# On the VM (admin):
sudo adduser --disabled-password --gecos '' deployer
sudo usermod -aG docker deployer
sudo mkdir -p /opt/student-lab
sudo chown deployer:deployer /opt/student-lab

# On your workstation:
ssh-keygen -t ed25519 -f ./student-deploy-key -C student-lab-actions
ssh-copy-id -i ./student-deploy-key.pub deployer@YOUR_VM_HOST
# Test key-based access before continuing.
ssh -i ./student-deploy-key deployer@YOUR_VM_HOST 'docker info >/dev/null'
```

If password login is disabled, install the public key through your cloud console/admin account into `/home/deployer/.ssh/authorized_keys`. Restrict permissions to 700 for `.ssh` and 600 for `authorized_keys`. Never add the private key to Git.

The workflow starts deployments as a systemd service so an SSH/runner disconnect does not terminate a canary halfway through. Configure the following lab sudo rule using `sudo visudo -f /etc/sudoers.d/student-lab`:

```text
deployer ALL=(root) NOPASSWD: /usr/bin/systemd-run
```

**This rule and Docker access are highly privileged.** The workflow specifies `User=deployer`, but the broad sudo rule is not a security boundary against a malicious account. For a real multi-user production host, replace it with a root-owned restricted deployment service/API and hardened permissions.

Copy configuration, create remote secrets, and provision data services:

```bash
# Workstation, in project root:
scp -i ./student-deploy-key -r deploy/. deployer@YOUR_VM_HOST:/opt/student-lab/
ssh -i ./student-deploy-key deployer@YOUR_VM_HOST
# Now on VM:
cd /opt/student-lab
cp .env.example .env
sed -i "s/REPLACE_WITH_RANDOM_PASSWORD/$(openssl rand -hex 24)/" .env
chmod 600 .env
bash bootstrap.sh
# Then configure GHCR login as described above if packages are private.
```

`bootstrap.sh` creates two external networks, starts PostgreSQL/Kafka, waits for them to be healthy, and pre-pulls the probe image. The PostgreSQL init script creates `students_staging` **only on first initialization**. If you reused an older existing volume, create that database explicitly with an administrator before deploying staging. PostgreSQL users/passwords are also initialized only once; changing `.env` alone does not change an existing DB password.

Stateful services have persistent named volumes. Do not run `down -v` on deployment infrastructure unless intentionally deleting its data. Back up the DB before migrations. Staging and production share a broker and database server on this lab VM, but use different databases/topics; this is logical separation, not security/resource isolation.

### Network and exposure

- Allow SSH only from trusted networks or an appropriately controlled Actions runner/VPN. GitHub-hosted runner IPs change; use a secure connectivity strategy rather than permanently opening unrestricted administrative access.
- Allow inbound port 80 for the demonstration, or restrict it to your own IP. DB, Kafka, backend and staging publish **no host ports**.
- Do not expose this unauthenticated demo with real/student data. For actual public production, add TLS at an external load balancer/reverse proxy, authentication, authorization, request limits and backups. A TLS terminator must forward traffic to the gateway, not bypass canary routing. DNS/TLS provisioning is intentionally not automated by this lab.

## 7. GitHub Secrets and deployment environments

In repository **Settings → Environments**, create **staging** and **production**. Restrict production deployment to `main`; optionally add required reviewers. Add these four environment secrets to **both** environments (same VM values for this example):

| Secret | Value |
|---|---|
| `DEPLOY_HOST` | VM DNS hostname or IPv4, not `https://...` |
| `DEPLOY_USER` | `deployer` |
| `DEPLOY_SSH_KEY` | Complete private key contents from `student-deploy-key`, including BEGIN/END lines |
| `DEPLOY_KNOWN_HOSTS` | Verified OpenSSH known_hosts entry/entries for the VM |

Retrieve the SSH host public key/fingerprint through the VM's trusted console:

```bash
# VM console:
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# Workstation: candidate entry, NOT automatically trusted:
ssh-keyscan -t ed25519 YOUR_VM_HOST > vm-known-hosts
ssh-keygen -lf vm-known-hosts
```

Compare fingerprints using the trusted console before storing `vm-known-hosts` contents as `DEPLOY_KNOWN_HOSTS`. The workflow uses strict host-key verification; it never trusts a runtime `ssh-keyscan`. The example uses SSH port 22 and deployment path `/opt/student-lab`; change the scripts deliberately if your infrastructure differs.

Do not put `DB_PASSWORD` in GitHub secrets for this implementation: it stays in `/opt/student-lab/.env` on the VM. `GITHUB_TOKEN` is provided automatically; no manual secret with that name is needed. Use `read:packages` login on the VM only if packages are private.

## 8. Canary behavior and rollback

Deployment files have three roles:

- `infra.compose.yml`: one shared stateful DB/broker stack; manually provisioned, not recreated per release.
- `release.compose.yml`: an isolated frontend/backend pair per `student-blue`, `student-green`, or `student-staging` project, with private release network.
- `gateway.compose.yml` + `render_gateway.py`: stable ingress with generated Nginx `split_clients` policy.

**First production deployment:** there is no previous version. Bootstrap at 100%, check readiness and the observation window. If it fails, remove the unsuccessful stack/gateway; there is no prior release to restore.

**Subsequent deployments:**

1. Keep the active slot running. Replace only the inactive slot with the immutable new images.
2. Compose waits for the candidate frontend's healthcheck. `/health` proxies backend readiness, which checks PostgreSQL **and a real Kafka broker request**.
3. Check the candidate release SHA and read API directly over the private edge network.
4. Route 10% to candidate; observe 30 seconds by default.
5. Repeat at 50% and 100%.
6. Only after the final successful window atomically update `runtime/active-slot`. Keep the old release running for operator rollback; replace it on the next deployment.

Routing is random per request, so percentages are statistical, not exactly 10 out of every 100. The router does **not** transparently retry a failed candidate request on the stable slot, which would hide candidate failures.

During each observation window, `monitor.py` sends two read-only probes every 0.2 seconds, reads **all new gateway log entries** (including real user requests), and fails if any of the following occurs:

- synthetic non-200/error rate exceeds 1%;
- either slot's HTTP 5xx rate exceeds 1% (not just the weighted aggregate);
- fewer than 100 total requests/probes or fewer than 5 candidate requests are observed;
- candidate readiness, API reachability, or release-version verification fails.

`CANARY_WINDOW_SECONDS` can increase observation time (minimum 20); `MAX_ERROR_RATE` changes the error budget. Defaults are appropriate for a short experiment, not statistically rigorous production SLOs. There is no latency regression analysis or external metrics store; add Prometheus/OpenTelemetry and longer traffic-aware windows for real production. Logs need an operational rotation/retention policy.

On failure, the ERR/TERM handler atomically restores the previous Nginx configuration, validates/reloads it, waits for old request workers to drain, and removes the rejected candidate. The deployment returns nonzero, failing the workflow. If router restoration itself fails, the script reports a **critical error** and preserves the candidate for manual recovery rather than removing a possibly still-routed service. A dead VM/Docker daemon, SIGKILL, or incompatible DB migration cannot be repaired by an application rollback trap.

Manual rollback after a successful promotion:

```bash
# VM:
cd /opt/student-lab
bash rollback.sh
cat runtime/active-slot
curl -f http://localhost/api/version
```

### Demonstrate automatic rollback

1. Complete one production deployment successfully (for example, active blue).
2. Push a small frontend change as a second commit. Watch the production Actions logs.
3. While green is receiving canary traffic, use a second VM SSH session:

```bash
docker stop student-green-backend-1
```

4. Readiness/probes/5xx checks should fail; the deployment must fail and restore blue. Verify:

```bash
cat /opt/student-lab/runtime/active-slot
curl -i http://localhost/api/version
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Reverse blue/green names if green was active initially. Do not stop the shared DB/broker to simulate a candidate-only failure: that breaks both versions.

### Deploy manually with the same release mechanism

From the VM, after pushing images and obtaining digests from the Actions build summary/GHCR:

```bash
cd /opt/student-lab
# Replace these values with a real full commit SHA and actual immutable GHCR digests.
RELEASE=YOUR_FULL_40_CHARACTER_GIT_SHA
BACKEND_IMAGE=ghcr.io/your-owner/your-repo-backend@sha256:YOUR_BACKEND_DIGEST
FRONTEND_IMAGE=ghcr.io/your-owner/your-repo-frontend@sha256:YOUR_FRONTEND_DIGEST
bash deploy.sh staging "$RELEASE" "$BACKEND_IMAGE" "$FRONTEND_IMAGE"
bash deploy.sh production "$RELEASE" "$BACKEND_IMAGE" "$FRONTEND_IMAGE"
```

These are clearly marked infrastructure-specific values, not executable placeholder image IDs. Prefer the workflow, which supplies actual SHA/digests automatically. Staging is deliberately private: its health checks execute via a temporary curl container on `student-edge`.

## 9. Important limitations and production decisions

- **Database compatibility:** both releases use the same production database. Flyway is versioned and Hibernate uses `validate`, but canary-safe migrations must be additive (expand/contract). Traffic rollback does not undo a schema migration or recover deleted data. Staging cannot prove compatibility with all existing production data.
- **Kafka delivery:** service writes run inside a database transaction and wait for Kafka acknowledgment. A Kafka failure rolls back the database transaction. This does **not** make DB + Kafka atomic: an event may publish before a later database commit fails. For reliable production integration use a transactional outbox and idempotent consumers. Do not claim exactly-once semantics from this demo.
- **Infrastructure:** one VM, one PostgreSQL instance, one Kafka broker, one router are single points of failure. Production requires replicas/managed services, TLS/SASL for Kafka, durable backups and recovery drills.
- **Security:** the CRUD API has no authentication. Credentials shown in local defaults are for experiments only. Images/action versions are pinned to readable versions, not immutable base/action SHAs; evaluate updates, scan/sign images, generate SBOMs, and pin reviewed digests/SHAs before real production.
- **Canary scope:** automatic rollback applies during deployment observation windows. It is not an always-running incident controller after promotion. An old tab's cached JavaScript may call a newer backend, so preserve backward-compatible API contracts.
- **Contract scope:** this mono-repo verifies this commit's consumer against its provider. Independent frontend/backend releases need versioned consumer compatibility checks via a Pact Broker. `contracts/README.md` explains a negative compatibility experiment.

## 10. File map — purpose of every major file

The complete generated tree is in `PROJECT_TREE.txt`.

| Path | Purpose |
|---|---|
| `backend/pom.xml` | All backend dependencies; Java 21, Surefire/Failsafe selection, Pact profile, JaCoCo gate |
| `backend/src/main/java/com/example/students/Application.java` | Spring Boot entry point |
| `Student.java`, `StudentRequest.java` | Persisted entity and validated input DTO |
| `StudentRepository.java` | Spring Data JPA CRUD |
| `StudentService.java` | Transactional business operations and event publishing |
| `StudentEvents.java` | Kafka topic creation and acknowledged event production |
| `KafkaHealthIndicator.java` | Broker-aware readiness, bounded request timeout and cleanup |
| `StudentController.java` | REST CRUD, HTTP statuses, release endpoint |
| `StudentNotFound.java`, `ApiErrors.java` | Missing-entity exception and stable HTTP error mapping |
| `backend/src/main/resources/application.yml` | Environment-driven database/Kafka/health configuration |
| `.../db/migration/V1__students.sql` | Versioned initial PostgreSQL schema and uniqueness constraint |
| `backend/src/test/.../StudentServiceTest.java` | Service Mockito/JUnit tests and DTO constraint tests |
| `.../ContainerTestBase.java` | Managed temporary PostgreSQL/Kafka containers and dynamic connection properties |
| `.../StudentApiIT.java` | API + database integration checks |
| `.../KafkaIT.java` | Actual producer/consumer verification |
| `.../StudentContractIT.java` | Pact provider states and HTTP verification against live Spring Boot |
| `frontend/package.json`, `package-lock.json` | Exact direct dependencies, reproducible npm dependency graph, scripts |
| `frontend/index.html`, `src/main.js`, `src/style.css` | CRUD UI, safe text rendering, styling |
| `frontend/src/api.js` | Shared real client used by UI, unit tests, and contract tests |
| `frontend/vite.config.js` | Host development API proxy |
| `frontend/test/api.test.js` | Frontend client unit tests |
| `frontend/test/contract.test.js` | Executable consumer contract definitions for CRUD and invalid/missing requests |
| `contracts/example-pact.json`, `contracts/README.md` | Inspectable sample contract and contract-testing explanation |
| `backend/Dockerfile`, `frontend/Dockerfile` | Multi-stage builds; non-root runtime processes |
| `frontend/nginx.conf.template` | SPA hosting, API/readiness proxy to the release-local backend |
| `compose.yml`, `.env.example` | Complete local stack and lab defaults |
| `.github/workflows/ci-cd.yml` | Test/build/publish/staging/production dependency graph |
| `deploy/*.compose.yml` | Shared infrastructure, isolated release pairs, stable ingress |
| `deploy/init-staging.sql` | Separate staging DB on first DB initialization |
| `deploy/.env.example` | Template for VM-only database credentials |
| `deploy/bootstrap.sh` | One-time network/data-service provisioning |
| `deploy/deploy.sh` | Locked staging deployment, weighted promotion and automatic failure rollback |
| `deploy/render_gateway.py` | Concrete Nginx weighted routing policy generator |
| `deploy/monitor.py` | Probe generation, sample sufficiency, per-slot error evaluation |
| `deploy/rollback.sh` | Operator-triggered traffic rollback to previous running release |
| `scripts/remote-deploy.sh` | Strict SSH verification, configuration upload and durable systemd execution |
| `scripts/verify-local.sh`, `scripts/check-stack.sh`, `scripts/smoke.py` | Full local verification and cleanup-safe CRUD smoke check |
| `scripts/test-routing.py` | Offline routing-policy regression assertions |
| `.gitignore`, `.dockerignore` | Exclude credentials, dependencies and generated artifacts |
| `VALIDATION.md` | What was actually run in the authoring workspace, and what was not |

## 11. Troubleshooting

- **Docker unavailable in tests:** run `docker info` as your user, start Docker/Desktop, allow its socket, check WSL integration. Never disable container tests just to get a green build.
- **Pact file missing:** run the frontend contract command first; run provider verification from `backend/` so `../frontend/pacts` resolves correctly.
- **Database login fails after changing `.env`:** existing volumes retain old users/passwords; migrate credentials properly, or reset only disposable local data with `down -v`.
- **Kafka connection refused:** distinguish host `localhost:29092` from Docker `kafka:9092`; Testcontainers supplies its own address automatically.
- **GHCR denied:** check VM login, package visibility/account access, PAT `read:packages`, SSO and repository package permissions. Lower-case image paths.
- **SSH verification failure:** verify your host key out of band and update the secret intentionally. Do not disable host-key checking.
- **Systemd deployment failure:** `sudo journalctl -u 'student-production-*' --no-pager` on the VM; inspect Docker logs and `runtime/active-slot`. Services are collected after exit but their journal records remain.
- **Canary fails on insufficient samples:** inspect gateway logs, resource capacity and configured window; increase observation time rather than bypassing the check.
- **Disk usage grows:** retain current/previous releases, inspect `docker system df`, rotate gateway logs, prune only known-unused images. Never delete active named volumes.
