# CI/CD Pipeline Project Using AWS Native SDLC Automation Tools

![Complete AWS Native CI/CD Project](https://github.com/awanmbandi/realworld-cicd-pipeline-project/blob/zdocs/images/aws_native_project_v2.2.png)

## Overview

A production‑grade CI/CD pipeline built **entirely with AWS native services**. It automates the SDLC from **code ➜ build ➜ test/quality ➜ artifacting ➜ deploy ➜ observe/notify**. The stack is cost‑efficient, scalable, and suitable for EC2 workloads (adaptable to ECS/EKS/Lambda).

---

## Project Toolbox

* **CodeCommit** – Private Git repo hosting
* **CodeBuild** – Builds, tests, packages app
* **CodeArtifact** – Package & artifact repository (npm/pip/maven/etc.)
* **CodeDeploy** – Automated EC2 app deployments (supports in‑place & blue/green)
* **CodePipeline** – Orchestrates stages end‑to‑end
* **Amazon S3** – Stores build artifacts, static files
* **Amazon EC2** – Target compute for the application
* **CloudWatch (Metrics & Logs)** – Monitoring/alerts/logs
* **SNS** – Notifications for pipeline/deploy/alarms
* **SonarCloud** – Static analysis & code quality gates

> **Notes**
>
> 1. Use a region that supports all Code\* services (e.g., `us-east-1`, `us-west-2`).
> 2. Log in as an **IAM user** with admin (or create minimal‑privilege roles/policies below).

---

## Architecture Workflow

1. **Source:** Push code to **CodeCommit**.
2. **Build & Test:** **CodeBuild** compiles/tests; runs **SonarCloud** analysis.
3. **Artifacting:** Dependencies & build outputs handled via **CodeArtifact** and/or **S3**.
4. **Deploy:** **CodeDeploy** releases to EC2 (rolling or **blue/green**).
5. **Orchestration:** **CodePipeline** wires it all together.
6. **Observe & Notify:** **CloudWatch** metrics/logs + **SNS** notifications.

---

## Prerequisites

* AWS account + IAM user/role with sufficient privileges
* AWS CLI v2 configured (`aws configure`)
* Git installed locally
* (Optional) SonarCloud org + project + token

---

## Repo Structure (recommended)

```
.
├── src/                     # application code
├── scripts/                 # deployment hook scripts
│   ├── before_install.sh
│   ├── after_install.sh
│   ├── start_server.sh
│   └── stop_server.sh
├── appspec.yml              # CodeDeploy spec
├── buildspec.yml            # CodeBuild spec
├── sonar-project.properties # SonarCloud project config (optional)
├── pipeline/pipeline.json   # CodePipeline definition (sample)
└── README.md
```

---

## Step 1 — Create a CodeCommit Repo & Push Code

```bash
# create a repo
aws codecommit create-repository \
  --repository-name aws-native-cicd-demo \
  --repository-description "AWS Native CI/CD demo"

# add CodeCommit remote (use the output clone URL HTTP or HTTPS-GRC if configured)
git init
git remote add codecommit https://git-codecommit.<REGION>.amazonaws.com/v1/repos/aws-native-cicd-demo

git add .
git commit -m "initial commit"
git push codecommit main
```

---

## Step 2 — (Optional) Configure CodeArtifact

**npm example**

```bash
# create domain & repo
aws codeartifact create-domain --domain team-domain
aws codeartifact create-repository --domain team-domain --repository team-repo \
  --upstreams repositoryName=public-npm --description "App deps & artifacts"

# login (fetch token then login npm)
export CODEART_TOKEN=$(aws codeartifact get-authorization-token \
  --domain team-domain --query authorizationToken --output text)

npm config set //$(aws codeartifact get-repository-endpoint \
  --domain team-domain --repository team-repo --format npm \
  --query repositoryEndpoint --output text | sed 's#https://##')/:_authToken $CODEART_TOKEN

npm config set registry $(aws codeartifact get-repository-endpoint \
  --domain team-domain --repository team-repo --format npm \
  --query repositoryEndpoint --output text)
```

> For **pip/maven/nuget** adjust commands accordingly.

---

## Step 3 — Provision Target EC2 & Install CodeDeploy Agent

**Security Group**: allow `80/tcp` (HTTP) from your network / ALB, and `22/tcp` (SSH) from admin IP.

**Amazon Linux 2**

```bash
sudo yum update -y
sudo yum install -y ruby wget
cd /home/ec2-user
wget https://aws-codedeploy-<REGION>.s3.<REGION>.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo systemctl status codedeploy-agent
```

**Ubuntu**

```bash
sudo apt update -y
sudo apt install -y ruby wget
cd /home/ubuntu
wget https://aws-codedeploy-<REGION>.s3.<REGION>.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo systemctl status codedeploy-agent
```

> Tag your instances (e.g., `Key=CodeDeploy,Value=Demo`) to target them in the Deployment Group.

---

## Step 4 — IAM Roles (Minimal Examples)

**CodeBuild role (trust policy)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "codebuild.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Attach policies** (examples): `AWSCodeBuildDeveloperAccess`, `AmazonS3FullAccess`, `CloudWatchLogsFullAccess`, `AWSCodeArtifactReadOnlyAccess`, `AmazonSNSFullAccess` (tailor to least-privilege).

**CodePipeline role (trust policy)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "codepipeline.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Attach policies**: `AWSCodePipelineFullAccess` (or tailor), plus pass‑role permissions for CodeBuild/CodeDeploy.

**CodeDeploy service role (trust policy)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "codedeploy.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Attach policy**: `AWSCodeDeployRole` (or the AWS managed `AWSCodeDeployRole` with EC2 permissions).

---

## Step 5 — CodeBuild: `buildspec.yml`

**Node.js example with SonarCloud**

```yaml
version: 0.2
env:
  variables:
    NODE_ENV: production
  secrets-manager:
    SONAR_TOKEN: sonarcloud/ci:token    # example path; store a key named 'token'
phases:
  install:
    commands:
      - echo "Installing deps"
      - npm ci
  pre_build:
    commands:
      - echo "Running unit tests"
      - npm test -- --ci --reporters=default
      - echo "SonarCloud scan"
      - npm install -g sonarqube-scanner
      - |
        sonarqube-scanner \
          -Dsonar.projectKey=$SONAR_PROJECT_KEY \
          -Dsonar.organization=$SONAR_ORG \
          -Dsonar.host.url=https://sonarcloud.io \
          -Dsonar.login=$SONAR_TOKEN || true  # don't fail build if quality gate check runs async
  build:
    commands:
      - echo "Building app"
      - npm run build
      - echo "Packaging artifact"
      - mkdir -p dist && cp -r build/* dist/
  post_build:
    commands:
      - echo "Zipping artifact"
      - zip -r artifact.zip dist appspec.yml scripts/
artifacts:
  files:
    - artifact.zip
  discard-paths: yes
```

**`sonar-project.properties`** (example)

```
sonar.projectKey=YOUR_PROJECT_KEY
sonar.organization=YOUR_ORG
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.sourceEncoding=UTF-8
```

---

## Step 6 — CodeDeploy: `appspec.yml` + Hook Scripts

**`appspec.yml`**

```yaml
version: 0.0
os: linux
files:
  - source: /dist
    destination: /var/www/app
hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 180
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 180
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 180
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 180
      runas: root
```

**`scripts/before_install.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
mkdir -p /var/www/app
# stop existing app if running
if pgrep -f "node /var/www/app" >/dev/null; then
  pkill -f "node /var/www/app" || true
fi
```

**`scripts/after_install.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
chown -R ec2-user:ec2-user /var/www/app || true
```

**`scripts/start_server.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
# example: serve static build with nginx OR run a node server
# here we just run a simple node server if your app supports it
cd /var/www/app
nohup node server.js > /var/log/app.log 2>&1 &
```

**`scripts/stop_server.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail
if pgrep -f "node /var/www/app" >/dev/null; then
  pkill -f "node /var/www/app"
fi
```

> Adjust scripts for your runtime (Node/Java/Python). For Nginx/PM2/systemd, replace `start_server.sh` with your service commands.

---

## Step 7 — Create CodeDeploy App & Deployment Group

```bash
# application
aws deploy create-application --application-name webapp-ec2 --compute-platform Server

# service role ARN from Step 4; pick a tag filter that matches your EC2
aws deploy create-deployment-group \
  --application-name webapp-ec2 \
  --deployment-group-name webapp-ec2-dg \
  --service-role-arn arn:aws:iam::<ACCOUNT_ID>:role/CodeDeployServiceRole \
  --deployment-config-name CodeDeployDefault.AllAtOnce \
  --ec2-tag-filters Key=CodeDeploy,Value=Demo,Type=KEY_AND_VALUE \
  --auto-scaling-groups "" \
  --deployment-style deploymentType=IN_PLACE,deploymentOption=WITHOUT_TRAFFIC_CONTROL
```

> Switch to `deploymentType=BLUE_GREEN` and attach a Load Balancer to enable blue/green.

---

## Step 8 — CodePipeline Definition (sample `pipeline/pipeline.json`)

```json
{
  "pipeline": {
    "name": "aws-native-cicd-demo",
    "roleArn": "arn:aws:iam::<ACCOUNT_ID>:role/CodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "<ARTIFACT_BUCKET_NAME>"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "CodeCommit_Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "configuration": {
              "RepositoryName": "aws-native-cicd-demo",
              "BranchName": "main",
              "PollForSourceChanges": "false"
            },
            "outputArtifacts": [{ "name": "SourceOutput" }],
            "runOrder": 1
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "CodeBuild_Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": { "ProjectName": "aws-native-cicd-build" },
            "inputArtifacts": [{ "name": "SourceOutput" }],
            "outputArtifacts": [{ "name": "BuildOutput" }],
            "runOrder": 1
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "CodeDeploy_Deploy",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeploy",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "webapp-ec2",
              "DeploymentGroupName": "webapp-ec2-dg"
            },
            "inputArtifacts": [{ "name": "BuildOutput" }],
            "runOrder": 1
          }
        ]
      }
    ],
    "version": 1
  }
}
```

**Create pipeline**

```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline/pipeline.json
```

**(Optional) Manual Approval Stage** – insert between Build and Deploy using the `Manual` action provider.

---

## Step 9 — CloudWatch Alarms + SNS Notifications

```bash
# SNS topic & subscription
aws sns create-topic --name cicd-alerts
aws sns subscribe --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:cicd-alerts \
  --protocol email --notification-endpoint YOUR_EMAIL@example.com

# Example alarm on EC2 CPUUtilization
aws cloudwatch put-metric-alarm \
  --alarm-name ec2-high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=<YOUR_ASG_NAME_OR_REMOVE_DIMENSION> \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:<REGION>:<ACCOUNT_ID>:cicd-alerts
```

> Also enable **CloudWatch Logs** for CodeBuild & CodeDeploy to trace builds/deploys.

---

## Step 10 — Run the Pipeline

1. Push a commit to `main` ➜ Source triggers pipeline (or enable event rule trigger).
2. Inspect CodeBuild logs; confirm artifact is produced.
3. CodeDeploy should roll out to EC2; verify app response (curl/ALB/URL).
4. Check CloudWatch dashboards/alarms; ensure SNS email delivery.

---

## Blue/Green Deployment (Quick Switch)

* Create/attach an **ALB** with two target groups (blue/green).
* In Deployment Group, set `deploymentType=BLUE_GREEN` & `deploymentOption=WITH_TRAFFIC_CONTROL`.
* Configure termination wait time & health checks. CodeDeploy will shift traffic when new version is healthy.

---

## Cleanup

* `aws codepipeline delete-pipeline --name aws-native-cicd-demo`
* Delete CodeBuild project, CodeDeploy app/group, SNS topic, CloudWatch alarms, S3 artifact bucket, EC2 instances, IAM roles/policies (in dependency order).

---

## FAQ

**Q:** Can I use Docker/ECS/EKS?
**A:** Yes. Swap EC2 + appspec hooks for container build/push (ECR) and ECS/EKS deploy actions.

**Q:** How do I enforce quality gates?
**A:** Use SonarCloud GitHub/PR decorators or poll quality gate via Web API; fail the build if gate fails.

**Q:** Can I split environments (Dev/Stage/Prod)?
**A:** Yes. Add multiple Deploy stages or multiple pipelines, with Manual Approval between environments.

---

## Author

**Akuphe Dieudonne** — Cloud & DevOps Engineer
GitHub: [https://github.com/akupheaws](https://github.com/akupheaws)
