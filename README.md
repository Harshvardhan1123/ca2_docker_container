# Docker & Container Security — Practical Assessment

## Student Information

| Field                 | Details                                                                          |
| --------------------- | -------------------------------------------------------------------------------- |
| Student Name          | Harsh Vardhan                                                                    |
| Registration Number   | 202401100100111                                                                  |
| Project               | FestPass — Containerized Event Registration Application                          |
| Application Port      | 8011                                                                             |
| Docker Hub Username   | `harsh403931d`                                                                   |
| Docker Hub Repository | [harsh403931d/fest-backend](https://hub.docker.com/r/harsh403931d/fest-backend)  |
| GitHub Repository     | [ca2_docker_container](https://github.com/Harshvardhan1123/ca2_docker_container) |

---

## 1. Project Overview

FestPass is a containerized event registration application developed to demonstrate Docker containerization, multi-container orchestration, network isolation, persistent storage, vulnerability scanning, image publishing, and cryptographic image signing.

The application uses a three-tier architecture:

* **Frontend:** Nginx serves the event registration interface.
* **Backend:** A Python Flask API handles health checks and event registrations.
* **Database:** PostgreSQL stores registration records persistently.

The application supports event registrations for Hackathon, Robo Race, and Tech Quiz.

The complete stack is managed using Docker Compose.

### Application URLs

| Endpoint      | URL                                       | Purpose                         |
| ------------- | ----------------------------------------- | ------------------------------- |
| Frontend      | `http://localhost:8011`                   | Event registration interface    |
| Health Check  | `http://localhost:8011/api/health`        | Application and database health |
| Registrations | `http://localhost:8011/api/registrations` | Retrieve saved registrations    |

---

## 2. Technology Stack

* Docker and Docker Compose
* Python 3.13.9
* Flask
* PostgreSQL 17 Alpine
* Nginx 1.30 Alpine
* Trivy
* Docker Hub
* Cosign
* Git and GitHub
* Ubuntu on WSL

---

## 3. Project Structure

```text
ca2/
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── frontend/
│   ├── index.html
│   └── nginx.conf
├── reports/
│   ├── trivy-before.txt
│   ├── trivy-after.txt
│   └── trivy-final.txt
├── screenshots/
│   └── Assessment evidence screenshots
├── compose.yaml
├── .env.example
├── .gitignore
├── cosign.pub
└── README.md
```

**Security note:** The private signing key (`cosign.key`) and `.env` file must not be committed to GitHub. The public verification key (`cosign.pub`) is included for signature verification.

---

## 4. Stage 1 — Backend Containerization

The Flask backend was containerized using a Dockerfile.

The container exposes port 5000 internally and provides API endpoints for health checks and registration management.

### Build and Run

```bash
docker build -t fest-backend:0.1 ./backend

docker run -d \
  --name fest-backend-test \
  -p 8011:5000 \
  fest-backend:0.1
```

### Health Check

```bash
curl http://localhost:8011/api/health
```

The endpoint was tested to confirm that the backend responded successfully.

The temporary standalone test container was removed after testing.

---

## 5. Stage 2 — Multi-Container Application with Docker Compose

The application was extended into a three-tier Docker Compose stack.

### Services

| Service  | Image                             | Responsibility                                     |
| -------- | --------------------------------- | -------------------------------------------------- |
| Frontend | `nginx:1.30-alpine`               | Serves the user interface and proxies API requests |
| Backend  | `harsh403931d/fest-backend:1.0.0` | Handles application logic and API requests         |
| Database | `postgres:17-alpine`              | Stores registration records                        |

### Network Isolation

Two bridge networks are configured:

1. `frontend_backend`

   * Connects the frontend to the backend.
   * Allows API requests to reach the backend.

2. `backend_db`

   * Connects the backend to PostgreSQL.
   * Configured as an internal network to restrict external connectivity.

The frontend is not directly connected to the database network.

### Persistent Storage

A named Docker volume, `postgres_data`, stores PostgreSQL data.

This allows registration records to survive container removal and recreation.

### Start the Application

```bash
docker compose up -d
```

### Verify Services

```bash
docker compose ps
```

### Verify Health

```bash
curl http://localhost:8011/api/health
```

### Verify Registrations

```bash
curl http://localhost:8011/api/registrations
```

The database health check ensures that PostgreSQL is ready before the backend starts.

---

## 6. Stage 3 — Container Vulnerability Scanning with Trivy

Trivy was used to scan the backend container image for operating-system and Python package vulnerabilities.

### Initial Image

The initial image was `fest-backend:0.1`.

The initial scan reported:

| Metric                     | Initial Image |
| -------------------------- | ------------: |
| Total findings             |        13,357 |
| Critical findings          |           234 |
| High findings              |         2,244 |
| Docker-reported disk usage |       1.61 GB |

### Security Improvements

The backend image was hardened through the following changes:

* Updated the Python base image to `python:3.13.9-slim-trixie`.
* Applied Debian package security updates during the image build.
* Removed package-manager lists after installation.
* Created a dedicated non-root application user.
* Configured the container to run as `appuser`.
* Disabled Python bytecode generation.
* Enabled unbuffered Python output.
* Disabled pip's package cache.
* Upgraded pip and installed the declared application dependencies.

### Final Image

The final hardened image is `fest-backend:0.3`.

The completed full scan reported:

| Metric                          | Final Image |
| ------------------------------- | ----------: |
| High findings — Debian OS       |          44 |
| High findings — Python packages |           5 |
| **Total High findings**         |      **49** |
| **Total Critical findings**     |       **0** |
| Docker-reported disk usage      |      288 MB |
| Runs as non-root                |         Yes |

The final image reduced Docker-reported disk usage from 1.61 GB to 288 MB.

The full scan still reported High-severity vulnerabilities. Therefore, the image should not be described as vulnerability-free.

### Critical Vulnerability Check

The following command checks for Critical vulnerabilities while ignoring unfixed findings:

```bash
trivy --timeout 15m image \
  --scanners vuln \
  --severity CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  fest-backend:0.3
```

The completed check reported zero Critical findings and exited with code `0`.

### Reports

The following scan reports are included in the repository:

* `reports/trivy-before.txt`
* `reports/trivy-after.txt`
* `reports/trivy-final.txt`

---

## 7. Stage 4 — Docker Hub Publishing and Deployment

The hardened application image was built with application version `1.0.0`.

### Published Images

Docker Hub repository:

`harsh403931d/fest-backend`

Published tags:

| Tag     | Purpose                                                       |
| ------- | ------------------------------------------------------------- |
| `1.0.0` | Versioned release                                             |
| `1.0`   | Additional release tag                                        |
| `1.0.1` | Unsigned image used for the negative Cosign verification test |

The `1.0` and `1.0.0` tags refer to the same published image digest.

### Release Image Digest

The published `1.0.0` image has the following registry digest:

```text
sha256:9fd56f87d3e443d67d639db47547607233a040569a61d50199c7e112a0737f7a
```

### Deploying the Published Image

The backend service in `compose.yaml` uses the published Docker Hub image:

```yaml
backend:
  image: harsh403931d/fest-backend:1.0.0
  environment:
    REGNO: "202401100100111"
    APP_VERSION: "1.0.0"
    DB_HOST: db
    DB_NAME: festpass
    DB_USER: festpass
    DB_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
  networks:
    - frontend_backend
    - backend_db
  depends_on:
    db:
      condition: service_healthy
  restart: unless-stopped
```

The backend's local `build:` configuration was removed from the Compose service.

The local backend images and tags were removed before pulling the published release.

### Pull and Deploy

```bash
docker compose pull backend

docker compose up -d

docker compose ps
```

### Deployment Verification

Health endpoint:

```bash
curl -s http://localhost:8011/api/health
```

Verified response:

```json
{
  "container": "8d44b66ef38c",
  "database": true,
  "regno": "202401100100111",
  "service": "festpass-backend",
  "version": "1.0.0"
}
```

The container identifier may differ after subsequent deployments.

### Persistence Verification

The following registrations were retrieved after redeployment:

```json
[
  {
    "event": "Hackathon",
    "id": 1,
    "name": "Harsh Vardhan"
  },
  {
    "event": "Tech Quiz",
    "id": 2,
    "name": "harshtest"
  }
]
```

This confirmed that the existing registration records remained available after the stack was stopped, its containers were removed, and the published image was deployed.

---

## 8. Stage 5 — Docker Image Signing with Cosign

Cosign was used to sign the published release image and verify its signature.

The signing operation uses the immutable image digest rather than relying solely on a mutable tag.

### Key Generation

A Cosign key pair was generated using:

```bash
cosign generate-key-pair
```

The generated files are:

* `cosign.key` — Private signing key.
* `cosign.pub` — Public verification key.

The private key was restricted to owner-only permissions:

```bash
chmod 600 cosign.key
```

The private key is excluded from Git using `.gitignore`.

**Security requirement:** Never commit, publish, or share `cosign.key` or its password. Only `cosign.pub` is included in the submission.

### Signing the Release Image

The signed release image is:

```text
docker.io/harsh403931d/fest-backend@sha256:9fd56f87d3e443d67d639db47547607233a040569a61d50199c7e112a0737f7a
```

The image was signed using:

```bash
cosign sign --key cosign.key "$IMAGE"
```

The signature was successfully published to the container registry.

### Signature Verification

The signature was verified using the public key:

```bash
cosign verify --key cosign.pub "$IMAGE"
```

The verification output confirmed that:

* The signature was verified against the specified public key.
* The Cosign claims were validated.
* The transparency-log entry was verified offline.

**Result: Signature verification passed.**

### Negative Test — Unsigned Image

A separate image, version `1.0.1`, was published without a Cosign signature.

Its registry digest is:

```text
sha256:32750b36eaec5893a572b3645e4fd3f23a23fcebd444f1dbd2ef300177c852c5
```

The unsigned image was tested with:

```bash
export UNSIGNED_IMAGE="docker.io/harsh403931d/fest-backend@sha256:32750b36eaec5893a572b3645e4fd3f23a23fcebd444f1dbd2ef300177c852c5"

cosign verify --key cosign.pub "$UNSIGNED_IMAGE"
```

The command returned:

```text
Error: no signatures found
error during command execution: no signatures found
```

This was the expected result because the image had not been signed.

**Result: The unsigned-image verification test behaved as expected.**

---

## 9. Security Controls

The following security measures were implemented:

* Multi-container architecture with separate services.
* Network isolation between frontend and database.
* Internal backend-to-database network.
* Persistent PostgreSQL storage using a named volume.
* Non-root backend execution.
* Operating-system package updates during image creation.
* Trivy vulnerability scanning.
* Docker Hub image publishing with versioned tags.
* Cosign image signing and public-key verification.
* `.env` excluded from Git.
* Private signing key excluded from Git.
* Public verification key included in the submission.

### Environment Variables

The application uses environment variables for database credentials.

The actual `.env` file is local and must not be committed.

The `.env.example` file contains a placeholder rather than the actual database password.

---

## 10. Useful Commands

### Start the Stack

```bash
docker compose up -d
```

### Stop the Stack

```bash
docker compose down
```

Do not use `docker compose down -v` when the existing database records must be preserved.

### Check Container Status

```bash
docker compose ps
```

### Check Logs

```bash
docker compose logs --tail=100
```

### Check Application Health

```bash
curl -s http://localhost:8011/api/health
```

### List Registrations

```bash
curl -s http://localhost:8011/api/registrations
```

### Verify the Published Image Signature

```bash
cosign verify --key cosign.pub \
  docker.io/harsh403931d/fest-backend@sha256:9fd56f87d3e443d67d639db47547607233a040569a61d50199c7e112a0737f7a
```

---

## 11. Assessment Evidence

The `screenshots/` directory should contain screenshots demonstrating the completed stages.

Recommended evidence:

| Screenshot                        | Evidence                                   |
| --------------------------------- | ------------------------------------------ |
| `S1-backend-containerization.png` | Backend image, container, and health check |
| `S2-network-isolation.png`        | Docker Compose network isolation           |
| `S2-persistence.png`              | Registration records and persistent volume |
| `S3-trivy-before.png`             | Initial vulnerability scan                 |
| `S3-trivy-after.png`              | Final vulnerability scan                   |
| `S4-1-dockerhub-push.png`         | Successful image publishing and digest     |
| `S4-2-dockerhub-tags.png`         | Docker Hub Tags page                       |
| `S4-3-deployment.png`             | Published image deployed with Compose      |
| `S4-3-persistence.png`            | Existing registrations after redeployment  |
| `S5-3-signing.png`                | Successful image signing                   |
| `S5-4-verification.png`           | Successful signature verification          |
| `S5-5-unsigned-rejected.png`      | Expected rejection of unsigned image       |

Only screenshots that have actually been captured should be included in the final submission.

---

## 12. Assessment Progress

| Stage   | Description                                 | Result    |
| ------- | ------------------------------------------- | --------- |
| Stage 1 | Backend containerization                    | Completed |
| Stage 2 | Docker Compose, networking, and persistence | Completed |
| Stage 3 | Trivy scanning and image hardening          | Completed |
| Stage 4 | Docker Hub publishing and deployment        | Completed |
| Stage 5 | Cosign signing and verification             | Completed |

---

## 13. AI Assistance

AI assistance was used to support the implementation, debugging, command preparation, and documentation of the practical.

The container configuration, image publishing, vulnerability scans, deployment checks, persistent-data verification, and Cosign operations were executed and tested in the working environment.

---

## 14. Conclusion

This practical demonstrates a complete container security workflow, from backend containerization to deployment of a published Docker image.

Docker Compose provides service orchestration, network separation, and persistent storage. Trivy identifies image vulnerabilities and helps measure the effects of image hardening. Docker Hub provides versioned image distribution, while Cosign provides cryptographic image signing and verification.

The final published release was deployed successfully, the existing registrations remained available, and Cosign successfully verified the signed release while rejecting the unsigned test image.

The final image scan still reported 49 High-severity findings, so additional vulnerability remediation and ongoing image maintenance remain important.
