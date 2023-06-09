# Continuous Integration Workflow

## Task

- This exercise will setup a CI workflow to for the project from the second exercise:

  - GitHub will be the VCS
  - GitHub Actions will be the build tool
  - Sonarcloud will be used for static code analysis

- The workflow will execute the unit tests of the source code and build the code.
- The workflow triggers, when a change to the codebase has been performed

## GitHub Actions

The setup of the GitHub Actions workflow is based on the instructions for ["Building and testing Go"](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-go) and ["Creating PostgreSQL service containers"](https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers) from the GitHub Actions documentation.

1. Create a new file in the `.github/workflows` directory of the project with the name `go.yml`
2. Add the following content to the file:

```yaml
name: Go

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: 1.19

      - name: Install dependencies
        run: go get .

      - name: Build
        run: go build -v ./...

      - name: Test with the Go CLI
        run: go test
        env:
          APP_DB_USERNAME: postgres
          APP_DB_PASSWORD: postgres
          APP_DB_NAME: postgres

    services:
      postgres:
        image: postgres
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
```

This will start a PostgreSQL database container, setup Go and run the unit tests.

## SonarCloud

SonarCloud is a cloud-based code quality and security service. It can be used to analyze the code quality of the project and to detect bugs and vulnerabilities.

Sign up for [SonarCloud](https://sonarcloud.io/) using your GitHub account and create a new project for the repository.
Public projects should already start to analyze on new commits.

To configure manual analysis of the project, a `sonar-project.properties` file can be added to the root directory of the project. The file requires at least the following properties:

```properties
sonar.projectKey=<project name>
sonar.organization=<organization or username>
```

Make sure to also disable automatic analysis in the SonarCloud settings.

### Adding SonarCloud to the GitHub Actions workflow

In order to add SonarCloud checks to the GitHub Actions workflow, add the following step to the `build` job:

```yaml
- name: SonarCloud Scan
  uses: SonarSource/sonarcloud-github-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

To set the token navigate to "Settings > Secrets and variables > Actions > Secrets (Tab)" and add a new secret with the name `SONAR_TOKEN` and the value of the token from the SonarCloud project settings.

Further information on using SonarCloud with GitHub Actions can be found in the [SonarCloud documentation](https://docs.sonarcloud.io/getting-started/github/).
