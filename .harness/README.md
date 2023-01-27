Build & Test GraphQL-JS on Harness CI
=======================================
This is a fork of GraphQL JS project. This project is used for testing Harness CI's capabilities by the CME team at Harness. This file contains instructions on how to run graphql/graphql-js on Harness CI.


- [Harness Fast CI Blog Announcement](https://harness.io/blog/announcing-speed-enhancements-and-hosted-builds-for-harness-ci)
- [Get Started with Harness CI](https://harness.io/products/continuous-integration)

## Setting up this pipeline on Harness CI Hosted Builds

1. Fork [this repository](https://github.com/harness-community/graphql-js) into your GitHub account. 

2. If you are new to Harness CI, signup for [Harness CI](https://app.harness.io/auth/#/signup)
 * Select the `Continuous Integration` module.
 * Follow the below steps to set up your pipeline:-
 
   1. Click on `Create Pipeline`. 
   2. Give a `Name` to your pipeline.
   3. Optionally, you can provide a `Description` and `Tags`
   4. Select `Remote` option on `How do you want to setup your pipeline`.
   5. Provide your `Git Connector`.
    
      - Select `Connectors` -> Click on `+ New Connector`
      - Connector Type: `Github`
      - Name: `reactor-core`
      - URL Type : `Repository`
      - Connection Type: `HTTP`
      - GitHub Repository URL: Paste the link of your forked repository
      - Username: (Your Github Username)
      - Personal Access Token: [Check out how to create personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
      - Secret Name: `Git-Token` 
      - Secret  Value: PAT value generated in Github
      - Click on `Save`.
      - This will allow the repository to be fetched click on it and click `Apply Selected`.
      - Make Enable API access (ON) with the secret token created
      - Click on `Connect through Harness Platform`.
      - To develop more understanding on Connectors check out the [docs](https://developer.harness.io/docs/platform/connectors/add-a-git-hub-connector/).

   6. Select your `Repository` here `reactor-core`.
   7. `Git Branch` will automatically be fetched. 
   8. Provide your `YAML Path` as `.harness/{PIPELINE_NAME}.yml`.The root folder `.harness` is required. 
   > _NOTE: Here `.harness/{PIPELINE_NAME}.yml`, {PIPELINE_NAME} is name of your `Pipeline`._
   9. Select `Start`.

3. Add the pipeline.yaml below `tags` in the YAML editor:

```
  properties:
    ci:
      codebase:
        connectorRef: git-connector
        repoName: graphql-js
        build: <+input>
  stages:
    - stage:
        name: ci
        identifier: ci
        description: ""
        type: CI
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  type: Run
                  name: Install Dependencies
                  identifier: Install_Dependencies
                  spec:
                    shell: Sh
                    command: npm ci --ignore-scripts
              - step:
                  type: Run
                  name: Lint Install and Tests
                  identifier: Lint_Install_and_Tests
                  spec:
                    shell: Sh
                    command: npm run <+matrix.deps>
                  failureStrategies: []
                  strategy:
                    matrix:
                      deps:
                        - lint
                        - check
                        - check:spelling
                      maxConcurrency: 1
    - stage:
        name: checks
        identifier: checks
        description: ""
        type: CI
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  type: Run
                  name: Check if commit contains files that should be ignored
                  identifier: Check_if_commit_contains_files_that_should_be_ignored
                  spec:
                    shell: Sh
                    command: |-
                      git clone --depth 1 https://github.com/github/gitignore.git
                                rm gitignore/Global/ModelSim.gitignore
                                rm gitignore/Global/Images.gitignore
                                cat gitignore/Node.gitignore gitignore/Global/*.gitignore > all.gitignore
                                IGNORED_FILES=$(git ls-files --cached --ignored --exclude-from=all.gitignore)
                                if  [[ "$IGNORED_FILES" != "" ]]; then
                                  echo -e "::error::Please remove these files:\n$IGNORED_FILES" | sed -z 's/\n/%0A/g'
                                  exit 1
                                fi
              - step:
                  type: Run
                  name: Run npm install
                  identifier: Run_npm_install
                  spec:
                    shell: Sh
                    command: npm install --ignore-scripts --force --package-lock-only --engine-strict --strict-peer-deps
              - step:
                  type: Run
                  name: package_lock sync with package
                  identifier: package_lock_sync_with_package
                  spec:
                    shell: Sh
                    command: |
                      git diff --exit-code package-lock.json
    - parallel:
        - stage:
            name: integration test
            identifier: integration_test
            description: ""
            type: CI
            spec:
              cloneCodebase: true
              platform:
                os: Linux
                arch: Amd64
              runtime:
                type: Cloud
                spec: {}
              execution:
                steps:
                  - step:
                      type: Run
                      name: Install Dependencies
                      identifier: Install_Dependencies
                      spec:
                        shell: Sh
                        command: |
                          npm ci --ignore-scripts
                  - step:
                      type: Run
                      name: Run Integration Tests
                      identifier: Run_Integration_Tests
                      spec:
                        shell: Sh
                        command: npm run check:integrations
        - stage:
            name: fuzzing tests
            identifier: fuzzing_tests
            description: ""
            type: CI
            spec:
              cloneCodebase: true
              platform:
                os: Linux
                arch: Amd64
              runtime:
                type: Cloud
                spec: {}
              execution:
                steps:
                  - step:
                      type: Run
                      name: Install Dependencies
                      identifier: Install_Dependencies
                      spec:
                        shell: Sh
                        command: npm ci --ignore-scripts
                  - step:
                      type: Run
                      name: Run Tests
                      identifier: Run_Tests
                      spec:
                        shell: Sh
                        command: |
                          npm run fuzzonly
        - stage:
            name: mutation test
            identifier: mutation_test
            description: ""
            type: CI
            spec:
              cloneCodebase: true
              platform:
                os: Linux
                arch: Amd64
              runtime:
                type: Cloud
                spec: {}
              execution:
                steps:
                  - step:
                      type: Run
                      name: Install Dependencies
                      identifier: Install_Dependencies
                      spec:
                        shell: Sh
                        command: npm ci --ignore-scripts
                  - step:
                      type: Run
                      name: Run mutation testing
                      identifier: Run_mutation_testing
                      spec:
                        shell: Sh
                        command: |
                          npm run testonly:mutate
                        reports:
                          type: JUnit
                          spec:
                            paths:
                              - ./reports/mutation/mutation.html
                  - step:
                      type: GCSUpload
                      name: upload test results
                      identifier: upload_test_results_test
                      spec:
                        connectorRef: gcpconnector
                        bucket: graphql-js
                        sourcePath: reports/
    - stage:
        name: test node
        identifier: test_node
        description: ""
        type: CI
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  type: Run
                  name: Install Dependencies
                  identifier: Install_Dependencies
                  spec:
                    shell: Sh
                    command: npm ci --ignore-scripts
              - step:
                  type: Run
                  name: Run Tests with coverage
                  identifier: Run_Tests_with_coverage
                  spec:
                    shell: Sh
                    command: npm run testonly:cover
    - parallel:
        - stage:
            name: build artifact
            identifier: build_artifact
            type: CI
            spec:
              cloneCodebase: true
              platform:
                os: Linux
                arch: Amd64
              runtime:
                type: Cloud
                spec: {}
              execution:
                steps:
                  - step:
                      type: Run
                      name: Install Dependencies
                      identifier: Install_Dependencies
                      spec:
                        shell: Sh
                        command: |
                          npm ci --ignore-scripts
                  - step:
                      type: Run
                      name: Build NPM package
                      identifier: Build_NPM_package
                      spec:
                        shell: Sh
                        command: |
                          npm run build:npm
                  - step:
                      type: Run
                      name: Build Deno package
                      identifier: Build_Deno_package
                      spec:
                        shell: Sh
                        command: |
                          npm run build:deno
              sharedPaths:
                - /npmDist
        - stage:
            name: build website
            identifier: build_website
            description: ""
            type: CI
            spec:
              cloneCodebase: true
              platform:
                os: Linux
                arch: Amd64
              runtime:
                type: Cloud
                spec: {}
              execution:
                steps:
                  - step:
                      type: Run
                      name: Install Dependencies
                      identifier: Install_Dependencies
                      spec:
                        shell: Sh
                        command: npm ci --ignore-scripts
                  - step:
                      type: Run
                      name: Build Docs
                      identifier: Build_Docs
                      spec:
                        shell: Sh
                        command: npm run build:website
  allowStageExecutions: true
```
> _NOTE: Make sure you modify the connectors with the connectors you create._

4.  Go to the newly created pipeline and hit the `Triggers` tab. If everything went well, you should see two triggers auto-created. A `Pull Request` trigger and a `Push` trigger. For this exercise, we only need `Pull Request` trigger to be enabled.

5. Create a Pull Request in a new branch by updating the project. (e.g. add a comment or new line). This should invoke a built-in Harness CI

6. Merge the PR after the pipeline execution is successful.

7. Enable GitHub Actions: The repository forked in _**Step 2**_ already has a GitHub Actions workflow file added. You can choose to enable this workflow from the Actions tab on GitHub.

8. Create any other Pull Request with a few source or test file changes. You can consider cherry-picking any of the commits from the main repository.

9. This PR will trigger the Harness CI pipeline (as well as GitHub Actions workflow if enabled in Step-7). Here we are implementing multiple workflows for Github Actions. (CI + Mutation)
