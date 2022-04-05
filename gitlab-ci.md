# 🦊 GitLab CI

## Overview
- [Caching Dependencies with GitLab](#caching-dependencies-with-gitlab)
    - [Utilising Artifacts and Dependencies](#utilising-artifacts-and-dependencies)
    - [Caching Dependencies on a Per-Branch Basis](#caching-dependencies-on-a-per-branch-basis)
- [Integration Testing with a Postgres Test Database](#integration-testing-with-a-postgres-test-database)

## Caching Dependencies with GitLab
Running `npm install` can take a long time, especially if you are installing a lot of packages. If a pipeline has to do this every time, it can slow down the build and deploy process as a whole.

```yaml
stages:
    - build
    - test
    - deploy

build:website:
    stage: build
    script:
        - npm install
        - npm run build

test:website:
    stage: test
    script:
        - npm install
        - npm run test

lint:website:
    stage: test
    script:
        - npm install
        - npm run lint
```
### Utilising Artifacts and Dependencies
We can optimise this by pre-installing the dependencies before the build stage, passing the `node_modules` folder down to the stages that require dependencies installed.

```yaml
stages:
    - install
    - build
    - test
    - deploy

install:website:
    stage: install
    script:
        - npm install
    artifacts:
        paths:
            - node_modules

build:website:
    stage: build
    script:
        - npm run build
    dependencies:
        - install:website

test:website:
    stage: test
    script:
        - npm run test
    dependencies:
        - install:website

lint:website:
    stage: test
    script:
        - npm run lint
    dependencies:
        - install:website
```

This approach can help reduce the number of `npm install` commands that need to be run to just once per commit, and can also reduce the total time taken to run the build and test stages. This is done by first installing all dependencies during the `install:module` job, and then passing the `node_modules` folder down to the other jobs. The `node_modules` artifact will remain until it [expires](https://docs.gitlab.com/ee/user/admin_area/settings/continuous_integration.html#default-artifacts-expiration) but it will only be accessible for the pipeline at that commit e.g for re-running the `build` and `test` stages of that commit at a later date.

### Caching Dependencies on a Per-Branch Basis

Sometimes, the number of dependencies installed can cause the time taken to upload the artifact to GitLab and then download it each commit can be much longer than the time taken to individually run `npm install` for each commit.

Generally speaking, most commits will rarely update the dependencies of a repository. However, if a feature branch does require installation of a new dependency, this change should be reflected immediately in the artifact. Luckily, it is possible to employ a per-branch caching strategy to reduce the time taken to install the dependencies as a whole when developing on a branch.

```yaml
stages:
    - install
    - build
    - test
    - deploy

cache-dependencies:
  stage: install
  cache: 
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules
  script:
    - npm ci
  only:
    changes:
      - package-lock.json
      - package.json

build:website:
    stage: build
    script:
        - npm run build
    cache: 
        key: $CI_COMMIT_REF_SLUG
        paths:
            - node_modules
        policy: pull

# If the rest of your pipeline uses the same dependencies, you can define this at the top-level rather than in each job.
# cache: 
#     key: $CI_COMMIT_REF_SLUG
#     paths:
#         - node_modules/
#     policy: pull

test:website:
    stage: test
    script:
        - npm run test
    cache: 
        key: $CI_COMMIT_REF_SLUG
        paths:
            - node_modules
        policy: pull

lint:website:
    stage: test
    script:
        - npm run lint
    cache: 
        key: $CI_COMMIT_REF_SLUG
        paths:
            - node_modules
        policy: pull
```

When the `package.json` or `package-lock.json` file is changed, this means that a dependency has either been added, removed or updated. The `cache-dependencies` job will only run when a commit contains such a change within a branch.

It will create a cache keyed under the branch name using `$CI_COMMIT_REF_SLUG` for the `node_modules` path. Other jobs will be able to use this key to pull down the cached dependencies by specifying `cache:key` as `$CI_COMMIT_REF_SLUG`. To ensure that the cache is only downloaded for usage and time is not wasted by the other jobs re-updating the cache themselves with a duplicate, `cache:policy` is set to `pull`.

Though more time will be spent installing the dependencies and preparing the on the first commit, subsequent commits will be much faster as the cache is pulled down.

A pipeline I worked on that previously ran at a consistent `9 minutes` always re-installing dependencies was able to be brought down to just under `5 minutes` on each cached commit. The time spent for the initial commit whenever a package was changed was around `11 minutes`, so the benefits were very much immediate after 2 commits on a new branch.

[🔗 Source: How to Speed Up Your GitLab CI Pipelines for Node Apps by 40%](https://www.addthis.com/blog/2019/05/06/how-to-speed-up-your-gitlab-ci-pipelines-for-node-apps-by-40/#.YRYSGC2ZPEY)

## Integration Testing with a Postgres Test Database
Integration testing can ensure that your application is configured correctly with the database as it is used with different queries.

For example, to check that [Prisma](https://prisma.io/) is correctly configured to a Postgres database, you may use integration testing that does not mock database connections. You can then run your mutations against your database to test for validation checks. As you don't want to use an actual Postgres database due to the cost, you can use a test database that is created and used just for testing within GitLab CI.

```yaml
# Create a test postgres database 
services:
    - postgres:latest
    - ...

# Set database variables for connection
variables:
    POSTGRES_DB: postgres
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: password
    POSTGRES_HOST: postgres

test:
    stage: test
    variables:
        # Environment variable to set in Prisma for the location of the database to use
        DATABASE_URL: postgres://${POSTGRES_HOST}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}/${POSTGRES_DB}
    script:
        - ...
```

[🔗 Source: Using PostgreSQL | GitLab](https://docs.gitlab.com/ee/ci/services/postgres.html)