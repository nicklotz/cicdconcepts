# Lab 05: Continuous Deployment

> This lab covers deploying applications through Jenkins pipelines, including staging environments, approval gates, and rollback strategies.

## A. Deployment Philosophy

### Continuous Delivery vs Continuous Deployment

| Aspect | Continuous Delivery | Continuous Deployment |
|--------|--------------------|-----------------------|
| Definition | Code is always deployable | Code is automatically deployed |
| Manual Step | Approval before production | None |
| Risk | Lower (human review) | Higher (but faster) |
| Frequency | On-demand | Every commit |

### Deployment Principles

1. **Automate everything** - No manual steps
2. **Deploy frequently** - Small changes, less risk
3. **Test in production-like environments** - Staging mirrors production
4. **Easy rollback** - Always have a path back
5. **Monitor everything** - Know immediately if something breaks

## B. Setting Up Deployment Targets

> We'll create local directories to simulate staging and production environments.

1. Create deployment target directories.
```
sudo mkdir -p /opt/myapp/staging
sudo mkdir -p /opt/myapp/production
sudo chown -R $USER:$USER /opt/myapp
```

2. Create a deployment info file.
```
cat << 'EOF' > /opt/myapp/staging/info.txt
Environment: Staging
Purpose: Pre-production testing
EOF

cat << 'EOF' > /opt/myapp/production/info.txt
Environment: Production
Purpose: Live application
EOF
```

3. Give Jenkins and your user access to deployment directories.
```
sudo chown -R jenkins:jenkins /opt/myapp
sudo chmod -R 775 /opt/myapp
sudo usermod -aG jenkins $USER
```

> **Note:** You may need to log out and back in for the group membership to take effect.

## C. Creating a Deployable Application

1. Navigate to the project.
```
cd ~/myciproject
```

2. Create an application entry point.
```
cat << 'EOF' > app.py
#!/usr/bin/env python3
"""
Simple web-like application for deployment demonstration.
"""

import os
import datetime
from calculator import add, subtract, multiply, divide

VERSION = "1.0.0"

def get_app_info():
    """Return application information."""
    return {
        "name": "MyCI Calculator App",
        "version": VERSION,
        "environment": os.getenv("APP_ENV", "development"),
        "deployed_at": datetime.datetime.now().isoformat(),
        "host": os.uname().nodename
    }

def health_check():
    """Perform basic health check."""
    try:
        # Test calculator functions
        assert add(2, 2) == 4
        assert subtract(5, 3) == 2
        assert multiply(3, 4) == 12
        assert divide(10, 2) == 5
        return {"status": "healthy", "checks_passed": 4}
    except AssertionError:
        return {"status": "unhealthy", "error": "Calculator test failed"}

def main():
    """Main application entry point."""
    info = get_app_info()
    print("=" * 50)
    print(f"Application: {info['name']}")
    print(f"Version: {info['version']}")
    print(f"Environment: {info['environment']}")
    print(f"Host: {info['host']}")
    print(f"Deployed: {info['deployed_at']}")
    print("=" * 50)

    health = health_check()
    print(f"\nHealth Check: {health['status'].upper()}")

    if health['status'] == 'healthy':
        print(f"All {health['checks_passed']} checks passed!")
        return 0
    else:
        print(f"Error: {health.get('error', 'Unknown')}")
        return 1

if __name__ == "__main__":
    exit(main())
EOF
```

3. Create a version file that will be updated during deployment.
```
cat << 'EOF' > version.txt
1.0.0
EOF
```

4. Create a deployment manifest.
```
cat << 'EOF' > manifest.json
{
    "app_name": "myci-calculator",
    "version": "1.0.0",
    "files": [
        "app.py",
        "calculator.py",
        "version.txt"
    ],
    "health_check": "python3 app.py",
    "rollback_enabled": true
}
EOF
```

## D. Creating a Deployment Script

1. Create a reusable deployment script.
```
cat << 'EOF' > deploy.sh
#!/bin/bash
set -euo pipefail

# Deployment script for MyCI Calculator App

ENVIRONMENT="${1:-staging}"
APP_DIR="/opt/myapp/${ENVIRONMENT}"
BACKUP_DIR="/opt/myapp/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=========================================="
echo "Deploying to: ${ENVIRONMENT}"
echo "Target: ${APP_DIR}"
echo "Time: ${TIMESTAMP}"
echo "=========================================="

# Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(staging|production)$ ]]; then
    echo "ERROR: Invalid environment. Use 'staging' or 'production'"
    exit 1
fi

# Create backup directory if it doesn't exist
mkdir -p "${BACKUP_DIR}"

# Backup current deployment
if [ -f "${APP_DIR}/app.py" ]; then
    echo "Creating backup..."
    BACKUP_FILE="${BACKUP_DIR}/${ENVIRONMENT}_${TIMESTAMP}.tar.gz"
    tar -czf "${BACKUP_FILE}" -C "${APP_DIR}" . 2>/dev/null || true
    echo "Backup created: ${BACKUP_FILE}"

    # Keep only last 5 backups
    ls -t "${BACKUP_DIR}/${ENVIRONMENT}_"*.tar.gz 2>/dev/null | tail -n +6 | xargs -r rm
fi

# Deploy new version
echo "Copying files..."
cp -v app.py calculator.py version.txt manifest.json "${APP_DIR}/"

# Create deployment record
cat << DEPLOY_EOF > "${APP_DIR}/deployment.txt
Deployed: ${TIMESTAMP}
Environment: ${ENVIRONMENT}
Version: $(cat version.txt)
By: Jenkins CI
DEPLOY_EOF

# Run health check
echo "Running health check..."
cd "${APP_DIR}"
export APP_ENV="${ENVIRONMENT}"
if python3 app.py; then
    echo "=========================================="
    echo "Deployment to ${ENVIRONMENT} SUCCESSFUL!"
    echo "=========================================="
    exit 0
else
    echo "=========================================="
    echo "Deployment FAILED! Health check did not pass."
    echo "=========================================="
    exit 1
fi
EOF
chmod +x deploy.sh
```

2. Create a rollback script.
```
cat << 'EOF' > rollback.sh
#!/bin/bash
set -euo pipefail

# Rollback script for MyCI Calculator App

ENVIRONMENT="${1:-staging}"
APP_DIR="/opt/myapp/${ENVIRONMENT}"
BACKUP_DIR="/opt/myapp/backups"

echo "=========================================="
echo "Rolling back: ${ENVIRONMENT}"
echo "=========================================="

# Find the latest backup
LATEST_BACKUP=$(ls -t "${BACKUP_DIR}/${ENVIRONMENT}_"*.tar.gz 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    echo "ERROR: No backup found for ${ENVIRONMENT}"
    exit 1
fi

echo "Restoring from: ${LATEST_BACKUP}"

# Clear current deployment
rm -rf "${APP_DIR:?}"/*

# Restore backup
tar -xzf "${LATEST_BACKUP}" -C "${APP_DIR}"

# Run health check
echo "Running health check after rollback..."
cd "${APP_DIR}"
export APP_ENV="${ENVIRONMENT}"
if python3 app.py; then
    echo "=========================================="
    echo "Rollback SUCCESSFUL!"
    echo "=========================================="
    exit 0
else
    echo "=========================================="
    echo "Rollback completed but health check failed!"
    echo "=========================================="
    exit 1
fi
EOF
chmod +x rollback.sh
```

## E. Multi-Stage Deployment Pipeline

1. Update the Jenkinsfile with deployment stages.
```
cat << 'EOF' > Jenkinsfile
pipeline {
    agent any

    environment {
        PYTHONPATH = "${WORKSPACE}"
    }

    parameters {
        choice(name: 'DEPLOY_TO', choices: ['none', 'staging', 'production'], description: 'Deploy to environment')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests before deployment')
        booleanParam(name: 'SKIP_APPROVAL', defaultValue: false, description: 'Skip production approval (USE WITH CAUTION)')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.BUILD_VERSION = readFile('version.txt').trim()
                }
                echo "Building version: ${env.BUILD_VERSION} (${env.GIT_COMMIT_SHORT})"
            }
        }

        stage('Build') {
            steps {
                echo 'Compiling Python files...'
                sh 'python3 -m py_compile calculator.py app.py'
                echo 'Build successful!'
            }
        }

        stage('Test') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                echo 'Running tests...'
                sh '''
                    pip3 install pytest pytest-cov --break-system-packages || true
                    python3 -m pytest tests/ -v --junitxml=test-results.xml || exit 1
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
            }
        }

        stage('Quality Check') {
            steps {
                echo 'Running quality checks...'
                sh '''
                    pip3 install pylint flake8 --break-system-packages || true
                    pylint calculator.py app.py --exit-zero --output-format=parseable > pylint.txt || true
                    flake8 calculator.py app.py --exit-zero > flake8.txt || true
                '''
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { params.DEPLOY_TO == 'staging' || params.DEPLOY_TO == 'production' }
            }
            steps {
                echo 'Deploying to Staging...'
                sh './deploy.sh staging'
            }
            post {
                success {
                    echo 'Staging deployment successful!'
                }
                failure {
                    echo 'Staging deployment failed!'
                }
            }
        }

        stage('Staging Verification') {
            when {
                expression { params.DEPLOY_TO == 'staging' || params.DEPLOY_TO == 'production' }
            }
            steps {
                echo 'Verifying staging deployment...'
                sh '''
                    cd /opt/myapp/staging
                    export APP_ENV=staging
                    python3 app.py
                '''
                echo 'Staging verification passed!'
            }
        }

        stage('Production Approval') {
            when {
                expression { params.DEPLOY_TO == 'production' && params.SKIP_APPROVAL == false }
            }
            steps {
                script {
                    def approver = input(
                        message: 'Deploy to Production?',
                        ok: 'Deploy',
                        submitterParameter: 'approver',
                        parameters: [
                            string(
                                name: 'APPROVAL_NOTES',
                                defaultValue: '',
                                description: 'Notes for this deployment'
                            )
                        ]
                    )
                    echo "Approved by: ${approver}"
                }
            }
        }

        stage('Deploy to Production') {
            when {
                expression { params.DEPLOY_TO == 'production' }
            }
            steps {
                echo 'Deploying to Production...'
                sh './deploy.sh production'
            }
            post {
                success {
                    echo '=========================================='
                    echo 'PRODUCTION DEPLOYMENT SUCCESSFUL!'
                    echo '=========================================='
                }
                failure {
                    echo 'Production deployment failed! Initiating rollback...'
                    sh './rollback.sh production || true'
                }
            }
        }

        stage('Production Verification') {
            when {
                expression { params.DEPLOY_TO == 'production' }
            }
            steps {
                echo 'Verifying production deployment...'
                sh '''
                    cd /opt/myapp/production
                    export APP_ENV=production
                    python3 app.py
                '''
                echo 'Production verification passed!'
            }
        }
    }

    post {
        always {
            echo "Pipeline completed for version ${env.BUILD_VERSION}"
            archiveArtifacts artifacts: '*.txt', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
EOF
```

2. Commit all deployment files.
```
git add .
git commit -m "Add deployment pipeline with staging and production"
git push origin main || git push origin master
```

## F. Testing the Deployment Pipeline

1. In Jenkins, open the `calculator-pipeline` job.

2. Click **Build with Parameters**.

3. Test staging deployment:
   - Set `DEPLOY_TO` to `staging`
   - Keep `RUN_TESTS` checked
   - Click **Build**

4. Verify the staging deployment.
```
cat /opt/myapp/staging/deployment.txt
cat /opt/myapp/staging/version.txt
```

5. Test production deployment:
   - Set `DEPLOY_TO` to `production`
   - The pipeline will pause for approval
   - Approve the deployment
   - Watch it complete

6. Verify the production deployment.
```
cat /opt/myapp/production/deployment.txt
```

## G. Implementing Version Bumping

1. Create a version bump script.
```
cat << 'EOF' > bump-version.sh
#!/bin/bash
set -euo pipefail

# Version bump script
BUMP_TYPE="${1:-patch}"  # major, minor, patch

CURRENT_VERSION=$(cat version.txt)
IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

case "$BUMP_TYPE" in
    major)
        MAJOR=$((MAJOR + 1))
        MINOR=0
        PATCH=0
        ;;
    minor)
        MINOR=$((MINOR + 1))
        PATCH=0
        ;;
    patch)
        PATCH=$((PATCH + 1))
        ;;
    *)
        echo "Usage: $0 [major|minor|patch]"
        exit 1
        ;;
esac

NEW_VERSION="${MAJOR}.${MINOR}.${PATCH}"
echo "$NEW_VERSION" > version.txt

# Update app.py version
sed -i "s/VERSION = \".*\"/VERSION = \"$NEW_VERSION\"/" app.py

echo "Version bumped: $CURRENT_VERSION -> $NEW_VERSION"
EOF
chmod +x bump-version.sh
```

2. Test version bumping.
```
./bump-version.sh patch
cat version.txt
```

## H. Rollback Demonstration

1. Create a "bad" version to demonstrate rollback.
```
# First, deploy a working version
echo "1.1.0" > version.txt
sed -i 's/VERSION = ".*"/VERSION = "1.1.0"/' app.py
git add .
git commit -m "Version 1.1.0"
git push origin main || git push origin master
```

2. Build and deploy to staging.

3. Now simulate a broken deployment.
```
cat << 'EOF' > app.py
#!/usr/bin/env python3
"""BROKEN VERSION - for rollback demonstration."""

def main():
    raise Exception("This version is broken!")
    return 1

if __name__ == "__main__":
    exit(main())
EOF

echo "1.2.0-broken" > version.txt
git add .
git commit -m "Broken version 1.2.0 (for rollback demo)"
git push origin main || git push origin master
```

4. Attempt to deploy - it should fail health check.

5. Manually trigger rollback.
```
./rollback.sh staging
```

6. Verify the rollback worked.
```
cd /opt/myapp/staging
python3 app.py
```

7. Restore the working version in the repository.
```
cd ~/myciproject
git revert HEAD --no-edit
git push origin main || git push origin master
```

## I. Blue-Green Deployment Concept

1. Create blue-green deployment directories.
```
sudo mkdir -p /opt/myapp/blue /opt/myapp/green
sudo chown -R jenkins:jenkins /opt/myapp/blue /opt/myapp/green
```

2. Create a blue-green deployment script.
```
cat << 'EOF' > deploy-blue-green.sh
#!/bin/bash
set -euo pipefail

# Blue-Green Deployment Script
CURRENT_LINK="/opt/myapp/current"
BLUE_DIR="/opt/myapp/blue"
GREEN_DIR="/opt/myapp/green"

# Determine which environment is currently active
if [ -L "$CURRENT_LINK" ]; then
    CURRENT=$(readlink "$CURRENT_LINK")
    if [ "$CURRENT" = "$BLUE_DIR" ]; then
        DEPLOY_TO="$GREEN_DIR"
        ACTIVE="blue"
        DEPLOYING="green"
    else
        DEPLOY_TO="$BLUE_DIR"
        ACTIVE="green"
        DEPLOYING="blue"
    fi
else
    # First deployment
    DEPLOY_TO="$BLUE_DIR"
    ACTIVE="none"
    DEPLOYING="blue"
fi

echo "Currently active: $ACTIVE"
echo "Deploying to: $DEPLOYING"

# Deploy to inactive environment
mkdir -p "$DEPLOY_TO"
cp -v app.py calculator.py version.txt manifest.json "$DEPLOY_TO/"

# Health check new deployment
echo "Health check on $DEPLOYING..."
cd "$DEPLOY_TO"
export APP_ENV="$DEPLOYING"
if python3 app.py; then
    echo "Health check passed!"

    # Switch traffic
    echo "Switching traffic to $DEPLOYING..."
    rm -f "$CURRENT_LINK"
    ln -s "$DEPLOY_TO" "$CURRENT_LINK"

    echo "=========================================="
    echo "Blue-Green deployment complete!"
    echo "Active environment: $DEPLOYING"
    echo "=========================================="
else
    echo "Health check failed! Traffic NOT switched."
    exit 1
fi
EOF
chmod +x deploy-blue-green.sh
```

## J. Clean Up

1. Verify all deployments.
```
echo "=== Staging ===" && cat /opt/myapp/staging/deployment.txt
echo "=== Production ===" && cat /opt/myapp/production/deployment.txt 2>/dev/null || echo "Not deployed"
```

2. List backups.
```
ls -la /opt/myapp/backups/
```

## K. Key Takeaways

- **Deployment pipelines** automate the release process
- **Staging environments** catch issues before production
- **Approval gates** provide human oversight for critical deployments
- **Health checks** verify deployments are working
- **Rollback scripts** provide quick recovery from failures
- **Blue-Green deployments** enable zero-downtime releases
- **Version tracking** maintains deployment history

## L. What's Next

In the next lab, you'll learn to:
- Set up CI/CD metrics and monitoring
- Create dashboards for pipeline visibility
- Implement notifications and alerts
- Apply CI/CD best practices
