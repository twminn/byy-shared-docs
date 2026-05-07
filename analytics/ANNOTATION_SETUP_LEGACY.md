# Annotation Setup — Legacy Team

This guide walks the Legacy team through adding deploy annotations to the BYY
marketing analytics dashboard. Once configured, every production deploy of the
legacy system will appear as an **amber ▼** marker on all dashboard time-series
charts, letting the whole organization correlate traffic and revenue shifts with
legacy releases.

## Why This Matters

The dashboard overlays deploy markers on Traffic, Search, Revenue, Landing
Pages, AEO/GEO, and ad-platform charts. Without annotations from every team,
we can only see _our own_ deploys when investigating metric changes — making it
easy to miss that a legacy system change affected downstream metrics.

## Prerequisites

- A GitHub Actions deploy workflow that runs on production deploys.
- An `ubuntu-latest` (or compatible) runner — `gh` CLI is pre-installed.
- AWS credentials with `dynamodb:PutItem` on the annotations table (see Step 2).

If your deployment does not use GitHub Actions (e.g. a manual or script-based
process), see the "Non-GitHub-Actions Deployments" section at the end.

## Step 1 — Copy the Composite Action

Copy the `.github/actions/annotate-deploy/` directory from the
`byy-marketing-analytics` repository into the root of your legacy repo so the
path is:

```
your-legacy-repo/
  .github/
    actions/
      annotate-deploy/
        action.yml
```

You only need the single `action.yml` file inside that directory.

## Step 2 — IAM Permissions

Your deploy workflow's AWS credentials need permission to write to the shared
DynamoDB table. Add this statement to the IAM role or user:

```json
{
  "Effect": "Allow",
  "Action": "dynamodb:PutItem",
  "Resource": "arn:aws:dynamodb:us-east-1:876524020257:table/byy-annotations"
}
```

If your workflow already configures AWS credentials for deployment, attach this
policy to the same role. If not, create a dedicated IAM user or role and
proceed to Step 3.

## Step 3 — GitHub Secrets

Add the following secrets to your legacy repository (Settings → Secrets and
variables → Actions):

| Secret                  | Value                                    |
| ----------------------- | ---------------------------------------- |
| `AWS_ACCESS_KEY_ID`     | Access key for the IAM user/role above   |
| `AWS_SECRET_ACCESS_KEY` | Corresponding secret key                 |

If your workflow already has AWS credentials configured, you can reuse them as
long as the IAM policy from Step 2 is attached.

## Step 4 — Add the Annotation Step

In your deploy workflow (e.g. `.github/workflows/deploy.yml`), add these steps
**after** your deploy succeeds. The annotation should only run when the deploy
is confirmed stable.

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: read          # needed for gh pr list

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ... your existing checkout, build, deploy steps ...

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Annotate deployment
        uses: ./.github/actions/annotate-deploy
        with:
          source: byy-legacy
          category: deployment
```

If your workflow already configures AWS credentials earlier in the job, you do
not need to add the credentials step again — just add the "Annotate deployment"
step after your deploy completes.

### Using a Different Category

For releases that are better described as a bug fix or feature release, override
the `category` input:

```yaml
- name: Annotate deployment
  uses: ./.github/actions/annotate-deploy
  with:
    source: byy-legacy
    category: bug_fix            # or feature_release, site_update, etc.
```

Available categories: `deployment`, `feature_release`, `campaign_change`,
`bug_fix`, `site_update`, `seasonal_event`.

## Step 5 — Verify

After the first deploy with the annotation step:

1. Open the marketing analytics dashboard.
2. Navigate to the **Annotations** page (`/annotations`).
3. Confirm that a new row appears with source `byy-legacy`.
4. Check any time-series chart — you should see an **amber ▼** marker on
   today's date with a dotted vertical line.

## Non-GitHub-Actions Deployments

If your deploy process does not use GitHub Actions, you can write the annotation
directly from any environment that has the AWS CLI configured:

```bash
DATE=$(date -u +%Y-%m-%d)
SK="${DATE}#$(uuidgen)"

aws dynamodb put-item \
  --table-name byy-annotations \
  --item "{
    \"pk\": {\"S\": \"ANNOTATION\"},
    \"sk\": {\"S\": \"$SK\"},
    \"date\": {\"S\": \"$DATE\"},
    \"category\": {\"S\": \"deployment\"},
    \"title\": {\"S\": \"Legacy deploy — describe the change\"},
    \"description\": {\"S\": \"Commit: <sha> | Repo: <repo>\"},
    \"source\": {\"S\": \"byy-legacy\"},
    \"created_at\": {\"S\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}
  }" \
  --region us-east-1
```

Add this command to the end of your deploy script or CI pipeline. Replace the
`title` and `description` values with the appropriate commit or release info.

## Reference

| Field          | Value             |
| -------------- | ----------------- |
| Source          | `byy-legacy`       |
| Marker color   | Amber (`#F59E0B`)  |
| DynamoDB table | `byy-annotations`  |
| AWS region     | `us-east-1`        |

For the full annotation schema and all categories, see
[DEPLOY_ANNOTATIONS.md](DEPLOY_ANNOTATIONS.md).
