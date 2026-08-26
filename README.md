# seated-resuable-workflows

main branch would be the latest version across environments

## Workflows

| File | For |
|---|---|
| `nextjs-ecs.yml` | Next.js apps on ECS Fargate — seated-web, restaurant-portal, seated-lab. One file, `environment` input. See [docs/nextjs-ecs.md](docs/nextjs-ecs.md). |
| `staging.yml`, `production.yml`, `uat.yml` | Java/Maven + Jib on ECS. |
| `staging-java-17.yml`, `production-java-17.yml`, `uat-java-17.yml` | Java 17 variants of the above. |
| `lambda-py.yml`, `lambda-py38.yml`, `lambda-py-docker.yml` | Python Lambdas. |
| `notify-slack.yml` | Standalone Slack thread notifier. |

Changelog:
- 2026-08-26:
1. `nextjs-ecs.yml` — consolidates the six hand-maintained workflow files across
   seated-web, restaurant-portal and seated-lab into one reusable pipeline
   (Test → Build → gated Deploy, with cyberdog notifications)
- 2022-06-12:
1. build and unit test
2. multi-region deployment
3. slack notification
