# Coding Guidelines for Cloud Test Framework

## 1. Introduction

This document outlines the coding guidelines and best practices for the Cloud Test Framework. Following these guidelines ensures code consistency, maintainability, and adherence to Python standards across the project.

## 2. Python Version

- Use Python 3.11 or higher for all development
- Ensure compatibility with Python 3.8+ for broader adoption
- Use type hints throughout the codebase

## 3. Code Style

### 3.1 PEP 8 Compliance

All code must comply with [PEP 8](https://www.python.org/dev/peps/pep-0008/) standards with the following specifications:

- Maximum line length: 100 characters
- Indentation: 4 spaces (no tabs)
- Use snake_case for variables, functions, and methods
- Use CamelCase for classes
- Use UPPER_CASE for constants
- Use leading underscore for private attributes and methods (e.g., `_private_method`)

### 3.2 Black Formatting

- All code must be formatted using [Black](https://black.readthedocs.io/) with default settings
- Black configuration in `pyproject.toml`:

```toml
[tool.black]
line-length = 100
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | _build
  | buck-out
  | build
  | dist
)/
'''
```

### 3.3 Import Ordering

- Use [isort](https://pycqa.github.io/isort/) to organize imports
- isort configuration in `pyproject.toml`:

```toml
[tool.isort]
profile = "black"
line_length = 100
multi_line_output = 3
include_trailing_comma = true
force_grid_wrap = 0
use_parentheses = true
ensure_newline_before_comments = true
```

### 3.4 Docstrings

- Use [Google style docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)
- All modules, classes, methods, and functions must have docstrings
- Include type information in docstrings even when using type hints
- Example:

```python
def create_vm(instance_type: str, image_id: str, region: str) -> Dict[str, Any]:
    """Create a virtual machine instance.
    
    Args:
        instance_type: The type of instance to create (e.g., 't3.micro')
        image_id: The ID of the image to use (e.g., 'ami-12345')
        region: The region to create the instance in (e.g., 'us-west-2')
        
    Returns:
        Dict containing the created instance details
        
    Raises:
        CloudProviderError: If instance creation fails
    """
```

### 3.5 Type Hints

- Use type hints for all function parameters and return values
- Use `typing` module for complex types (e.g., `List`, `Dict`, `Optional`)
- Use `Any` sparingly and only when necessary
- For Python 3.9+, use built-in generics (e.g., `list[str]` instead of `List[str]`)
- Example:

```python
from typing import Dict, List, Optional, Any

def get_instances(region: str, filters: Optional[Dict[str, str]] = None) -> List[Dict[str, Any]]:
    """Get instances in the specified region."""
```

## 4. Project Structure

### 4.1 Directory Structure

```
cloud_test_framework/
├── src/
│   └── cloudtestlib/
│       ├── core/
│       │   ├── __init__.py
│       │   ├── logging.py
│       │   ├── config.py
│       │   └── ...
│       ├── providers/
│       │   ├── __init__.py
│       │   ├── aws/
│       │   ├── azure/
│       │   └── gcp/
│       ├── monitoring/
│       │   ├── __init__.py
│       │   ├── cloudwatch.py
│       │   ├── azure_monitor.py
│       │   └── ...
│       ├── kubernetes/
│       │   ├── __init__.py
│       │   ├── client.py
│       │   └── ...
│       ├── reporting/
│       │   ├── __init__.py
│       │   ├── allure.py
│       │   ├── testrail.py
│       │   └── ...
│       ├── sedai/
│       │   ├── __init__.py
│       │   ├── client.py
│       │   └── ...
│       └── pytest_plugins/
│           ├── __init__.py
│           ├── pytest_parallel.py
│           └── ...
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   ├── performance/
│   ├── security/
│   ├── cross_validation/
│   └── validation/
├── docs/
│   ├── api/
│   ├── guides/
│   └── examples/
├── examples/
├── scripts/
├── pyproject.toml
├── setup.py
├── setup.cfg
├── README.md
└── LICENSE
```

### 4.2 Module Organization

- Each module should have a single, well-defined responsibility
- Related functionality should be grouped together
- Public API should be exposed through `__init__.py` files
- Example:

```python
# src/cloudtestlib/providers/aws/__init__.py
from .ec2 import EC2Client
from .s3 import S3Client
from .rds import RDSClient
from .provider import AWSProvider

__all__ = ["EC2Client", "S3Client", "RDSClient", "AWSProvider"]
```

## 5. Design Patterns

### 5.1 Factory Pattern

Use factory pattern for creating provider-specific implementations:

```python
class ProviderFactory:
    """Factory for creating cloud provider instances."""
    
    @staticmethod
    def create_provider(provider_type: str, **kwargs) -> CloudProvider:
        """Create a cloud provider instance.
        
        Args:
            provider_type: Type of provider ('aws', 'azure', 'gcp')
            **kwargs: Provider-specific configuration
            
        Returns:
            CloudProvider instance
            
        Raises:
            ValueError: If provider_type is not supported
        """
        if provider_type == "aws":
            return AWSProvider(**kwargs)
        elif provider_type == "azure":
            return AzureProvider(**kwargs)
        elif provider_type == "gcp":
            return GCPProvider(**kwargs)
        else:
            raise ValueError(f"Unsupported provider type: {provider_type}")
```

### 5.2 Strategy Pattern

Use strategy pattern for implementing different algorithms or behaviors:

```python
class TestDistributionStrategy(ABC):
    """Base class for test distribution strategies."""
    
    @abstractmethod
    def distribute(self, tests: List[str], num_pipelines: int) -> List[List[str]]:
        """Distribute tests across pipelines."""
        pass

class ModuleBasedDistribution(TestDistributionStrategy):
    """Distribute tests by module."""
    
    def distribute(self, tests: List[str], num_pipelines: int) -> List[List[str]]:
        """Distribute tests by module."""
        # Implementation
        
class ComplexityBasedDistribution(TestDistributionStrategy):
    """Distribute tests by complexity."""
    
    def distribute(self, tests: List[str], num_pipelines: int) -> List[List[str]]:
        """Distribute tests by complexity."""
        # Implementation
```

### 5.3 Singleton Pattern

Use singleton pattern for managers and shared resources:

```python
class LoggingManager:
    """Singleton manager for logging configuration."""
    
    _instance = None
    
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super(LoggingManager, cls).__new__(cls)
            cls._instance._initialized = False
        return cls._instance
    
    def __init__(self, config_file=None):
        if self._initialized:
            return
        # Initialize logging
        self._initialized = True
```

### 5.4 Adapter Pattern

Use adapter pattern for integrating with external systems:

```python
class TestRailAdapter:
    """Adapter for TestRail integration."""
    
    def __init__(self, url: str, username: str, password: str):
        self.client = TestRailClient(url, username, password)
    
    def add_result(self, test_result: TestResult) -> None:
        """Add test result to TestRail."""
        # Convert TestResult to TestRail format
        testrail_result = {
            "test_id": test_result.test_id,
            "status_id": self._convert_status(test_result.status),
            "comment": test_result.message,
            "elapsed": f"{test_result.duration}s"
        }
        self.client.add_result(**testrail_result)
    
    def _convert_status(self, status: str) -> int:
        """Convert status to TestRail status ID."""
        # Implementation
```

### 5.5 Builder Pattern

Use builder pattern for complex object construction:

```python
class TestCaseBuilder:
    """Builder for test case configuration."""
    
    def __init__(self, name: str):
        self.test_case = {"name": name, "steps": []}
    
    def with_description(self, description: str) -> "TestCaseBuilder":
        """Add description to test case."""
        self.test_case["description"] = description
        return self
    
    def with_priority(self, priority: int) -> "TestCaseBuilder":
        """Add priority to test case."""
        self.test_case["priority"] = priority
        return self
    
    def with_step(self, description: str, expected: str) -> "TestCaseBuilder":
        """Add step to test case."""
        self.test_case["steps"].append({
            "description": description,
            "expected": expected
        })
        return self
    
    def build(self) -> Dict[str, Any]:
        """Build the test case."""
        return self.test_case
```

## 6. Error Handling

### 6.1 Exception Hierarchy

Define a clear exception hierarchy:

```python
class CloudTestError(Exception):
    """Base exception for all cloud test framework errors."""
    pass

class CloudProviderError(CloudTestError):
    """Base exception for cloud provider errors."""
    pass

class AWSError(CloudProviderError):
    """AWS-specific errors."""
    pass

class AzureError(CloudProviderError):
    """Azure-specific errors."""
    pass

class GCPError(CloudProviderError):
    """GCP-specific errors."""
    pass

class MonitoringError(CloudTestError):
    """Base exception for monitoring errors."""
    pass

class ConfigurationError(CloudTestError):
    """Configuration-related errors."""
    pass
```

### 6.2 Error Handling Guidelines

- Be specific about which exceptions to catch
- Avoid catching `Exception` or `BaseException` directly
- Re-raise exceptions with context when appropriate
- Log exceptions with sufficient context
- Use context managers for resource cleanup
- Example:

```python
try:
    result = aws_client.create_instance(instance_type, image_id)
except ClientError as e:
    if e.response["Error"]["Code"] == "InvalidParameterValue":
        logger.error(f"Invalid parameter: {e}")
        raise AWSError(f"Invalid parameter for instance creation: {e}") from e
    elif e.response["Error"]["Code"] == "UnauthorizedOperation":
        logger.error(f"Unauthorized operation: {e}")
        raise AWSError(f"Unauthorized instance creation: {e}") from e
    else:
        logger.error(f"Unexpected error creating instance: {e}")
        raise AWSError(f"Failed to create instance: {e}") from e
```

## 7. Logging

### 7.1 Logging Guidelines

- Use the framework's logging module instead of `print` statements
- Log at appropriate levels:
  - DEBUG: Detailed information for debugging
  - INFO: Confirmation that things are working as expected
  - WARNING: Indication that something unexpected happened, but the application still works
  - ERROR: Due to a more serious problem, the application has not been able to perform a function
  - CRITICAL: A serious error, indicating that the application itself may be unable to continue running
- Include context in log messages
- Example:

```python
from cloudtestlib.core.logging import get_logger

logger = get_logger(__name__)

def create_vm(instance_type: str, image_id: str, region: str) -> Dict[str, Any]:
    """Create a virtual machine instance."""
    logger.info(f"Creating VM with type={instance_type}, image={image_id} in region={region}")
    try:
        # Create VM
        logger.debug(f"Sending create request to provider API")
        result = provider_client.create_vm(instance_type, image_id, region)
        logger.info(f"VM created successfully: id={result['id']}")
        return result
    except Exception as e:
        logger.error(f"Failed to create VM: {e}", exc_info=True)
        raise
```

### 7.2 Structured Logging

Use structured logging for machine-readable logs:

```python
logger.info("VM created", extra={
    "vm_id": vm.id,
    "instance_type": vm.instance_type,
    "region": vm.region,
    "creation_time": vm.creation_time.isoformat()
})
```

## 8. Testing

### 8.1 Test Organization

- Organize tests to mirror the structure of the code being tested
- Use descriptive test names that explain what is being tested
- Group related tests in test classes
- Example:

```python
class TestEC2Client:
    """Tests for EC2Client."""
    
    def test_create_instance_with_valid_parameters(self):
        """Test creating an instance with valid parameters."""
        
    def test_create_instance_with_invalid_instance_type(self):
        """Test creating an instance with invalid instance type."""
        
    def test_start_instance_success(self):
        """Test starting an instance successfully."""
```

### 8.2 Test Fixtures

- Use pytest fixtures for test setup and teardown
- Share fixtures across test modules when appropriate
- Use fixture scope appropriately (function, class, module, session)
- Example:

```python
@pytest.fixture(scope="module")
def aws_client():
    """Create AWS client for testing."""
    client = AWSClient(
        region="us-west-2",
        access_key="test-key",
        secret_key="test-secret"
    )
    yield client
    # Cleanup resources
    client.cleanup()

@pytest.fixture
def ec2_instance(aws_client):
    """Create EC2 instance for testing."""
    instance = aws_client.create_instance(
        instance_type="t3.micro",
        image_id="ami-12345"
    )
    yield instance
    # Terminate instance
    aws_client.terminate_instance(instance["id"])
```

### 8.3 Mocking

- Use `unittest.mock` or `pytest-mock` for mocking
- Mock external dependencies, not the code being tested
- Verify mock calls when behavior is important
- Example:

```python
def test_create_instance(mocker):
    """Test creating an instance."""
    # Mock boto3 client
    mock_ec2 = mocker.patch("boto3.client")
    mock_ec2.return_value.run_instances.return_value = {
        "Instances": [{"InstanceId": "i-12345"}]
    }
    
    # Create client and call method
    client = EC2Client(region="us-west-2")
    result = client.create_instance(instance_type="t3.micro", image_id="ami-12345")
    
    # Verify result
    assert result["id"] == "i-12345"
    
    # Verify mock was called correctly
    mock_ec2.return_value.run_instances.assert_called_once_with(
        ImageId="ami-12345",
        InstanceType="t3.micro",
        MinCount=1,
        MaxCount=1
    )
```

### 8.4 Parameterized Tests

Use parameterized tests for testing multiple scenarios:

```python
@pytest.mark.parametrize("instance_type,expected_error", [
    ("t3.micro", None),
    ("invalid-type", ValueError),
    ("t3.2xlarge", None),
])
def test_validate_instance_type(instance_type, expected_error):
    """Test validating instance types."""
    if expected_error:
        with pytest.raises(expected_error):
            EC2Client.validate_instance_type(instance_type)
    else:
        EC2Client.validate_instance_type(instance_type)  # Should not raise
```

### 8.5 Test Markers

Use pytest markers to categorize tests:

```python
@pytest.mark.aws
@pytest.mark.slow
def test_create_and_terminate_instance():
    """Test creating and terminating an instance."""
    # Test implementation
```

## 9. Documentation

### 9.1 Code Documentation

- Document all public APIs
- Include examples in docstrings
- Document parameters, return values, and exceptions
- Document complex algorithms and non-obvious behavior

### 9.2 Project Documentation

- Maintain up-to-date README with installation and usage instructions
- Provide comprehensive API documentation
- Include examples and tutorials
- Document design decisions and architecture

## 10. Performance Considerations

### 10.1 Resource Management

- Use context managers for resource cleanup
- Close connections and file handles explicitly
- Release resources in finally blocks
- Example:

```python
def download_file(url: str, output_path: str) -> None:
    """Download file from URL."""
    try:
        response = requests.get(url, stream=True)
        response.raise_for_status()
        
        with open(output_path, "wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
    finally:
        response.close()
```

### 10.2 Optimization Guidelines

- Profile code before optimizing
- Focus on algorithmic improvements over micro-optimizations
- Use appropriate data structures for the task
- Consider memory usage for large datasets
- Use generators for processing large datasets
- Example:

```python
# Instead of loading all instances into memory
all_instances = []
for region in regions:
    all_instances.extend(client.get_instances(region))

# Use a generator to process instances one at a time
def get_all_instances(regions):
    for region in regions:
        yield from client.get_instances(region)

for instance in get_all_instances(regions):
    process_instance(instance)
```

## 11. Security Considerations

### 11.1 Credential Management

- Never hardcode credentials in source code
- Use environment variables or secure credential stores
- Rotate credentials regularly
- Use least privilege principle for service accounts
- Example:

```python
def get_aws_credentials():
    """Get AWS credentials from environment or credential store."""
    # Try environment variables
    access_key = os.environ.get("AWS_ACCESS_KEY_ID")
    secret_key = os.environ.get("AWS_SECRET_ACCESS_KEY")
    
    if access_key and secret_key:
        return access_key, secret_key
    
    # Try AWS credentials file
    session = boto3.Session()
    credentials = session.get_credentials()
    
    if credentials:
        return credentials.access_key, credentials.secret_key
    
    # Try AWS Secrets Manager
    secrets_client = boto3.client("secretsmanager")
    response = secrets_client.get_secret_value(SecretId="aws/credentials")
    secret = json.loads(response["SecretString"])
    
    return secret["access_key"], secret["secret_key"]
```

### 11.2 Input Validation

- Validate all inputs, especially those from external sources
- Use type hints and runtime type checking
- Sanitize inputs before using them in queries or commands
- Example:

```python
def create_instance(instance_type: str, image_id: str, region: str) -> Dict[str, Any]:
    """Create an instance."""
    # Validate inputs
    if not isinstance(instance_type, str) or not instance_type:
        raise ValueError("instance_type must be a non-empty string")
    
    if not isinstance(image_id, str) or not image_id:
        raise ValueError("image_id must be a non-empty string")
    
    if not isinstance(region, str) or not region:
        raise ValueError("region must be a non-empty string")
    
    # Validate instance type format
    if not re.match(r'^[a-z][1-9][.][a-z0-9]+$', instance_type):
        raise ValueError(f"Invalid instance type format: {instance_type}")
    
    # Validate image ID format
    if not re.match(r'^ami-[a-f0-9]+$', image_id):
        raise ValueError(f"Invalid image ID format: {image_id}")
    
    # Validate region format
    if not re.match(r'^[a-z]+-[a-z]+-[0-9]+$', region):
        raise ValueError(f"Invalid region format: {region}")
    
    # Create instance
    return client.create_instance(instance_type, image_id, region)
```

## 12. Continuous Integration

### 12.1 CI Checks

Implement the following CI checks:

- Linting with flake8
- Formatting with Black
- Type checking with mypy
- Unit tests with pytest
- Code coverage with pytest-cov
- Security scanning with bandit
- Example CI configuration:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install flake8 black isort mypy bandit
    - name: Lint with flake8
      run: flake8 src tests
    - name: Check formatting with Black
      run: black --check src tests
    - name: Check import order with isort
      run: isort --check-only --profile black src tests
    - name: Type check with mypy
      run: mypy src
    - name: Security scan with bandit
      run: bandit -r src

  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -e .[dev,test]
    - name: Test with pytest
      run: pytest tests/unit --cov=src --cov-report=xml
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
```

### 12.2 Pre-commit Hooks

Use pre-commit hooks to enforce standards:

```yaml
# .pre-commit-config.yaml
repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
    -   id: trailing-whitespace
    -   id: end-of-file-fixer
    -   id: check-yaml
    -   id: check-added-large-files

-   repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
    -   id: isort
        args: ["--profile", "black"]

-   repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
    -   id: black

-   repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
    -   id: flake8
        additional_dependencies: [flake8-docstrings]

-   repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.3.0
    hooks:
    -   id: mypy
        additional_dependencies: [types-requests, types-PyYAML]
```

## 13. Dependency Management

### 13.1 Package Dependencies

- Specify dependencies in `pyproject.toml` and `setup.py`
- Use version ranges for dependencies
- Pin exact versions in `requirements.txt` for reproducible builds
- Example:

```python
# setup.py
setup(
    name="cloudtestlib",
    version="0.1.0",
    packages=find_packages(where="src"),
    package_dir={"": "src"},
    python_requires=">=3.8",
    install_requires=[
        "boto3>=1.26.0,<2.0.0",
        "azure-mgmt-compute>=30.0.0,<31.0.0",
        "google-cloud-compute>=1.10.0,<2.0.0",
        "kubernetes>=26.1.0,<27.0.0",
        "requests>=2.28.0,<3.0.0",
        "pyyaml>=6.0,<7.0",
    ],
    extras_require={
        "dev": [
            "black",
            "flake8",
            "isort",
            "mypy",
            "pre-commit",
        ],
        "test": [
            "pytest",
            "pytest-cov",
            "pytest-mock",
            "pytest-xdist",
        ],
    },
)
```

### 13.2 Virtual Environments

- Use virtual environments for development
- Document environment setup in README
- Provide environment setup scripts
- Example:

```bash
#!/bin/bash
# setup_dev_env.sh

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -e .[dev,test]

# Install pre-commit hooks
pre-commit install
```

## 14. Code Review Guidelines

### 14.1 Code Review Checklist

- Code follows style guidelines
- Tests are included and pass
- Documentation is updated
- Error handling is appropriate
- Security considerations are addressed
- Performance implications are considered
- No unnecessary dependencies
- No hardcoded credentials or sensitive information

### 14.2 Pull Request Template

```markdown
## Description
<!-- Describe the changes in this PR -->

## Related Issue
<!-- Link to the related issue -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## How Has This Been Tested?
<!-- Describe the tests that you ran -->

## Checklist
- [ ] My code follows the style guidelines of this project
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] Any dependent changes have been merged and published in downstream modules
```

## 15. Versioning and Releases

### 15.1 Versioning Scheme

- Follow [Semantic Versioning](https://semver.org/)
- MAJOR.MINOR.PATCH format
- Increment MAJOR for incompatible API changes
- Increment MINOR for new functionality in a backward compatible manner
- Increment PATCH for backward compatible bug fixes

### 15.2 Release Process

- Update version in `pyproject.toml` and `setup.py`
- Update CHANGELOG.md
- Create a release branch
- Tag the release
- Build and publish package
- Example:

```bash
#!/bin/bash
# release.sh

# Get version from argument
VERSION=$1

# Update version in files
sed -i "s/version=\".*\"/version=\"$VERSION\"/" setup.py
sed -i "s/version = \".*\"/version = \"$VERSION\"/" pyproject.toml

# Commit changes
git add setup.py pyproject.toml
git commit -m "Bump version to $VERSION"

# Create release branch
git checkout -b release/$VERSION

# Create tag
git tag -a v$VERSION -m "Release $VERSION"

# Push branch and tag
git push origin release/$VERSION
git push origin v$VERSION

# Build package
python -m build

# Publish package
python -m twine upload dist/*
```

## 16. Conclusion

Following these coding guidelines ensures that the Cloud Test Framework codebase remains consistent, maintainable, and adheres to Python best practices. All contributors should familiarize themselves with these guidelines and apply them in their work.

## Appendix A: Tools and Resources

### A.1 Recommended Tools

- **Linting**: flake8, pylint
- **Formatting**: black, isort
- **Type Checking**: mypy
- **Testing**: pytest, pytest-cov, pytest-mock, pytest-xdist
- **Documentation**: Sphinx, Read the Docs
- **Security**: bandit, safety
- **Pre-commit**: pre-commit

### A.2 Resources

- [PEP 8 -- Style Guide for Python Code](https://www.python.org/dev/peps/pep-0008/)
- [PEP 484 -- Type Hints](https://www.python.org/dev/peps/pep-0484/)
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [Black Documentation](https://black.readthedocs.io/)
- [mypy Documentation](https://mypy.readthedocs.io/)
- [pytest Documentation](https://docs.pytest.org/)
- [Semantic Versioning](https://semver.org/)
