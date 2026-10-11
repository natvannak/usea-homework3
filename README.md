# Homework 3
## Release Management with GitHub Tags, Jenkins, ECR and Docker Swarm

## Architecture

GitHub
→ Jenkins
→ Docker
→ Amazon ECR
→ EC2 Docker Swarm

## Versioning

This project uses Semantic Versioning:

vMAJOR.MINOR.PATCH

Releases:

- v1.0.0
- v1.1.0
- v1.2.0

## Release

Create tag:

git tag -a v1.2.0 -m "Release v1.2.0"

Push:

git push origin v1.2.0

The GitHub tag webhook triggers Jenkins.

## Jenkins Actions

ACTION:

- build
- push
- deploy
- rollback

VERSION:

Example:

v1.0.0

## Rollback

Example:

ACTION=rollback
VERSION=v1.0.0

Jenkins checks that the image exists in Amazon ECR and deploys the existing image to Docker Swarm without rebuilding.

## ECR

Repository:

usea-homework3

Images:

v1.0.0
v1.1.0
v1.2.0

## Deployment History

| Version | Action   | Result  |
|---------|----------|---------|
| v1.0.0  | Deploy   | Success |
| v1.1.0  | Deploy   | Success |
| v1.2.0  | Deploy   | Success |
| v1.0.0  | Rollback | Success |

## Live Deployment

http://54.82.194.125