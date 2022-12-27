# Unit tests and Integration tests in GitLab CI

## Prerequiste

1. test projects in .NET 6
2. [JunitXml.TestLogger](https://www.nuget.org/packages/JunitXml.TestLogger) in your test projects (for test reports)
3. A GitLab Runner
4. `.gitlab-ci.yml` for GitLab Runner

## Purpose

Integrating tests, including unit test and integration test, in your CI pipeline in GitLab.

## Configuration

Let me give you some examples first, and I will explain the details latter.

In directory, `NineYi.Portal.API.Tests` is unit test project and `NineYi.Portal.Shell.IntegrationTest` is integration test project.

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

In test.yml, we go to different...
<!-- TODO: -->

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
      - /^v\d+\.\d+\.\d+-dev-b\d+-\d+$/
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
  tags:
    - docker
    - linux
  before_script: 
    - export TEST_HOME=$(pwd)
    - tmp=$(mktemp)
    - cat "${TEST_HOME}/pipeline/NineYi.Portal.Shell/env/nineyi.portal.shell.integration.test.gitlab.env" > "$tmp" && mv "$tmp" "${TEST_HOME}/pipeline/NineYi.Portal.Shell/env/nineyi.portal.shell.integration.test.env"
  script:
    - dotnet restore --configfile "${TEST_HOME}/src/nuget.config"
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
      - /^v\d{1,3}\.\d{1,3}\.\d{1,3}-rel$/
      - merge_requests
```

## Results

## Conclusion