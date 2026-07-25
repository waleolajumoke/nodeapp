# Deploying a Node.js API to AWS EC2 with Jenkins, Docker Hub, and SSH

## 1. Overview

This guide explains how to build a complete Continuous Integration and Continuous Deployment pipeline for a Node.js API.

The pipeline will perform the following operations whenever code is pushed to GitHub:

1. GitHub notifies Jenkins through a webhook.
2. Jenkins pulls the latest source code.
3. Jenkins installs dependencies and runs tests.
4. Jenkins builds a Docker image.
5. Jenkins signs in to Docker Hub using stored credentials.
6. Jenkins pushes the Docker image to Docker Hub.
7. Jenkins connects to an AWS EC2 instance through SSH.
8. The EC2 instance pulls the new Docker image.
9. The old container is replaced with the updated container.

## 2. Deployment Architecture

```text
Developer
   |
   | git push
   v
GitHub Repository
   |
   | Webhook
   v
Jenkins Server
   |
   | Build and test
   | Build Docker image
   | Push image
   v
Docker Hub
   |
   | docker pull
   v
AWS EC2 Instance
   |
   v
Node.js API Container
```

## 3. Tools Used

| Tool | Purpose |
|---|---|
| GitHub | Stores the application source code |
| Jenkins | Automates testing, building, publishing, and deployment |
| Docker | Packages the Node.js API as a portable image |
| Docker Hub | Stores the Docker image |
| AWS EC2 | Hosts the running application |
| SSH | Allows Jenkins to securely execute deployment commands on EC2 |

## 4. Assumptions

This guide assumes:

- The application is a Node.js API.
- The application listens on port `3000`.
- The main branch is named `main`.
- Jenkins runs on a Linux server.
- The deployment server is an Ubuntu EC2 instance.
- A Docker Hub repository has already been created.
- Jenkins can reach the EC2 instance through port `22`.

Replace all placeholder values with your real values.

## 5. Recommended Project Structure

```text
nodejs-api/
├── src/
│   └── server.js
├── tests/
├── .dockerignore
├── .gitignore
├── Dockerfile
├── Jenkinsfile
├── package.json
├── package-lock.json
└── README.md
```

## 6. Prepare the Node.js Application

A basic Express application can look like this:

```javascript
const express = require("express");

const app = express();
const port = process.env.PORT || 3000;

app.use(express.json());

app.get("/", (req, res) => {
  res.json({
    message: "Node.js API is running",
    environment: process.env.NODE_ENV || "development"
  });
});

app.get("/health", (req, res) => {
  res.status(200).json({ status: "healthy" });
});

app.listen(port, "0.0.0.0", () => {
  console.log(`API listening on port ${port}`);
});
```

The application must listen on `0.0.0.0`, not only `localhost`, so that traffic can reach it from outside the container.

### Example `package.json`

```json
{
  "name": "nodejs-api",
  "version": "1.0.0",
  "description": "Node.js API deployed with Jenkins and Docker",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "test": "node --test"
  },
  "dependencies": {
    "express": "^5.1.0"
  },
  "engines": {
    "node": ">=20"
  }
}
```

Install dependencies locally:

```bash
npm install
```

Run the API:

```bash
npm start
```

Test it locally:

```bash
curl http://localhost:3000/health
```

Expected response:

```json
{"status":"healthy"}
```

## 7. Create the Dockerfile

Create a file named `Dockerfile` in the root of the project:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

### Why this Dockerfile is structured this way

- `node:22-alpine` provides a smaller Node.js image.
- `WORKDIR /app` creates the working directory inside the container.
- Dependency files are copied before application files to improve Docker layer caching.
- `npm ci` provides a predictable installation based on `package-lock.json`.
- `--omit=dev` avoids installing development-only packages in the production image.
- `USER node` avoids running the application as the root user.
- `EXPOSE 3000` documents the port used by the application.

## 8. Create `.dockerignore`

Create `.dockerignore`:

```text
node_modules
npm-debug.log
.git
.gitignore
Jenkinsfile
.env
.env.*
coverage
README.md
Dockerfile*
docker-compose*.yml
```

Do not copy `.env` files into the image. Runtime secrets should be supplied when the container starts.

## 9. Test the Docker Image Locally

Build the image:

```bash
docker build -t nodejs-api:local .
```

Run the container:

```bash
docker run -d \
  --name nodejs-api-local \
  -p 3000:3000 \
  nodejs-api:local
```

Check the running container:

```bash
docker ps
```

Test the API:

```bash
curl http://localhost:3000/health
```

View container logs:

```bash
docker logs nodejs-api-local
```

Remove the test container:

```bash
docker rm -f nodejs-api-local
```

## 10. Create a Docker Hub Repository

1. Sign in to Docker Hub.
2. Select **Create repository**.
3. Enter a repository name such as `nodejs-api`.
4. Choose public or private visibility.
5. Create the repository.

The full image name will look like this:

```text
DOCKERHUB_USERNAME/nodejs-api
```

Example:

```text
wale123/nodejs-api
```

For Jenkins automation, use a Docker Hub access token instead of your main Docker Hub password.

## 11. Launch the AWS EC2 Deployment Server

### Recommended starting configuration

- Operating system: Ubuntu Server
- Instance type: `t3.micro` or another size suitable for the workload
- Storage: At least 10 GB
- Public IP: Enabled, or attach an Elastic IP
- Key pair: Create or select an existing key pair

### Security group inbound rules

| Type | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | Jenkins server public IP | Jenkins deployment access |
| Custom TCP | 3000 | Your trusted IP or `0.0.0.0/0` temporarily | Direct API access |
| HTTP | 80 | `0.0.0.0/0` | Optional reverse proxy access |
| HTTPS | 443 | `0.0.0.0/0` | Optional secure production access |

For better security, do not leave SSH port `22` open to the entire internet. Restrict it to the Jenkins server IP whenever possible.

## 12. Connect to the EC2 Instance

Change the private key permission:

```bash
chmod 400 nodejs-api-key.pem
```

Connect to Ubuntu EC2:

```bash
ssh -i nodejs-api-key.pem ubuntu@EC2_PUBLIC_IP
```

For Amazon Linux, the default username is commonly:

```text
ec2-user
```

For Ubuntu, the default username is commonly:

```text
ubuntu
```

## 13. Install Docker on the Ubuntu EC2 Instance

Run the following commands on the EC2 instance:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"${UBUNTU_CODENAME:-$VERSION_CODENAME}\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start and enable Docker:

```bash
sudo systemctl enable --now docker
```

Add the EC2 user to the Docker group:

```bash
sudo usermod -aG docker ubuntu
```

Log out and reconnect so the group change becomes active:

```bash
exit
```

Reconnect:

```bash
ssh -i nodejs-api-key.pem ubuntu@EC2_PUBLIC_IP
```

Confirm Docker works:

```bash
docker --version
docker run --rm hello-world
```

## 14. Prepare the EC2 Application Directory

Create a directory for deployment configuration:

```bash
mkdir -p /home/ubuntu/nodejs-api
cd /home/ubuntu/nodejs-api
```

### Optional environment file

Create a runtime environment file:

```bash
nano .env
```

Example:

```env
NODE_ENV=production
PORT=3000
DATABASE_URL=replace-with-real-database-url
JWT_SECRET=replace-with-a-strong-secret
```

Protect the file:

```bash
chmod 600 .env
```

Do not commit this file to GitHub.

## 15. Prepare the Jenkins Server

Jenkins requires the following software:

- Git
- Docker Engine and Docker CLI
- SSH client
- `ssh-agent`
- Jenkins Pipeline plugins

### Install Docker on the Jenkins server

On Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y docker.io openssh-client
sudo systemctl enable --now docker
```

Allow the Jenkins user to use Docker:

```bash
sudo usermod -aG docker jenkins
```

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

You may also need to log out and restart the Jenkins server before group membership is refreshed.

Test Docker as the Jenkins user:

```bash
sudo -u jenkins docker version
```

If Jenkins cannot access Docker, inspect the Docker socket:

```bash
ls -l /var/run/docker.sock
id jenkins
```

Avoid using `chmod 777 /var/run/docker.sock`. Add Jenkins to the Docker group instead.

## 16. Install Jenkins Plugins

Open:

```text
Manage Jenkins > Plugins
```

Install or confirm these plugins:

- Pipeline
- Git
- GitHub Integration
- Credentials Binding
- SSH Agent
- Docker Pipeline, optional for advanced Docker syntax

Restart Jenkins when required.

## 17. Configure Docker Hub Credentials in Jenkins

Open:

```text
Manage Jenkins > Credentials > System > Global credentials > Add Credentials
```

Use these settings:

| Setting | Value |
|---|---|
| Kind | Username with password |
| Username | Docker Hub username |
| Password | Docker Hub access token |
| ID | `dockerhub-credentials` |
| Description | Docker Hub access token |

The credential ID must match the value used inside the `Jenkinsfile`.

## 18. Configure the EC2 SSH Credential in Jenkins

Use the private key that matches the public key authorized on the EC2 instance.

Open:

```text
Manage Jenkins > Credentials > System > Global credentials > Add Credentials
```

Use these settings:

| Setting | Value |
|---|---|
| Kind | SSH Username with private key |
| ID | `ec2-ssh-key` |
| Username | `ubuntu` |
| Private Key | Enter directly, then paste the private key content |
| Passphrase | Enter it only if the key is protected |
| Description | SSH key for Node.js EC2 deployment |

To display the private key locally:

```bash
cat nodejs-api-key.pem
```

Copy the full content, including:

```text
-----BEGIN ... PRIVATE KEY-----
...
-----END ... PRIVATE KEY-----
```

Never add the private key to GitHub.

## 19. Configure Jenkins Environment Values

The Jenkinsfile below uses the following values:

| Variable | Example |
|---|---|
| `DOCKER_IMAGE` | `dockerhubusername/nodejs-api` |
| `EC2_HOST` | `203.0.113.10` |
| `EC2_USER` | `ubuntu` |
| `CONTAINER_NAME` | `nodejs-api` |
| `APP_PORT` | `3000` |

You can place non-secret values directly in the Jenkinsfile. Secrets must be stored in Jenkins Credentials.

## 20. Create the Jenkinsfile

Create a file named `Jenkinsfile` in the project root:

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        DOCKER_IMAGE   = 'DOCKERHUB_USERNAME/nodejs-api'
        DOCKER_CREDS   = 'dockerhub-credentials'
        EC2_SSH_CREDS  = 'ec2-ssh-key'
        EC2_USER       = 'ubuntu'
        EC2_HOST       = 'EC2_PUBLIC_IP_OR_DNS'
        CONTAINER_NAME = 'nodejs-api'
        APP_PORT       = '3000'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"
                }

                sh '''
                    docker build \
                      --tag ${DOCKER_IMAGE}:${IMAGE_TAG} \
                      --tag ${DOCKER_IMAGE}:latest \
                      .
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.DOCKER_CREDS,
                        usernameVariable: 'DOCKERHUB_USERNAME',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DOCKERHUB_TOKEN" | \
                          docker login --username "$DOCKERHUB_USERNAME" --password-stdin

                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Prepare SSH Host Key') {
            steps {
                sh '''
                    mkdir -p ~/.ssh
                    chmod 700 ~/.ssh
                    ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts
                    chmod 600 ~/.ssh/known_hosts
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: [env.EC2_SSH_CREDS]) {
                    sh '''
                        ssh "${EC2_USER}@${EC2_HOST}" \
                          DOCKER_IMAGE="${DOCKER_IMAGE}" \
                          IMAGE_TAG="${IMAGE_TAG}" \
                          CONTAINER_NAME="${CONTAINER_NAME}" \
                          APP_PORT="${APP_PORT}" \
                          'bash -s' <<'REMOTE_SCRIPT'
                            set -euo pipefail

                            echo "Pulling image ${DOCKER_IMAGE}:${IMAGE_TAG}"
                            docker pull "${DOCKER_IMAGE}:${IMAGE_TAG}"

                            echo "Removing existing container when present"
                            docker rm -f "${CONTAINER_NAME}" 2>/dev/null || true

                            echo "Starting updated container"
                            docker run -d \
                              --name "${CONTAINER_NAME}" \
                              --restart unless-stopped \
                              --env-file /home/ubuntu/nodejs-api/.env \
                              -p "${APP_PORT}:3000" \
                              "${DOCKER_IMAGE}:${IMAGE_TAG}"

                            echo "Checking container state"
                            docker ps --filter "name=${CONTAINER_NAME}"

                            echo "Removing unused images"
                            docker image prune -f
REMOTE_SCRIPT
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    attempts=12

                    until curl --fail --silent \
                      "http://${EC2_HOST}:${APP_PORT}/health" > /dev/null; do
                        attempts=$((attempts - 1))

                        if [ "$attempts" -le 0 ]; then
                            echo "Application health check failed"
                            exit 1
                        fi

                        echo "Waiting for the API to become healthy"
                        sleep 5
                    done

                    echo "Deployment health check passed"
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully"
        }

        failure {
            echo "Pipeline failed. Review the Jenkins console output."
        }

        always {
            sh 'docker logout >/dev/null 2>&1 || true'
            cleanWs(deleteDirs: true, notFailBuild: true)
        }
    }
}
```

### Important replacements

Change:

```groovy
DOCKER_IMAGE = 'DOCKERHUB_USERNAME/nodejs-api'
```

To your real Docker Hub image name:

```groovy
DOCKER_IMAGE = 'wale123/nodejs-api'
```

Change:

```groovy
EC2_HOST = 'EC2_PUBLIC_IP_OR_DNS'
```

To the EC2 public IP or DNS name:

```groovy
EC2_HOST = 'ec2-203-0-113-10.compute-1.amazonaws.com'
```

## 21. Why `ssh-keyscan` Is Used

Before Jenkins connects to EC2, the EC2 host key must be present in the Jenkins user's `known_hosts` file.

Correct command:

```bash
ssh-keyscan -H "$EC2_HOST" >> ~/.ssh/known_hosts
```

Do not use:

```bash
ssh-keygen -H hostname
```

`ssh-keygen -H` hashes entries already stored in a known-hosts file. It does not retrieve a remote server host key. Using a hostname as an extra argument can produce a `Too many arguments` error.

For a stricter production setup, manually verify and store the EC2 host fingerprint instead of automatically trusting the result of `ssh-keyscan` during every build.

## 22. Alternative Deployment Stage for a Public Docker Hub Repository

When the image is public, EC2 does not need Docker Hub credentials to pull it.

The deployment command can simply run:

```bash
docker pull DOCKERHUB_USERNAME/nodejs-api:latest
```

## 23. Deploying a Private Docker Hub Image

For a private image, the EC2 instance must also authenticate to Docker Hub.

One option is to store a separate Docker Hub token in Jenkins and pass it temporarily during deployment:

```groovy
stage('Deploy Private Image to EC2') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-credentials',
                usernameVariable: 'DOCKERHUB_USERNAME',
                passwordVariable: 'DOCKERHUB_TOKEN'
            )
        ]) {
            sshagent(credentials: ['ec2-ssh-key']) {
                sh '''
                    printf '%s' "$DOCKERHUB_TOKEN" | \
                    ssh "${EC2_USER}@${EC2_HOST}" \
                      "docker login --username '$DOCKERHUB_USERNAME' --password-stdin && \
                       docker pull '${DOCKER_IMAGE}:${IMAGE_TAG}' && \
                       docker logout"
                '''
            }
        }
    }
}
```

A more secure production design is to use a deployment-specific registry token with the minimum required permissions.

## 24. Create the Jenkins Pipeline Job

1. Open the Jenkins dashboard.
2. Select **New Item**.
3. Enter a name such as `nodejs-api-pipeline`.
4. Select **Pipeline**.
5. Select **OK**.
6. Under **Pipeline**, choose **Pipeline script from SCM**.
7. Select **Git** as the SCM.
8. Enter the GitHub repository URL.
9. Add GitHub credentials if the repository is private.
10. Set the branch specifier to:

```text
*/main
```

11. Set the script path to:

```text
Jenkinsfile
```

12. Save the job.
13. Select **Build Now** for the first manual test.

## 25. Configure the GitHub Webhook

The Jenkins server must be reachable from GitHub.

In GitHub:

1. Open the repository.
2. Select **Settings**.
3. Select **Webhooks**.
4. Select **Add webhook**.
5. Set the payload URL to:

```text
http://JENKINS_PUBLIC_IP:8080/github-webhook/
```

For production, use HTTPS and a domain:

```text
https://jenkins.example.com/github-webhook/
```

6. Set content type to:

```text
application/json
```

7. Select push events.
8. Save the webhook.

In the Jenkins job, enable the GitHub hook trigger when required by your Jenkins configuration.

## 26. First Pipeline Run

During the first run, monitor these stages:

```text
Checkout
Install Dependencies
Test
Build Docker Image
Push Image to Docker Hub
Prepare SSH Host Key
Deploy to EC2
Health Check
```

After a successful run, check Docker Hub for these tags:

```text
latest
BUILD_NUMBER-COMMIT_HASH
```

Example:

```text
latest
15-a71b2cd
```

## 27. Verify the Deployment on EC2

Connect to EC2:

```bash
ssh -i nodejs-api-key.pem ubuntu@EC2_PUBLIC_IP
```

Check the container:

```bash
docker ps
```

View logs:

```bash
docker logs nodejs-api
```

Follow logs continuously:

```bash
docker logs -f nodejs-api
```

Inspect the container:

```bash
docker inspect nodejs-api
```

Test from the EC2 server:

```bash
curl http://localhost:3000/health
```

Test from your computer:

```bash
curl http://EC2_PUBLIC_IP:3000/health
```

## 28. Common Errors and Solutions

### Error: `docker: permission denied`

Example:

```text
permission denied while trying to connect to the Docker daemon socket
```

Confirm that Jenkins belongs to the Docker group:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
id jenkins
```

On EC2:

```bash
sudo usermod -aG docker ubuntu
```

Log out and reconnect after changing group membership.

### Error: `sshagent` is not recognized

Install the SSH Agent plugin:

```text
Manage Jenkins > Plugins > Available plugins > SSH Agent
```

### Error: `ssh-agent executable could not be found`

Install the OpenSSH client on the Jenkins machine:

```bash
sudo apt-get update
sudo apt-get install -y openssh-client
```

Confirm:

```bash
which ssh-agent
which ssh
```

### Error: `Too many arguments` from `ssh-keygen`

Wrong:

```bash
ssh-keygen -H ec2-hostname
```

Correct:

```bash
ssh-keyscan -H ec2-hostname >> ~/.ssh/known_hosts
```

### Error: `Host key verification failed`

Create and update the `known_hosts` file as the Jenkins user:

```bash
sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh
sudo -u jenkins ssh-keyscan -H EC2_HOST \
  >> /var/lib/jenkins/.ssh/known_hosts
sudo chmod 700 /var/lib/jenkins/.ssh
sudo chmod 600 /var/lib/jenkins/.ssh/known_hosts
```

### Error: `Permission denied (publickey)`

Check the following:

- The Jenkins credential contains the correct private key.
- The Jenkins SSH username is correct.
- The matching public key exists in `~/.ssh/authorized_keys` on EC2.
- Port `22` is allowed from the Jenkins server IP.
- The private key format was copied completely.

Test from the Jenkins server:

```bash
sudo -u jenkins ssh ubuntu@EC2_HOST
```

When using a credential-managed key, test through a Jenkins pipeline rather than exposing the private key on disk.

### Error: `docker login` fails

Confirm:

- The username is the Docker Hub username, not the email address.
- The password field contains a valid Docker Hub access token.
- The Jenkins credential ID matches `dockerhub-credentials`.
- The Jenkins server can reach Docker Hub through HTTPS.

### Error: Docker image push is denied

Example:

```text
requested access to the resource is denied
```

Confirm that `DOCKER_IMAGE` begins with the same Docker Hub username stored in Jenkins:

```text
correctusername/nodejs-api
```

### Error: Container exits immediately

Check logs:

```bash
docker logs nodejs-api
```

Check the application command:

```bash
npm start
```

Confirm that the application listens on:

```javascript
app.listen(port, "0.0.0.0");
```

### Error: API cannot be reached externally

Check:

```bash
docker ps
sudo ss -lntp | grep 3000
curl http://localhost:3000/health
```

Also confirm that the EC2 security group allows the application port.

### Error: `.env` file does not exist

The pipeline uses:

```bash
--env-file /home/ubuntu/nodejs-api/.env
```

Create the file on EC2, or remove the `--env-file` option when the application does not require it.

## 29. Rollback Procedure

Each build is pushed with a unique tag such as:

```text
25-a71b2cd
```

To roll back, connect to EC2 and run:

```bash
export DOCKER_IMAGE="DOCKERHUB_USERNAME/nodejs-api"
export PREVIOUS_TAG="24-f91c812"

docker pull "${DOCKER_IMAGE}:${PREVIOUS_TAG}"
docker rm -f nodejs-api || true

docker run -d \
  --name nodejs-api \
  --restart unless-stopped \
  --env-file /home/ubuntu/nodejs-api/.env \
  -p 3000:3000 \
  "${DOCKER_IMAGE}:${PREVIOUS_TAG}"
```

Verify:

```bash
curl http://localhost:3000/health
```

Unique image tags are important because `latest` alone does not clearly identify the previously working release.

## 30. Optional Docker Compose Deployment

Create `/home/ubuntu/nodejs-api/compose.yaml` on EC2:

```yaml
services:
  api:
    image: DOCKERHUB_USERNAME/nodejs-api:${IMAGE_TAG:-latest}
    container_name: nodejs-api
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "3000:3000"
```

Deployment commands:

```bash
cd /home/ubuntu/nodejs-api
export IMAGE_TAG="25-a71b2cd"
docker compose pull
docker compose up -d --remove-orphans
```

The Jenkins deployment stage can execute these commands remotely.

## 31. Recommended Production Improvements

### Use a reverse proxy

Place Nginx, an Application Load Balancer, or another reverse proxy in front of the container. This allows the public API to use ports `80` and `443` while the container remains on an internal port.

### Enable HTTPS

Use an SSL certificate and expose the API through HTTPS rather than sending production traffic directly to port `3000`.

### Use an Elastic IP or DNS name

A normal EC2 public IP can change when the instance is stopped and started. Use an Elastic IP or Route 53 DNS record for a stable endpoint.

### Restrict SSH access

Allow SSH only from the Jenkins server IP, a VPN, a bastion host, or another trusted network.

### Protect secrets

Do not place the following in the repository or Docker image:

- Database passwords
- API keys
- JWT secrets
- SSH private keys
- Docker Hub access tokens
- Production `.env` files

Use Jenkins Credentials, AWS Systems Manager Parameter Store, or AWS Secrets Manager.

### Add image scanning

Scan container images before deployment using a supported image security scanner.

### Use immutable tags

Deploy a unique build tag instead of relying only on `latest`.

### Add automated rollback

Save the currently running image tag before replacement. If the health check fails, restart the previous image.

### Use separate environments

Create separate deployment settings and credentials for:

```text
development
staging
production
```

### Avoid building directly on the Jenkins controller

For larger production environments, use dedicated Jenkins agents for application builds.

## 32. Simplified Pipeline Flow

```text
Git push
   ↓
GitHub webhook
   ↓
Jenkins checkout
   ↓
npm ci
   ↓
npm test
   ↓
docker build
   ↓
docker login
   ↓
docker push
   ↓
SSH into EC2
   ↓
docker pull
   ↓
replace container
   ↓
health check
```

## 33. Deployment Checklist

### GitHub

- [ ] Node.js source code has been pushed.
- [ ] `Dockerfile` has been committed.
- [ ] `.dockerignore` has been committed.
- [ ] `Jenkinsfile` has been committed.
- [ ] GitHub webhook has been created.

### Docker Hub

- [ ] Repository has been created.
- [ ] Access token has been created.
- [ ] Image repository name matches the Jenkinsfile.

### AWS EC2

- [ ] EC2 instance is running.
- [ ] Docker is installed.
- [ ] Docker starts automatically.
- [ ] EC2 user can execute Docker commands.
- [ ] SSH port is restricted to the Jenkins server IP.
- [ ] Application port is allowed where required.
- [ ] `.env` exists and has restrictive permissions.

### Jenkins

- [ ] Jenkins can execute Docker commands.
- [ ] SSH Agent plugin is installed.
- [ ] Docker Hub credential ID is `dockerhub-credentials`.
- [ ] EC2 SSH credential ID is `ec2-ssh-key`.
- [ ] GitHub repository is configured.
- [ ] Pipeline branch is `main`.
- [ ] First manual build succeeds.

## 34. Official Reference Notes

- Jenkins recommends storing pipeline definitions in a `Jenkinsfile` checked into source control.
- Jenkins credentials can be bound temporarily to environment variables within a pipeline step.
- The Jenkins SSH Agent plugin supplies private-key credentials to an `ssh-agent` during a build.
- Docker supports non-interactive registry authentication through `docker login --password-stdin`.
- Docker Hub access tokens are intended for automation and avoid exposing the account password.
- EC2 security groups control inbound and outbound traffic, including SSH and application ports.

## 35. Final Result

After the setup is complete, every push to the `main` branch can automatically:

```text
Test the Node.js API
Build a Docker image
Push the image to Docker Hub
Connect securely to AWS EC2
Pull and run the latest release
Verify the API health endpoint
```

This provides a practical Continuous Integration and Continuous Deployment workflow while keeping Docker Hub tokens and EC2 SSH private keys inside Jenkins Credentials rather than inside the source code.
