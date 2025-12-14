# Lab 03: Automated Testing with Jenkins

> This lab covers integrating automated testing into Jenkins pipelines, including unit tests, test reports, and code coverage.

## A. The Testing Pyramid Review

```
        /\
       /  \       E2E Tests (Few, Slow, Expensive)
      /----\
     /      \     Integration Tests (Some, Medium)
    /--------\
   /          \   Unit Tests (Many, Fast, Cheap)
  /__________\
```

### Test Categories

| Type | Scope | Speed | Quantity |
|------|-------|-------|----------|
| Unit | Single function/class | Milliseconds | Hundreds |
| Integration | Multiple components | Seconds | Dozens |
| E2E/System | Full application | Minutes | Few |

## B. Setting Up a Python Test Project

1. Navigate to the project directory.
```
cd ~/myciproject
```

2. Create a proper test directory structure.
```
mkdir -p tests
touch tests/__init__.py
```

3. Create comprehensive unit tests for the calculator.
```
cat << 'EOF' > tests/test_calculator.py
#!/usr/bin/env python3
"""Unit tests for the calculator module."""

import unittest
import sys
sys.path.insert(0, '..')
from calculator import add, subtract, multiply, divide


class TestAddition(unittest.TestCase):
    """Tests for the add function."""

    def test_add_positive_numbers(self):
        """Test adding two positive numbers."""
        self.assertEqual(add(2, 3), 5)

    def test_add_negative_numbers(self):
        """Test adding two negative numbers."""
        self.assertEqual(add(-1, -1), -2)

    def test_add_mixed_numbers(self):
        """Test adding positive and negative numbers."""
        self.assertEqual(add(-1, 1), 0)

    def test_add_zeros(self):
        """Test adding zeros."""
        self.assertEqual(add(0, 0), 0)

    def test_add_floats(self):
        """Test adding floating point numbers."""
        self.assertAlmostEqual(add(0.1, 0.2), 0.3, places=1)


class TestSubtraction(unittest.TestCase):
    """Tests for the subtract function."""

    def test_subtract_positive_numbers(self):
        """Test subtracting positive numbers."""
        self.assertEqual(subtract(5, 3), 2)

    def test_subtract_negative_result(self):
        """Test subtraction resulting in negative number."""
        self.assertEqual(subtract(3, 5), -2)

    def test_subtract_from_zero(self):
        """Test subtracting from zero."""
        self.assertEqual(subtract(0, 5), -5)


class TestMultiplication(unittest.TestCase):
    """Tests for the multiply function."""

    def test_multiply_positive_numbers(self):
        """Test multiplying positive numbers."""
        self.assertEqual(multiply(3, 4), 12)

    def test_multiply_by_zero(self):
        """Test multiplying by zero."""
        self.assertEqual(multiply(5, 0), 0)

    def test_multiply_negative_numbers(self):
        """Test multiplying negative numbers."""
        self.assertEqual(multiply(-2, -3), 6)

    def test_multiply_mixed_signs(self):
        """Test multiplying numbers with different signs."""
        self.assertEqual(multiply(-2, 3), -6)


class TestDivision(unittest.TestCase):
    """Tests for the divide function."""

    def test_divide_evenly(self):
        """Test division that divides evenly."""
        self.assertEqual(divide(10, 2), 5)

    def test_divide_with_remainder(self):
        """Test division with remainder."""
        self.assertAlmostEqual(divide(7, 2), 3.5)

    def test_divide_by_zero(self):
        """Test that division by zero raises an error."""
        with self.assertRaises(ValueError) as context:
            divide(10, 0)
        self.assertIn("zero", str(context.exception).lower())

    def test_divide_zero_by_number(self):
        """Test dividing zero by a number."""
        self.assertEqual(divide(0, 5), 0)


if __name__ == '__main__':
    unittest.main()
EOF
```

4. Install pytest and coverage tools.
```
sudo apt-get install -y python3-pip python3-venv
pip3 install pytest pytest-cov pytest-html --break-system-packages
```

5. Run tests locally to verify they work.
```
cd ~/myciproject
python3 -m pytest tests/ -v
```

## C. Creating Test Reports

1. Run pytest with JUnit XML output (Jenkins-compatible format).
```
python3 -m pytest tests/ -v --junitxml=test-results.xml
```

2. Examine the generated XML report.
```
cat test-results.xml
```

3. Generate an HTML report.
```
python3 -m pytest tests/ -v --html=test-report.html --self-contained-html
```

4. View the HTML report.
```
ls -la test-report.html
```

## D. Code Coverage

1. Run tests with coverage measurement.
```
python3 -m pytest tests/ -v --cov=. --cov-report=xml --cov-report=html --cov-report=term
```

2. Examine coverage output.
```
cat coverage.xml | head -50
```

3. View the HTML coverage report directory.
```
ls -la htmlcov/
```

## E. Update Jenkinsfile for Testing

1. Update the Jenkinsfile with comprehensive testing stages.
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
                echo 'Setting up Python environment...'
                sh '''
                    python3 --version
                    pip3 --version
                    pip3 install pytest pytest-cov pytest-html --break-system-packages || true
                '''
            }
        }

        stage('Lint') {
            steps {
                echo 'Running basic syntax check...'
                sh 'python3 -m py_compile calculator.py'
                sh 'python3 -m py_compile tests/test_calculator.py'
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh '''
                    python3 -m pytest tests/ -v \
                        --junitxml=test-results.xml \
                        --html=test-report.html \
                        --self-contained-html \
                        --cov=. \
                        --cov-report=xml \
                        --cov-report=html \
                        --cov-report=term
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'test-report.html',
                        reportName: 'Test Report'
                    ])
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Coverage Check') {
            steps {
                echo 'Checking coverage threshold...'
                script {
                    def coverage = sh(
                        script: "python3 -m coverage report | grep TOTAL | awk '{print \$4}' | sed 's/%//'",
                        returnStdout: true
                    ).trim()
                    echo "Code coverage: ${coverage}%"
                    if (coverage.toInteger() < 80) {
                        echo "WARNING: Coverage ${coverage}% is below 80% threshold"
                        // Uncomment to fail build: error("Coverage below threshold")
                    }
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'test-results.xml, test-report.html, coverage.xml', fingerprint: true
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'rm -rf __pycache__ .pytest_cache .coverage htmlcov || true'
        }
        success {
            echo 'All tests passed!'
        }
        failure {
            echo 'Tests failed! Check the reports for details.'
        }
    }
}
EOF
```

2. Commit the changes.
```
git add .
git commit -m "Add comprehensive testing with reports and coverage"
git push origin main || git push origin master
```

## F. Install Jenkins Plugins for Reports

1. In Jenkins, go to **Manage Jenkins** → **Plugins** → **Available plugins**.

2. Search for and install:
   - **JUnit** (usually pre-installed)
   - **HTML Publisher**
   - **Code Coverage API** (optional, for coverage trends)

3. Restart Jenkins after plugin installation.
```
sudo systemctl restart jenkins
```

## G. Run the Test Pipeline

1. Open the `calculator-pipeline` job in Jenkins.

2. Click **Build Now**.

3. After the build completes, explore:
   - **Test Result** link (JUnit report)
   - **Test Report** link (HTML test report)
   - **Coverage Report** link (HTML coverage report)

4. Click on the test results to see individual test details.

## H. Writing Tests for Defects (Bug-Driven Testing)

> When a bug is found, write a test that reproduces it first.

1. Let's simulate finding a bug. Add a "buggy" function to calculator.py.
```
cat << 'EOF' >> calculator.py

def percentage(value, percent):
    """Calculate percentage of a value.
    BUG: This implementation is wrong!
    """
    return value * percent  # Should be value * percent / 100
EOF
```

2. Add a test that exposes the bug.
```
cat << 'EOF' >> tests/test_calculator.py


class TestPercentage(unittest.TestCase):
    """Tests for the percentage function - demonstrates bug-first testing."""

    def test_percentage_of_hundred(self):
        """10% of 100 should be 10."""
        from calculator import percentage
        # This test will FAIL because the function is buggy
        self.assertEqual(percentage(100, 10), 10)

    def test_percentage_of_fifty(self):
        """50% of 200 should be 100."""
        from calculator import percentage
        self.assertEqual(percentage(200, 50), 100)
EOF
```

3. Run tests locally - they should fail.
```
python3 -m pytest tests/test_calculator.py::TestPercentage -v
```

4. Now fix the bug in calculator.py.
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

def percentage(value, percent):
    """Calculate percentage of a value."""
    return value * percent / 100  # Fixed!

if __name__ == "__main__":
    print(f"2 + 3 = {add(2, 3)}")
    print(f"5 - 2 = {subtract(5, 2)}")
    print(f"4 * 3 = {multiply(4, 3)}")
    print(f"10 / 2 = {divide(10, 2)}")
    print(f"10% of 100 = {percentage(100, 10)}")
EOF
```

5. Run tests again - they should pass.
```
python3 -m pytest tests/test_calculator.py::TestPercentage -v
```

6. Commit the fix with tests.
```
git add .
git commit -m "Add percentage function with tests (bug fixed)"
git push origin main || git push origin master
```

7. Trigger a Jenkins build and verify all tests pass.

## I. Test Failure Handling

1. Create a pipeline that handles test failures gracefully.

2. Create a new pipeline job named `test-failure-demo`.

3. Add this pipeline script:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }

        stage('Test') {
            steps {
                script {
                    try {
                        sh '''
                            echo "Running tests..."
                            # Simulate a test failure
                            exit 1
                        '''
                    } catch (Exception e) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Tests failed but continuing..."
                    }
                }
            }
        }

        stage('Report') {
            steps {
                echo 'Generating reports...'
            }
        }
    }

    post {
        unstable {
            echo 'Build is UNSTABLE - some tests failed'
        }
    }
}
```

4. Build and observe the unstable status.

## J. Integration Test Example

1. Create an integration test file.
```
cat << 'EOF' > tests/test_integration.py
#!/usr/bin/env python3
"""Integration tests - test components working together."""

import unittest
import sys
sys.path.insert(0, '..')
from calculator import add, subtract, multiply, divide, percentage


class TestCalculatorIntegration(unittest.TestCase):
    """Integration tests for calculator operations."""

    def test_complex_calculation(self):
        """Test a complex multi-step calculation."""
        # (10 + 5) * 2 - 3 = 27
        result = add(10, 5)
        result = multiply(result, 2)
        result = subtract(result, 3)
        self.assertEqual(result, 27)

    def test_financial_calculation(self):
        """Test a financial calculation scenario."""
        # Calculate 20% discount on $150
        price = 150
        discount_percent = 20
        discount_amount = percentage(price, discount_percent)
        final_price = subtract(price, discount_amount)
        self.assertEqual(final_price, 120)

    def test_division_with_validation(self):
        """Test division with proper error handling."""
        # First divide, then validate result
        result = divide(100, 4)
        self.assertEqual(result, 25)

        # Verify multiplication reverses it
        reversed_result = multiply(result, 4)
        self.assertEqual(reversed_result, 100)


if __name__ == '__main__':
    unittest.main()
EOF
```

2. Run integration tests.
```
python3 -m pytest tests/test_integration.py -v
```

3. Commit the integration tests.
```
git add .
git commit -m "Add integration tests"
git push origin main || git push origin master
```

## K. Test Organization Best Practices

### Directory Structure
```
myciproject/
├── calculator.py          # Source code
├── Jenkinsfile            # Pipeline definition
├── tests/
│   ├── __init__.py
│   ├── test_calculator.py # Unit tests
│   ├── test_integration.py # Integration tests
│   └── conftest.py        # Shared fixtures (optional)
└── requirements.txt       # Dependencies
```

### Create requirements.txt
```
cat << 'EOF' > requirements.txt
pytest>=7.0.0
pytest-cov>=4.0.0
pytest-html>=3.0.0
EOF
```

### Commit the requirements file.
```
git add requirements.txt
git commit -m "Add requirements.txt"
git push origin main || git push origin master
```

## L. Clean Up

1. Remove temporary test files.
```
cd ~/myciproject
rm -rf __pycache__ tests/__pycache__ .pytest_cache .coverage htmlcov test-results.xml test-report.html coverage.xml
```

## M. Key Takeaways

- **pytest** is a powerful Python testing framework
- **JUnit XML** is the standard format for CI test reports
- **Code coverage** measures what percentage of code is tested
- **HTML reports** provide detailed, browsable test results
- **Bug-driven testing** ensures bugs don't return
- **Test stages** in Jenkins provide clear visibility into test results

## N. What's Next

In the next lab, you'll learn to:
- Run static code analysis
- Integrate linters (pylint, flake8, shellcheck)
- Set up quality gates that fail builds on violations
