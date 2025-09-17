# CodeCommit to GitHub Migration Guide

## Overview

This guide provides step-by-step instructions for migrating the AWS CodePipeline DevSecOps solution from AWS CodeCommit to GitHub integration. Since AWS CodeCommit is no longer available to new customers, this migration enables the same DevSecOps functionality using GitHub as the source repository.

## Prerequisites

- Existing AWS account with appropriate permissions
- GitHub account with repository access
- AWS CLI configured
- CloudFormation deployment experience

## Migration Architecture

### Before (CodeCommit)
- AWS CodeCommit repository as source
- Direct CodeCommit integration with CodePipeline
- CodeCommit-specific IAM permissions

### After (GitHub)
- GitHub repository as source
- AWS CodeStar Connections for GitHub integration
- GitHub OAuth token stored in AWS Secrets Manager
- Webhook-based triggering from GitHub

## Step 1: GitHub Setup

### 1.1 Create GitHub OAuth App

1. Go to GitHub Settings → Developer settings → OAuth Apps
2. Click "New OAuth App"
3. Fill in the details:
   - **Application name**: `AWS-CodePipeline-DevSecOps`
   - **Homepage URL**: `https://aws.amazon.com/codepipeline/`
   - **Authorization callback URL**: `https://console.aws.amazon.com/`
4. Click "Register application"
5. Note down the **Client ID** and generate a **Client Secret**

### 1.2 Generate Personal Access Token (Alternative)

If OAuth App is not preferred, create a Personal Access Token:

1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Click "Generate new token"
3. Select scopes:
   - `repo` (Full control of private repositories)
   - `admin:repo_hook` (Full control of repository hooks)
4. Generate and securely store the token

## Step 2: Prepare Migration

### 2.1 Backup Current Setup

```bash
# Export current CloudFormation templates
aws cloudformation get-template --stack-name <your-codecommit-stack> > codecommit-backup.json
aws cloudformation get-template --stack-name <your-pipeline-stack> > pipeline-backup.json
```

### 2.2 Repository Migration

If migrating from CodeCommit to GitHub:

```bash
# Clone from CodeCommit
git clone https://git-codecommit.<region>.amazonaws.com/v1/repos/<repo-name>

# Add GitHub remote
git remote add github https://github.com/<owner>/<repo-name>.git

# Push to GitHub
git push github main
```

## Step 3: Deploy New Infrastructure

### 3.1 Deploy GitHub ECR Stack

Use the new `github_ecr.yaml` template:

```bash
aws cloudformation create-stack \
  --stack-name github-ecr-stack \
  --template-body file://cf_templates/github_ecr.yaml \
  --parameters \
    ParameterKey=GitHubRepositoryOwner,ParameterValue=<your-github-username> \
    ParameterKey=GitHubRepositoryName,ParameterValue=<your-repo-name> \
    ParameterKey=GitHubBranchName,ParameterValue=main \
    ParameterKey=GitHubOAuthToken,ParameterValue=<your-oauth-token> \
    ParameterKey=ECRRepositoryName,ParameterValue=<your-ecr-repo-name>
```

### 3.2 Get Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name github-ecr-stack \
  --query 'Stacks[0].Outputs'
```

Note the following outputs:
- `GitHubOAuthSecretArn`
- `GitHubRepositoryOwner`
- `GitHubRepositoryName`

## Step 4: Deploy Pipeline Stack

### 4.1 Modified Parameters

The new pipeline stack requires these parameters:

| Parameter | Description | Example |
|-----------|-------------|---------|
| GitHubRepositoryOwner | GitHub username/org | `aws-samples` |
| GitHubRepositoryName | Repository name | `aws-codepipeline-devsecops-amazoneks` |
| GitHubBranchName | Branch to monitor | `main` |
| GitHubOAuthSecretArn | ARN from Step 3.2 | `arn:aws:secretsmanager:...` |
| EKSClusterName | Your EKS cluster name | `my-eks-cluster` |
| EcrDockerRepository | ECR repo name | `my-app-repo` |
| EmailRecipient | Notification email | `user@domain.com` |

### 4.2 Deploy Command

```bash
aws cloudformation create-stack \
  --stack-name github-pipeline-stack \
  --template-body file://cf_templates/build_deployment_github.yaml \
  --parameters file://github-pipeline-parameters.json \
  --capabilities CAPABILITY_IAM
```

## Step 5: Manual Configuration

### 5.1 Complete GitHub Connection

1. Go to AWS Console → Developer Tools → Settings → Connections
2. Find the connection created by CloudFormation
3. Click "Update pending connection"
4. Authorize the GitHub connection
5. Select your GitHub account/organization

### 5.2 Verify CodeGuru Reviewer

1. Go to AWS Console → CodeGuru → Reviewer
2. Verify the GitHub repository association is active
3. Test by creating a pull request in GitHub

## Step 6: Testing

### 6.1 Manual Pipeline Trigger

1. Go to AWS Console → CodePipeline
2. Find your new pipeline
3. Click "Release change" to trigger manually
4. Verify all stages complete successfully

### 6.2 Webhook Testing

1. Make a small change to your GitHub repository
2. Push to the monitored branch
3. Verify the pipeline triggers automatically
4. Check CloudWatch Events for webhook delivery

## Step 7: Cleanup Old Resources

After successful migration and testing:

### 7.1 Delete CodeCommit Stack

```bash
aws cloudformation delete-stack --stack-name <old-codecommit-stack>
```

### 7.2 Remove CodeCommit Repository

```bash
aws codecommit delete-repository --repository-name <old-repo-name>
```

## Troubleshooting

### Common Issues

1. **Connection Pending**: Complete manual authorization in AWS Console
2. **Webhook Not Working**: Check GitHub webhook configuration
3. **Permission Denied**: Verify OAuth token has correct scopes
4. **CodeGuru Not Working**: Ensure GitHub connection is active

### Verification Commands

```bash
# Check connection status
aws codestar-connections list-connections

# Check pipeline status
aws codepipeline get-pipeline-state --name <pipeline-name>

# Check CodeGuru associations
aws codeguru-reviewer list-repository-associations
```

## Key Differences from CodeCommit

| Aspect | CodeCommit | GitHub |
|--------|------------|--------|
| Authentication | IAM roles | OAuth tokens |
| Triggering | CloudWatch Events | Webhooks |
| Connection Type | Direct | CodeStar Connections |
| Manual Setup | None | Connection authorization |
| Token Management | None | Periodic renewal |

## Security Considerations

1. **OAuth Token Security**: Store in Secrets Manager, rotate regularly
2. **Connection Permissions**: Limit to specific repositories
3. **Webhook Security**: Use GitHub webhook secrets if needed
4. **IAM Permissions**: Follow least privilege principle

## Next Steps

After successful migration:

1. Update documentation to reflect GitHub integration
2. Train team on new GitHub-based workflow
3. Set up monitoring for GitHub webhook delivery
4. Plan for OAuth token rotation schedule
5. Consider implementing GitHub branch protection rules
