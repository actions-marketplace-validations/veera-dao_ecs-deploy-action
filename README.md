# ECS Deploy Action

A reusable GitHub Action to build a Docker image, push to Amazon ECR, and deploy to multiple ECS services.

## Features

- Build and push Docker images to ECR
- Deploy to multiple ECS services (each with its own task definition)
- Update multiple containers per task definition
- Optionally update version file from git tags
- Slack notifications
- Multi-arch builds via buildx

## Usage

### Beta Deployment (on push to main)

```yaml
name: Deploy to Beta

on:
  push:
    branches: [main]

jobs:
  deploy:
    environment: beta
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/ecs-deploy-action@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          ecr-repository: my-app
          ecs-cluster: beta-cluster
          services: |
            [
              {"task_def": "api-task", "service": "api-svc", "containers": ["api"]},
              {"task_def": "worker-task", "service": "worker-svc", "containers": ["worker"]}
            ]
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel: deployments-beta
```

### Production Deployment (on version tag)

```yaml
name: Deploy to Production

on:
  push:
    tags: ['v*']

permissions:
  contents: write

jobs:
  deploy:
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.REPO_PUSH_TOKEN }}

      - uses: your-org/ecs-deploy-action@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          ecr-repository: my-app
          ecs-cluster: prod-cluster
          services: |
            [
              {"task_def": "api-task", "service": "api-svc", "containers": ["api"]},
              {"task_def": "consumer-task", "service": "consumer-svc", "containers": ["consumer"]}
            ]
          update-version-file: 'true'
          slack-bot-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel: deployments-prod
```

### Multiple Containers (Same Image)

When all containers use the built image:

```yaml
services: |
  [
    {
      "task_def": "web-task",
      "service": "web-svc",
      "containers": ["app", "worker", "sidecar"]
    }
  ]
```

### Multiple Containers (Different Images)

When containers need different images:

```yaml
services: |
  [
    {
      "task_def": "web-task",
      "service": "web-svc",
      "containers": [
        {"name": "app"},
        {"name": "nginx", "image": "nginx:1.25-alpine"},
        {"name": "datadog", "image": "datadog/agent:7"},
        {"name": "fluentbit", "image": "skip"}
      ]
    }
  ]
```

- `{"name": "app"}` → uses the built image
- `{"name": "nginx", "image": "nginx:1.25"}` → uses custom image
- `{"name": "fluentbit", "image": "skip"}` → container is not updated

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `aws-access-key-id` | AWS Access Key ID | Yes | - |
| `aws-secret-access-key` | AWS Secret Access Key | Yes | - |
| `aws-region` | AWS Region | Yes | - |
| `ecr-repository` | ECR Repository name | Yes | - |
| `ecs-cluster` | ECS Cluster name | Yes | - |
| `services` | JSON array of services (see format below) | Yes | - |
| `dockerfile-path` | Path to Dockerfile context | No | `.` |
| `docker-build-args` | Docker build arguments | No | `''` |
| `docker-platform` | Docker platform | No | `linux/arm64` |
| `image-tag` | Image tag | No | `github.sha` |
| `wait-for-service-stability` | Wait for ECS stability | No | `true` |
| `update-version-file` | Update version.txt from tag | No | `false` |
| `version-file-path` | Path to version file | No | `version.txt` |
| `version-target-branch` | Branch to push version to | No | `main` |
| `slack-bot-token` | Slack Bot Token | No | `''` |
| `slack-channel` | Slack channel for notifications | No | `''` |

### Services JSON Format

**Simple format** (all containers use the built image):
```json
[
  {
    "task_def": "task-definition-family-name",
    "service": "ecs-service-name",
    "containers": ["container1", "container2"]
  }
]
```

**Advanced format** (different images per container):
```json
[
  {
    "task_def": "task-definition-family-name",
    "service": "ecs-service-name",
    "containers": [
      {"name": "app"},
      {"name": "nginx", "image": "nginx:latest"},
      {"name": "redis", "image": "redis:7-alpine"},
      {"name": "unchanged", "image": "skip"}
    ]
  }
]
```

| Container Format | Behavior |
|-----------------|----------|
| `"container-name"` | Uses built image |
| `{"name": "x"}` | Uses built image |
| `{"name": "x", "image": "foo:tag"}` | Uses specified image |
| `{"name": "x", "image": "skip"}` | Container not updated |

## Outputs

| Output | Description |
|--------|-------------|
| `image` | Full ECR image URI that was built and pushed |
| `version` | Version from git tag (if `update-version-file` is enabled) |

## Required AWS Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
        }
      }
    }
  ]
}
```

## License

MIT
