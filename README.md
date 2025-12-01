# Python Application Deployment Using Jenkins Pipeline

This document provides a structured approach for deploying a Python application to a remote server using **Jenkins**.
It follows an automated CI/CD workflow suitable for beginners as well as intermediate DevOps practitioners.

---

## Project Overview

The deployment architecture consists of:

* **Jenkins Server**: Executes pipeline stages and coordinates the deployment workflow.
  
* **Application Server**: Hosts and runs the Python application.
  
* **Deployment Method**: Jenkins declarative pipeline using Git, SSH, and Python virtual environments.
  
**Pipeline Stages**:
1. Clone source code from the Git repository

2. Transfer project files to the application server

3. Install dependencies and start the application

**Plugins Used**:

* [Git Plugin](https://plugins.jenkins.io/git/)
* [SSH Agent Plugin](https://plugins.jenkins.io/ssh-agent/)
* [Pipeline Plugin](https://plugins.jenkins.io/workflow-aggregator/) (Workflow Aggregator)

---

##  Prerequisites

### 1. Infrastructure

* **Two servers** (can be AWS EC2, Azure VM, etc.):

  * **Jenkins Server** (to run the pipeline)
  * **App Server** (to host the Python application)
* Both servers should be launched using the same SSH key (PEM) to simplify access management.

### 2. Software Requirements

| Component         | Jenkins Server    | App Server   |
| ----------------- | --------------    | ------------ |
| Java 17           | ✔ Required       | ✔ Required   |
| Python3 & venv    | ✘ Not needed     | ✔ Required   |
| pip               | ✘ Not needed     | ✔ Required   |
| Git               | ✔ Required       | ✔ Required   |
| Jenkins           | ✔ Required       | ✘ Not needed |

---

## Network & Firewall Configuration

Configure your security group or firewall with the following inbound rules:

| Port   | Purpose                         | Where to Open                                      |
| ------ | ------------------------------- | -------------------------------------------------- |
| 22     | SSH access                      | Jenkins → App Server, Local Machine → BothServer |
| 8080   | Jenkins web interface           | Local Machine → Jenkins Server                           |
| 5000\* | Python app (or your app's port) | Public or specific IPs ranges                            |

> **Tip:** Update the port number depending on your application’s configuration.

---

## Adding SSH Credentials in Jenkins

Since both machines use the same PEM key, Jenkins can directly SSH into the application server.
This means Jenkins can SSH into the app server without generating a new key.

### Step 1 — Upload PEM Key to Jenkins Server

From your **local machine**:

```bash
scp -i pem-key-server.pem pem-key-server.pem ubuntu@<JENKINS_SERVER_PUBLIC_IP>:/home/ubuntu/
```

### Step 2 — Configure Jenkins Credentials

1. Navigate to:
Manage Jenkins → Credentials → Global → Add Credentials

2. Fill in the following:

   * **Kind**: SSH Username with private key
   * **Username**: `ubuntu` (or your respective SSH user)
   * **Private Key**: Enter directly → Paste PEM contents `pem-key-server.pem`
   * **Host**: Enter the public IP of your app server (e.g., `98.81.210.251`)
   * **ID**: `python-app-credentials`
3. Save the configuration.
   

---

## Jenkins Declarative Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any

    environment {
        SSH_CRED = 'ssh-key-pair-credentials'
        SERVER_IP = 'python-app-server-ip'
        REMOTE_USER = 'ubuntu'
        APP_DIR = '/home/ubuntu/pythonapp'
    }

    stages {
        stage('Clone Repo') {
            steps {
                git url: 'github-pythonapp-url', branch: 'main'
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(credentials: ["${SSH_CRED}"]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} "mkdir -p ${APP_DIR}"
                        scp -r Dockerfile README.md app.py requirements.txt test ${REMOTE_USER}@${SERVER_IP}:${APP_DIR}/
                    '''
                }
            }
        }

        stage('Install & Run App') {
            steps {
                sshagent(["${SSH_CRED}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} '
                            sudo apt update &&
                            sudo apt install -y python3-venv python3-pip &&
                            cd ${APP_DIR} &&
                            python3 -m venv venv &&
                            source venv/bin/activate &&
                            pip install --upgrade pip &&
                            pip install -r requirements.txt &&
                            nohup python3 app.py --host=0.0.0.0 > app.log 2>&1 &
                            exit 0
                        '
                    """
                }
            }
        }
    }
}
```

---

1. **Job Configuration** 
  
2. **Build Console Output** 
  
3. **Browser** showing your running Python app.


---

## Deployment Flow

1. Jenkins fetches the latest code from the Git repository.

2. Required project files are securely copied to the application server.

3. A virtual environment is created, dependencies are installed, and the application is started using nohup.

4. The application becomes accessible at:
```
http://<public-ip of app-server>:5000
```


##  Enabling Automated Builds with GitHub Webhooks

To trigger the Jenkins build automatically on code push, ensure the **GitHub Plugin** is installed on Jenkins.

**Step 1 — Enable GitHub Hook Trigger in Jenkins**

* Open your Jenkins job → Configure
* Under **Build Triggers**, check:

  * `GitHub hook trigger for GITScm polling`



**Step 2 — Add Webhook in GitHub**

* Go to your repository → **Settings** → **Webhooks** → **Add webhook**
* Payload URL:

```
http://<JENKINS_SERVER_IP>:8080/github-webhook/
```


This ensures Jenkins automatically triggers a pipeline run whenever new code is pushed.

