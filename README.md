# GitHub Actions Training Quiz

A simple Python Flask backend project used for GitHub Actions CI/CD training.

## CI

On every push to the `main` branch, the workflow:

- Prints environment variables
- Installs dependencies
- Compiles and packages the Python project
- Builds a Docker image
- Tags the image with a `YYYYmmddHHMMSS` timestamp
- Pushes the image to Docker Hub
- Saves the Docker image as a `.tar` archive
- Uploads the archive as a GitHub Actions artifact
- Prints the final image version in the workflow summary

## CD

The workflow deploys to two GitHub Environments:

- DEV (`suffix=dev`)
- PROD (`suffix=prod`)

Each CD job downloads the Docker image artifact, loads it, retags it with the environment suffix, and lists the local Docker images.

## Docker Hub

Image repository:

`chengfe/github-actions-quiz`
