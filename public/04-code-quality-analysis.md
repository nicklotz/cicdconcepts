# Lab 04: Code Quality and Analysis

> This lab covers static code analysis, linting, and quality gates in Jenkins pipelines.

## A. Code Inspection vs Code Testing

| Aspect | Code Testing | Code Inspection |
|--------|-------------|-----------------|
| Purpose | Verify behavior | Analyze structure |
| Execution | Runs code | Static analysis |
| Finds | Bugs, logic errors | Style issues, code smells |
| Speed | Varies | Very fast |
| Examples | pytest, JUnit | pylint, ESLint, ShellCheck |

> **Best Practice**: Use both! Testing ensures correctness; inspection ensures maintainability.

## B. Installing Code Analysis Tools

1. Install Python linting tools.
```
pip3 install pylint flake8 black bandit --break-system-packages
```

2. Install ShellCheck for bash script analysis.
```
sudo apt-get install -y shellcheck
```

3. Verify installations.
```
pylint --version
flake8 --version
shellcheck --version
```

## C. Running Pylint

> Pylint is a comprehensive Python linter that checks for errors, style issues, and code complexity.

1. Navigate to the project.
```
cd ~/myciproject
```

2. Run pylint on the calculator module.
```
pylint calculator.py
```

3. Observe the output - you'll see:
   - Convention (C) messages: Style violations
   - Refactor (R) messages: Code smell suggestions
   - Warning (W) messages: Potential issues
   - Error (E) messages: Probable bugs
   - A score out of 10

4. Run pylint with a specific format for CI.
```
pylint calculator.py --output-format=parseable
```

5. Generate a detailed report.
```
pylint calculator.py --reports=y
```

## D. Configuring Pylint

1. Create a pylint configuration file.
```
cat << 'EOF' > .pylintrc
[MASTER]
# Ignore test files for some checks
ignore=tests

[MESSAGES CONTROL]
# Disable specific warnings
disable=
    C0114,  # missing-module-docstring (we have them, but flexible)
    C0116,  # missing-function-docstring (optional for simple functions)
    W0511,  # fixme (allow TODO comments)

[FORMAT]
# Maximum line length
max-line-length=100

[DESIGN]
# Maximum number of arguments for function
max-args=6
# Maximum number of local variables
max-locals=15

[BASIC]
# Good variable names that should always be accepted
good-names=a,b,i,j,k,x,y,_
EOF
```

2. Run pylint with the configuration.
```
pylint calculator.py --rcfile=.pylintrc
```

## E. Running Flake8

> Flake8 is a simpler, faster linter that combines pyflakes, pycodestyle, and mccabe.

1. Run flake8 on the project.
```
flake8 calculator.py
```

2. Run with more verbose output.
```
flake8 calculator.py --show-source --statistics
```

3. Create a flake8 configuration.
```
cat << 'EOF' > .flake8
[flake8]
max-line-length = 100
exclude =
    .git,
    __pycache__,
    build,
    dist,
    htmlcov
ignore =
    E501, 
    W503, 
per-file-ignores =
    tests/*: E501
EOF
```

4. Run flake8 with the configuration.
```
flake8 calculator.py
```

## F. ShellCheck for Bash Scripts

1. Create a sample deployment script.
```
cat << 'EOF' > deploy.sh
#!/bin/bash
# Sample deployment script with intentional issues

APP_DIR=/opt/myapp
VERSION=$1

# Bad: Unquoted variable
cd $APP_DIR

# Bad: Using backticks instead of $()
DATE=`date`

# Bad: Not checking if directory exists
rm -rf $APP_DIR/*

# Bad: Useless use of cat
cat config.txt | grep "setting"

# Good practice
if [ -z "$VERSION" ]; then
    echo "Error: Version required"
    exit 1
fi

echo "Deploying version $VERSION at $DATE"
EOF
chmod +x deploy.sh
```

2. Run ShellCheck on the script.
```
shellcheck deploy.sh
```

3. Notice the warnings and suggestions.

4. Create a fixed version of the script.
```
cat << 'EOF' > deploy_fixed.sh
#!/bin/bash
# Sample deployment script - fixed version

set -euo pipefail  # Exit on error, undefined vars, pipe failures

APP_DIR="/opt/myapp"
VERSION="${1:-}"

# Check required argument
if [ -z "$VERSION" ]; then
    echo "Error: Version required" >&2
    exit 1
fi

# Check if directory exists
if [ ! -d "$APP_DIR" ]; then
    echo "Error: $APP_DIR does not exist" >&2
    exit 1
fi

# Safe directory change
cd "$APP_DIR" || exit 1

# Modern command substitution
DATE=$(date)

# Proper way to search file
grep "setting" config.txt || echo "Setting not found"

echo "Deploying version $VERSION at $DATE"
EOF
chmod +x deploy_fixed.sh
```

5. Run ShellCheck on the fixed script.
```
shellcheck deploy_fixed.sh
```

## G. Security Analysis with Bandit

> Bandit finds common security issues in Python code.

1. Run bandit on the calculator.
```
bandit calculator.py
```

2. Create a file with security issues for demonstration.
```
cat << 'EOF' > security_example.py
#!/usr/bin/env python3
"""Example file with security issues - for demonstration only."""

import subprocess
import pickle

def run_command(user_input):
    """BAD: Shell injection vulnerability."""
    subprocess.call(user_input, shell=True)  # B602: subprocess_popen_with_shell_equals_true

def load_data(filename):
    """BAD: Pickle deserialization vulnerability."""
    with open(filename, 'rb') as f:
        return pickle.load(f)  # B301: pickle

def get_password():
    """BAD: Hardcoded password."""
    password = "admin123"  # B105: hardcoded_password_string
    return password

# BAD: Assert used for security check
def check_admin(user):
    """BAD: Using assert for security."""
    assert user.is_admin  # B101: assert_used
    return True
EOF
```

3. Run bandit on the security example.
```
bandit security_example.py -ll
```

4. Run bandit with detailed output.
```
bandit security_example.py -f json | python3 -m json.tool
```

5. Clean up the security example.
```
rm security_example.py
```

## H. Integrating Quality Checks into Jenkins

1. Update the Jenkinsfile with quality analysis stages.
```
cat << 'EOF' > Jenkinsfile
pipeline {
    agent any

    environment {
        PYTHONPATH = "${WORKSPACE}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup') {
            steps {
                echo 'Installing dependencies...'
                sh '''
                    pip3 install pytest pytest-cov pytest-html pylint flake8 bandit --break-system-packages || true
                '''
            }
        }

        stage('Quality Analysis') {
            parallel {
                stage('Pylint') {
                    steps {
                        echo 'Running Pylint...'
                        sh '''
                            python3 -m pylint calculator.py \
                                --output-format=parseable \
                                --rcfile=.pylintrc \
                                --exit-zero \
                                > pylint-report.txt || true
                            cat pylint-report.txt
                        '''
                        // Extract score
                        script {
                            def pylintOutput = readFile('pylint-report.txt')
                            def scoreMatch = pylintOutput =~ /Your code has been rated at ([0-9.]+)/
                            if (scoreMatch) {
                                def score = scoreMatch[0][1]
                                echo "Pylint Score: ${score}/10"
                                if (score.toFloat() < 7.0) {
                                    echo "WARNING: Pylint score below 7.0"
                                }
                            }
                        }
                    }
                }

                stage('Flake8') {
                    steps {
                        echo 'Running Flake8...'
                        sh '''
                            python3 -m flake8 calculator.py \
                                --output-file=flake8-report.txt \
                                --tee \
                                --exit-zero || true
                        '''
                    }
                }

                stage('Bandit') {
                    steps {
                        echo 'Running Bandit security scan...'
                        sh '''
                            python3 -m bandit calculator.py \
                                -f txt \
                                -o bandit-report.txt \
                                --exit-zero || true
                            cat bandit-report.txt
                        '''
                    }
                }

                stage('ShellCheck') {
                    steps {
                        echo 'Running ShellCheck...'
                        sh '''
                            if ls *.sh 1> /dev/null 2>&1; then
                                shellcheck *.sh > shellcheck-report.txt 2>&1 || true
                                cat shellcheck-report.txt
                            else
                                echo "No shell scripts found" > shellcheck-report.txt
                            fi
                        '''
                    }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh '''
                    python3 -m pytest tests/ -v \
                        --junitxml=test-results.xml \
                        --cov=. \
                        --cov-report=xml \
                        --cov-report=term
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Checking quality gates...'
                script {
                    def pylintReport = readFile('pylint-report.txt')
                    def flake8Report = readFile('flake8-report.txt')

                    // Check for critical issues
                    if (pylintReport.contains('E0001') || pylintReport.contains('E1')) {
                        error('Critical Pylint errors found!')
                    }

                    // Count flake8 issues
                    def flake8Lines = flake8Report.split('\n').findAll { it.trim() }
                    def issueCount = flake8Lines.size()
                    echo "Flake8 issues: ${issueCount}"

                    if (issueCount > 10) {
                        echo "WARNING: High number of style issues"
                        // Uncomment to fail: error('Too many style issues')
                    }
                }
                echo 'Quality gates passed!'
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: '*-report.txt, test-results.xml, coverage.xml', fingerprint: true, allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed'
            sh 'rm -rf __pycache__ .pytest_cache .coverage || true'
        }
        success {
            echo 'All quality checks passed!'
        }
        failure {
            echo 'Quality checks failed - review the reports'
        }
    }
}
EOF
```

2. Commit all the changes.
```
git add .
git commit -m "Add code quality analysis with pylint, flake8, bandit, shellcheck"
git push origin main || git push origin master
```

## I. Running the Quality Pipeline

1. Trigger a build in Jenkins.

2. Observe the parallel execution of quality tools.

3. Review the archived reports after the build.

## J. Setting Up Quality Thresholds

1. Create a quality configuration file.
```
cat << 'EOF' > quality-config.json
{
    "pylint": {
        "min_score": 7.0,
        "fail_on_error": true,
        "fail_on_warning": false
    },
    "flake8": {
        "max_issues": 10,
        "fail_on_error": true
    },
    "coverage": {
        "min_percentage": 80
    },
    "bandit": {
        "fail_on_high": true,
        "fail_on_medium": false
    }
}
EOF
```

2. Add the configuration to the repository.
```
git add quality-config.json
git commit -m "Add quality threshold configuration"
git push origin main || git push origin master
```

## K. Code Complexity Analysis

1. Check cyclomatic complexity with pylint.
```
pylint calculator.py --reports=y | grep -A 20 "Raw metrics"
```

2. Use radon for detailed complexity analysis.
```
pip3 install radon --break-system-packages
```

3. Check cyclomatic complexity.
```
radon cc calculator.py -s
```

4. Check maintainability index.
```
radon mi calculator.py -s
```

### Complexity Grades

| Grade | Complexity | Risk |
|-------|------------|------|
| A | 1-5 | Low |
| B | 6-10 | Low |
| C | 11-20 | Moderate |
| D | 21-30 | High |
| E | 31-40 | Very High |
| F | 40+ | Untestable |

## L. Pre-commit Hooks (Local Quality Gates)

1. Install pre-commit.
```
pip3 install pre-commit --break-system-packages
```

2. Create a pre-commit configuration.
```
cat << 'EOF' > .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: pylint
        name: pylint
        entry: pylint
        language: system
        types: [python]
        args: ['--rcfile=.pylintrc', '--fail-under=7']

      - id: flake8
        name: flake8
        entry: flake8
        language: system
        types: [python]

      - id: shellcheck
        name: shellcheck
        entry: shellcheck
        language: system
        types: [shell]
EOF
```

3. Install the git hooks.
```
sudo apt-get install pre-commit
cd ~/myciproject
pre-commit install
```

4. Test the pre-commit hooks.
```
pre-commit run --all-files
```

5. Add pre-commit config to the repository.
```
git add .pre-commit-config.yaml
git commit -m "Add pre-commit hooks for local quality gates"
git push origin main || git push origin master
```

## M. Viewing Quality Trends (Optional)

> Jenkins can track quality metrics over time with the Warnings Next Generation plugin.

1. In Jenkins, install **Warnings Next Generation** plugin.

2. Update Jenkinsfile to publish warnings:

```groovy
// Add to post section of your pipeline
recordIssues(
    tools: [
        pyLint(pattern: 'pylint-report.txt'),
        flake8(pattern: 'flake8-report.txt')
    ]
)
```

## N. Clean Up

1. Remove temporary files.
```
cd ~/myciproject
rm -f *-report.txt deploy.sh deploy_fixed.sh
rm -rf __pycache__ .pytest_cache .coverage htmlcov
```

## O. Key Takeaways

- **Pylint** provides comprehensive Python analysis with scoring
- **Flake8** is fast and good for style enforcement
- **ShellCheck** catches common bash scripting mistakes
- **Bandit** finds security vulnerabilities in Python
- **Quality gates** fail builds that don't meet standards
- **Pre-commit hooks** catch issues before they're committed
- Run quality checks **in parallel** to speed up pipelines

## P. What's Next

In the next lab, you'll learn to:
- Create deployment pipelines
- Implement staging and production environments
- Add manual approval gates
- Practice rollback procedures
