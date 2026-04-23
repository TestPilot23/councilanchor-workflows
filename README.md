# councilanchor-workflows

Reusable GitHub Actions workflows shared across all CouncilAnchor service repositories.

## Available Workflows

### `deploy-lambda.yml`

Builds and deploys a Go Lambda service.

**Inputs:**

| Input | Required | Description |
|---|---|---|
| `service-name` | Yes | e.g. `council`, `member` |
| `environment` | Yes | `dev` or `prod` |
| `aws-region` | No | Default: `us-east-1` |

**Secrets:**

| Secret | Description |
|---|---|
| `aws-role-arn` | IAM role ARN for OIDC authentication |

**Pipeline stages:**
1. Unit tests (`go test ./... -race`)
2. Build arm64 Lambda binary (`GOOS=linux GOARCH=arm64`)
3. Zip and upload artifact
4. Deploy via `aws lambda update-function-code`
5. Smoke test (`/health` invocation must return 200)

**Usage from a service repo:**

```yaml
jobs:
  deploy:
    uses: councilanchor/councilanchor-workflows/.github/workflows/deploy-lambda.yml@main
    with:
      service-name: council
      environment: dev
    secrets:
      aws-role-arn: ${{ secrets.DEV_AWS_ROLE_ARN }}
```

---

### `deploy-frontend.yml`

Builds and deploys the React Vite frontend to S3 + CloudFront.

**Inputs:**

| Input | Required | Description |
|---|---|---|
| `environment` | Yes | `dev` or `prod` |
| `aws-region` | No | Default: `us-east-1` |

**Secrets:**

| Secret | Description |
|---|---|
| `aws-role-arn` | IAM role ARN for OIDC authentication |
| `bucket-name` | S3 bucket name |
| `cloudfront-id` | CloudFront distribution ID |
| `vite-env` | Multiline string of all `VITE_*` env vars |

**Pipeline stages:**
1. `npm ci` + `npm run build`
2. S3 sync — hashed assets get `max-age=31536000,immutable`; `index.html` and `sw.js` get `no-cache`
3. CloudFront invalidation of `/index.html`, `/sw.js`, `/manifest.webmanifest`

**Usage from the frontend repo:**

```yaml
jobs:
  deploy:
    uses: councilanchor/councilanchor-workflows/.github/workflows/deploy-frontend.yml@main
    with:
      environment: dev
    secrets:
      aws-role-arn: ${{ secrets.DEV_AWS_ROLE_ARN }}
      bucket-name: ${{ secrets.DEV_FRONTEND_BUCKET }}
      cloudfront-id: ${{ secrets.DEV_CLOUDFRONT_ID }}
      vite-env: ${{ secrets.DEV_VITE_ENV }}
```

## GitHub Organization Secrets

These org-level secrets must exist before any pipeline can run:

| Secret | Description |
|---|---|
| `DEV_AWS_ROLE_ARN` | OIDC role in the dev AWS account |
| `PROD_AWS_ROLE_ARN` | OIDC role in the prod AWS account |
| `DEV_FRONTEND_BUCKET` | `councilanchor-frontend-dev` |
| `PROD_FRONTEND_BUCKET` | `councilanchor-frontend-prod` |
| `DEV_CLOUDFRONT_ID` | Dev CloudFront distribution ID |
| `PROD_CLOUDFRONT_ID` | Prod CloudFront distribution ID |
| `DEV_VITE_ENV` | All `VITE_*` env vars for dev (multiline) |
| `PROD_VITE_ENV` | All `VITE_*` env vars for prod (multiline) |

## Adding a New Service

1. Create the new service repo under the `councilanchor` org
2. Add a `.github/workflows/deploy.yml` that calls `deploy-lambda.yml` (copy from an existing service)
3. Add the Lambda function name to the IAM role's allowed resources in `councilanchor-infrastructure/terraform/modules/iam/main.tf`
4. Wire the new Lambda into `councilanchor-infrastructure/terraform/modules/api-gateway/main.tf`
