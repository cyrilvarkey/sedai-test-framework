[requirements_analysis.md](../../../../Downloads/Framework/Creating%20Cloudtestlib%20Monitoring%20Files%20and%20Correcting%20the%20Solution%20(5)/doc_cloud_test_framework/requirements_analysis.md)# External Test Reporting Integration

This document outlines the implementation of external test reporting integration for the enhanced cloud testing framework, including TestRail, Zephyr, and other test management systems.

## 1. External Reporting Architecture

### 1.1 Reporting Architecture Overview

We'll implement a modular reporting architecture that supports multiple external reporting systems:

```
┌─────────────────────────────────────────────────────┐
│                  Reporting Manager                  │
├─────────────────────────────────────────────────────┤
│                  Reporter Factory                   │
├───────────────┬───────────────┬───────────────┬─────┤
│   TestRail    │    Zephyr     │    Allure     │ ... │
│   Reporter    │   Reporter    │   Reporter    │     │
└───────────────┴───────────────┴───────────────┴─────┘
```

### 1.2 Core Reporting Interfaces

```python
# src/cloudtestlib/reporting/base.py

from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional
import pytest

class TestResult:
    """
    Class representing a test result.
    """
    
    def __init__(self, 
                 test_id: str,
                 name: str,
                 outcome: str,
                 duration: float,
                 message: Optional[str] = None,
                 traceback: Optional[str] = None,
                 metadata: Optional[Dict[str, Any]] = None):
        """
        Initialize a test result.
        
        Args:
            test_id: Unique identifier for the test
            name: Test name
            outcome: Test outcome (passed, failed, skipped)
            duration: Test duration in seconds
            message: Test message (e.g., failure message)
            traceback: Test traceback (for failures)
            metadata: Additional test metadata
        """
        self.test_id = test_id
        self.name = name
        self.outcome = outcome
        self.duration = duration
        self.message = message
        self.traceback = traceback
        self.metadata = metadata or {}
        
    @classmethod
    def from_pytest_report(cls, report):
        """
        Create a TestResult from a pytest report.
        
        Args:
            report: pytest report object
            
        Returns:
            TestResult instance
        """
        # Extract test ID and name
        test_id = report.nodeid
        name = report.nodeid.split("::")[-1]
        
        # Extract outcome
        if report.passed:
            outcome = "passed"
        elif report.failed:
            outcome = "failed"
        elif report.skipped:
            outcome = "skipped"
        else:
            outcome = "unknown"
            
        # Extract duration
        duration = getattr(report, "duration", 0.0)
        
        # Extract message and traceback
        message = None
        traceback = None
        
        if hasattr(report, "longrepr"):
            if isinstance(report.longrepr, tuple):
                message = report.longrepr[2]
                traceback = "\n".join(report.longreprtext.splitlines())
            elif isinstance(report.longrepr, str):
                message = report.longrepr
                
        # Extract metadata
        metadata = {}
        
        # Add markers as metadata
        if hasattr(report, "keywords"):
            markers = [k for k, v in report.keywords.items() if isinstance(v, pytest.mark.MarkDecorator)]
            metadata["markers"] = markers
            
        # Add test parameters as metadata
        if hasattr(report, "callspec"):
            params = getattr(report.callspec, "params", {})
            metadata["parameters"] = params
            
        return cls(
            test_id=test_id,
            name=name,
            outcome=outcome,
            duration=duration,
            message=message,
            traceback=traceback,
            metadata=metadata
        )

class ExternalReporter(ABC):
    """
    Abstract base class for external test reporters.
    """
    
    @abstractmethod
    def initialize(self, **kwargs) -> bool:
        """
        Initialize the reporter.
        
        Args:
            **kwargs: Reporter-specific initialization arguments
            
        Returns:
            True if initialization was successful, False otherwise
        """
        pass
        
    @abstractmethod
    def report_result(self, result: TestResult) -> bool:
        """
        Report a test result to the external system.
        
        Args:
            result: Test result to report
            
        Returns:
            True if reporting was successful, False otherwise
        """
        pass
        
    @abstractmethod
    def report_results(self, results: List[TestResult]) -> Dict[str, bool]:
        """
        Report multiple test results to the external system.
        
        Args:
            results: List of test results to report
            
        Returns:
            Dictionary mapping test IDs to reporting success
        """
        pass
        
    @abstractmethod
    def finalize(self) -> bool:
        """
        Finalize reporting (e.g., close connections, submit final reports).
        
        Returns:
            True if finalization was successful, False otherwise
        """
        pass
```

### 1.3 Reporter Factory

```python
# src/cloudtestlib/reporting/factory.py

from typing import Dict, Optional, Type
from cloudtestlib.reporting.base import ExternalReporter

class ReporterFactory:
    """
    Factory for creating external reporters.
    """
    
    _reporters = {}
    
    @classmethod
    def register_reporter(cls, reporter_id: str, reporter_class: Type[ExternalReporter]) -> None:
        """
        Register a reporter class.
        
        Args:
            reporter_id: Reporter identifier
            reporter_class: Reporter class
        """
        cls._reporters[reporter_id] = reporter_class
        
    @classmethod
    def create_reporter(cls, reporter_id: str, **kwargs) -> ExternalReporter:
        """
        Create a reporter instance.
        
        Args:
            reporter_id: Reporter identifier
            **kwargs: Reporter-specific initialization arguments
            
        Returns:
            Reporter instance
        """
        if reporter_id not in cls._reporters:
            raise ValueError(f"Unknown reporter: {reporter_id}")
            
        reporter_class = cls._reporters[reporter_id]
        reporter = reporter_class()
        reporter.initialize(**kwargs)
        
        return reporter
        
    @classmethod
    def list_reporters(cls) -> Dict[str, Type[ExternalReporter]]:
        """
        List all registered reporters.
        
        Returns:
            Dictionary mapping reporter IDs to reporter classes
        """
        return cls._reporters.copy()
```

### 1.4 Reporting Manager

```python
# src/cloudtestlib/reporting/manager.py

from typing import Dict, List, Optional, Set
import logging
from cloudtestlib.reporting.base import ExternalReporter, TestResult
from cloudtestlib.reporting.factory import ReporterFactory

class ReportingManager:
    """
    Manager for external test reporting.
    """
    
    def __init__(self):
        """Initialize the reporting manager."""
        self._reporters = {}
        self._logger = logging.getLogger(__name__)
        
    def add_reporter(self, reporter_id: str, reporter: ExternalReporter) -> None:
        """
        Add a reporter to the manager.
        
        Args:
            reporter_id: Reporter identifier
            reporter: Reporter instance
        """
        self._reporters[reporter_id] = reporter
        
    def create_and_add_reporter(self, reporter_id: str, **kwargs) -> ExternalReporter:
        """
        Create and add a reporter to the manager.
        
        Args:
            reporter_id: Reporter identifier
            **kwargs: Reporter-specific initialization arguments
            
        Returns:
            Created reporter instance
        """
        reporter = ReporterFactory.create_reporter(reporter_id, **kwargs)
        self.add_reporter(reporter_id, reporter)
        return reporter
        
    def remove_reporter(self, reporter_id: str) -> bool:
        """
        Remove a reporter from the manager.
        
        Args:
            reporter_id: Reporter identifier
            
        Returns:
            True if reporter was removed, False otherwise
        """
        if reporter_id in self._reporters:
            reporter = self._reporters.pop(reporter_id)
            reporter.finalize()
            return True
        return False
        
    def report_result(self, result: TestResult, reporter_ids: Optional[Set[str]] = None) -> Dict[str, bool]:
        """
        Report a test result to specified reporters.
        
        Args:
            result: Test result to report
            reporter_ids: Set of reporter IDs to report to (None for all)
            
        Returns:
            Dictionary mapping reporter IDs to reporting success
        """
        if reporter_ids is None:
            reporter_ids = set(self._reporters.keys())
            
        results = {}
        
        for reporter_id in reporter_ids:
            if reporter_id in self._reporters:
                try:
                    success = self._reporters[reporter_id].report_result(result)
                    results[reporter_id] = success
                except Exception as e:
                    self._logger.error(f"Error reporting to {reporter_id}: {e}")
                    results[reporter_id] = False
            else:
                results[reporter_id] = False
                
        return results
        
    def report_results(self, results: List[TestResult], reporter_ids: Optional[Set[str]] = None) -> Dict[str, Dict[str, bool]]:
        """
        Report multiple test results to specified reporters.
        
        Args:
            results: List of test results to report
            reporter_ids: Set of reporter IDs to report to (None for all)
            
        Returns:
            Dictionary mapping reporter IDs to dictionaries mapping test IDs to reporting success
        """
        if reporter_ids is None:
            reporter_ids = set(self._reporters.keys())
            
        report_results = {}
        
        for reporter_id in reporter_ids:
            if reporter_id in self._reporters:
                try:
                    success_dict = self._reporters[reporter_id].report_results(results)
                    report_results[reporter_id] = success_dict
                except Exception as e:
                    self._logger.error(f"Error reporting to {reporter_id}: {e}")
                    report_results[reporter_id] = {result.test_id: False for result in results}
            else:
                report_results[reporter_id] = {result.test_id: False for result in results}
                
        return report_results
        
    def finalize_all(self) -> Dict[str, bool]:
        """
        Finalize all reporters.
        
        Returns:
            Dictionary mapping reporter IDs to finalization success
        """
        results = {}
        
        for reporter_id, reporter in self._reporters.items():
            try:
                success = reporter.finalize()
                results[reporter_id] = success
            except Exception as e:
                self._logger.error(f"Error finalizing {reporter_id}: {e}")
                results[reporter_id] = False
                
        return results
```

## 2. TestRail Integration

### 2.1 TestRail Reporter Implementation

```python
# src/cloudtestlib/reporting/testrail.py

import re
import requests
from typing import Dict, List, Optional, Tuple, Union
from cloudtestlib.reporting.base import ExternalReporter, TestResult

class TestRailReporter(ExternalReporter):
    """
    Reporter for TestRail integration.
    """
    
    # TestRail API status IDs
    STATUS_PASSED = 1
    STATUS_BLOCKED = 2
    STATUS_UNTESTED = 3
    STATUS_RETEST = 4
    STATUS_FAILED = 5
    STATUS_SKIPPED = 6
    
    def __init__(self):
        """Initialize the TestRail reporter."""
        self._url = None
        self._username = None
        self._password = None
        self._project_id = None
        self._suite_id = None
        self._run_id = None
        self._case_id_pattern = re.compile(r'C(\d+)')
        self._session = None
        
    def initialize(self, **kwargs) -> bool:
        """
        Initialize the TestRail reporter.
        
        Args:
            url: TestRail URL
            username: TestRail username
            password: TestRail password or API key
            project_id: TestRail project ID
            suite_id: TestRail test suite ID
            run_id: TestRail test run ID (optional)
            run_name: Name for a new test run (if run_id not provided)
            run_description: Description for a new test run (if run_id not provided)
            
        Returns:
            True if initialization was successful, False otherwise
        """
        self._url = kwargs.get('url')
        self._username = kwargs.get('username')
        self._password = kwargs.get('password')
        self._project_id = kwargs.get('project_id')
        self._suite_id = kwargs.get('suite_id')
        self._run_id = kwargs.get('run_id')
        
        if not all([self._url, self._username, self._password, self._project_id]):
            return False
            
        # Initialize session
        self._session = requests.Session()
        self._session.auth = (self._username, self._password)
        self._session.headers.update({'Content-Type': 'application/json'})
        
        # Create a new test run if run_id not provided
        if not self._run_id:
            run_name = kwargs.get('run_name', 'Automated Test Run')
            run_description = kwargs.get('run_description', 'Created by cloudtestlib')
            
            self._run_id = self._create_test_run(run_name, run_description)
            
        return self._run_id is not None
        
    def report_result(self, result: TestResult) -> bool:
        """
        Report a test result to TestRail.
        
        Args:
            result: Test result to report
            
        Returns:
            True if reporting was successful, False otherwise
        """
        case_id = self._extract_case_id(result)
        if not case_id:
            return False
            
        status_id = self._map_outcome_to_status_id(result.outcome)
        comment = self._format_comment(result)
        
        return self._add_test_result(case_id, status_id, comment, result.duration)
        
    def report_results(self, results: List[TestResult]) -> Dict[str, bool]:
        """
        Report multiple test results to TestRail.
        
        Args:
            results: List of test results to report
            
        Returns:
            Dictionary mapping test IDs to reporting success
        """
        success_dict = {}
        
        for result in results:
            success = self.report_result(result)
            success_dict[result.test_id] = success
            
        return success_dict
        
    def finalize(self) -> bool:
        """
        Finalize reporting to TestRail.
        
        Returns:
            True if finalization was successful, False otherwise
        """
        # Close the session
        if self._session:
            self._session.close()
            self._session = None
            
        return True
        
    def _extract_case_id(self, result: TestResult) -> Optional[int]:
        """
        Extract TestRail case ID from test result.
        
        Args:
            result: Test result
            
        Returns:
            TestRail case ID or None if not found
        """
        # Check if case ID is in metadata
        if 'testrail_case_id' in result.metadata:
            return int(result.metadata['testrail_case_id'])
            
        # Check if case ID is in markers
        if 'markers' in result.metadata:
            for marker in result.metadata['markers']:
                if marker.startswith('testrail'):
                    match = self._case_id_pattern.search(marker)
                    if match:
                        return int(match.group(1))
                        
        # Check if case ID is in test ID
        match = self._case_id_pattern.search(result.test_id)
        if match:
            return int(match.group(1))
            
        return None
        
    def _map_outcome_to_status_id(self, outcome: str) -> int:
        """
        Map pytest outcome to TestRail status ID.
        
        Args:
            outcome: pytest outcome
            
        Returns:
            TestRail status ID
        """
        if outcome == 'passed':
            return self.STATUS_PASSED
        elif outcome == 'failed':
            return self.STATUS_FAILED
        elif outcome == 'skipped':
            return self.STATUS_SKIPPED
        else:
            return self.STATUS_UNTESTED
            
    def _format_comment(self, result: TestResult) -> str:
        """
        Format a comment for TestRail.
        
        Args:
            result: Test result
            
        Returns:
            Formatted comment
        """
        comment = f"Test: {result.name}\n"
        comment += f"Duration: {result.duration:.2f}s\n"
        
        if result.message:
            comment += f"\nMessage:\n{result.message}\n"
            
        if result.traceback:
            comment += f"\nTraceback:\n{result.traceback}\n"
            
        return comment
        
    def _create_test_run(self, name: str, description: str) -> Optional[int]:
        """
        Create a new test run in TestRail.
        
        Args:
            name: Test run name
            description: Test run description
            
        Returns:
            Test run ID or None if creation failed
        """
        url = f"{self._url}/index.php?/api/v2/add_run/{self._project_id}"
        data = {
            'suite_id': self._suite_id,
            'name': name,
            'description': description,
            'include_all': True
        }
        
        try:
            response = self._session.post(url, json=data)
            response.raise_for_status()
            return response.json()['id']
        except Exception:
            return None
            
    def _add_test_result(self, case_id: int, status_id: int, comment: str, duration: float) -> bool:
        """
        Add a test result to TestRail.
        
        Args:
            case_id: TestRail case ID
            status_id: TestRail status ID
            comment: Test result comment
            duration: Test duration in seconds
            
        Returns:
            True if result was added successfully, False otherwise
        """
        url = f"{self._url}/index.php?/api/v2/add_result_for_case/{self._run_id}/{case_id}"
        data = {
            'status_id': status_id,
            'comment': comment,
            'elapsed': self._format_elapsed_time(duration)
        }
        
        try:
            response = self._session.post(url, json=data)
            response.raise_for_status()
            return True
        except Exception:
            return False
            
    def _format_elapsed_time(self, duration: float) -> str:
        """
        Format elapsed time for TestRail.
        
        Args:
            duration: Duration in seconds
            
        Returns:
            Formatted elapsed time (e.g., "1m 30s")
        """
        minutes = int(duration // 60)
        seconds = int(duration % 60)
        
        if minutes > 0:
            return f"{minutes}m {seconds}s"
        else:
            return f"{seconds}s"
```

### 2.2 TestRail Marker

```python
# src/cloudtestlib/markers.py (continued)

import pytest

def testrail(case_id):
    """
    Mark a test with a TestRail case ID.
    
    Args:
        case_id: TestRail case ID
        
    Returns:
        pytest.mark: A pytest mark for the TestRail case
    """
    return pytest.mark.testrail(f"C{case_id}")
```

## 3. Zephyr Integration

### 3.1 Zephyr Reporter Implementation

```python
# src/cloudtestlib/reporting/zephyr.py

import re
import requests
import json
from typing import Dict, List, Optional, Tuple, Union
from cloudtestlib.reporting.base import ExternalReporter, TestResult

class ZephyrReporter(ExternalReporter):
    """
    Reporter for Zephyr Scale (formerly Zephyr for Jira) integration.
    """
    
    # Zephyr status mappings
    STATUS_PASS = "Pass"
    STATUS_FAIL = "Fail"
    STATUS_WIP = "Work in progress"
    STATUS_BLOCKED = "Blocked"
    STATUS_UNEXECUTED = "Unexecuted"
    
    def __init__(self):
        """Initialize the Zephyr reporter."""
        self._url = None
        self._api_key = None
        self._project_key = None
        self._cycle_key = None
        self._folder_key = None
        self._test_key_pattern = re.compile(r'([A-Z]+-\d+)')
        self._session = None
        
    def initialize(self, **kwargs) -> bool:
        """
        Initialize the Zephyr reporter.
        
        Args:
            url: Jira URL
            api_key: Zephyr API key
            project_key: Jira project key
            cycle_key: Zephyr test cycle key
            folder_key: Zephyr folder key (optional)
            cycle_name: Name for a new test cycle (if cycle_key not provided)
            
        Returns:
            True if initialization was successful, False otherwise
        """
        self._url = kwargs.get('url')
        self._api_key = kwargs.get('api_key')
        self._project_key = kwargs.get('project_key')
        self._cycle_key = kwargs.get('cycle_key')
        self._folder_key = kwargs.get('folder_key')
        
        if not all([self._url, self._api_key, self._project_key]):
            return False
            
        # Initialize session
        self._session = requests.Session()
        self._session.headers.update({
            'Content-Type': 'application/json',
            'Authorization': f"Bearer {self._api_key}"
        })
        
        # Create a new test cycle if cycle_key not provided
        if not self._cycle_key:
            cycle_name = kwargs.get('cycle_name', 'Automated Test Cycle')
            self._cycle_key = self._create_test_cycle(cycle_name)
            
        return self._cycle_key is not None
        
    def report_result(self, result: TestResult) -> bool:
        """
        Report a test result to Zephyr.
        
        Args:
            result: Test result to report
            
        Returns:
            True if reporting was successful, False otherwise
        """
        test_key = self._extract_test_key(result)
        if not test_key:
            return False
            
        status = self._map_outcome_to_status(result.outcome)
        comment = self._format_comment(result)
        
        return self._add_test_execution(test_key, status, comment)
        
    def report_results(self, results: List[TestResult]) -> Dict[str, bool]:
        """
        Report multiple test results to Zephyr.
        
        Args:
            results: List of test results to report
            
        Returns:
            Dictionary mapping test IDs to reporting success
        """
        success_dict = {}
        
        for result in results:
            success = self.report_result(result)
            success_dict[result.test_id] = success
            
        return success_dict
        
    def finalize(self) -> bool:
        """
        Finalize reporting to Zephyr.
        
        Returns:
            True if finalization was successful, False otherwise
        """
        # Close the session
        if self._session:
            self._session.close()
            self._session = None
            
        return True
        
    def _extract_test_key(self, result: TestResult) -> Optional[str]:
        """
        Extract Jira test key from test result.
        
        Args:
            result: Test result
            
        Returns:
            Jira test key or None if not found
        """
        # Check if test key is in metadata
        if 'zephyr_test_key' in result.metadata:
            return result.metadata['zephyr_test_key']
            
        # Check if test key is in markers
        if 'markers' in result.metadata:
            for marker in result.metadata['markers']:
                if marker.startswith('zephyr'):
                    match = self._test_key_pattern.search(marker)
                    if match:
                        return match.group(1)
                        
        # Check if test key is in test ID
        match = self._test_key_pattern.search(result.test_id)
        if match:
            return match.group(1)
            
        return None
        
    def _map_outcome_to_status(self, outcome: str) -> str:
        """
        Map pytest outcome to Zephyr status.
        
        Args:
            outcome: pytest outcome
            
        Returns:
            Zephyr status
        """
        if outcome == 'passed':
            return self.STATUS_PASS
        elif outcome == 'failed':
            return self.STATUS_FAIL
        elif outcome == 'skipped':
            return self.STATUS_BLOCKED
        else:
            return self.STATUS_UNEXECUTED
            
    def _format_comment(self, result: TestResult) -> str:
        """
        Format a comment for Zephyr.
        
        Args:
            result: Test result
            
        Returns:
            Formatted comment
        """
        comment = f"Test: {result.name}\n"
        comment += f"Duration: {result.duration:.2f}s\n"
        
        if result.message:
            comment += f"\nMessage:\n{result.message}\n"
            
        if result.traceback:
            comment += f"\nTraceback:\n{result.traceback}\n"
            
        return comment
        
    def _create_test_cycle(self, name: str) -> Optional[str]:
        """
        Create a new test cycle in Zephyr.
        
        Args:
            name: Test cycle name
            
        Returns:
            Test cycle key or None if creation failed
        """
        url = f"{self._url}/rest/api/2/cycle"
        data = {
            'projectKey': self._project_key,
            'name': name,
            'description': 'Created by cloudtestlib'
        }
        
        try:
            response = self._session.post(url, json=data)
            response.raise_for_status()
            return response.json()['key']
        except Exception:
            return None
            
    def _add_test_execution(self, test_key: str, status: str, comment: str) -> bool:
        """
        Add a test execution to Zephyr.
        
        Args:
            test_key: Jira test key
            status: Zephyr status
            comment: Test result comment
            
        Returns:
            True if execution was added successfully, False otherwise
        """
        url = f"{self._url}/rest/api/2/execution"
        data = {
            'projectKey': self._project_key,
            'testKey': test_key,
            'cycleKey': self._cycle_key,
            'status': status,
            'comment': comment
        }
        
        if self._folder_key:
            data['folderKey'] = self._folder_key
            
        try:
            response = self._session.post(url, json=data)
            response.raise_for_status()
            return True
        except Exception:
            return False
```

### 3.2 Zephyr Marker

```python
# src/cloudtestlib/markers.py (continued)

import pytest

def zephyr(test_key):
    """
    Mark a test with a Zephyr test key.
    
    Args:
        test_key: Zephyr test key (e.g., "PROJ-123")
        
    Returns:
        pytest.mark: A pytest mark for the Zephyr test
    """
    return pytest.mark.zephyr(test_key)
```

## 4. Allure Integration

### 4.1 Allure Reporter Implementation

```python
# src/cloudtestlib/reporting/allure.py

import os
import json
import uuid
from datetime import datetime
from typing import Dict, List, Optional
from cloudtestlib.reporting.base import ExternalReporter, TestResult

class AllureReporter(ExternalReporter):
    """
    Reporter for Allure integration.
    """
    
    def __init__(self):
        """Initialize the Allure reporter."""
        self._results_dir = None
        self._test_containers = {}
        
    def initialize(self, **kwargs) -> bool:
        """
        Initialize the Allure reporter.
        
        Args:
            results_dir: Directory to store Allure results
            
        Returns:
            True if initialization was successful, False otherwise
        """
        self._results_dir = kwargs.get('results_dir')
        
        if not self._results_dir:
            return False
            
        # Create results directory if it doesn't exist
        os.makedirs(self._results_dir, exist_ok=True)
        
        return True
        
    def report_result(self, result: TestResult) -> bool:
        """
        Report a test result to Allure.
        
        Args:
            result: Test result to report
            
        Returns:
            True if reporting was successful, False otherwise
        """
        try:
            # Create test container if it doesn't exist
            if result.test_id not in self._test_containers:
                container_uuid = str(uuid.uuid4())
                self._test_containers[result.test_id] = container_uuid
                self._write_test_container(container_uuid, result)
                
            # Write test result
            self._write_test_result(result)
            
            return True
        except Exception:
            return False
        
    def report_results(self, results: List[TestResult]) -> Dict[str, bool]:
        """
        Report multiple test results to Allure.
        
        Args:
            results: List of test results to report
            
        Returns:
            Dictionary mapping test IDs to reporting success
        """
        success_dict = {}
        
        for result in results:
            success = self.report_result(result)
            success_dict[result.test_id] = success
            
        return success_dict
        
    def finalize(self) -> bool:
        """
        Finalize reporting to Allure.
        
        Returns:
            True if finalization was successful, False otherwise
        """
        # Nothing to do for Allure
        return True
        
    def _write_test_container(self, container_uuid: str, result: TestResult) -> None:
        """
        Write a test container to the Allure results directory.
        
        Args:
            container_uuid: Container UUID
            result: Test result
        """
        container = {
            'uuid': container_uuid,
            'name': result.name,
            'children': [str(uuid.uuid4())],
            'befores': [],
            'afters': [],
            'start': int(datetime.now().timestamp() * 1000),
        }
        
        file_path = os.path.join(self._results_dir, f"{container_uuid}-container.json")
        with open(file_path, 'w') as f:
            json.dump(container, f)
            
    def _write_test_result(self, result: TestResult) -> None:
        """
        Write a test result to the Allure results directory.
        
        Args:
            result: Test result
        """
        test_uuid = str(uuid.uuid4())
        container_uuid = self._test_containers[result.test_id]
        
        # Convert pytest outcome to Allure status
        if result.outcome == 'passed':
            status = 'passed'
        elif result.outcome == 'failed':
            status = 'failed'
        elif result.outcome == 'skipped':
            status = 'skipped'
        else:
            status = 'broken'
            
        # Create test result
        test_result = {
            'uuid': test_uuid,
            'historyId': result.test_id,
            'name': result.name,
            'fullName': result.test_id,
            'status': status,
            'start': int((datetime.now().timestamp() - result.duration) * 1000),
            'stop': int(datetime.now().timestamp() * 1000),
            'labels': self._create_labels(result),
            'links': [],
        }
        
        # Add failure details
        if result.outcome == 'failed':
            test_result['statusDetails'] = {
                'message': result.message or 'Test failed',
                'trace': result.traceback or '',
            }
            
        # Write test result to file
        file_path = os.path.join(self._results_dir, f"{test_uuid}-result.json")
        with open(file_path, 'w') as f:
            json.dump(test_result, f)
            
    def _create_labels(self, result: TestResult) -> List[Dict[str, str]]:
        """
        Create Allure labels from test result.
        
        Args:
            result: Test result
            
        Returns:
            List of Allure labels
        """
        labels = [
            {'name': 'framework', 'value': 'pytest'},
            {'name': 'language', 'value': 'python'},
        ]
        
        # Add markers as labels
        if 'markers' in result.metadata:
            for marker in result.metadata['markers']:
                if marker in ['aws', 'azure', 'gcp', 'kubernetes']:
                    labels.append({'name': 'provider', 'value': marker})
                elif marker in ['compute', 'storage', 'database', 'serverless', 'network']:
                    labels.append({'name': 'service', 'value': marker})
                elif marker in ['unit', 'integration']:
                    labels.append({'name': 'testType', 'value': marker})
                    
        return labels
```

## 5. PyTest Plugin Integration

### 5.1 PyTest Hooks

```python
# src/cloudtestlib/plugins/reporting_plugin.py

import pytest
from typing import Dict, List, Optional, Set
from cloudtestlib.reporting.base import TestResult
from cloudtestlib.reporting.manager import ReportingManager
from cloudtestlib.reporting.testrail import TestRailReporter
from cloudtestlib.reporting.zephyr import ZephyrReporter
from cloudtestlib.reporting.allure import AllureReporter
from cloudtestlib.reporting.factory import ReporterFactory

class ReportingPlugin:
    """
    PyTest plugin for external test reporting.
    """
    
    def __init__(self):
        """Initialize the reporting plugin."""
        self.manager = ReportingManager()
        self.results = []
        
    def pytest_configure(self, config):
        """
        Configure the plugin.
        
        Args:
            config: PyTest config
        """
        # Register reporters
        ReporterFactory.register_reporter('testrail', TestRailReporter)
        ReporterFactory.register_reporter('zephyr', ZephyrReporter)
        ReporterFactory.register_reporter('allure', AllureReporter)
        
        # Add reporters based on command-line options
        if config.getoption('--testrail'):
            self.manager.create_and_add_reporter('testrail', **self._get_testrail_options(config))
            
        if config.getoption('--zephyr'):
            self.manager.create_and_add_reporter('zephyr', **self._get_zephyr_options(config))
            
        if config.getoption('--allure'):
            self.manager.create_and_add_reporter('allure', **self._get_allure_options(config))
            
    def pytest_runtest_logreport(self, report):
        """
        Process test reports.
        
        Args:
            report: PyTest report
        """
        # Only process call reports (not setup/teardown)
        if report.when == 'call' or (report.when == 'setup' and report.skipped):
            # Create test result
            result = TestResult.from_pytest_report(report)
            
            # Store result
            self.results.append(result)
            
            # Report result immediately if real-time reporting is enabled
            if pytest.config.getoption('--report-immediately', False):
                self.manager.report_result(result)
                
    def pytest_terminal_summary(self, terminalreporter):
        """
        Add reporting summary to terminal output.
        
        Args:
            terminalreporter: Terminal reporter
        """
        if not self.results:
            return
            
        # Report all results
        report_results = self.manager.report_results(self.results)
        
        # Finalize reporting
        finalize_results = self.manager.finalize_all()
        
        # Print summary
        terminalreporter.write_sep('=', 'External Test Reporting Summary')
        
        for reporter_id, success_dict in report_results.items():
            success_count = sum(1 for success in success_dict.values() if success)
            total_count = len(success_dict)
            
            terminalreporter.write_line(
                f"{reporter_id}: Reported {success_count}/{total_count} tests successfully"
            )
            
    def _get_testrail_options(self, config) -> Dict:
        """
        Get TestRail options from config.
        
        Args:
            config: PyTest config
            
        Returns:
            Dictionary of TestRail options
        """
        return {
            'url': config.getoption('--testrail-url'),
            'username': config.getoption('--testrail-username'),
            'password': config.getoption('--testrail-password'),
            'project_id': config.getoption('--testrail-project-id'),
            'suite_id': config.getoption('--testrail-suite-id'),
            'run_id': config.getoption('--testrail-run-id'),
            'run_name': config.getoption('--testrail-run-name'),
        }
        
    def _get_zephyr_options(self, config) -> Dict:
        """
        Get Zephyr options from config.
        
        Args:
            config: PyTest config
            
        Returns:
            Dictionary of Zephyr options
        """
        return {
            'url': config.getoption('--zephyr-url'),
            'api_key': config.getoption('--zephyr-api-key'),
            'project_key': config.getoption('--zephyr-project-key'),
            'cycle_key': config.getoption('--zephyr-cycle-key'),
            'folder_key': config.getoption('--zephyr-folder-key'),
            'cycle_name': config.getoption('--zephyr-cycle-name'),
        }
        
    def _get_allure_options(self, config) -> Dict:
        """
        Get Allure options from config.
        
        Args:
            config: PyTest config
            
        Returns:
            Dictionary of Allure options
        """
        return {
            'results_dir': config.getoption('--allure-results-dir'),
        }

def pytest_addoption(parser):
    """
    Add command-line options for reporting.
    
    Args:
        parser: PyTest parser
    """
    group = parser.getgroup('reporting', 'External test reporting')
    
    # General options
    group.addoption(
        '--report-immediately',
        action='store_true',
        help='Report test results immediately after each test'
    )
    
    # TestRail options
    group.addoption('--testrail', action='store_true', help='Enable TestRail reporting')
    group.addoption('--testrail-url', help='TestRail URL')
    group.addoption('--testrail-username', help='TestRail username')
    group.addoption('--testrail-password', help='TestRail password or API key')
    group.addoption('--testrail-project-id', help='TestRail project ID')
    group.addoption('--testrail-suite-id', help='TestRail test suite ID')
    group.addoption('--testrail-run-id', help='TestRail test run ID')
    group.addoption('--testrail-run-name', help='Name for a new test run')
    
    # Zephyr options
    group.addoption('--zephyr', action='store_true', help='Enable Zephyr reporting')
    group.addoption('--zephyr-url', help='Jira URL')
    group.addoption('--zephyr-api-key', help='Zephyr API key')
    group.addoption('--zephyr-project-key', help='Jira project key')
    group.addoption('--zephyr-cycle-key', help='Zephyr test cycle key')
    group.addoption('--zephyr-folder-key', help='Zephyr folder key')
    group.addoption('--zephyr-cycle-name', help='Name for a new test cycle')
    
    # Allure options
    group.addoption('--allure', action='store_true', help='Enable Allure reporting')
    group.addoption('--allure-results-dir', help='Directory to store Allure results')

def pytest_configure(config):
    """
    Configure the plugin.
    
    Args:
        config: PyTest config
    """
    # Register the plugin
    plugin = ReportingPlugin()
    config.pluginmanager.register(plugin, 'reporting_plugin')
    
    # Register markers
    config.addinivalue_line('markers', 'testrail(case_id): Mark test with TestRail case ID')
    config.addinivalue_line('markers', 'zephyr(test_key): Mark test with Zephyr test key')
```

## 6. Usage Examples

### 6.1 TestRail Example

```python
# tests/examples/test_testrail_example.py

import pytest
from cloudtestlib.markers import testrail, aws, compute

@testrail(12345)  # TestRail case ID
@aws
@compute
def test_ec2_instance_creation():
    """Test EC2 instance creation."""
    # Test code here
    assert True
    
@testrail(12346)  # TestRail case ID
@aws
@compute
def test_ec2_instance_termination():
    """Test EC2 instance termination."""
    # Test code here
    assert True
```

### 6.2 Zephyr Example

```python
# tests/examples/test_zephyr_example.py

import pytest
from cloudtestlib.markers import zephyr, azure, storage

@zephyr("CLOUD-123")  # Zephyr test key
@azure
@storage
def test_blob_storage_upload():
    """Test Azure Blob Storage upload."""
    # Test code here
    assert True
    
@zephyr("CLOUD-124")  # Zephyr test key
@azure
@storage
def test_blob_storage_download():
    """Test Azure Blob Storage download."""
    # Test code here
    assert True
```

### 6.3 Combined Example

```python
# tests/examples/test_combined_example.py

import pytest
from cloudtestlib.markers import testrail, zephyr, gcp, database

@testrail(12347)  # TestRail case ID
@zephyr("CLOUD-125")  # Zephyr test key
@gcp
@database
def test_cloud_sql_creation():
    """Test Cloud SQL instance creation."""
    # Test code here
    assert True
    
@testrail(12348)  # TestRail case ID
@zephyr("CLOUD-126")  # Zephyr test key
@gcp
@database
def test_cloud_sql_backup():
    """Test Cloud SQL backup."""
    # Test code here
    assert True
```

## 7. Command-Line Usage

```bash
# Report to TestRail
pytest tests/ --testrail --testrail-url=https://example.testrail.io --testrail-username=user --testrail-password=pass --testrail-project-id=1 --testrail-suite-id=2 --testrail-run-name="Automated Test Run"

# Report to Zephyr
pytest tests/ --zephyr --zephyr-url=https://example.atlassian.net --zephyr-api-key=api_key --zephyr-project-key=CLOUD --zephyr-cycle-name="Automated Test Cycle"

# Report to Allure
pytest tests/ --allure --allure-results-dir=./allure-results

# Report to multiple systems
pytest tests/ --testrail --testrail-url=... --zephyr --zephyr-url=... --allure --allure-results-dir=...

# Report immediately after each test
pytest tests/ --testrail --testrail-url=... --report-immediately
```

## 8. Implementation Plan

### 8.1 Phase 1: Core Reporting Framework
1. Implement TestResult class
2. Implement ExternalReporter interface
3. Implement ReporterFactory
4. Implement ReportingManager

### 8.2 Phase 2: TestRail Integration
1. Implement TestRailReporter
2. Implement testrail marker
3. Add TestRail command-line options
4. Create TestRail usage examples

### 8.3 Phase 3: Zephyr Integration
1. Implement ZephyrReporter
2. Implement zephyr marker
3. Add Zephyr command-line options
4. Create Zephyr usage examples

### 8.4 Phase 4: Allure Integration
1. Implement AllureReporter
2. Add Allure command-line options
3. Create Allure usage examples

### 8.5 Phase 5: PyTest Plugin Integration
1. Implement ReportingPlugin
2. Add pytest_addoption hook
3. Add pytest_configure hook
4. Add pytest_runtest_logreport hook
5. Add pytest_terminal_summary hook

### 8.6 Phase 6: Testing and Documentation
1. Test all reporters
2. Create comprehensive documentation
3. Add usage examples
4. Document command-line options
