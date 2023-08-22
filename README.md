# PGWeb Testing Harness Suite - Documentation

This is the project report for the PGWeb Testing Harness Suite Developed for the organisation PostgreSQL.

## About Project
This project is an automated testing harness suite for the <a href="https://www.postgresql.org">official PostgreSQL website</a>. Previously, many bugs were encountered whenever new changes were introduced to the website. These bugs had to be manually debugged, which could take hours of trial and error to identify the root cause. To address this problem, this testing suite was developed under the Google Summer of Code 2023 program. 

### How does it work?

While developing this suite, we had three major requirements:
 - Keep the testing suite codebase completely seperate from the <a href="https://git.postgresql.org/gitweb/?p=pgweb.git;a=summary">pgweb codebase</a>
 - The test suite should automatically detect any changes in the pgweb codebase and trigger the tests without the need for any human intervention.
 - Test suite should be developed in such a way that it complements the way new modules are added to the main website.
 - Proper Error reporting mechanisms.

Let's get into each of these points!

#### Seperating the codebase
The testing harness codebase has been hosted on GitHub and it runs entirely within GitHub actions. To write comprehensive tests for the website we decided to go with the default <a href="https://docs.djangoproject.com/en/4.2/topics/testing/overview/">TestCase class</a> provided in Django. This allows us to utilize the full benefits of <a href="https://www.selenium.dev/">Selenium</a> in a test build of the website instead of manually spinning up a server instance and then testing via bruteforce.

In the action workflow run, the pgweb repository  is cloned first and the test folders from the testing harness are copied to the cloned repository. After this all the tests aer run using the default Django commands. To get a closer look on this process checkout <a href="https://github.com/destrex271/pgweb-testing-harness/blob/main/src/workflow_utils/setup.sh">this file</a>.

#### Pgweb repository monitoring mechanism
To check for any changes in the main website, a seperate GitHub Actions workflow has been setup <a href="https://github.com/destrex271/pgweb-testing-harness/blob/main/.github/workflows/monitor.yml">here</a>.

This workflow runs as a cronjob every six hours i.e. checks the main repository 4 times a day. Since the main website does not recieve any frequent updates we decided to keep the cronjob scheduled in this way.
As this workflow runs, it extracts the last commit Id from the repository and checks it against the previous commit Id which is stored in the commit_id.txt file on the main repository. If the commit Id is different, the testing suite is triggered and the value of the prev commit is updated in the commit_id.txt file.

This sophisticated mechanism had to be taken since we wanted to make little to no changes in the main repository and use GitHub actions to keep everything at the same place.

#### 'LEGO' structure to easily add new tests

All the tests have been seperated in folders pertaining to their categories. For now functional_tests houses all the Functionality related tests for the website. 
To add any new tests that use django you can use the following template:
NAME SCHEME FOR FILES : tests_<name or purpose of test>
```
from selenium.webdriver.common.by import By
from selenium.webdriver.firefox.service import Service
from webdriver_manager.firefox import GeckoDriverManager
from selenium import webdriver

from django.test.testcases import call_command, connections
from django.contrib.staticfiles.testing import LiveServerTestCase
import random

# Utility functions
from .extra_utils.util_functions import create_permitted_user_with_org_email, generate_session_cookie, varnish_cache

# Any required models from pgweb codebase
# Kindly follow this import scheme strictly to avoid any errors in test runs
from .app.models import ModelClass


# any required constants


# Mandatory code to run tests with postgresql to avoid Contraint errors
# Fix for CASCADE TRUNCATE FK error


def _fixture_teardown(self):
    # Allow TRUNCATE ... CASCADE and don't emit the post_migrate signal
    # when flushing only a subset of the apps
    for db_name in self._databases_names(include_mirrors=False):
        # Flush the database
        inhibit_post_migrate = (
            self.available_apps is not None
            or (  # Inhibit the post_migrate signal when using serialized
                # rollback to avoid trying to recreate the serialized data.
                self.serialized_rollback
                and hasattr(connections[db_name], "_test_serialized_contents")
            )
        )
        call_command(
            "flush",
            verbosity=0,
            interactive=False,
            database=db_name,
            reset_sequences=False,
            # In the real TransactionTestCase this is conditionally set to False.
            allow_cascade=True,
            inhibit_post_migrate=inhibit_post_migrate,
        )


LiveServerTestCase._fixture_teardown = _fixture_teardown
# ---------------------------



# Main test class begins
class <ItemToBeTested>Tests(LiveServerTestCase):

    fixtures = ['pgweb/fixtures/users.json',
                'pgweb/fixtures/org_types.json', 'pgweb/fixtures/organisations.json', 'pgweb/downloads/fixtures/data.json']

    @ classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Required constants
        # . . . . .
        # . . . . .

        # Webdriver Configuration
        options = webdriver.FirefoxOptions()
        options.headless = True
        serv = Service(executable_path=GeckoDriverManager().install())
        cls.selenium = webdriver.Firefox(
            service=serv, options=options)

        # dummy data
        # . . . . . .

        # Call SQL for simulating local varnish cache
        varnish_cache()
        # DO NOT REMOVE -> Reuqired to simulate varnish cache

    @ classmethod
    def tearDownClass(cls) -> None:
        cls.selenium.quit()
        return super().tearDownClass()

    def test_<testname>(self):
        # Test body begins
        # You can use selenium as self.selenium here to
        # perform any tests
```

To add any tests that use bash scripts kindly add them to the required action workflow job. You can also setup new jobs if your tests take too long to run.

If you are generating any extra reports kindly add them in the workflow to be uploaded as artifacts.


#### Error Reporting Mechanism

Once the tests are run, the logs are analyzed by <a href="https://github.com/destrex271/pgweb-testing-harness/blob/main/src/utils/process_logs.py">this script</a>. If any tests are failed, an email is prepared with the respective reports and a link to the action run is also added in the content. This email is then to the concerned authorities.


## Features that were left/ Features that can be added later

- A Web interface to view individual reports for each build.
- Persistence of artifacts by storing them in a database or storage service to create a log archive.
- Crawling service to check the website regualarly for any expired/broken links.
- More tests for other processes that are run on each build of the website.

## Contributing to the project
Tools required to run the testing harness locally:
The testing suite can also be run locally. To do so you need the following prerequisites:
- **act** - CLI Tool GitHub Actions Emulation. Installation source can be found [here](https://github.com/nektos/act).
- **Docker** - Required by act to build containers.Installation source can be found [here](https://docs.docker.com/engine/install/).

After you fork and clone the repository kindly install all the required packages using

`pip install -r requirements.txt`

**_We suggest that you do so in a virtual environment to avoid any unnecessary conflicts._**

To run the tests use:

`act -j run-tests`

The following prerequisites are required for contributing:
 - Python
 - Github Action(Just basic knowledge)
 - Bash Scripting
 - Django (basic idea about the framework)
 - Selenium and BeautifulSoup4

Kindly follow the template given above if you are writing tests w.r.t Djanfo TestCase class and follow the given rules:
- All tests files should be in the format **tests_*.py**
- New Tests should be added via a new branch with the name in format <test_type>/<test_name> where test type can be functional, accessibility and security.
- All the pull requests should be to the develop branch. **No pull request would be merged directly into main**.
- Kindly refrain from pushing any unwanted files and folders generated by your text editors and IDEs(for eg .idea/ .code/ etc..)


## Bi-Weekly reports for Coding period for additional details

- <a href="https://medium.com/dev-genius/week-1-2-at-postgresql-gsoc-2023-914c689984f3">Week 1 - 2</a>: CI/CD Integration & Base
- <a href="https://medium.com/dev-genius/week-3-4-postgresql-gsoc23-13b4bbb4c54a">Week 3 - 4</a>: Report Generation & Basic Form Tests
- <a href="https://medium.com/dev-genius/week-5-6-postgresql-google-summer-of-code23-d74ec1dff8f8">Week 5 - 6</a>: Document Render Tests
- <a href="https://medium.com/towardsdev/week-7-8-postgresql-gsoc23-ecf87803e9fd">Week 7 - 8</a>: Crawler Service 
- <a href="https://medium.com/@destrex271/week-9-10-postgresql-gsoc23-9e3fc29890e9">Week 9 - 10</a>: Url validation
- <a href="https://medium.com/@destrex271/week-11-12-postgresql-gsoc23-the-final-leg-119813aa121c">Week 11 - 12</a>: Accessibility Tests & Error Reporting