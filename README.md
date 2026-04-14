# Flask CI/CD Pipeline using Jenkins and GitHub Actions

## Project Overview

This project demonstrates a complete CI/CD implementation for a Flask web application integrated with MongoDB Atlas. The assignment is implemented in two parts:

* Part 1: Jenkins CI/CD Pipeline
* Part 2: GitHub Actions CI/CD Pipeline

The application performs CRUD operations on student records and stores data in MongoDB Atlas.

---

## Architecture Diagram

```
                +----------------------+
                |      Developer       |
                |   (Code Commit)      |
                +----------+-----------+
                           |
                           v
                +----------------------+
                |      GitHub Repo     |
                +----------+-----------+
                           |
        +------------------+------------------+
        |                                     |
        v                                     v
+----------------------+          +----------------------+
|   GitHub Actions     |          |       Jenkins        |
|   (CI/CD Pipeline)   |          |   (Docker on EC2)    |
+----------+-----------+          +----------+-----------+
           |                                 |
           v                                 v
   +------------------+             +------------------+
   |  Build & Test    |             |  Build & Test    |
   +------------------+             +------------------+
           |
           v
   +--------------------------+
   |  Deployment Strategy     |
   +--------------------------+
        |              |
        |              |
        v              v
+----------------+   +----------------------+
| Staging Deploy |   | Production Deploy    |
| (staging branch)|  | (Git Tag / Release) |
+----------------+   +----------------------+
        |                      |
        +----------+-----------+
                   |
                   v
        +----------------------+
        |   Flask Application  |
        |   (EC2 - Port 5000)  |
        +----------+-----------+
                   |
                   v
        +----------------------+
        |   MongoDB Atlas      |
        |   (Cloud Database)   |
        +----------------------+
```

---

## Environment Variables

```
MONGO_URI=<your_mongodb_connection_string>
SECRET_KEY=<your_secret_key>
```

---

# PART 1: Jenkins CI/CD Pipeline

## Step 1: Launch EC2

* Ubuntu 22.04
* Open ports:

  * 22 (SSH)
  * 8080 (Jenkins)
  * 5000 (Flask)

---

## Step 2: Connect to EC2

```
chmod 400 key.pem
ssh -i key.pem ubuntu@<EC2-IP>
```

---

## Step 3: Install Docker

```
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
newgrp docker
```

---

## Step 4: Create Jenkins Dockerfile

```
nano Dockerfile
```

```
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    python3 python3-pip python3-venv git && \
    apt-get clean

USER jenkins
```

---

## Step 5: Build Jenkins Image

```
docker build -t my-jenkins .
```

---

## Step 6: Run Jenkins Container

```
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 5000:5000 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  my-jenkins
```

---

## Step 7: Access Jenkins

```
http://<EC2-IP>:8080
```

Get password:

```
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

---

## Step 8: Install Plugins

Install suggested plugins

---

## Step 9: Add Credentials

Manage Jenkins → Credentials

```
ID: mongo-uri
Type: Secret Text

ID: secret-key
Type: Secret Text
```

---

## Step 10: Create Pipeline Job

* New Item → Pipeline
* Pipeline from SCM

```
Repository: https://github.com/hariprn/flask_practice.git
Branch: */main
Script Path: Jenkinsfile
```

---

## Step 11: Jenkinsfile

```
pipeline {
    agent any

    environment {
        MONGO_URI = credentials('mongo-uri')
        SECRET_KEY = credentials('secret-key')
        VENV = "venv"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/hariprn/flask_practice.git'
            }
        }

        stage('Setup') {
            steps {
                sh '''
                python3 -m venv $VENV
                . $VENV/bin/activate
                pip install --upgrade pip
                '''
            }
        }

        stage('Install') {
            steps {
                sh '''
                . $VENV/bin/activate
                pip install -r requirements.txt
                pip install pytest python-dotenv
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                . $VENV/bin/activate
                pytest
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                . $VENV/bin/activate
                pkill -f app.py || true
                nohup python app.py > app.log 2>&1 &
                '''
            }
        }
    }
}
```

---

## Step 12: Run Pipeline

Click Build Now

---

## Access Application

```
http://<EC2-IP>:5000
```

---

# PART 2: GitHub Actions CI/CD Pipeline

## Step 1: Create Workflow

```
mkdir -p .github/workflows
touch .github/workflows/ci-cd.yml
```

---

## Workflow File

```
name: Flask CI/CD Pipeline

on:
  push:
    branches:
      - main
      - staging
  pull_request:
    branches:
      - main
  release:
    types: [created]

jobs:

  build-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-python@v4
        with:
          python-version: "3.10"

      - run: |
          pip install -r requirements.txt
          pip install pytest python-dotenv

      - run: pytest
        env:
          MONGO_URI: ${{ secrets.MONGO_URI }}
          SECRET_KEY: ${{ secrets.SECRET_KEY }}

  deploy-staging:
    needs: build-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/staging'
    steps:
      - run: echo "Deploying to staging..."

  deploy-production:
    needs: build-test
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - run: echo "Deploying to production..."
```

---

## GitHub Secrets

Repository → Settings → Secrets → Actions

```
MONGO_URI
SECRET_KEY
```

---

## Trigger Pipeline

### Build

```
git push origin main
```

### Staging

```
git checkout staging
git push origin staging
```

### Production

```
git tag v1.0
git push origin v1.0
```

---

## Key Learnings

* Implemented CI/CD using Jenkins and GitHub Actions
* Managed secrets securely
* Used Docker-based Jenkins setup
* Implemented branch-based and tag-based deployment
* Debugged real-world issues in pipelines and infrastructure

---
