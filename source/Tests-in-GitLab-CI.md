# Unit tests and Integration tests in GitLab CI

In this article, you can learn how to  

1. Include unit tests in your GitLab CI pipeline.
2. Include integration tests and external dependencies used in your integration test projects, such as PostgreSQL, in your GitLab CI pipeline.
3. Preview your test reports on GitLab.

## Prerequiste

1. test projects in .NET 6
2. [JunitXml.TestLogger](https://www.nuget.org/packages/JunitXml.TestLogger) in your test projects for test reports
3. A GitLab Runner
4. `.gitlab-ci.yml` for GitLab Runner

## Purpose

Integrating tests, including unit tests and integration tests, in your CI pipeline in GitLab.

## Configuration

Let me give you some examples first, and I will explain the details later.

In the directory, `NineYi.Portal.API.Tests` is a unit test project and `NineYi.Portal.Shell.IntegrationTest` is a integration test project.

```plain
$tree -a
.
├──.gitlab
│   └───test.yml
├──.gitlab-ci.yml
└──test
   ├───NineYi.Portal.API.Tests
   │   ├───NineYi.Portal.API.Tests.csproj
   │   └───snip...
   └───NineYi.Portal.Shell.IntegrationTest
       ├───NineYi.Portal.Shell.IntegrationTest.csproj
       └───snip...
```

In .gitlab-ci.yml

```yaml
stages:
  - unit-test

include:
  # run unit/integration test
  - local: .gitlab/test.yml
```

In `test.yml`, we have two [jobs](https://docs.gitlab.com/ee/ci/jobs/) (terminology in GitLab), one is for unit tests and the other one is for integration tests.  

First, let us talk about the details of job `unit_test` blocks by blocks.

- stage: stage name(here is `unit-test`)
- image: image used in GitLab runner.
- script: We will go to different directories which include `.Tests` at the end of the directory name. Then, executing each test project through the `dotnet test` command. Significantly, we have some special arguments in command, such as `--test-adapter-path` and `--logger`, and those arguments are for [test reports in .NET](https://docs.gitlab.com/ee/ci/testing/unit_test_report_examples.html#net).
- artifacts: You can store your test reports as artifacts in specific paths in GitLab. Thus, you can see your test reports on GitLab. Notice that GitLab would classify your test reports according to test reports file names.
- only: We only trigger our tests pipeline in GitLab CI when we send a Merge Request.

Second, let us talk about the details of job `integration_test` blocks by blocks.

- stage: stage name(here is `integration-test`)
- script: We will go to different directories which include `.IntegrationTest` at the end of the directory name. Then, execute each test project through the `dotnet test` command. Significantly, we have some special arguments in command, such as `--test-adapter-path` and `--logger`, and those arguments are for [test reports in .NET](https://docs.gitlab.com/ee/ci/testing/unit_test_report_examples.html#net).
- services: Because we rely on PostgreSQL in our integration test project, we have to set up an extra container for PostgreSQL. Here, we can prepare for PostgreSQL easily by using a feature provided by GitLab called [services](https://docs.gitlab.com/ee/ci/services). In services, we can set up different external dependencies, such as Redis, MySQL and PostgreSQL, by specifying the image name. We only use PostgreSQL here.
- variables: You can set up your own environment variables for GitLab runner, but here, we only set up [necessary environment variables](https://docs.gitlab.com/ee/ci/services/postgres.html) for PostgreSQL.

```yml
unit_test:
  stage: unit-test
  image: mcr.microsoft.com/dotnet/sdk:6.0 
  before_script: 
    - export TEST_HOME=$(pwd)
  script:
    - cd ${TEST_HOME}/test
    - |
      for foldername in *.Tests; do
          [ -e "${foldername}" ] || continue
          cd ./${foldername}
          dotnet test --test-adapter-path:. --logger:'"junit;LogFilePath=..\artifacts\'${foldername}'-unit-test-result.xml;MethodFormat=Class;FailureBodyFormat=Verbose"'
          cd ../
      done
  artifacts:
    when: always
    paths: 
      - ./test/artifacts/*unit-test-result.xml
    reports:
      junit: 
        - ./test/artifacts/*unit-test-result.xml
    expire_in: 1 week
  when: always
  only:
    refs:
      - merge_requests

integration_test:
  stage: integration-test
  services:
      - name: postgres:11.12
  variables:
    Host: postgres
    POSTGRES_DB: portal_shell
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: medusa
  image: mcr.microsoft.com/dotnet/sdk:6.0 
  before_script: 
    - export TEST_HOME=$(pwd)
  script:
    - cd ${TEST_HOME}/test
    - |
      for foldername in *.IntegrationTest; do
          [ -e "${foldername}" ] || continue
          cd ./${foldername}
          dotnet test --test-adapter-path:. --logger:'"junit;LogFilePath=..\artifacts\'${foldername}'-integration-test-result.xml;MethodFormat=Class;FailureBodyFormat=Verbose"'
          cd ../
      done
  artifacts:
    when: always
    paths: 
      - ./test/artifacts/*integration-test-result.xml
    reports:
      junit: 
        - ./test/artifacts/*integration-test-result.xml
    expire_in: 1 week
  when: always
  only:
    refs:
      - merge_requests
```

## Results

### CI Pipeline

![image](images/CI_pipeline_in_GitLab.PNG)

### Preview your test reports on GitLab

![image](images/test_reports.PNG)

## Conclusion

Still under contruction...

<!-- TODO: -->

## References

- [Test reports in .NET in GitLab](https://docs.gitlab.com/ee/ci/testing/unit_test_report_examples.html#net)
- [Services in GitLab](https://docs.gitlab.com/ee/ci/services/)
- [Using PostgreSQL](https://docs.gitlab.com/ee/ci/services/postgres.html)
