# Jenkins Docker CI/CD Template

This repository provides a reusable Jenkins-in-Docker setup for running CI/CD pipelines with:

* Jenkins running from Docker Compose
* Docker CLI support inside Jenkins
* AWS CLI support for pushing images to Amazon ECR
* GitHub App authentication for repository scanning
* Multibranch pipelines
* Optional Portainer webhook deployment
* Branch and release-tag based image publishing

## Repository Structure

```text
.
├── docker-compose.yml
├── Dockerfile
├── Jenkinsfile.sample
└── README.md
```

## What This Template Provides

This setup is intended to run a Jenkins controller container that can:

1. Authenticate to GitHub using a GitHub App.
2. Scan GitHub repositories using Multibranch Pipeline.
3. Build Docker images.
4. Push Docker images to Amazon ECR.
5. Trigger Portainer stack redeployments using webhooks.

## Prerequisites

You need the following before starting:

* A Docker host
* Docker Compose v2
* A GitHub organization or user account
* An AWS account with ECR access
* A Portainer instance if you want automated stack redeploys
* A public HTTPS URL for Jenkins if GitHub webhooks need to reach it

## 1. Start Jenkins

Clone this repository onto your Docker host.

```bash
git clone <this-repo-url>
cd <this-repo>
```

If your `docker-compose.yml` uses the host Docker socket, create a `.env` file with the Docker group ID:

```bash
echo "DOCKER_GID=$(getent group docker | cut -d: -f3)" > .env
```

Start Jenkins:

```bash
docker compose up -d --build
```

View the logs:

```bash
docker logs -f jenkins
```

Get the initial Jenkins admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open Jenkins:

```text
http://YOUR_SERVER_IP:8080
```

Complete the initial setup wizard and install suggested plugins.

## 2. Recommended Jenkins Plugins

Install these plugins:

```text
Git
Pipeline
GitHub
GitHub Branch Source
Docker Pipeline
Credentials Binding
AWS Credentials
```

Optional:

```text
Blue Ocean
```

The required plugins allow Jenkins to scan GitHub repositories, run `Jenkinsfile` pipelines, bind AWS credentials, and build/push Docker images.

## 3. Create a GitHub App

In GitHub, go to:

```text
Organization Settings → Developer settings → GitHub Apps → New GitHub App
```

Use values similar to these:

```text
GitHub App name: Jenkins CI
Homepage URL: https://jenkins.example.com/
Callback URL: https://jenkins.example.com/securityRealm/finishLogin
Webhook URL: https://jenkins.example.com/github-webhook/
```

Use your real Jenkins URL.

### GitHub App Repository Permissions

Set the following repository permissions:

```text
Contents: Read-only
Metadata: Read-only
Pull requests: Read-only
Commit statuses: Read and write
Checks: Read and write
```

If Jenkins needs to create tags or GitHub releases, grant:

```text
Contents: Read and write
```

### Webhook Events

Subscribe to:

```text
Check run
Check suite
Pull request
Push
Repository
```

Create the app.

## 4. Generate the GitHub App Private Key

After creating the app, open the GitHub App settings and go to:

```text
Private keys → Generate a private key
```

GitHub will download a `.pem` file.

Some Jenkins versions require the key to be in PKCS#8 format. If Jenkins rejects the key with a PKCS#8 error, convert it:

```bash
openssl pkcs8 \
  -topk8 \
  -inform PEM \
  -outform PEM \
  -in current-key.pem \
  -out new-key.pem \
  -nocrypt
```

Use the converted `new-key.pem` value in Jenkins.

The converted key should start with:

```text
-----BEGIN PRIVATE KEY-----
```

not:

```text
-----BEGIN RSA PRIVATE KEY-----
```

## 5. Install the GitHub App

In GitHub, go to your GitHub App and click:

```text
Install App
```

Install it into the organization or account that owns the repositories Jenkins should build.

For least privilege, choose:

```text
Only select repositories
```

Then select the repositories Jenkins should access.

## 6. Add GitHub App Credentials to Jenkins

In Jenkins, go to:

```text
Manage Jenkins → Credentials → System → Global credentials → Add Credentials
```

Create a credential:

```text
Kind: GitHub App
Scope: Global
App ID: Your numeric GitHub App ID
Private Key: Paste the full private key contents
ID: github-app
Description: GitHub App for Jenkins
```

Click **Test Connection**.

If the connection fails, verify:

* You used the numeric App ID, not the Client ID.
* The app is installed on the target organization or repo.
* The private key is complete.
* The key format is PKCS#8 if required by Jenkins.

## 7. Add AWS ECR Credentials to Jenkins

Create or use an AWS IAM user/role with ECR push permissions.

In Jenkins, go to:

```text
Manage Jenkins → Credentials → System → Global credentials → Add Credentials
```

Create:

```text
Kind: AWS Credentials
Scope: Global
ID: aws-ecr-creds
Access Key ID: your AWS access key
Secret Access Key: your AWS secret key
Description: AWS credentials for ECR image pushes
```

The pipeline expects the credential ID:

```text
aws-ecr-creds
```

## 8. Add Portainer Webhook Credentials

In Portainer, enable webhooks on the stack you want Jenkins to redeploy.

For a dev stack, create this Jenkins credential:

```text
Kind: Secret text
Scope: Global
Secret: Portainer dev stack webhook URL
ID: portainer-dev-webhook
Description: Portainer dev stack webhook
```

For a production stack, create:

```text
Kind: Secret text
Scope: Global
Secret: Portainer prod stack webhook URL
ID: portainer-prod-webhook
Description: Portainer prod stack webhook
```

If a project uses custom credential IDs, update the `Jenkinsfile` accordingly.

## 9. Create an ECR Repository

Create an ECR repository for your application.

Example:

```bash
aws ecr create-repository \
  --repository-name my-org/my-app \
  --region us-east-1
```

Your image URI will look like:

```text
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-org/my-app
```

## 10. Create a Multibranch Pipeline

In Jenkins:

```text
New Item → Multibranch Pipeline
```

Give it a name, then configure:

```text
Branch Sources → Add source → GitHub
```

Use:

```text
Credentials: github-app
Repository HTTPS URL: https://github.com/YOUR_ORG/YOUR_REPO
```

Or, depending on the Jenkins UI:

```text
Owner: YOUR_ORG
Repository: YOUR_REPO
Credentials: github-app
```

## 11. Configure Branch and Tag Discovery

In the Multibranch Pipeline configuration:

```text
Branch Sources → GitHub → Behaviors
```

Recommended behaviors:

```text
Discover branches
Discover pull requests from origin
Discover tags
```

To limit branch builds, add:

```text
Filter by name with wildcards
```

Use:

```text
Include: main dev
Exclude:
```

If using regex:

```regex
^(main|dev)$
```

Tags are handled separately from branches. The sample Jenkinsfile only treats tags matching this pattern as release tags:

```regex
^v[0-9]+\.[0-9]+\.[0-9]+.*$
```

Examples:

```text
v1.0.0
v1.1.1
v2.0.0
v1.2.3-rc1
```

## 12. Add a Jenkinsfile to Your Application Repo

Copy `Jenkinsfile.sample` into the root of your application repository as:

```text
Jenkinsfile
```

Then update the environment variables:

```groovy
AWS_REGION = "us-east-1"
AWS_ACCOUNT_ID = "123456789012"
ECR_REPOSITORY = "my-org/my-app"
```

Update Portainer credential IDs if needed:

```groovy
PORTAINER_WEBHOOK_CREDENTIAL = "portainer-dev-webhook"
PORTAINER_WEBHOOK_CREDENTIAL = "portainer-prod-webhook"
```

Commit and push.

## 13. Pipeline Behavior

The sample pipeline behaves like this:

### `dev` branch

Builds and pushes:

```text
:dev
:dev-BUILD_NUMBER
:sha-COMMIT
```

Then triggers the dev Portainer webhook.

### `main` branch

Builds and pushes:

```text
:main
:sha-COMMIT
```

It does not trigger Portainer.

This is useful when `main` should validate and publish an image, but production should only deploy from a release tag.

### Release tags

For a tag like:

```text
v1.1.1
```

The pipeline builds and pushes:

```text
:latest
:v1.1.1
:sha-COMMIT
```

Then triggers the production Portainer webhook.

This allows Portainer production stacks to continue using:

```text
:latest
```

while ECR still keeps immutable release tags like:

```text
:v1.1.1
```

### Other branches or tags

The pipeline exits successfully without building an image.

This avoids failed GitHub checks for branches that should not produce images.

## 14. Example Portainer Stack Images

Development stack:

```yaml
services:
  app:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-org/my-app:dev
    restart: unless-stopped
```

Production stack using latest:

```yaml
services:
  app:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-org/my-app:latest
    restart: unless-stopped
```

Production rollback example:

```yaml
services:
  app:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-org/my-app:v1.1.0
    restart: unless-stopped
```

## 15. Important ECR Pull Note for Portainer

If Portainer runs on a different Docker host than Jenkins, that host must be able to pull from ECR.

You can manually test on the Portainer Docker host:

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

For long-term production use, consider one of these:

* Amazon ECR Docker credential helper
* Scheduled ECR Docker login refresh
* Jenkins deploying directly to the target host
* ECS instead of Portainer

## 16. Common Issues

### Jenkins says GitHub App key must be PKCS#8

Convert the key:

```bash
openssl pkcs8 \
  -topk8 \
  -inform PEM \
  -outform PEM \
  -in current-key.pem \
  -out new-key.pem \
  -nocrypt
```

Use `new-key.pem` in Jenkins.

### Credential type mismatch for AWS

If the Jenkinsfile uses:

```groovy
AmazonWebServicesCredentialsBinding
```

then the credential must be:

```text
Kind: AWS Credentials
```

If the Jenkinsfile uses:

```groovy
usernamePassword(...)
```

then the credential must be:

```text
Kind: Username with password
```

This template expects:

```text
Kind: AWS Credentials
ID: aws-ecr-creds
```

### Portainer credential not found

Create a Jenkins Secret Text credential with the exact ID used in the Jenkinsfile.

Examples:

```text
portainer-dev-webhook
portainer-prod-webhook
```

### Dev branch does not build

Confirm the pipeline log shows:

```text
Branch: dev
Should build image: true
Deploy to Portainer: true
```

If not, confirm the Jenkinsfile committed to the `dev` branch contains the correct metadata logic.

### Main branch deploys unexpectedly

The sample pipeline intentionally does not deploy from `main`.

Only these deploy to Portainer:

```text
dev branch
release tags like v1.1.1
```

## 17. Recommended Release Flow

For development:

```bash
git checkout dev
git commit -am "Update app"
git push origin dev
```

This pushes the `:dev` image and redeploys the dev Portainer stack.

For production:

```bash
git checkout main
git pull
git tag v1.1.1
git push origin v1.1.1
```

Or create a GitHub Release with tag:

```text
v1.1.1
```

This pushes:

```text
:latest
:v1.1.1
:sha-COMMIT
```

and redeploys the production Portainer stack.

## 18. Security Notes

Mounting `/var/run/docker.sock` into Jenkins gives Jenkins powerful control over the Docker host.

This is convenient for a home lab, internal CI/CD server, or simple single-host setup, but for higher security environments consider:

* Jenkins agents instead of building on the controller
* Rootless Docker or isolated build workers
* Kaniko, BuildKit, or other daemonless build tools
* Separate deployment credentials per environment
* Least-privilege IAM policies for ECR
* Separate Portainer webhooks for dev and production
