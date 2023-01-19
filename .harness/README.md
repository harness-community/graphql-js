Build & Test GraphQL-JS on Harness CI
=======================================
This is a fork of GraphQL JS project. This project is used for testing Harness CI's capabilities by the CME team at Harness. Please follow [these instructions](https://github.com/harness-community/graphql-js/blob/trunk/.harness/README.md) on how to clone this repo and see the results yourself.


- [Harness Fast CI Blog Announcement](https://harness.io/blog/announcing-speed-enhancements-and-hosted-builds-for-harness-ci)
- [Get Started with Harness CI](https://harness.io/products/continuous-integration)

## Setting up this pipeline on Harness CI Hosted Builds

1. Create a [GitHub Account](https://github.com) or use an existing one

2. Fork [this repository](https://github.com/harness-community/graphql/graphql-js) into your GitHub account. 

3. If you are new to Harness CI, signup for [Harness CI](https://app.harness.io/auth/#/signup)
  * Select the `Continuous Integration` module and choose the `Starter pipeline` wizard to create your first pipeline using the forked repo from #2.
  * Go to the newly created pipeline and hit the `Triggers`tab. If everything went well, you should see two triggers auto-created. A `Pull Request`trigger and a `Push`trigger. For this exercise, we only need `Pull Request`trigger to be enabled. So, please disable or delete the `Push`trigger.

4. If you are an existing Harness CI user, create a new pipeline to use the cloud option for infrastructure and setup the PR trigger.

5. Add the pipeline.yaml stages in the YAML editor:

```
  properties:
    ci:
      codebase:
        connectorRef: dhrubaaccountconnector
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
> Make sure you modify the connectors with the connectors you create. 

6. Create a Pull Request in a new branch by updating the project. (e.g. add a comment or new line). This should invoke a build in Harness CI

7. Merge the PR after the pipeline execution is successfull.

8. Enable GitHub Actions : The repository forked in Step 2 already has a GitHub Actions workflow file added. You can choose to enable this workflow from the Actions tab on GitHub.

9. Create any other Pull Request with a few source or test file changes. You can consider cherry-picking any of the commits from the main repository.

10. This PR will trigger the Harness CI pipeline (as well as GitHub Actions workflow if enabled in Step-9). Here we are implementing multiple workflows for Github Actions. (CI + Mutation)
