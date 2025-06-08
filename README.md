# 🧪 AWS CI/CD for Flask App using CodeBuild & CodePipeline

This project demonstrates a professional Continuous Integration (CI) setup for a Python Flask application using **AWS CodeBuild**, **AWS CodePipeline**, and **System Manager (SSM) Parameter Store**. The goal is to automate the build and Docker image push process to a private Docker registry.

---

## 📁 Project Structure


```
aws-devops-zero-to-hero/
├── day-14/
│   └── simple-python-app/
│       ├── app.py
│       ├── requirements.txt
│       └── Dockerfile
└── buildspec.yml
```

---

## 🚀 Step-by-Step CI Setup

### 🔧 1. **AWS CodeBuild Setup**

* **Project Name**: `continuous-integration-project`
* **Environment**:

  * **Provisioning model**: On-demand
  * **Image**: Managed image - `aws/codebuild/standard:7.0-25.04.04`
  * **Operating system**: Ubuntu
  * **Runtime**: Python 3.11
  * **Compute Type**: EC2
  * **Buildspec Location**: Insert build commands (inline)
* **Source**: GitHub (public repo connected via OAuth)

  * **Repository URL**: `https://github.com/iam-veeramalla/aws-devops-zero-to-hero`

### 🔐 2. **AWS Systems Manager (SSM) Parameter Store**

Store sensitive Docker credentials securely:

| Parameter Name                       | Type     | Description                       |
| ------------------------------------ | -------- | --------------------------------- |
| `/myapp/docker-credentials/username` | Standard | Docker registry username          |
| `/myapp/docker-credentials/password` | Standard | Docker registry password (secret) |
| `/myapp/docker-registry/url`         | Standard | Docker registry URL               |

> 🔒 Make sure the type is `SecureString` for sensitive data.

### 🛡️ 3. **IAM Role Permissions**

Grant CodeBuild access to SSM:

1. Go to **IAM > Roles**.
2. Select the role: `codebuild-continuous-integration-project-service-role`.
3. Attach Policy: `AmazonSSMFullAccess`.

> This allows the build to fetch credentials stored in the Parameter Store.

---

## 📄 `buildspec.yml` (Inline in CodeBuild)

```yaml
version: 0.2                 # Buildspec syntax version enabling latest features

env:
  parameter-store:            # Pull environment variables securely from SSM Parameter Store
    DOCKER_REGISTRY_USERNAME: /myapp/docker-credentials/username    # Docker registry username
    DOCKER_REGISTRY_PASSWORD: /myapp/docker-credentials/password    # Docker registry password (secure)
    DOCKER_REGISTRY_URL: /myapp/docker-registry/url                # Docker registry URL

phases:
  install:
    runtime-versions:
      python: 3.11           # Specify Python version for consistent runtime environment
    commands:
      - echo "Installing dependencies..."  # Log for clarity

  pre_build:
    commands:
      - echo "Running pre-build tasks..."
      - pip install -r day-14/simple-python-app/requirements.txt  # Install Python dependencies for Flask app
      - echo "Completed pre-build tasks..."

  build:
    commands:
      - echo "Running tests..."  # Placeholder for test commands (add actual tests here if any)
      - cd day-14/simple-python-app/ # Move to app directory for Docker build context
      - echo "Building Docker image..."
      - echo "$DOCKER_REGISTRY_PASSWORD" | docker login -u "$DOCKER_REGISTRY_USERNAME" --password-stdin "$DOCKER_REGISTRY_URL"
        # Secure login to Docker registry using credentials from Parameter Store
      - docker build -t "$DOCKER_REGISTRY_URL/$DOCKER_REGISTRY_USERNAME/simple-python-flask-app:latest" .
        # Build Docker image and tag with repo and username info
      - docker push "$DOCKER_REGISTRY_URL/$DOCKER_REGISTRY_USERNAME/simple-python-flask-app:latest"
        # Push Docker image to private registry
      - echo "Successfully completed the building process..."

  post_build:
    commands:
      - echo "Build completed successfully!"   # Final confirmation log
```

---

## ⚙️ Triggering the Build

Initially, we are **manually starting builds** from the CodeBuild console. This helps test CI in isolation.

Once verified, we integrate it with **CodePipeline** for automation.

---

## 🔄 AWS CodePipeline Setup

1. **Pipeline Name**: `flask-app-ci-pipeline`
2. **Source Provider**: GitHub (Version 2)

   * Create GitHub connection (OAuth)
   * Choose the repository and branch
3. **Build Provider**: AWS CodeBuild

   * Select the `continuous-integration-project`
4. **Deploy Provider**: (Optional at this stage)

   * Will be added using **CodeDeploy** in later stages

> Now every `git push` will trigger a build via CodePipeline automatically.

---



### 🔹 Key Features

* End-to-end CI for a Flask app using AWS-native tools
* Secrets managed securely with SSM Parameter Store
* Dockerized build process with image push to private registry
* Automated via CodePipeline

### 🧑‍💻 Technologies Used

* Python (Flask), Docker
* AWS CodeBuild, CodePipeline
* AWS Systems Manager (SSM), IAM

---

## 📘 How to Run This Project

```bash
# Clone the project
$ git clone https://github.com/iam-veeramalla/aws-devops-zero-to-hero

# Navigate to Flask app folder
$ cd day-14/simple-python-app/

# (Optional) Run locally
$ pip install -r requirements.txt
$ python app.py
```

---


````markdown
# 🚀 Continuous Deployment (CD) with AWS CodeDeploy, EC2, and CodePipeline

This guide walks you through a **step-by-step setup** of **Continuous Deployment (CD) in AWS using CodeDeploy**, integrated with EC2 and CodePipeline.

## 📋 What's Covered?

- Creating a CodeDeploy application
- Configuring an EC2 instance
- Installing CodeDeploy agent
- Writing deployment scripts & `appspec.yml`
- Triggering deployment manually
- Automating with CodePipeline

---

## 🧩 Why Use These Services?

| AWS Service      | Purpose                                                  |
| ---------------- | -------------------------------------------------------- |
| **EC2**          | Server (host) where your application runs                |
| **CodeDeploy**   | Automates deployment to EC2                              |
| **CodePipeline** | Automates the full CI/CD flow (build → test → deploy)    |
| **GitHub**       | Code repository (source code + deployment files/scripts) |

---

## 🛠️ Step-by-Step Deployment Process

---

### ✅ Step 1: Launch an EC2 Instance (Ubuntu)

1. Go to EC2 Console → Launch instance  
2. Choose **Ubuntu Server 22.04 LTS**  
3. Select instance type: `t2.micro`  
4. Configure security group:  
   - Allow ports: **22 (SSH)**, **80 (HTTP)**, **443 (HTTPS)**, **5000 (Flask)**  
5. Add tag: `Name = MyAppServer`  
6. Save key-pair and launch instance  

---

### ✅ Step 2: Install CodeDeploy Agent on EC2 (Ubuntu)

SSH into your EC2 instance and run:

```bash
sudo apt update
sudo apt install ruby -y
sudo apt install wget -y

cd /home/ubuntu
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

# Start and enable CodeDeploy agent
sudo systemctl start codedeploy-agent
sudo systemctl enable codedeploy-agent
sudo systemctl status codedeploy-agent
````

> ✅ CodeDeploy agent receives commands from CodeDeploy to deploy your application to EC2.

---

### ✅ Step 3: IAM Role for EC2 (Instance Profile)

Create an IAM Role:

* **Name:** `CodeDeployEC2Role`
* **Policy:** `AmazonEC2RoleforAWSCodeDeploy`, `AmazonS3ReadOnlyAccess`
* **Attach to EC2 Instance**

> ✅ Allows CodeDeploy to access EC2 and pull code from S3.

---

### ✅ Step 4: IAM Role for CodeDeploy

Create another IAM role:

* **Name:** `CodeDeployServiceRole`
* **Policy:** `AWSCodeDeployRole`
* **Use Case:** CodeDeploy

---

### ✅ Step 5: Create CodeDeploy Application

1. Go to AWS CodeDeploy → Applications → **Create Application**
2. Application name: `MyApp`
3. Compute platform: **EC2/On-Premises**

---

### ✅ Step 6: Create Deployment Group

1. Inside your app → **Create Deployment Group**
2. Name: `MyAppDeploymentGroup`
3. Service role: `CodeDeployServiceRole`
4. Deployment type: **In-place**
5. Environment config: select EC2 instance with tag `Name=MyAppServer`
6. Deployment config: `CodeDeployDefault.AllAtOnce`
7. Load Balancer: Skip (unless needed)

---


---

### ✅ Step 8: Write `appspec.yml`

```yaml
version: 0.0
os: linux

hooks:
  ApplicationStop:
    - location: scripts/stop_container.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/start_container.sh
      timeout: 300
      runas: root
```

---

### ✅ Step 9: Write Bash Scripts

**scripts/start\_container.sh**

```bash
#!/bin/bash
set -e
echo "Starting Docker container..."

# Pull the Docker image from Docker Hub
docker pull khanmohammedyasir/simple-python-flask-app

# Run the Docker image as a container
docker run -d -p 5000:5000 khanmohammedyasir/simple-python-flask-app



```

**scripts/stop\_container.sh**

```bash
#!/bin/bash
set -e
echo "Stopping Docker container..."

# Stop the running container (if any)
containerId = `docker ps | awk -F " " '{print $1}'`
docker rm -f $containerId
```

---



---

### ✅ Step 11: Trigger Manual Deployment

1. Go to **CodeDeploy → Application → Deployment Groups**
2. Click **Create Deployment**
3. Choose revision from **GitHub**
4. Select your branch (e.g., `main`)
5. Click **Deploy**

---

### ✅ Step 12: Automate with CodePipeline

1. Create pipeline with stages:

   * **Source**: GitHub (connect and select branch)
   * **Build**: Optional (if not using CodeBuild, skip)
   * **Deploy**: AWS CodeDeploy → `MyAppDeploymentGroup`

---

## ✅ CD Architecture

```text
GitHub (Source Code)
     ⬇️
CodePipeline (CI/CD)
     ⬇️
CodeDeploy (Delivery)
     ⬇️
EC2 (Docker container)
```

---




