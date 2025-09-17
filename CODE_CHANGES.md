# Specific Code Changes for GitHub Migration

## Overview

This document provides the exact code changes needed to migrate from CodeCommit to GitHub integration.

## 1. CloudFormation Template Changes

### A. New Template: `cf_templates/github_ecr.yaml`

**Purpose**: Replaces `codecommit_ecr.yaml` - creates GitHub integration and ECR repository

**Key Changes**:
- Removes `AWS::CodeCommit::Repository` resource
- Adds GitHub parameters
- Creates `AWS::SecretsManager::Secret` for OAuth token
- Maintains ECR repository unchanged

### B. Modified Template: `cf_templates/build_deployment.yaml`

#### Parameters Section Changes

**REMOVE these parameters**:
```yaml
SourceRepoName:
  Type: String
  Description: AWS CodeCommit RepoName where code resides
```

**ADD these parameters**:
```yaml
GitHubRepositoryOwner:
  Type: String
  Description: GitHub repository owner (username or organization)
GitHubRepositoryName:
  Type: String
  Description: GitHub repository name
GitHubBranchName:
  Type: String
  Default: main
  Description: Branch Name
GitHubOAuthSecretArn:
  Type: String
  Description: ARN of the AWS Secrets Manager secret containing GitHub OAuth token
```

#### Resources Section Changes

**1. ADD GitHub Connection Resource**:
```yaml
GitHubConnection:
  Type: AWS::CodeStarConnections::Connection
  Properties:
    ConnectionName: !Sub "${AWS::StackName}-github-connection"
    ProviderType: GitHub
```

**2. MODIFY CodeGuru Reviewer Association**:

**FROM**:
```yaml
SourceRepositoryAssociation:
  Type: AWS::CodeGuruReviewer::RepositoryAssociation
  Properties:
    Name: !Ref SourceRepoName
    Type: CodeCommit
    BucketName: !Ref CodeGuruReviewerBucket
```

**TO**:
```yaml
SourceRepositoryAssociation:
  Type: AWS::CodeGuruReviewer::RepositoryAssociation
  Properties:
    Name: !Sub "${GitHubRepositoryOwner}/${GitHubRepositoryName}"
    Type: GitHub
    ConnectionArn: !Ref GitHubConnection
    BucketName: !Ref CodeGuruReviewerBucket
```

**3. MODIFY CodePipeline Service Role IAM Policy**:

**REMOVE these statements**:
```yaml
- Resource: !Sub arn:${AWS::Partition}:codecommit:${AWS::Region}:${AWS::AccountId}:${SourceRepoName}
  Effect: Allow
  Action:
    - codecommit:GetBranch
    - codecommit:GetCommit
    - codecommit:ListRepositories
    - codecommit:GetRepository
    - codecommit:UploadArchive
    - codecommit:GetUploadArchiveStatus
    - codecommit:CancelUploadArchive
```

**ADD these statements**:
```yaml
- Resource: !Ref GitHubConnection
  Effect: Allow
  Action:
    - codestar-connections:UseConnection
- Resource: !Ref GitHubOAuthSecretArn
  Effect: Allow
  Action:
    - secretsmanager:GetSecretValue
```

**4. MODIFY CodeBuild Service Role IAM Policy**:

**REMOVE this statement**:
```yaml
- Resource: !Sub arn:${AWS::Partition}:codecommit:${AWS::Region}:${AWS::AccountId}:${SourceRepoName}
  Effect: Allow
  Action:
    - codecommit:GitPull
    - codecommit:TagResource
```

**5. MODIFY CodeBuild Projects Source Configuration**:

**REMOVE Location from all CodeBuild projects**:

**FROM**:
```yaml
Source:
  Location:
    Fn::Sub: https://git-codecommit.${AWS::Region}.amazonaws.com/v1/repos/${SourceRepoName}
  Type: CODEPIPELINE
  BuildSpec: "buildspec/buildspec_secscan.yaml"
```

**TO**:
```yaml
Source:
  Type: CODEPIPELINE
  BuildSpec: "buildspec/buildspec_secscan.yaml"
```

Apply this change to:
- `CodeSecScanProject`
- `CodeBuildImageProject` 
- `CodeDeployImageProject`

**6. MODIFY CodePipeline Source Stage**:

**CHANGE Pipeline Resource Name**:
```yaml
# FROM:
CodePipelineCodeCommit:

# TO:
CodePipelineGitHub:
```

**MODIFY Source Action**:

**FROM**:
```yaml
- Name: App
  ActionTypeId:
    Category: Source
    Owner: AWS
    Version: 1
    Provider: CodeCommit
  Configuration:
    BranchName: !Ref CodeBranchName
    OutputArtifactFormat: CODEBUILD_CLONE_REF
    RepositoryName:
      Ref: SourceRepoName
```

**TO**:
```yaml
- Name: App
  ActionTypeId:
    Category: Source
    Owner: AWS
    Version: 1
    Provider: CodeStarSourceConnection
  Configuration:
    ConnectionArn: !Ref GitHubConnection
    FullRepositoryId: !Sub "${GitHubRepositoryOwner}/${GitHubRepositoryName}"
    BranchName: !Ref GitHubBranchName
    OutputArtifactFormat: CODEBUILD_CLONE_REF
```

**7. UPDATE Event Rule Dependencies**:

**CHANGE DependsOn references**:
```yaml
# FROM:
DependsOn: [ "CodePipelineCodeCommit" ]

# TO:
DependsOn: [ "CodePipelineGitHub" ]
```

**8. UPDATE Outputs Section**:

**MODIFY Pipeline URL Output**:
```yaml
CodePipelineURL:
  Description: Codepipeline URL for this stack
  Value:
    Fn::Join:
      - ""
      - - "https://console.aws.amazon.com/codepipeline/home?region="
        - Ref: AWS::Region
        - "#/view/"
        - Ref: CodePipelineGitHub  # Changed from CodePipelineCodeCommit
```

## 2. Parameter Files

### Create `github-pipeline-parameters.json`:

```json
[
  {
    "ParameterKey": "GitHubRepositoryOwner",
    "ParameterValue": "aws-samples"
  },
  {
    "ParameterKey": "GitHubRepositoryName", 
    "ParameterValue": "aws-codepipeline-devsecops-amazoneks"
  },
  {
    "ParameterKey": "GitHubBranchName",
    "ParameterValue": "main"
  },
  {
    "ParameterKey": "GitHubOAuthSecretArn",
    "ParameterValue": "arn:aws:secretsmanager:region:account:secret:name"
  },
  {
    "ParameterKey": "EKSClusterName",
    "ParameterValue": "your-eks-cluster"
  },
  {
    "ParameterKey": "EcrDockerRepository",
    "ParameterValue": "your-ecr-repo"
  },
  {
    "ParameterKey": "EmailRecipient",
    "ParameterValue": "your-email@domain.com"
  },
  {
    "ParameterKey": "EKSWorkerNodeRoleName",
    "ParameterValue": "your-worker-node-role"
  },
  {
    "ParameterKey": "EKSWorkerNodeRoleARN",
    "ParameterValue": "arn:aws:iam::account:role/worker-node-role"
  }
]
```

## 3. Documentation Updates

### Update `README.md`:

**REPLACE CodeCommit references**:

**FROM**:
```markdown
1. Developer will update the Java application code in the base branch of the AWS CodeCommit repository, creating a Pull Request (PR).
```

**TO**:
```markdown
1. Developer will update the Java application code in the base branch of the GitHub repository, creating a Pull Request (PR).
```

**UPDATE Setup Instructions**:

**REPLACE**:
```markdown
2) **CodeCommitECR Creation**:  
   Ensure you have previously created AWS CodeCommit and Amazon ECR...
   Run the CloudFormation template **cf_templates/codecommit_ecr.yaml**
```

**WITH**:
```markdown
2) **GitHub ECR Creation**:  
   Ensure you have a GitHub repository and Amazon ECR setup...
   Run the CloudFormation template **cf_templates/github_ecr.yaml**
```

**UPDATE Parameter Tables**:

**REPLACE CodeCommit parameters with GitHub parameters** in all documentation tables.

## 4. Validation Steps

After making these changes:

1. **Validate CloudFormation Templates**:
```bash
aws cloudformation validate-template --template-body file://cf_templates/github_ecr.yaml
aws cloudformation validate-template --template-body file://cf_templates/build_deployment_github.yaml
```

2. **Check for Remaining CodeCommit References**:
```bash
grep -r "codecommit\|CodeCommit" cf_templates/
grep -r "git-codecommit" cf_templates/
```

3. **Verify Parameter Consistency**:
- Ensure all GitHub parameters are used consistently
- Check that no CodeCommit parameters remain
- Validate parameter types and defaults

## 5. Testing Checklist

- [ ] CloudFormation templates validate successfully
- [ ] No CodeCommit references remain in templates
- [ ] GitHub connection can be established
- [ ] Pipeline triggers from GitHub webhooks
- [ ] CodeGuru Reviewer works with GitHub
- [ ] All build stages complete successfully
- [ ] Security scanning continues to work
- [ ] Deployment to EKS functions properly
