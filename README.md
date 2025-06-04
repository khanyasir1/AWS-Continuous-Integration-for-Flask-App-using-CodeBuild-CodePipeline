# 🧪 AWS Continuous Integration for Flask App using CodeBuild & CodePipeline

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


