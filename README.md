# Task Jenkins



## 1. Agent Configuration

1.1. Static (Windows vs Linux)

<details>
  <summary>Click for more information</summary>

First let's setup a master Jenkins. In this homework I'm gonna use AWS EC2 servers.

To setup a master Jenkins server I'm gonna execute this bash script:

```bash
#!/bin/bash

# Step 1: Update the System
sudo apt-get update -y && sudo apt-get upgrade -y

# Step 3: Install Java
sudo apt install openjdk-11-jdk -y

# Verify Java installation
java --version

# Step 4: Install Jenkins

# Add the Jenkins repository and key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update the system and install Jenkins
sudo apt update -y
sudo apt install jenkins -y

# Start and enable the Jenkins service
sudo systemctl start jenkins && sudo systemctl enable jenkins

# Check the status of the Jenkins service
sudo systemctl status jenkins

# Output the initial admin password
echo "Initial Admin Password:"
sudo cat /var/lib/jenkins/secrets/initialAdminPassword


```
Here is our Jenkins master setting up:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013183339.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013183612.png>)

In the parallel let's set up our Jenkins agent on a separate Linux Ubuntu machine. 
To do this I'm gonna execute this script:

```bash
#!/bin/bash

# Update the system
sudo apt-get update -y && sudo apt-get upgrade -y

# Install Java (OpenJDK 11)
sudo apt install -y openjdk-11-jdk

# Create a directory for the Jenkins agent
sudo mkdir -p /var/lib/jenkins-agent

# Change ownership to the current user (assuming it's the user that will run the Jenkins agent)
sudo chown -R $(whoami):$(whoami) /var/lib/jenkins-agent

# Output instructions to connect the agent to the master
echo "-----------------------------------------------------------"
echo "To connect this agent to the Jenkins master, follow these steps:"
echo "1. In the Jenkins dashboard, navigate to 'Manage Jenkins' -> 'Manage Nodes and Clouds' -> 'New Node'."
echo "2. Provide a name for the new node, select 'Permanent Agent', and configure the node to use the '/var/lib/jenkins-agent' directory."
echo "3. Once the node is created, Jenkins will provide a launch command. Run that command on this machine to connect the agent."
echo "-----------------------------------------------------------"


```
Now let's create a new node on the Jenkins master and run the provided commands on the Jenkins agent to establish connection:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013203551.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013203618.png>)

The connection is established:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013203710.png>)

Now let's create a simple job to test our new agent:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013204908.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013205038.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013205613.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013205811.png>)

Now let's set up our Jenkins agent on a separate Windows machine. 
To do this I'm gonna execute this script:

```bash
# PowerShell Script to Prepare Windows Machine as a Jenkins Agent

# Parameters
param (
    [string]$jenkinsMasterUrl = "http://18.185.100.32:8080",  # Replace with your Jenkins master URL
    [string]$nodeName = "windows-agent",  # Replace with your desired node name
    [string]$agentWorkDir = "C:\Jenkins\windowsagent"  # Replace with your desired agent working directory
)

# Run this script with administrative privileges
if (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Host "Please run this script as an Administrator!" -ForegroundColor Red
    Exit
}

# Check if Chocolatey is installed
$chocoInstalled = $true
try {
    Get-Command choco -ErrorAction Stop | Out-Null
} catch {
    $chocoInstalled = $false
}

# Install Chocolatey if not installed
if (-not $chocoInstalled) {
    Set-ExecutionPolicy Bypass -Scope Process -Force;
    [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072;
    iex ((Invoke-WebRequest -Uri https://chocolatey.org/install.ps1).Content)
    $env:Path += ";C:\ProgramData\chocolatey\bin"  # Add Chocolatey to PATH
}

# Install Java using Chocolatey
choco install openjdk -y

# Set JAVA_HOME environment variable
$javaHome = "C:\Program Files\OpenJDK\jdk-*"
[System.Environment]::SetEnvironmentVariable("JAVA_HOME", $javaHome, [System.EnvironmentVariableTarget]::Machine)

# Download Jenkins Agent JAR
$agentJarUrl = "${jenkinsMasterUrl}/jnlpJars/agent.jar"
$agentJarPath = Join-Path $agentWorkDir "agent.jar"
if (-not (Test-Path $agentWorkDir)) {
    New-Item -ItemType Directory -Path $agentWorkDir | Out-Null
}
Invoke-WebRequest -Uri $agentJarUrl -OutFile $agentJarPath

# Create a script to launch the Jenkins agent
$launchScriptContent = @"
java -jar $agentJarPath -jnlpUrl ${jenkinsMasterUrl}/computer/${nodeName}/slave-agent.jnlp
"@
$launchScriptPath = Join-Path $agentWorkDir "LaunchAgent.ps1"
$launchScriptContent | Out-File -FilePath $launchScriptPath

# Create a scheduled task to run the Jenkins agent on startup
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-File $launchScriptPath"
$trigger = New-ScheduledTaskTrigger -AtStartup
$settings = New-ScheduledTaskSettingsSet
$principal = New-ScheduledTaskPrincipal -GroupId "BUILTIN\Administrators" -RunLevel Highest
$task = New-ScheduledTask -Action $action -Principal $principal -Trigger $trigger -Settings $settings
Register-ScheduledTask -TaskName "Launch Jenkins Agent" -InputObject $task

# Output instructions to connect the agent to Jenkins
Write-Output "1. Go to $jenkinsMasterUrl/computer/ and click on '$nodeName'."
Write-Output "2. Follow the instructions to authenticate and connect the agent to Jenkins."

```

Let's create and connect the Windows agent to the master:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013223713.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013224108.png>)

Let's create a pipeline that will involve our Linux and Windows agents:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013224320.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013224418.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231013224456.png>)




</details>

---

1.2. Dynamic (e.g., https://www.jenkins.io/doc/book/pipeline/docker/)

<details>
  <summary>Click for more information</summary>

  First let's install Docker on our master server. For that I'm gonna use this script:

```bash
#!/bin/bash

# Update package information
sudo apt-get update -y

# Install necessary packages
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package information for docker
sudo apt-get update -y

# Install Docker Engine
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Add current user to docker group
sudo usermod -aG docker $USER
```

Than create new Docker cloud, connect it to the Jenkins master and configure Docker agent template:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014164134.png>) 

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014165542.png>) 

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014165604.png>) 

I'm gonna use this pipeline script to test our Docker agent:

```bash
pipeline {
    agent {
        docker {
            image 'jenkins/inbound-agent'
            args '-v /usr/bin/docker:/usr/bin/docker'
            args '-u root'
            label 'docker-agent'
        }
    }
    stages {
        stage('Prepare Environment') {
            steps {
                sh 'apt-get update && apt-get install -y git'
            }
        }
        stage('Clone Repository') {
            steps {
                sh 'git clone https://github.com/jenkinsci/docker.git'
            }
        }
        stage('List Directory') {
            steps {
                sh 'ls -al'
            }
        }
    }
}
```

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014181630.png>) 

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014181706.png>)



</details>

---

## 2. Place sensitive data in credentials
(e.g., GitHub/GitLab connection details, etc.)

<details>
  <summary>Click for more information</summary>

In my case I'm gonna store Dockerhub credentials:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014203607.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014203632.png>)

</details>

---

## 3. Set up access rights
- Create three groups: dev, qa, devops, and grant different access rights.

<details>
  <summary>Click for more information</summary>

First let's install Role-based Authorization Strategy plugin. Once the plugin is installed and activated in Manage Jenkins > Security, navigate to "Manage Jenkins" > "Manage and Assign Roles" > "Manage Roles".
Under the “Roles” section, we'll see a sub-section named "Global roles". Here, click on the "Add" button to create new roles. Create three roles: dev, qa, and devops.

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014211208.png>)


</details>

---

## 4. Create a multi-branch pipeline that:
1. Triggers on changes in any branch of the git repository. A separate pipeline is created for each branch.
2. The pipeline consists of the following steps:

    a. Clone the repository from the feature branch.

    b. Optionally: Check the commit message for adherence to best practices (message length, JIRA ticket code first, etc.)

    c. Lint Dockerfiles.

3. If the pipeline fails, block the possibility of merging the feature branch into the main branch.

<details>
  <summary>Click for more information</summary>

You can check out the GitHub repository for this task here: https://github.com/maximdove/lint-dockerfile.git.
Below is a step-by-step breakdown of the task accomplishment.

To create a multi-branch pipeline and test it according to the task, I'm going to set up a repository consisting of two branches, namely `main` and `feature-1`. I'll be incorporating a straightforward `Dockerfile` for linting purposes and a `Jenkinsfile` to outline the pipeline.

### Here's how the repository structure will look:

```plaintext
repository-name/
|-- .git/
|-- Dockerfile
|-- Jenkinsfile
```

### Setting up the Main Branch:

1. Initially, I'll create a new repository on the Git server.
2. Next, I'll navigate to the directory of the repository.
3. Now, I'll create a `Dockerfile` with the following content:

```dockerfile
# I'll start by using an official Python runtime as a base image
FROM python:3.8-slim-buster

# I'll set the working directory in the container to /app
WORKDIR /app

# Now, I'll copy the contents of the current directory into the container at /app
COPY . /app

# I'll install any necessary packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
```

4. Following that, I'll create a `Jenkinsfile` with the pipeline script:

```jenkinsfile
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Check Commit Message') {
            steps {
                script {
                    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    // Assuming a simple check for the presence of a JIRA ticket code at the start of the commit message
                    if (!commitMessage.matches('^[A-Z]+-[0-9]+.*')) {
                        error("Commit message does not adhere to best practices")
                    }
                }
            }
        }

        stage('Lint Dockerfiles') {
            steps {
                script {
                    def lintResult = sh(script: 'docker run --rm -v $(pwd):/app -w /app hadolint/hadolint hadolint Dockerfile', returnStdout: true).trim()
                    if (lintResult) {
                        echo lintResult
                        error("Linting errors found")
                    }
                }
            }
        }
    }

    post {
        failure {
            echo 'Build failed!'
        }
    }
}
```

5. I'll commit these files to the `main` branch.

```bash
git add Dockerfile Jenkinsfile
git commit -m "JIRA-1234: Update Commit Message so it adheres to the defined best practices."
git push origin main
```

### Setting up the Feature Branch:

1. Now, I'll create a new branch named `feature-1`:

```bash
git checkout -b feature-1
```

2. I'll modify the `Dockerfile` slightly to introduce a linting error, like a long comment and missing tags:

```dockerfile
# This is a very long comment that should trigger a linting error due to its length exceeding the recommended maximum length for a single line in a Dockerfile...............$
FROM python
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -r requirements.txt
```

3. I'll commit this change to the `feature-1` branch.

```bash
git commit -am "JIRA-1235: Update Commit Message so it adheres to the defined best practices."
git push origin feature-1
```

Now, I have a repository with two distinct branches: `main` (which has a lint-passing `Dockerfile` and the `Jenkinsfile`), and `feature-1` (which has a lint-failing `Dockerfile`). This setup is ideal for testing the Jenkins multi-branch pipeline, as it will display a passing build for the `main` branch and a failing build for the `feature-1` branch.

Here are the screenshots from Jenkins:

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014225710.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014225741.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231014225810.png>)


</details>

---

## 5. Create a CI (Continuous Integration) pipeline that:
1. Watches the main branch of the repository (main/master/whatever).
2. Triggers on a merge into the main branch.
3. Clones the repository.
4. Launches static code analysis. Bugs, Vulnerabilities, Security Hotspots, and Code Smells are available on the SonarQube server.
5. Builds the Docker image.
6. Tags the image twice (as "latest" and with the build version).
7. Pushes the image to Docker Hub.

<details>
  <summary>Click for more information</summary>

You can check out the GitHub repository for this task here: https://github.com/maximdove/SonarQube-Pipeline.git.
Below is a step-by-step breakdown of the task accomplishment.

### 1. Crafting a Simple Python Application

To start, I'll be crafting a straightforward Python application. I'll create a file dubbed `app.py` with the following script:

```python
def greet(name):
    return f"Hello, {name}!"

if __name__ == "__main__":
    print(greet("World"))
```

### 2. Crafting a Dockerfile

Now, it’s time to create a `Dockerfile` to build a Docker image of our application. I’ll generate a file called `Dockerfile` with the script below:

```Dockerfile
FROM python:3.8-slim

COPY app.py /app.py

CMD ["python", "/app.py"]
```

### 3. Setting Up SonarQube Analysis

For the sake of simplicity, I’ll use the default `sonar-project.properties` file. I'll create a file named `sonar-project.properties` and populate it with:

```properties
sonar.projectKey=my-simple-app
sonar.sources=.
sonar.host.url=http://3.68.70.79:9000
sonar.language=py
```



### 4. Crafting the Jenkinsfile

Now, onto creating a `Jenkinsfile` to outline the CI pipeline. I’ll create a file named `Jenkinsfile` with the script below:

```groovy
pipeline {
    agent any

    stages {
        stage('Check for Merge Commit') {
            steps {
                script {
                    def isMergeCommit = sh(script: 'git rev-parse HEAD^2', returnStatus: true) == 0
                    if (!isMergeCommit) {
                        currentBuild.result = 'ABORTED'
                        error('This is not a merge commit. Aborting the build.')
                    }
                }
            }
        }

        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Static Code Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQube-pipeline';
                    withSonarQubeEnv('SonarQube-pipeline') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def app = docker.build("maximdove/my-app:${env.BUILD_ID}")
                }
            }
        }

        stage('Tag Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        def app = docker.image("maximdove/my-app:${env.BUILD_ID}")
                        app.push('latest')
                        app.push("${env.BUILD_ID}")
                    }
                }
            }
        }
    }
}
```

### 5. Syncing My Code to a Git Repository

Now, I'll initialize a git repository, commit the files (`app.py`, `Dockerfile`, `sonar-project.properties`, `Jenkinsfile`) to the feature-branch and sync them to a remote repository (GitHub).


### 6. Tweaking Jenkins Settings

- Now, I'll head to my Jenkins dashboard and create a new Pipeline job.
- In the `Pipeline` segment, I’ll select `Pipeline script from SCM`, opt for `Git`, and input my repository URL.
- In the `Build Triggers’ section, I’ll tick `GitHub hook trigger for GITScm polling`.

### 7. Establishing a Webhook

I'll establish a webhook within my GitHub repository to prompt Jenkins upon a merge to the main branch.

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017182226.png>)

### 8. Setting up a Separate SonarQube Server

To install SonarQube on a separate server, install all the necessary dependencies, create a dedicated user and setup a sonarqube.service I'll use this script:

```bash
#!/bin/bash

# Update the system and install necessary packages
sudo yum update -y
sudo yum install wget unzip -y

# Download and Install Java 17 (as SonarQube requires a more recent version of Java)
wget -P /tmp https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm
sudo rpm -ivh /tmp/jdk-17_linux-x64_bin.rpm
rm /tmp/jdk-17_linux-x64_bin.rpm

# Verify Java installation
java -version

# Download SonarQube (updated to version 9.9.2.77730 as per your request)
wget -P /tmp https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.2.77730.zip

# Create directory for SonarQube and extract the downloaded file
sudo mkdir /opt/sonarqube
sudo unzip /tmp/sonarqube-9.9.2.77730.zip -d /opt/sonarqube
rm /tmp/sonarqube-9.9.2.77730.zip

# Set necessary system limits
# These configurations are set permanently by modifying system files
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=65536" | sudo tee -a /etc/sysctl.conf
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf
sudo sysctl -p

# Create a new user named 'sonar'
sudo adduser sonar

# Set the ownership of the SonarQube directory to the 'sonar' user
sudo chown -R sonar:sonar /opt/sonarqube

# Create a systemd service file for SonarQube
echo "[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=simple
User=sonar
Group=sonar
ExecStart=/opt/sonarqube/sonarqube-9.9.2.77730/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/sonarqube-9.9.2.77730/bin/linux-x86-64/sonar.sh stop
Restart=always

[Install]
WantedBy=multi-user.target
" | sudo tee /etc/systemd/system/sonarqube.service

# Reload systemd to recognize the new service
sudo systemctl daemon-reload

# Enable SonarQube service to start on boot
sudo systemctl enable sonarqube.service

# Start SonarQube service
sudo systemctl start sonarqube.service
```

Then I have to create a new project (SonarQube-pipeline), install a Python plugin, create access token to connect Jenkins server with SonarQube server.

### 9. Tweaking SonarQube Settings on the Jenkins Server and adding Dockerhub Credentials

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017181435.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017181519.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017181538.png>)

### 10. Triggering our Piplene with the Branch Merge

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017182014.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017181758.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017181836.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231017181933.png>)

</details>

---


## 6. Create a CD (Continuous Deployment) pipeline that:
1. Parameters during launch:

    a. Environment name (dev/qa).

    b. Version number (i.e., image tag, version, or latest). A bonus if this list is dynamically retrieved from Dockerhub.

2. Deploys the image to the selected environment.
3. A stage with a health check of the deployed environment (e.g., curl an endpoint or something similar).

**Bonus:**
+ Notifications via email or Teams.

<details>
  <summary>Click for more information</summary>

You can check out the GitHub repository for this task here: https://github.com/maximdove/Parameters-Notification-CD-Pipeline.git.
Below is a step-by-step breakdown of the task accomplishment.

### 1. **Crafting a Simple Web Application:**
Our main file named `app.py` would look something like this:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/health', methods=['GET'])
def health_check():
    return "OK", 200

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

### 2. **Drafting a Dockerfile:**

Now, to containerize this application, we'll craft a `Dockerfile` as follows:

```dockerfile
# Use the official Python image for the AMD64 architecture
FROM python:3.8-slim-buster

# Set the working directory
WORKDIR /app

# Copy the application files to the container
COPY app/app.py .
COPY requirements.txt .

# Install the application dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Expose the application port
EXPOSE 80

# Run the application
CMD ["python", "app.py"]
```

Our `requirements.txt` will have:

```plaintext
Flask==2.0.1
Werkzeug==2.0.1
```

### 3. **Constructing a Jenkins Pipeline:**
   
The pipeline will use this Jenkinsfile store on GitHub with the other files mentioned above:

```groovy
pipeline {
    agent any
    parameters {
        choice(
            name: 'Environment',
            choices: ['dev', 'qa'],
            description: 'Select Environment'
        )
        string(
            name: 'Version',
            defaultValue: '1.0',
            description: 'Enter the version number'
        )
    }
    environment {
        DOCKER = credentials('docker-hub-credentials')
    }
    stages {
        stage('Build and Push') {
            steps {
                script {
                    // This block automatically logs in to Docker Hub, handles the push, and then logs out
                    withDockerRegistry([ credentialsId: 'docker-hub-credentials', url: '' ]) {
                        def image = docker.build("maximdove/my-web-app:${params.Version}")
                        image.push()
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // Remove any existing container with the same name
                    sh "docker rm -f my-web-app-${params.Environment} || true"
                    // Deploy the Docker image to the selected environment
                    sh "docker run -d -e ENV=${params.Environment} --name my-web-app-${params.Environment} -p 9090:80 maximdove/my-web-app:${params.Version}"
                }
            }
        }
        stage('Health Check') {
            steps {
                script {
                    sh "sleep 10" // Give the container some time to start
                    try {
                        sh "curl http://localhost:9090/health"
                    } catch (Exception e) {
                        error "Health check failed: ${e.message}"
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                emailext(
                    subject: "Deployment Successful",
                    body: "The deployment to ${params.Environment} was successful.",
                    to: 'maximdove@gmail.com',
                    from: 'maximfeb@gmail.com'
                )
            }
        }
        failure {
            script {
                emailext(
                    subject: "Deployment Failed",
                    body: "The deployment to ${params.Environment} failed.",
                    to: 'maximdove@gmail.com',
                    from: 'maximfeb@gmail.com'
                )
            }
        }
    }
}
```

### 4. **Starting a Jenkins Pipeline:**

![Alt text](<Screenshots/iScreen Shoter - Safari - 231018140511.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231018140322.png>)

![Alt text](<Screenshots/iScreen Shoter - Safari - 231018140647.png>)


</details>

---