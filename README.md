# FestPass — Docker & Container Security Assessment

A containerized three-tier web application developed as part of the Docker & Container Security practical assessment (CA-2). The application allows students to register for college technical-fest events and demonstrates containerization, network isolation, persistent storage, vulnerability scanning, image publishing, and container image signing.

## Student Information

| Field                       | Details                                                  |
| --------------------------- | -------------------------------------------------------- |
| Name                        | Harsh Vardhan                                            |
| Registration Number (REGNO) | `202401100100111`                                        |
| Application Port (PORT)     | `8011`                                                   |
| Docker Hub Username         | `harsh403931d`                                           |
| Docker Hub Repository       | https://hub.docker.com/r/harsh403931d/fest-backend       |
| GitHub Repository           | https://github.com/Harshvardhan1123/ca2_docker_container |

## Project Overview

FestPass consists of three services:

* **Frontend:** Nginx serves the registration interface and forwards API requests to the backend.
* **Backend:** A Python Flask API provides health-check and registration endpoints.
* **Database:** PostgreSQL stores event registrations persistently.

The application supports registering participants for Hackathon, Robo Race, and Tech Quiz events.

## Architecture

```text
                   Browser
                      |
                      | http://localhost:8011
                      v
               Frontend (Nginx)
                      |
                frontend_backend
                      |
                      v
               Backend (Flask)
                /           \
               /             \
      frontend_backend    backend_db
                                |
                                v
                         PostgreSQL
                                |
                                v
                       postgres_data volume
```

The frontend and backend share the `frontend_backend` network. The backend and database share the `backend_db` network. The database network is internal, and the frontend is not directly connected to the database network.

PostgreSQL data is stored in the named Docker volume `postgres_data`, allowing registrations to survive container recreation.

## Technology Stack

* Docker and Docker Compose
* Python 3.13.9
* Flask
* PostgreSQL 17 Alpine
* Nginx 1.30 Alpine
* Trivy vulnerability scanner
* Docker Hub
* Cosign for container image signing and verification

## Project Structure

```text
ca2/
├── README.md
├── compose.yaml
├── .env.example
├── .gitignore
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
└── screenshots/
```

Additional files, including `cosign.pub` and evidence screenshots, will be included as the remaining assessment stages are completed.

## Running the Application

### Prerequisites

* Docker Engine or Docker Desktop with Docker Compose
* Git

### 1. Clone the repository

```bash
git clone https://github.com/Harshvardhan1123/ca2_docker_container.git
cd ca2_docker_container
```

### 2. Configure the database password

Create a local `.env` file using the example:

```bash
cp .env.example .env
nano .env
```

Set a strong value for `POSTGRES_PASSWORD`. Do not commit the `.env` file or publish its contents.

### 3. Start the services

```bash
docker compose up -d --build
```

### 4. Check service status

```bash
docker compose ps
```

### 5. Test the backend health endpoint

```bash
curl http://localhost:8011/api/health
```

The response should contain the registration number, application version, and database connection status.

### 6. Open the frontend

Visit:

http://localhost:8011

Submit registrations through the web interface.

### 7. Stop the application

```bash
docker compose down
```

The named PostgreSQL volume is retained when the stack is stopped this way. Do not use `docker compose down -v` if you want to preserve the registration data.

> After Stage 4, the Compose configuration will use the published Docker Hub backend image instead of building the backend locally.

## Container Security

The initial backend image, `fest-backend:0.1`, used the Python 3.9 base image. The hardened image, `fest-backend:0.3`, uses a versioned slim Python base, installs available Debian package updates during the build, upgrades the Python dependencies, and runs the application as a non-root user.

Security improvements include:

* Use of a slim Python base image.
* Installation of available operating-system package updates during image construction.
* Updated Flask and Werkzeug dependencies.
* Installation of Python dependencies without retaining the pip download cache.
* Execution as the unprivileged `appuser` account.
* Removal of unnecessary APT package lists after package installation.
* Vulnerability scanning with Trivy.

### Vulnerability Scan Results

The initial scan of `fest-backend:0.1` reported the following findings:

| Metric                               | Initial image (`fest-backend:0.1`) | Hardened image (`fest-backend:0.3`) |
| ------------------------------------ | ---------------------------------: | ----------------------------------: |
| Critical findings — full scan        |                                234 |    **Update after final full scan** |
| High findings — full scan            |                              2,244 |    **Update after final full scan** |
| Critical findings — assessment check |        Not yet recorded separately |                               **0** |
| Disk usage                           |                            1.61 GB |                              288 MB |
| Runs as non-root                     |                                 No |                                 Yes |

The final image's targeted Trivy scan reported **zero Critical findings**. The assessment check using `--severity CRITICAL --ignore-unfixed --exit-code 1` also exited with code `0`.

The final full-scan High and Critical counts must be recorded from the completed scan report. Image sizes are Docker's reported disk usage and may vary with local image layers.

## Published Image

The secured backend image will be published to Docker Hub using versioned tags.

| Item                   | Value                                          |
| ---------------------- | ---------------------------------------------- |
| Docker Hub repository  | `harsh403931d/fest-backend`                    |
| Release version        | `1.0.0`                                        |
| Additional release tag | `1.0`                                          |
| Image digest           | **To be recorded after publishing in Stage 4** |
| Signature verification | To be documented after Stage 5                 |

The release image will be signed by digest using Cosign. The public verification key, `cosign.pub`, will be included in the repository. The private signing key must never be committed.

## AI Tools Used

**ChatGPT (OpenAI):**

* Assisted with understanding the practical assessment requirements.
* Helped develop and troubleshoot the Dockerfile and Docker Compose configuration.
* Explained Trivy vulnerability reports and suggested image-hardening improvements.
* Assisted with command-line troubleshooting and README documentation.

All commands and tests are executed on my own development environment.

## Assessment Progress

* [x] Stage 1 — Containerize the backend.
* [x] Stage 2 — Run the three-tier Compose stack with network isolation and persistent database storage.
* [x] Stage 3 — Scan the initial image, harden the backend image, and pass the targeted Critical-vulnerability check.
* [ ] Stage 4 — Publish versioned backend images to Docker Hub, record the `1.0.0` digest, and verify the application using the registry image.
* [ ] Stage 5 — Sign version `1.0.0` with Cosign, verify the signature, and demonstrate rejection of unsigned version `1.0.1`.

## Incomplete Steps

* Stage 4: Docker Hub publishing, recording the final image digest, and registry-based deployment verification are pending.
* Stage 5: Cosign signing, signature verification, and unsigned-image rejection are pending.

Update this section and the published-image details after completing the remaining stages.
