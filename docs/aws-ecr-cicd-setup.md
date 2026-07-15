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

## 2. Add GitHub as an IAM OIDC provider

This is a one-time setup per AWS account. If you already created this provider,
skip to the next section.

1. Open **AWS Console > IAM > Identity providers**.
2. Choose **Add provider** and select **OpenID Connect**.
3. Enter provider URL `https://token.actions.githubusercontent.com`.
4. Enter audience `sts.amazonaws.com`.
5. Add the provider.

## 3. Create a least-privilege ECR push policy

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

## 4. Create the GitHub Actions IAM role

Creating the identity provider is only the first half of OIDC setup. GitHub also
needs an IAM role that trusts that provider and has the ECR push policy.

1. Open **IAM > Roles > Create role**.
2. Choose **Web identity**.
3. Select `token.actions.githubusercontent.com`.
4. Select audience `sts.amazonaws.com`.
5. For GitHub organization enter `koushik-0513`.
6. For GitHub repository enter `te`.
7. For GitHub branch enter `main`.
8. Attach the `GitHubActionsPushTeEcr` policy from the previous section.
9. Name the role `GitHubActionsTeEcrPush` and create it.

Open the role's **Trust relationships** tab and confirm the trust policy is
restricted to this repository and branch. It should have this shape (replace
the account ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:koushik-0513/te:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Copy the role ARN, which looks like
`arn:aws:iam::123456789012:role/GitHubActionsTeEcrPush`.

## 5. Configure GitHub repository variables

Open the GitHub repository, then go to **Settings > Secrets and variables >
Actions > Variables** and create these repository variables:

| Variable | Example | Purpose |
| --- | --- | --- |
| `AWS_REGION` | `ap-southeast-2` | Region containing the ECR repository |
| `ECR_REPOSITORY` | `te` | ECR repository name only, not its full URI |
| `AWS_ROLE_ARN` | `arn:aws:iam::123456789012:role/GitHubActionsTeEcrPush` | Role assumed through GitHub OIDC |

These values are identifiers rather than credentials, so repository variables
are appropriate. No AWS access-key secrets are required.

## 6. Run and verify the pipeline

1. Commit and push `.github/workflows/build-and-push-ecr.yml` to `main`, or open
   **GitHub > Actions > Build and push image to Amazon ECR > Run workflow**.
2. Open the workflow run and verify that `Build and push image` succeeds.
3. Open **AWS Console > ECR > Private repositories > your repository > Images**.
4. Confirm that the same image digest has both the commit SHA and `latest` tags.

Use the commit-SHA image URI when configuring ECS, EKS, App Runner, or another
deployment target. That keeps deployments reproducible even after `latest`
moves to a newer image.

## Troubleshooting

- `Not authorized to perform sts:AssumeRoleWithWebIdentity`: check the role ARN,
  repository owner/name, `main` branch condition, OIDC provider, and audience.
- `repository ... not found`: create the ECR repository in `AWS_REGION` and make
  sure `ECR_REPOSITORY` contains only its name.
- `not authorized to perform ecr:PutImage`: confirm that the policy repository
  ARN exactly matches the selected Region, account, and repository.
- Docker build failures should also reproduce locally with `docker build .`.
