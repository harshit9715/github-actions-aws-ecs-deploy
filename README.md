# Setup CI/CD using Github Actions and Deploy the app in AWS ECS

## Prerequisites

For deploying an application image to the registry you should have:

- An application (We are deploying a simple python app, check [here](https://aws.amazon.com/blogs/containers/create-a-ci-cd-pipeline-for-amazon-ecs-with-github-actions-and-aws-codebuild-tests/))
- ECS infrastructure (existing registry to push image)

  - We created it using [AWS CDK for python](https://docs.aws.amazon.com/cdk/latest/guide/work-with-cdk-python.html) by following above guide.
  - And configured by replacing the contents of the file "ecs_devops_sandbox_cdk/ecs_devops_sandbox_cdk_stack.py" with [ECS_INFRA_setup](ECS_INFRA_setup.py)
  - [this](https://aws.amazon.com/blogs/containers/create-a-ci-cd-pipeline-for-amazon-ecs-with-github-actions-and-aws-codebuild-tests/) guide (same as above) should setup your app as well as your Infrastructure.

## After you push your code to Github

### These are the secrets that you need to configure in your repository

- __AWS_ACCESS_KEY_ID__ - Create a new AWS user with only programmatic access and no permissions initially.
- __AWS_SECRET_ACCESS_KEY__ - Secret of this user
- __AWS_ROLE_EXTERNAL_ID__ - A simple secret string to only allow this user to assume role and restrict other users.
- __AWS_ROLE_TO_ASSUME__ - ARN of the assumed role (even ROLE_NAME will work if AWS_USER and the ROLE_TO_BE_ASSUMED belong to same account.)

## Policy and Roles configuration to be done in AWS account

### The Assumed role should have the policy to build, tag and deploy image to the registry

#### Go to AWS -> IAM -> Roles -> Select Assumed Role -> and attach this policy (inline_policy, make sure to replace the required files inside it.)

```json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ecr:PutImageTagMutability",
                "ecr:DescribeImageScanFindings",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetDownloadUrlForLayer",
                "ecr:PutImageScanningConfiguration",
                "ecr:ListTagsForResource",
                "ecr:UploadLayerPart",
                "ecr:ListImages",
                "ecr:PutImage",
                "ecs:UpdateService",
                "iam:PassRole",
                "ecr:BatchGetImage",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeImages",
                "ecr:DescribeRepositories",
                "ecs:DescribeServices",
                "ecr:InitiateLayerUpload",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetRepositoryPolicy",
                "ecr:GetLifecyclePolicy"
            ],
            "Resource": [
                "arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecs-devops-sandbox-execution-role",
                "arn:aws:ecr:<AWS_REGION>:<AWS_ACCOUNT_ID>:repository/<AWS_REGISTRY_NAME>",
                "arn:aws:ecs:<AWS_REGION>:<AWS_ACCOUNT_ID>:service/<AWS_ECS_CLUSTER_NAME>/<AWS_ECS_CLUSTER_SERVICE_NAME>"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "ecr:GetRegistryPolicy",
                "ecr:DescribeRegistry",
                "ecs:RegisterTaskDefinition",
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}
```

### Assumed role should have a Trust Relationship with our AWS GITHUB ACTION USER

#### Go to AWS -> IAM -> Roles -> Select Assumed Role -> Trust Relationship and attach this policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowIamUserAssumeRole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/<AWS_GITHUB_ACTION_USER_NAME>"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<SIMPLE_SECRET_STRING_FOR_LIMITING_ASSUME_ROLE_ACCESS>"
        }
      }
    },
    {
      "Sid": "AllowPassSessionTags",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<AWS_ACCOUNT_ID>:user/<AWS_GITHUB_ACTION_USER_NAME>"
      },
      "Action": "sts:TagSession"
    }
  ]
}

```

### Our AWS GITHUB ACTION USER must have assume role permissions

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ],
            "Resource": [
                "arn:aws:iam::<AWS_ACCOUNT_ID>:role/<ASSUMED_ROLE_NAME>",
                "arn:aws:iam::<AWS_ACCOUNT_ID>:user/<AWS_GITHUB_ACTION_USER_NAME>"
            ]
        }
    ]
}
```

### Finally

- Go to github , click on actions
- Create new workflow and select Deploy to Amazon ECS
- Configure the settings such as repo name, cluster name and service name and you are done
