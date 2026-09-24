# GitHub Actions Training Quiz

[![CI/CD](https://github.com/xucf1/github-actions-quiz/actions/workflows/ci-cd.yml/badge.svg?branch=main&event=push)](https://github.com/xucf1/github-actions-quiz/actions/workflows/ci-cd.yml)

A simple Python Flask backend with a GitHub Actions pipeline for building, packaging, and distributing a Docker image.

The pipeline builds the image once in CI, publishes it to Docker Hub, and passes the same image to separate DEV and PROD jobs through a GitHub Actions artifact.

> **Scope:** The CD phase demonstrates image artifact transfer, environment-specific retagging, and local image inspection. It does not deploy a running service to an external server.

## Repository Links

- **GitHub:** [xucf1/github-actions-quiz](https://github.com/xucf1/github-actions-quiz)
- **GitHub Actions:** [Workflow runs](https://github.com/xucf1/github-actions-quiz/actions/workflows/ci-cd.yml)
- **Docker Hub:** [chengfe/github-actions-quiz](https://hub.docker.com/r/chengfe/github-actions-quiz)

## Project Structure

```text
github-actions-quiz/
├── .github/
│   └── workflows/
│       └── ci-cd.yml       # CI and environment-specific CD jobs
├── .dockerignore          # Files excluded from the Docker build context
├── Dockerfile             # Container image definition
├── README.md              # Project documentation
├── app.py                 # Flask backend
├── pyproject.toml         # Python project and packaging configuration
└── requirements.txt       # Application dependencies
```

## Backend Application

The application uses Flask and listens on port `8000`. Python `3.12` is used in both the CI setup and the Dockerfile.

| Endpoint | Expected response |
|---|---|
| `GET /` | `{"message":"GitHub Actions Training Quiz"}` |
| `GET /health` | `{"status":"ok"}` |

The application uses Flask's built-in server for this training exercise.

## Required Configuration

### Repository Variable and Secret

Configure these under:

**Repository → Settings → Secrets and variables → Actions**

| Type | Name | Value or purpose |
|---|---|---|
| Repository Variable | `DOCKER_USERNAME` | `chengfe` — the Docker Hub username |
| Repository Secret | `DOCKER_PASSWORD` | A Docker Hub Personal Access Token with `Repo Read & Write` permissions |

The workflow reads these values using:

```yaml
username: ${{ vars.DOCKER_USERNAME }}
password: ${{ secrets.DOCKER_PASSWORD }}
```

Although the secret is named `DOCKER_PASSWORD`, its value is a Docker Hub Personal Access Token, not the account password.

Never put the token in source code, the Dockerfile, or this README. Do not add it to the environment variables printed by the workflow.

### GitHub Environments

Configure two environments under:

**Repository → Settings → Environments**

Each environment has its own variable named `suffix`.

| Environment | Variable name | Value |
|---|---|---|
| `DEV` | `suffix` | `dev` |
| `PROD` | `suffix` | `prod` |

Each CD job is associated with its corresponding GitHub Environment and reads the value through `${{ vars.suffix }}`.

## Workflow Overview

Workflow file:

```text
.github/workflows/ci-cd.yml
```

The workflow runs automatically on every push to the `main` branch:

```yaml
on:
  push:
    branches:
      - main
```

This includes changes made and committed directly through the GitHub web interface.

```text
Push to main
     |
     v
CI: Install, compile, package, and build
     |
     +--> Tag and push the image to Docker Hub
     |
     +--> Save image.tar and upload the artifact
                  |
          +-------+-------+
          |               |
          v               v
       CD - DEV        CD - PROD
       Download        Download
       Load image      Load image
       Add -dev tag    Add -prod tag
       List images     List images
```

Both CD jobs depend on successful completion of CI. They do not depend on each other.

## CI Phase

| Stage | Implementation |
|---|---|
| Print environment variables | Runs `printenv \| sort` |
| Install dependencies | Installs the application dependencies and the Python `build` package |
| Compile source code | Runs `python -m compileall app.py` |
| Package the Python project | Runs `python -m build` and lists the generated files in `dist/` |
| Build the Docker image | Builds the image using the root-level `Dockerfile` |
| Generate the image version | Creates a UTC timestamp after the Docker build succeeds |
| Tag the image | Tags the built image as `DOCKER_USERNAME/github-actions-quiz:<timestamp>` |
| Authenticate and publish | Uses `docker/login-action` and pushes the timestamp-tagged image to Docker Hub |
| Export the image | Runs `docker save` to create `image.tar` |
| Upload the artifact | Uses `actions/upload-artifact` to upload `image.tar` as `docker-image-tar` |
| Display the final version | Prints the image reference in the logs and writes it to `$GITHUB_STEP_SUMMARY` |

Python packaging produces a wheel and a source distribution in `dist/`. The Dockerfile separately builds the container from `app.py` and `requirements.txt`; it does not install the generated wheel.

The artifact transferred to the CD jobs is the Docker image archive, not the Python package.

### Timestamp Format

The timestamp is generated with:

```bash
date -u +"%Y%m%d%H%M%S"
```

The resulting tag follows the required `YYYYmmddHHMMSS` format and uses UTC.

Example:

```text
chengfe/github-actions-quiz:20260924094249
```

## CD Phase

A matrix creates separate CD jobs for `DEV` and `PROD`.

Each job:

1. Downloads `docker-image-tar` using `actions/download-artifact`.
2. Loads `downloaded-image/image.tar` into its local Docker image store using `docker load`.
3. Adds a new tag by appending `-` and the environment's `suffix` value to the original image tag.
4. Prints the local Docker image list and writes the retagged image reference to the job summary.

The resulting image references follow this pattern:

```text
Original: chengfe/github-actions-quiz:<timestamp>
DEV:      chengfe/github-actions-quiz:<timestamp>-dev
PROD:     chengfe/github-actions-quiz:<timestamp>-prod
```

DEV and PROD run on separate runners. Each job has its own local Docker image store.

The environment-specific tags are created locally and are **not pushed to Docker Hub**. The quiz requires the CD jobs to retag and list images, rather than publish the new tags or start a persistent service.

## Artifacts and Run Results

| Output | Location |
|---|---|
| Published timestamp-tagged image | Docker Hub repository → **Tags** |
| Docker image archive | Workflow run → **Artifacts** → `docker-image-tar` |
| Original image version | Workflow run → **Summary** → **Final Image Version** |
| DEV retagging result | **CD - DEV** → **Retag image with environment suffix** |
| PROD retagging result | **CD - PROD** → **Retag image with environment suffix** |
| Local image inventory | Each CD job → **List all local Docker images** |

The uploaded artifact contains:

```text
image.tar
```

## Verification Record

The first successful training run produced the following tags for `chengfe/github-actions-quiz`:

| Stage | Tag | Verified location |
|---|---|---|
| CI | `20260924094249` | Actions summary and Docker Hub |
| DEV | `20260924094249-dev` | DEV job logs and local image list |
| PROD | `20260924094249-prod` | PROD job logs and local image list |

In both CD jobs, the original tag and the environment-specific tag pointed to the same Docker Image ID:

```text
cd13b16df07d
```

This confirmed that the jobs reused the same image contents and added new tags without rebuilding the application.

These values document the first successful run. Later runs generate timestamps at build time; check their summaries for the corresponding image references.

## Run Locally with Docker

The following commands are an optional manual check. They are not additional steps in the GitHub Actions workflow.

With Git and Docker installed, clone the repository and build the image:

```bash
git clone https://github.com/xucf1/github-actions-quiz.git
cd github-actions-quiz

docker build -t github-actions-quiz:local .
```

Start the application:

```bash
docker run --rm -p 8000:8000 github-actions-quiz:local
```

In another terminal, check the health endpoint:

```bash
curl http://localhost:8000/health
```

Expected response:

```json
{"status":"ok"}
```

The application is also accessible at:

```text
http://localhost:8000/
```

Press `Ctrl+C` in the terminal running the container to stop it.

## Implementation Notes

### Build once, reuse across environments

The image is built once in CI. Both CD jobs consume the same saved image archive instead of independently rebuilding the application.

### Metadata and image contents are transferred separately

Job outputs pass the original image reference from CI to the CD jobs. The artifact carries the actual Docker image archive.

Downloading `image.tar` only restores a file. Each CD job must run `docker load` before the image can be retagged locally.

### Environment configuration stays outside the workflow logic

DEV and PROD use the same CD job definition. Their different suffix values come from GitHub Environment variables rather than separate hardcoded retagging commands.

### Retagging is not deployment

Adding a tag changes how an image is referenced; it does not start the application. This project deliberately keeps the CD implementation within the image-handling scope required by the training quiz.
