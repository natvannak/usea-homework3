# Release Management

## Versioning Convention

This project uses Semantic Versioning:

vMAJOR.MINOR.PATCH

Examples:

- v1.0.0
- v1.1.0
- v1.2.0

## Release Process

1. Make changes to the project.
2. Commit the changes.
3. Push the changes to GitHub.
4. Create a Git tag.
5. Push the tag to GitHub.
6. GitHub webhook triggers Jenkins.
7. Jenkins builds the Docker image.
8. Jenkins pushes the image to Amazon ECR.
9. Jenkins deploys the image to Docker Swarm.

Example:

git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

## Rollback

To rollback:

1. Open Jenkins.
2. Run the pipeline manually.
3. Select:

ACTION=rollback

4. Enter the previous version.

Example:

VERSION=v1.0.0

5. Jenkins verifies that the image exists in ECR.
6. Jenkins deploys the existing image to Docker Swarm.
7. No rebuilding is performed.