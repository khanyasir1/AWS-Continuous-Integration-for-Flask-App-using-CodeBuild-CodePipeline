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
# Update the package index to get the latest package metadata
sudo apt update

# Install Ruby (required for CodeDeploy agent to run)
sudo apt install ruby -y

# Install wget (used to download files over the internet)
sudo apt install wget -y

# Navigate to the home directory of the 'ubuntu' user
cd /home/ubuntu

# Download the AWS CodeDeploy agent installation script from the S3 bucket (US East 1 region)
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install

# Make the downloaded install script executable
chmod +x ./install

# Run the install script with 'auto' flag to install CodeDeploy agent non-interactively
sudo ./install auto

# Start the CodeDeploy agent service
sudo systemctl start codedeploy-agent

# Enable the service to start automatically on system boot
sudo systemctl enable codedeploy-agent

# Check and display the status of the CodeDeploy agent
sudo systemctl status codedeploy-agent
````

> ✅ CodeDeploy agent receives commands from CodeDeploy to deploy your application to EC2.

---

## 🔐 IAM Role Setup: Choose One of Two Options

To allow **CodeDeploy** to deploy your app onto **EC2**, you need the right IAM roles. You can either:

1. Use **two separate roles** (recommended for production, better security isolation)
2. Use **a single unified role** (simpler for small projects or learning)

---

### 🧭 Option 1: Two Separate IAM Roles (Recommended for Production)

#### ✅ Step 3: IAM Role for EC2 (Instance Profile)

Create an IAM Role:

* **Name:** `CodeDeployEC2Role`
* **Policies:**
  - `AmazonEC2RoleforAWSCodeDeploy`
  - `AmazonS3ReadOnlyAccess`
* **Attach** this role to your EC2 instance

✅ This allows EC2 to be targeted by CodeDeploy and access S3 artifacts.

---

#### ✅ Step 4: IAM Role for CodeDeploy

Create another IAM Role:

* **Name:** `CodeDeployServiceRole`
* **Policy:** `AWSCodeDeployRole`
* **Use Case:** CodeDeploy

✅ This allows the CodeDeploy service to orchestrate the deployment process.

---

### 🧭 Option 2: Use a Single Unified IAM Role (Simpler for Learning/Demo)

You can combine both roles into one to simplify configuration.

#### ✅ Step 3: Create a Unified IAM Role

1. Go to **IAM > Roles > Create Role**
2. Choose **Trusted entity type:** AWS service
3. Use **Custom trust policy**, and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "ec2.amazonaws.com",
          "codedeploy.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
````

4. Attach the following policies:

* `AWSCodeDeployFullAccess`
* `AmazonEC2FullAccess`
* `AmazonS3ReadOnlyAccess` (if needed)

5. Name the role: **`CodeDeployEC2Role`**
6. Create the role.

---

#### ✅ Step 4: Attach Unified Role to EC2 Instance

1. Go to **EC2 Console > Instances**
2. Select your EC2 instance
3. Click **Actions > Security > Modify IAM Role**
4. Attach the `CodeDeployEC2Role` you created

✅ Now EC2 and CodeDeploy can both operate using a single role.

---

📌 **Note:** While the unified role is easier to manage, the **two-role setup is preferred for production** environments to follow **least privilege** principles and maintain security boundaries.




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

### ✅ Step 7: Write `appspec.yml`

```yaml
# The version of the AppSpec file format
version: 0.0

# Target operating system (Linux for EC2 instances)
os: linux

# Lifecycle event hooks define custom scripts to run at different stages of deployment
hooks:
  # This hook runs BEFORE the new version is installed
  # Used to gracefully stop running containers or services
  ApplicationStop:
    - location: scripts/stop_container.sh  # Path to the script to stop the app or container
      timeout: 300                         # Maximum time (in seconds) allowed for the script to run
      runas: root                          # Run this script as the root user

  # This hook runs AFTER files are copied to the EC2 instance
  # Used to start services or containers with the updated code
  AfterInstall:
    - location: scripts/start_container.sh # Path to the script that starts the app or container
      timeout: 300                         # Maximum time allowed for the script to complete
      runas: root                          # Run this script as the root user

```

---

### ✅ Step 8: Write Bash Scripts

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

# checks is there any running container , then copy the container Id
containerId=`docker ps | awk -F " " '{print $1}'`

#forcefully remove the container
docker rm -f $containerId
```

---



---

### ✅ Step 9: Trigger Manual Deployment

1. Go to **CodeDeploy → Application → Deployment Groups**
2. Click **Create Deployment**
3. Choose revision from **GitHub**
4. Select your branch (e.g., `main`)
5. Click **Deploy**

---

### ✅ Step 10: Automate with CodePipeline

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




