# Lab 01: Introduction to CI and Jenkins Setup

> This lab introduces Continuous Integration concepts and guides you through installing Jenkins on Ubuntu 24.04 LTS.

## A. Understanding Continuous Integration

### What is CI?

**Continuous Integration (CI)** is a software development practice where:
- Developers frequently integrate code into a shared repository
- Each integration is verified by automated builds and tests
- Problems are detected and fixed early

### The CI Workflow

```
Developer Commits → CI Server Detects → Build Triggered → Tests Run → Results Reported
                                              ↓
                                    [Pass] Deploy/Merge
                                    [Fail] Fix Required
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| Early Bug Detection | Find issues when they're easiest to fix |
| Reduced Integration Risk | Small, frequent merges vs. big-bang integration |
| Always Deployable | Main branch is always in a working state |
| Fast Feedback | Know within minutes if something broke |
| Documentation | Build scripts document how to build the project |

## B. Setting Up the Lab Environment

1. Update your system packages.
```
sudo apt-get update && sudo apt-get upgrade -y
```

2. Install essential tools we'll need throughout the labs.
```
sudo apt-get install -y git curl wget gnupg2 software-properties-common
```

## C. Installing Java (Jenkins Requirement)

> Jenkins requires Java 17 or 21 to run. We'll install OpenJDK 17.

1. Install OpenJDK 17.
```
sudo apt-get install -y openjdk-17-jdk
```

2. Verify the Java installation.
```
java -version
```

3. Set JAVA_HOME environment variable.
```
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' | sudo tee -a /etc/profile.d/java.sh
echo 'export PATH=$PATH:$JAVA_HOME/bin' | sudo tee -a /etc/profile.d/java.sh
source /etc/profile.d/java.sh
```

4. Verify JAVA_HOME is set.
```
echo $JAVA_HOME
```

## D. Installing Jenkins

1. Add the Jenkins repository key.
```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
```

2. Add the Jenkins repository to your sources list.
```
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

3. Update package lists to include Jenkins.
```
sudo apt-get update
```

4. Install Jenkins.
```
sudo apt-get install -y jenkins
```

5. Start the Jenkins service.
```
sudo systemctl start jenkins
```

6. Enable Jenkins to start on boot.
```
sudo systemctl enable jenkins
```

7. Check Jenkins service status.
```
sudo systemctl status jenkins
```

> You should see "active (running)" in the output.

## E. Initial Jenkins Configuration

1. Get the initial admin password.
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

> Copy this password - you'll need it in the next step.

2. Open Jenkins in your web browser.
```
echo "Open http://$(hostname -I | awk '{print $1}'):8080 in your browser"
```

> If you're on the local machine, you can use http://localhost:8080

3. In the browser:
   - Paste the initial admin password
   - Click "Install suggested plugins" (this may take a few minutes)
   - Create your admin user when prompted
   - Accept the default Jenkins URL
   - Click "Start using Jenkins"

## F. Exploring the Jenkins Interface

> Take a moment to explore the Jenkins dashboard.

### Key Areas

| Area | Purpose |
|------|---------|
| Dashboard | Overview of all jobs and status |
| New Item | Create new jobs/pipelines |
| People | User management |
| Build History | Recent build activity |
| Manage Jenkins | System configuration |

1. Click on **Manage Jenkins** → **System Information** to view your Jenkins installation details.

2. Click on **Manage Jenkins** → **Plugins** to see available plugins.

## G. Creating Your First Freestyle Job

> A Freestyle job is the simplest type of Jenkins job, configured through the GUI.

1. From the dashboard, click **New Item**.

2. Enter the name: `hello-world-freestyle`

3. Select **Freestyle project** and click **OK**.

4. In the configuration page:
   - Scroll down to **Build Steps**
   - Click **Add build step** → **Execute shell**
   - Enter the following script:

```bash
echo "Hello from Jenkins!"
echo "Build Number: $BUILD_NUMBER"
echo "Job Name: $JOB_NAME"
echo "Workspace: $WORKSPACE"
date
```

5. Click **Save**.

6. Click **Build Now** in the left sidebar.

7. Once the build completes, click on the build number (e.g., #1) in **Build History**.

8. Click **Console Output** to see the results.

> You should see your echo statements and the current date printed.

## H. Creating Your First Pipeline Job

> Pipelines are defined as code, making them version-controllable and more powerful.

1. From the dashboard, click **New Item**.

2. Enter the name: `hello-world-pipeline`

3. Select **Pipeline** and click **OK**.

4. Scroll down to the **Pipeline** section.

5. In the **Script** text area, enter:

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello from Jenkins Pipeline!'
            }
        }

        stage('Environment') {
            steps {
                echo "Build Number: ${env.BUILD_NUMBER}"
                echo "Job Name: ${env.JOB_NAME}"
                sh 'whoami'
                sh 'pwd'
            }
        }

        stage('Date') {
            steps {
                sh 'date'
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed!'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

6. Click **Save**.

7. Click **Build Now**.

8. Watch the **Stage View** that appears - it shows each stage's progress.

9. Click on the build number and then **Console Output** to see detailed logs.

## I. Understanding Pipeline Syntax

### Pipeline Structure

```groovy
pipeline {
    agent any              // Where to run (any available agent)

    stages {               // Container for all stages
        stage('Name') {    // A single stage
            steps {        // Actions to perform
                // commands here
            }
        }
    }

    post {                 // Actions after pipeline completes
        always { }         // Always runs
        success { }        // Only on success
        failure { }        // Only on failure
    }
}
```

### Common Steps

| Step | Purpose | Example |
|------|---------|---------|
| `echo` | Print message | `echo 'Hello'` |
| `sh` | Run shell command | `sh 'ls -la'` |
| `git` | Clone repository | `git 'https://...'` |
| `dir` | Change directory | `dir('subdir') { }` |
| `timeout` | Set time limit | `timeout(5) { }` |

## J. Setting Up Git for Jenkins

> Jenkins needs Git to check out source code from repositories.

1. Install Git (if not already installed).
```
sudo apt-get install -y git
```

2. Verify Git is available to Jenkins by creating a test pipeline.

3. Create a new Pipeline job named `git-test-pipeline`.

4. Add this pipeline script:

```groovy
pipeline {
    agent any

    stages {
        stage('Check Git') {
            steps {
                sh 'git --version'
            }
        }

        stage('Clone Sample Repo') {
            steps {
                git branch: 'master', url: 'https://github.com/octocat/Hello-World.git'
                sh 'ls -la'
            }
        }
    }
}
```

5. Build the job and verify it can clone the repository.

## K. Jenkins File System

> Understanding where Jenkins stores data helps with troubleshooting.

1. Explore the Jenkins home directory.
```
ls -la /var/lib/jenkins/
```

| Directory | Contents |
|-----------|----------|
| `jobs/` | Job configurations and build history |
| `workspace/` | Working directories for builds |
| `plugins/` | Installed plugins |
| `secrets/` | Encrypted credentials |
| `logs/` | Jenkins logs |

2. View Jenkins logs.
```
sudo journalctl -u jenkins -f
```

> Press Ctrl+C to exit the log viewer.

3. Check disk usage by Jenkins.
```
sudo du -sh /var/lib/jenkins/
```

## L. Clean Up

> Keep your Jenkins instance clean by removing test jobs.

1. From the Jenkins dashboard, click on `hello-world-freestyle`.

2. Click **Delete Project** in the left sidebar and confirm.

3. Repeat for `hello-world-pipeline` and `git-test-pipeline` if desired, or keep them for reference.

## M. Key Takeaways

- **Jenkins** is an automation server that executes CI/CD pipelines
- **Freestyle jobs** are GUI-configured and simple but limited
- **Pipeline jobs** are code-defined, version-controllable, and powerful
- **Stages** organize pipeline work into logical units
- **Steps** are the individual actions within a stage
- Jenkins stores all data in `/var/lib/jenkins/`

## N. What's Next

In the next lab, you'll learn to:
- Create more complex pipelines
- Use Jenkinsfiles stored in Git repositories
- Work with build parameters
- Set up build triggers
