# 🚀 Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

## 📑 Table of Contents

* [Project Overview](#project-overview)
* [Architecture Diagram](#architecture-diagram)
* [Step 1: AWS EC2 Instance Preparation](#step-1-aws-ec2-instance-preparation)
* [Step 2: Install Dependencies on EC2](#step-2-install-dependencies-on-ec2)
* [Step 3: Jenkins Installation and Setup](#step-3-jenkins-installation-and-setup)
* [Step 4: GitHub Repository Configuration](#step-4-github-repository-configuration)

  * [Dockerfile](#dockerfile)
  * [docker-compose.yml](#docker-composeyml)
  * [Jenkinsfile](#jenkinsfile)
* [Step 5: Jenkins Pipeline Creation and Execution](#step-5-jenkins-pipeline-creation-and-execution)
* [Errors that I faced](#errors-that-i-faced)
* [Conclusion](#conclusion)
* [Infrastructure Diagram](#infrastructure-diagram)
* [Work flow Diagram](#work-flow-diagram)

---

## Project Overview

This project explains how I deployed a simple two-layer web app (Flask and MySQL) on an AWS EC2 server. 
The app runs inside Docker containers managed by Docker Compose, and Jenkins automatically rebuilds and updates the application whenever new code is pushed to GitHub.

The goal was not only to deploy an app, but to automate the entire lifecycle:

``Code Push → Build → Containerize → Deploy → Verify → Repeat``


**Key Concepts Demonstrated**

* AWS EC2 infrastructure provisioning
* Docker containerization
* Docker Compose multi‑container orchestration
* Jenkins CI/CD automation
* Auto deployment on every Git push
* Troubleshooting real production‑like issues

🎥 Demo Video

Watch the demo: 👉 https://drive.google.com/file/d/1AtWLi6w_6rZVL14ltFitBRpxuE8pHsoQ/view?usp=sharing
---

## Architecture Diagram

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same AWS EC2)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```
### Application Components

**Application Layer**
- Flask Web App (Python)

**Database Layer**
- MySQL Container


Both services run inside Docker containers on the same EC2 host.

---

## Step 1: AWS EC2 Instance Preparation

**Instance Configuration**

| Setting  | Value              |
| -------- | ------------------ |
| OS       | Ubuntu 22.04 LTS   |
| Instance | t3.small           |
| Storage  | 16GB               |
| Ports    | 22, 80, 5000, 8080 |

**Security Group Rules**

| Port | Purpose   |
| ---- | --------- |
| 22   | SSH       |
| 5000 | Flask App |
| 8080 | Jenkins   |
| 80   | HTTP      |

Launch EC2 Instance:
- Navigate to the AWS EC2 console.
- Launch a new instance using the Ubuntu 22.04 LTS AMI.
- Select the t3.small instance type.
- Create and assign a new key pair for SSH access.

<img width="1917" height="891" alt="Image" src="https://github.com/user-attachments/assets/6d8e5746-9083-4028-88d1-3445cc6f8906" />


Configure Security Group:
Create a security group with the following inbound rules:
- Type: SSH, Protocol: TCP, Port: 22, Source: Your IP
- Type: HTTP, Protocol: TCP, Port: 80, Source: Anywhere (0.0.0.0/0)
- Type: Custom TCP, Protocol: TCP, Port: 5000 (for Flask), Source: Anywhere (0.0.0.0/0)
- Type: Custom TCP, Protocol: TCP, Port: 8080 (for Jenkins), Source: Anywhere (0.0.0.0/0)
<img width="1918" height="878" alt="Image" src="https://github.com/user-attachments/assets/5361dfe5-b526-4fe1-8113-5c3575e8a29a" />

Connect to EC2 Instance:
- Use SSH to connect to the instance's public IP address.

```ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>```

---

## Step 2: Install Dependencies on EC2

1. Update System Packages:
   
```sudo apt update && sudo apt upgrade -y```

2. Install Git, Docker, and Docker Compose:

```sudo apt install git docker.io docker-compose-v2 -y```

3. Start and Enable Docker:

```
sudo systemctl start docker
sudo systemctl enable docker
```

4. Add User to Docker Group (to run docker without sudo):
```
sudo usermod -aG docker $USER
newgrp docker
Restart Docker as well
````


---

## Step 3: Jenkins Installation and Setup

1 Install Java (OpenJDK 17):

```
sudo apt install openjdk-17-jdk -y
```

2 Add Jenkins Repository and Install:
```
curl -fsSL [https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key](https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key) | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] [https://pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable) binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
```

3 Start and Enable Jenkins Service:

```
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

4 Initial Jenkins Setup:

- Retrieve the initial admin password:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Access the Jenkins dashboard at http://<ec2-public-ip>:8080.
- Paste the password, install suggested plugins, and create an admin user.
  
5 Grant Jenkins Docker Permissions:
```
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
<img width="1850" height="910" alt="Image" src="https://github.com/user-attachments/assets/0562bb36-7a77-48fb-a0ef-428ff97f5f13" />

---

## Step 4: GitHub Repository Configuration

Ensure your GitHub repository contains the following three files.

1) Dockerfile
This file defines the environment for the Flask application container.
Creates the Flask runtime environment:
- Python base image
- MySQL client dependencies
- Installs requirements
- Runs app on port 5000
```
# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Install system dependencies required for mysqlclient
RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*

# Copy the requirements file to leverage Docker cache
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on
EXPOSE 5000

# Command to run the application
CMD ["python", "app.py"]
```

2) docker-compose.yml

This file defines and orchestrates the multi-container application (Flask and MySQL).

Handles:
- Networking
- Health checks
- Restart policies
- Dependency order

```
version: "3.8"

services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_DATABASE: "devops"
      MYSQL_ROOT_PASSWORD: "root"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - two-tier
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask:
    build:
      context: .
    container_name: two-tier-app
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=devops
    networks:
      - two-tier
    depends_on:
      - mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

volumes:
  mysql-data:

networks:
  two-tier:
```

3) Jenkinsfile

This file contains the pipeline-as-code definition for Jenkins.

CI/CD Pipeline (Jenkinsfile)

Pipeline stages:
- Clone repository
- Build Docker image
- Stop old containers
- Deploy new containers
```
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                // Replace with your GitHub repository URL
                git branch: 'main', url: '[https://github.com/your-username/your-repo.git](https://github.com/your-username/your-repo.git)'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                // Stop existing containers if they are running
                sh 'docker compose down || true'
                // Start the application, rebuilding the flask image
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

---

## Step 5: Jenkins Pipeline Creation and Execution
1) Create a New Pipeline Job in Jenkins:

- From the Jenkins dashboard, select New Item.
- Name the project, choose Pipeline, and click OK.

Configure the Pipeline:
- In the project configuration, scroll to the Pipeline section.
- Set Definition to Pipeline script from SCM.
- Choose Git as the SCM.
- Enter your GitHub repository URL.
- Verify the Script Path is Jenkinsfile.
- Save the configuration.

<img width="1850" height="881" alt="Image" src="https://github.com/user-attachments/assets/5ec98a6d-fca5-4fdb-bd9a-bf3e63933736" />

Run the Pipeline:
- Click Build Now to trigger the pipeline manually for the first time.
- Monitor the execution through the Stage View or Console Output.

<img width="1841" height="853" alt="Image" src="https://github.com/user-attachments/assets/ab3526ec-8d07-44bb-8347-ad5d83411987" />
<img width="1845" height="856" alt="Image" src="https://github.com/user-attachments/assets/304108c9-fcbe-45a7-8c21-6273ead9e897" />

Verify Deployment:
- After a successful build, your Flask application will be accessible at http://<your-ec2-public-ip>:5000.
- Confirm the containers are running on the EC2 instance with docker ps.

<img width="1850" height="800" alt="Image" src="https://github.com/user-attachments/assets/522f659f-ae86-44ab-8d8d-727b8d48672b" />

---

## Errors that I faced

❌ Error 1 — Docker Permission Denied

Problem
Jenkins build failed:
```
Permission denied while trying to connect to the Docker daemon socket
/var/run/docker.sock
```

Even though Jenkins was already added to the Docker group.

Investigation

Tested Docker manually:
```
docker build -t test-permission.
```
Docker worked → The issue was the Jenkins permission application.

Root Cause

Docker group changes require a service restart.

Solution
```
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins
```
❌ Error 2 — EC2 Instance Freeze & Stuck Builds
Symptoms
- SSH froze
- Ping failed
- Jenkins build stuck:
```
Still waiting to schedule task
Waiting for next available executor
```
Observation
- AWS monitoring showed sustained high CPU usage.

<img width="1801" height="862" alt="Image" src="https://github.com/user-attachments/assets/d41b8ed5-e862-4eb9-a687-616f7bfe039f" />

Hidden Issue
- Jenkins node disk below threshold → builds blocked.

<img width="1507" height="413" alt="Image" src="https://github.com/user-attachments/assets/f7f4239d-7cca-4c7c-9bd3-6f3f62ac67d8" />

Cause

t3.micro resource exhaustion:

| Resource | Effect             |
| -------- | ------------------ |
| Low RAM  | Instance freeze    |
| Low Disk | Build blocked      |
| High CPU | Docker build stuck |


Docker images filled the storage.

Temporary Fix

- Cleared build history
- Deleted workspace
- Removed Docker images
- Restarted Jenkins

The issue returned again.

Final Solution
| Before   | After     |
| -------- | --------- |
| t3.micro | t3.small  |
| 8GB disk | 16GB disk |

After upgrade → stable builds.
---

## Conclusion

The CI/CD pipeline automatically builds and deploys the Flask application on every Git push, providing a fully automated deployment workflow.

---

## Infrastructure Diagram

![Image](https://github.com/user-attachments/assets/c6f92ff3-cf40-4bdf-902f-6c17472ab9cf)

---

## Work flow Diagram

![Image](https://github.com/user-attachments/assets/dfa544c9-85c5-495c-a77b-47cf141e41fb)
