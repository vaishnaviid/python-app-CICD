# Python Application Deployment Using Jenkins Pipeline

This document provides a structured approach for deploying a Python application to a remote server using **Jenkins**.
It follows an automated CI/CD workflow suitable for beginners as well as intermediate DevOps practitioners.

---

## Project Overview

We use:

* **Jenkins Server**: Executes pipeline stages and coordinates the deployment workflow.
* **Application Server**: Hosts and runs the Python application.
* **Deployment Method**: Jenkins declarative pipeline using Git, SSH, and Python virtual environments.
* **Pipeline Stages**:

  1. Clone source code from GitHub.
  2. Upload files to remote app server.
  3. Install dependencies & start the app.

**Plugins Used**:

* [Git Plugin](https://plugins.jenkins.io/git/)
* [SSH Agent Plugin](https://plugins.jenkins.io/ssh-agent/)
* [Pipeline Plugin](https://plugins.jenkins.io/workflow-aggregator/)

---

## 🛠 Prerequisites

### 1. Infrastructure

* **Two servers** (can be AWS EC2, Azure VM, etc.):

  * **Jenkins Server** (to run the pipeline)
  * **App Server** (to host the Python application)
* Both servers launched using the **same `.pem` key**.

### 2. Software Requirements

| Component         | Jenkins Server | App Server   |
| ----------------- | -------------- | ------------ |
| Java (OpenJDK 17) | ✅ Required     | ✅ Required   |
| Python3 & venv    | ❌ Not needed   | ✅ Required   |
| pip               | ❌ Not needed   | ✅ Required   |
| Git               | ✅ Required     | ✅ Required   |
| Jenkins           | ✅ Required     | ❌ Not needed |

---

## Security Group / Firewall Rules

Open these ports in your **cloud security group** (or firewall):

| Port   | Purpose                         | Where to Open                                      |
| ------ | ------------------------------- | -------------------------------------------------- |
| 22     | SSH access                      | Jenkins → App Server, Your PC → Jenkins/App Server |
| 8080   | Jenkins web interface           | Your PC → Jenkins Server                           |
| 5000\* | Python app (or your app's port) | Public or specific IPs                             |

> **Tip:** Replace `5000` with the actual port your app listens on.

---

## Setting Up Jenkins Credentials

We are reusing the **same `.pem` key** that was used to launch both the Jenkins server and the App server.
This means Jenkins can SSH into the app server without generating a new key.

### Step 1 — Copy Your PEM Key to Jenkins Server

From your **local machine**:

```bash
scp -i pem-key-server.pem pem-key-server.pem ubuntu@<JENKINS_SERVER_PUBLIC_IP>:/home/ubuntu/
```

### Step 2 — Add PEM Key to Jenkins Credentials

1. Under **Stores scoped to Jenkins**, click **(global)** → **Add Credentials**.
   ![](/python-app-img/credentials-1.png)
2. Fill in:

   * **Kind**: SSH Username with private key
   * **Username**: `ubuntu` (or the SSH username for your app server)
   * **Private Key**: Choose **"Enter directly"** and paste the contents of `pem-key-server.pem`
   * **Host**: Enter the public IP of your app server (e.g., `98.81.210.251`)
   * **ID**: `python-app-credentials`
3. Save.
   ![](/python-app-img/credentials-2.png)

---

## Jenkins Pipeline (Jenkinsfile)

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
   ![](/python-app-img/job-config.png)
2. **Build Console Output** 
   ![](/python-app-img/build-success.png)
3. **Browser** showing your running Python app.
   ![](/python-app-img/final-output-1.png)

   ![](/python-app-img/final-output-2.png)

---

## How It Works

1. **Clone Repository** – Pulls Python app code from GitHub.
2. **Upload Files** – Uses SSH to copy files to app server.
3. **Install & Run** – Creates a virtual environment, installs dependencies, and runs the app in the background.
4. **Access App** – Open `http://<APP_SERVER_IP>:<PORT>` in your browser.

---

## Example Access URL

```
http://<public-ip of app-server>:5000
```


## ⚡ Automating Builds with GitHub Webhooks

To trigger the Jenkins build automatically on code push, ensure the **GitHub Plugin** is installed on Jenkins.

**Step 1 — Enable GitHub Hook Trigger in Jenkins**

* Open your Jenkins job → Configure
* Under **Build Triggers**, check:

  * `GitHub hook trigger for GITScm polling`

![](/python-app-img/webhook-jenkins-1.png)


**Step 2 — Add Webhook in GitHub**

* Go to your repository → **Settings** → **Webhooks** → **Add webhook**
* Payload URL:

```
http://<JENKINS_SERVER_IP>:8080/github-webhook/
```

![](/python-app-img/webhook-GIThub.png)

Now, whenever you push code to the repo, Jenkins will automatically pull changes and deploy them.

