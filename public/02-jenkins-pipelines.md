# Lab 02: Jenkins Pipelines

> This lab explores Jenkins Pipelines in depth, including Jenkinsfiles, build triggers, and parameters.

## A. Pipeline Fundamentals Review

### Declarative vs Scripted Pipelines

| Aspect | Declarative | Scripted |
|--------|-------------|----------|
| Syntax | Structured, opinionated | Full Groovy |
| Learning Curve | Easier | Steeper |
| Flexibility | Limited but sufficient | Unlimited |
| Validation | Built-in | Manual |
| Recommendation | Start here | Advanced use |

### Basic Declarative Structure

```groovy
pipeline {
    agent any

    environment {
        // Environment variables
    }

    stages {
        stage('Stage Name') {
            steps {
                // Commands
            }
        }
    }

    post {
        // Post-build actions
    }
}
```

## B. Creating a Project with a Jenkinsfile

> Best practice is to store your pipeline definition in your source code repository.

1. Create a project directory.
```
mkdir -p ~/myciproject
cd ~/myciproject
```

2. Initialize a Git repository.
```
git init
```

3. Create a simple application - a Python calculator.
```
cat << 'EOF' > calculator.py
#!/usr/bin/env python3
"""A simple calculator module for CI demonstration."""

def add(a, b):
    """Add two numbers."""
    return a + b

def subtract(a, b):
    """Subtract b from a."""
    return a - b

def multiply(a, b):
    """Multiply two numbers."""
    return a * b

def divide(a, b):
    """Divide a by b."""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

if __name__ == "__main__":
    print(f"2 + 3 = {add(2, 3)}")
    print(f"5 - 2 = {subtract(5, 2)}")
    print(f"4 * 3 = {multiply(4, 3)}")
    print(f"10 / 2 = {divide(10, 2)}")
EOF
```

4. Create a Jenkinsfile in the project root.
```
cat << 'EOF' > Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'python3 --version'
                sh 'python3 -m py_compile calculator.py'
                echo 'Build complete!'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'python3 calculator.py'
                echo 'Tests passed!'
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo 'SUCCESS: All stages completed successfully!'
        }
        failure {
            echo 'FAILURE: Pipeline failed. Check the logs.'
        }
    }
}
EOF
```

5. Add and commit the files.
```
git add .
git commit -m "Initial commit with calculator and Jenkinsfile"
```

## C. Setting Up a Local Git Server (Optional)

> For a fully local setup, we can use Jenkins' built-in Git support with local paths.

1. Create a bare Git repository to act as a "remote".
```
mkdir -p ~/git-repos
git clone --bare ~/myciproject ~/git-repos/myciproject.git
```

2. Configure the original project to use this as a remote.
```
cd ~/myciproject
git remote add origin ~/git-repos/myciproject.git
git push -u origin master || git push -u origin main
```

3. Give Jenkins permission to access the "remote" project.
```
chmod 755 ~
chmod -R 755 ~/git-repos/myciproject.git

# Configure git safe directory for Jenkins user (required for Git 2.35.2+)
# This allows Jenkins to access repositories owned by other users
sudo -u jenkins git config --global --add safe.directory '*'

echo "============================================"
echo "Repository path for Jenkins:"
echo "file:///home/YOUR_USER/git-repos/myciproject.git"
echo "============================================"
echo ""
echo "Directory permissions:"
ls -la ~/git-repos/myciproject.git
```

4. Allow Jenkins to check out local directories (normally considered insecure).
```
sudo mkdir -p /etc/systemd/system/jenkins.service.d

sudo tee /etc/systemd/system/jenkins.service.d/override.conf >/dev/null <<'EOF'
[Service]
Environment="JAVA_OPTS=-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true"
EOF

sudo systemctl daemon-reload
sudo systemctl restart jenkins
```   

## D. Creating a Pipeline Job from SCM

1. In Jenkins, click **New Item**.

2. Name it `calculator-pipeline` and select **Pipeline**.

3. In the configuration:
   - Scroll to **Pipeline** section
   - Change **Definition** to "Pipeline script from SCM"
   - Select **Git** as SCM
   - Enter Repository URL: `/home/YOUR_USERNAME/git-repos/myciproject.git`
     (Replace YOUR_USERNAME with your actual username)
   - Branch Specifier: `*/main` or `*/master`
   - Script Path: `Jenkinsfile`

4. Click **Save**.

5. Click **Build Now** and watch the build execute.

> **Note**: If using the local path gives permission errors, you may need to:
```
sudo usermod -aG $USER jenkins
sudo chmod -R 755 ~/git-repos
sudo systemctl restart jenkins
```

## E. Pipeline with Parameters

> Parameters allow you to customize builds at runtime.

1. Create a new pipeline job named `parameterized-pipeline`.

2. Check **This project is parameterized**.

3. Add the following parameters:
   - **String Parameter**: Name: `GREETING`, Default: `Hello`
   - **Choice Parameter**: Name: `ENVIRONMENT`, Choices: `dev`, `staging`, `prod`
   - **Boolean Parameter**: Name: `RUN_TESTS`, Default: checked

4. Add this pipeline script:

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'GREETING', defaultValue: 'Hello', description: 'Greeting message')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests?')
    }

    stages {
        stage('Greet') {
            steps {
                echo "${params.GREETING}, World!"
                echo "Deploying to: ${params.ENVIRONMENT}"
            }
        }

        stage('Build') {
            steps {
                echo "Building for ${params.ENVIRONMENT}..."
                sh 'sleep 2'
                echo 'Build complete!'
            }
        }

        stage('Test') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                echo 'Running tests...'
                sh 'sleep 2'
                echo 'Tests passed!'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying to ${params.ENVIRONMENT}..."
                script {
                    if (params.ENVIRONMENT == 'prod') {
                        echo 'WARNING: Production deployment!'
                    }
                }
            }
        }
    }
}
```

5. Click **Save**, then **Build with Parameters**.

6. Try different parameter combinations and observe the behavior.

## F. Build Triggers

> Triggers automatically start builds based on events.

### Poll SCM

1. Open the `calculator-pipeline` job configuration.

2. Under **Build Triggers**, check **Poll SCM**.

3. Enter schedule: `H/5 * * * *` (every 5 minutes)

> **Cron Syntax**: `MINUTE HOUR DOM MONTH DOW`
> - `H` means "hash" - Jenkins picks a time to spread load

4. Save and make a change to the repository:
```
cd ~/myciproject
echo "# Updated" >> calculator.py
git add .
git commit -m "Minor update"
git push origin main || git push origin master
```

5. Wait up to 5 minutes and observe the build trigger.

### Webhook Triggers (Concept)

In real environments, webhooks provide instant triggers:

```
Repository Push → Webhook → Jenkins → Build Starts
```

> For local development, Poll SCM works fine. Production setups should use webhooks.

## G. Pipeline Stages and Parallel Execution

1. Create a new pipeline job named `parallel-pipeline`.

2. Add this pipeline script:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building...'
                sh 'sleep 2'
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo 'Running unit tests...'
                        sh 'sleep 3'
                        echo 'Unit tests passed!'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        echo 'Running integration tests...'
                        sh 'sleep 4'
                        echo 'Integration tests passed!'
                    }
                }
                stage('Lint') {
                    steps {
                        echo 'Running linter...'
                        sh 'sleep 2'
                        echo 'Lint passed!'
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying...'
                sh 'sleep 2'
                echo 'Deployed!'
            }
        }
    }
}
```

3. Build and observe the **Stage View** - parallel stages run simultaneously.

## H. Environment Variables

1. Create a new pipeline job named `environment-pipeline`.

2. Add this pipeline script:

```groovy
pipeline {
    agent any

    environment {
        APP_NAME = 'MyApp'
        APP_VERSION = '1.0.0'
        DEPLOY_ENV = 'development'
    }

    stages {
        stage('Show Environment') {
            steps {
                echo "Application: ${env.APP_NAME}"
                echo "Version: ${env.APP_VERSION}"
                echo "Environment: ${env.DEPLOY_ENV}"
                echo "---"
                echo "Jenkins Built-in Variables:"
                echo "BUILD_NUMBER: ${env.BUILD_NUMBER}"
                echo "BUILD_ID: ${env.BUILD_ID}"
                echo "JOB_NAME: ${env.JOB_NAME}"
                echo "WORKSPACE: ${env.WORKSPACE}"
                echo "JENKINS_HOME: ${env.JENKINS_HOME}"
            }
        }

        stage('Use in Shell') {
            steps {
                sh '''
                    echo "From shell script:"
                    echo "App: $APP_NAME v$APP_VERSION"
                    echo "Build: $BUILD_NUMBER"
                '''
            }
        }

        stage('Dynamic Variables') {
            steps {
                script {
                    env.BUILD_TIME = sh(script: 'date', returnStdout: true).trim()
                    env.GIT_SHORT = sh(script: 'echo "abc123"', returnStdout: true).trim()
                }
                echo "Build time: ${env.BUILD_TIME}"
                echo "Git short hash: ${env.GIT_SHORT}"
            }
        }
    }
}
```

3. Build and review the console output.

## I. Conditional Execution

1. Create a new pipeline job named `conditional-pipeline`.

2. Add this pipeline script:

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Environment')
    }

    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }

        stage('Deploy to Dev') {
            when {
                expression { params.ENVIRONMENT == 'dev' }
            }
            steps {
                echo 'Deploying to DEV environment'
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { params.ENVIRONMENT == 'staging' }
            }
            steps {
                echo 'Deploying to STAGING environment'
            }
        }

        stage('Deploy to Prod') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                echo 'Deploying to PRODUCTION environment'
                echo 'Running extra safety checks...'
            }
        }

        stage('Always Runs') {
            steps {
                echo "Deployment to ${params.ENVIRONMENT} complete!"
            }
        }
    }
}
```

3. Build with different environment choices and observe which stages run.

## J. Input and Approval Gates

1. Create a new pipeline job named `approval-pipeline`.

2. Add this pipeline script:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'sleep 2'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'sleep 2'
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to staging...'
                sh 'sleep 2'
            }
        }

        stage('Approval') {
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
            }
        }

        stage('Deploy to Production') {
            steps {
                echo 'Deploying to production...'
                sh 'sleep 2'
                echo 'Production deployment complete!'
            }
        }
    }
}
```

3. Build the pipeline and watch it pause at the Approval stage.

4. Click on the paused stage and approve or abort the deployment.

## K. Error Handling

1. Create a new pipeline job named `error-handling-pipeline`.

2. Add this pipeline script:

```groovy
pipeline {
    agent any

    stages {
        stage('Success Stage') {
            steps {
                echo 'This will succeed'
            }
        }

        stage('Potential Failure') {
            steps {
                script {
                    try {
                        echo 'Attempting risky operation...'
                        sh 'exit 1'  // This will fail
                    } catch (Exception e) {
                        echo "Caught error: ${e.message}"
                        echo 'Continuing anyway...'
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                echo 'Cleanup stage'
            }
        }
    }

    post {
        always {
            echo 'This always runs'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        unstable {
            echo 'Pipeline is unstable'
        }
    }
}
```

3. Build and observe error handling in action.

## L. Workspace and Artifacts

1. Create a new pipeline job named `artifacts-pipeline`.

2. Add this pipeline script:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh '''
                    echo "Build output" > build-output.txt
                    echo "Build Number: $BUILD_NUMBER" >> build-output.txt
                    echo "Build Date: $(date)" >> build-output.txt
                    mkdir -p dist
                    echo "Application binary placeholder" > dist/app.bin
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "Test Results" > test-results.txt
                    echo "Tests: 10" >> test-results.txt
                    echo "Passed: 10" >> test-results.txt
                    echo "Failed: 0" >> test-results.txt
                '''
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'build-output.txt, test-results.txt, dist/*', fingerprint: true
            }
        }
    }
}
```

3. Build the pipeline.

4. After the build, click on the build number and find **Archived Artifacts**.

5. Download and examine the archived files.

## M. Clean Up Test Jobs

```
# From Jenkins dashboard, delete test jobs you no longer need
# Keep calculator-pipeline for future labs
```

## N. Key Takeaways

- **Jenkinsfiles** store pipeline configuration as code
- **Parameters** make pipelines flexible and reusable
- **Triggers** automate build initiation
- **Parallel stages** speed up pipelines
- **Conditionals** control which stages run
- **Input steps** provide manual approval gates
- **Artifacts** preserve build outputs

## O. What's Next

In the next lab, you'll learn to:
- Integrate automated testing frameworks
- Generate test reports
- Set up code coverage tracking
