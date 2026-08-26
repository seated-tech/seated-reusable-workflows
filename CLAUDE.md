# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains reusable GitHub Actions workflows for Seated's deployment pipelines. These workflows handle:
- Java/Maven application deployments to AWS ECS
- Python Lambda function deployments
- Slack notifications for deployment status

## Key Architecture

### Workflow Types

0. **Next.js ECS Workflow** (`nextjs-ecs.yml`)
   - The shared pipeline for every Next.js app on ECS Fargate: seated-web,
     restaurant-portal, seated-lab
   - ONE file parameterised by an `environment` input, not a per-environment split
   - Test (lint/typecheck/unit/e2e) → Build (docker + ECR) → gated Deploy (ECS)
   - Full reference and copy-paste callers: `docs/nextjs-ecs.md`
   - Note its `container_name` default is `<repo>`, NOT `<repo>-<env>`. The
     web-services terraform module does not env-suffix the container name; the
     Java workflows' suffixed form fails these task definitions outright.

1. **ECS Deployment Workflows** (staging.yml, production.yml, and Java-17 variants)
   - Build Java applications using Maven and Jib
   - Deploy Docker containers to AWS ECS
   - Support multi-region deployments
   - Include Slack notifications at each stage

2. **Lambda Deployment Workflows** (lambda-py.yml, lambda-py38.yml, lambda-py-docker.yml)
   - Package Python Lambda functions
   - Deploy to AWS Lambda
   - Support both zip and Docker-based deployments

3. **Notification Workflow** (notify-slack.yml)
   - Reusable Slack notification system
   - Creates or updates thread messages
   - Integrates with cyberdog.seatedapp.io API

### Deployment Strategy

- **Staging**: Deploys from `master` branch with `-staging` suffix
- **Production**: Deploys from `release` branch with `-production` suffix
- **UAT**: Separate workflow for user acceptance testing environment

### Common Patterns

All workflows use:
- `workflow_call` trigger for reusability
- Semantic versioning with automatic Git tagging
- AWS authentication via GitHub OIDC
- Slack notifications for build/deploy status
- Multi-region deployment support

## Common Development Tasks

Since this is a workflow repository, there are no traditional build/test commands. When modifying workflows:

1. Test workflow changes in a separate repository that uses these workflows
2. Ensure all required secrets are documented in workflow inputs
3. Maintain consistency in naming patterns (e.g., `app-name-environment`)
4. Update both staging and production workflows when making changes

## Workflow Dependencies

Required GitHub secrets for repositories using these workflows:
- `SLACK_API`: Slack API endpoint
- `AWS_*`: AWS credentials and configuration
- `DOCKERHUB_*`: Docker Hub credentials (for Java workflows)
- Environment-specific secrets (regions, channels, etc.)