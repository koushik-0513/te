# Amazon ECR CI/CD setup

This repository publishes its production Docker image to a private Amazon ECR
repository whenever a commit is pushed to `main`. Pull requests build the image
without authenticating to AWS or pushing it.

Published tags:

- `<commit-sha>`: immutable identifier for a specific build; use this for deployments.
- `latest`: convenient pointer to the newest successful `main` build.

## 1. Create the ECR repository

1. Open **AWS Console > Elastic Container Registry > Private repositories**.
2. Select the AWS Region where the image will be used.
3. Choose **Create repository**.
4. Select **Private** and enter a name, for example `te`.
5. Keep tag mutability **Mutable** because this workflow updates `latest` on
   every successful build. The commit SHA tag remains unique.
6. Keep AES-256 encryption, unless your organization requires a customer-managed
   KMS key, and create the repository.
7. Recommended: under **Private registry > Scanning**, enable basic scan-on-push
   for this repository (or configure enhanced scanning).

Record the repository name and Region. The repository must exist before the
first workflow run.

## 2. Create a least-privilege ECR push policy

Open **IAM > Policies > Create policy > JSON** and paste the policy below.
Replace `<AWS_ACCOUNT_ID>`, `<AWS_REGION>`, and `<ECR_REPOSITORY>` with the real
values. The repository ARN is visible on the ECR repository details page.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GetEcrLoginToken",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "PushOnlyToApplicationRepository",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:<AWS_REGION>:<AWS_ACCOUNT_ID>:repository/<ECR_REPOSITORY>"
    }
  ]
}
```

Name it `GitHubActionsPushTeEcr` (or another clear name) and create it.

## 3. Create the CI IAM user and access key

If the access keys you added to GitHub already belong to a dedicated IAM user
with the policy above, skip this section. Do not use root-account access keys.

1. Open **IAM > Users > Create user**.
2. Name the user `github-actions-te-ecr` and create it without AWS Console access.
3. Open the user, choose **Add permissions > Attach policies directly**, and
   attach `GitHubActionsPushTeEcr`.
4. Open **Security credentials > Access keys > Create access key**.
5. Select **Third-party service**, acknowledge the recommendation, and create
   the key.
6. Copy the access key ID and secret access key. AWS only displays the secret
   once.

The IAM user should have only the ECR push policy above. Do not attach
`AdministratorAccess` or broad ECR permissions.

## 4. Configure GitHub secrets and variables

Open the GitHub repository, then go to **Settings > Secrets and variables >
Actions > Secrets**. Create these repository secrets:

| Secret | Value |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | Access key ID for `github-actions-te-ecr` |
| `AWS_SECRET_ACCESS_KEY` | Secret access key for `github-actions-te-ecr` |

Never commit either value to this repository or paste it into the workflow.

Then open the **Variables** tab and create these repository variables:

| Variable | Example | Purpose |
| --- | --- | --- |
| `AWS_REGION` | `ap-southeast-2` | Region containing the ECR repository |
| `ECR_REPOSITORY` | `te` | ECR repository name only, not its full URI |

The Region and repository name are identifiers rather than credentials, so
repository variables are appropriate for them.

## 5. Run and verify the pipeline

1. Commit and push `.github/workflows/build-and-push-ecr.yml` to `main`, or open
   **GitHub > Actions > Build and push image to Amazon ECR > Run workflow**.
2. Open the workflow run and verify that `Build and push image` succeeds.
3. Open **AWS Console > ECR > Private repositories > your repository > Images**.
4. Confirm that the same image digest has both the commit SHA and `latest` tags.

Use the commit-SHA image URI when configuring ECS, EKS, App Runner, or another
deployment target. That keeps deployments reproducible even after `latest`
moves to a newer image.

## Troubleshooting

- `Credentials could not be loaded`: confirm both GitHub secret names exactly
  match `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
- `repository ... not found`: create the ECR repository in `AWS_REGION` and make
  sure `ECR_REPOSITORY` contains only its name.
- `not authorized to perform ecr:PutImage`: confirm that the policy repository
  ARN exactly matches the selected Region, account, and repository.
- Docker build failures should also reproduce locally with `docker build .`.
