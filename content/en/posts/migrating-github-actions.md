---
title: "Migrating our CI to Github Actions"
date: 2019-12-16
tags: ["continuous integration", "github actions", "tourlane", "software development", "continuous deployment"]
comments: true
---

And how we did that in [Tourlane](https://www.tourlane.com/)

Now in Tourlane we are switching our deployment system to use GitHub Actions. So, in this post, we are sharing our experience with the tool so far, and how we are migrating our projects, little by little to the tool.

## What is CI/CD?

Martin Fowler, one of the original authors of the Agile Manifesto, describes them like this in [this book](https://martinfowler.com/books/duvall.html):

> Continuous Integration (CI) is a development practice that requires developers to integrate code into a shared repository several times a day. Each check-in is then verified by an automated build, allowing teams to detect problems early.

> Continuous Delivery (CD) is the natural extension of Continuous Integration: an approach in which teams ensure that every change to the system is releasable, and that we can release any version at the push of a button. Continuous Delivery aims to make releases boring, so we can deliver frequently and get fast feedback on what users care about.

##What are Github Actions

GitHub has just released a version of its own CI/CD tool, which is called GitHub Actions. More information can be found in [this blog post](https://github.blog/2019-08-08-github-actions-now-supports-ci-cd/). It helps you to:

> orchestrate any workflow, based on any event, while GitHub manages the execution, provides rich feedback, and secures every step along the way.

For Tourlane, it looked a quite convenient tool to use — We already use GitHub for repository storage and collaboration, thus Actions facilitates the efforts of integrating the tools.

![](https://media.giphy.com/media/5r5J4JD9miis/giphy.gif)

## How we used to do it before

At the beginning of the company, we used to use [Jenkins](https://jenkins.io/doc/book/pipeline/) as a CI tool to deploy our applications. But then we switched to CircleCI because at that point it looked **simple, faster and cheaper**.

![image](https://user-images.githubusercontent.com/4131432/97111186-b5495e80-16dd-11eb-8b57-1f0807949e9b.png)

![image](https://user-images.githubusercontent.com/4131432/97111198-c7c39800-16dd-11eb-828d-d84e88028be4.png)

So after that, even though each repository has its own specific rules, we basically used to use [CircleCI](https://circleci.com/) to handle our continuous integrations. We had a file under the `.circleci` folder, called `config.yml`. Also, we had some environment keys configured for each project.

On top of that, we have organized `Makefiles` that made the task of migrating even easier, so in the end, just had to migrate the commands to GitHub Actions’ own config file.

![image](https://user-images.githubusercontent.com/4131432/97111221-ee81ce80-16dd-11eb-871f-1d7ebcf4cbd6.png)

The structure of the configuration didn’t have to change much, where we also have a file to coordinate how the build process should be and manage the deploy when in a staging/master branch.

## Why we decided to switch

CircleCI served us well (and still is) but we decided to experiment with Github Actions in order to have a single platform where our codebase and CI/CD setup live as it makes onboarding easier as its one less tool to onboard new developers. While Github Actions is still new it comes with a lot of integrations that made our workflows faster and simpler.

![image](https://user-images.githubusercontent.com/4131432/97111238-05c0bc00-16de-11eb-83ea-80c7483186dd.png)

## How are doing it now

We started the migration slowly, first migrating some projects with fewer dependencies until we move to a project with more needs, this was the result:

```
name: CI

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: ruby:2.6.5

    services:
      <db_service_name>:
        image: postgres:10.6
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@master

      - uses: actions/cache@v1
        with:
          path: /usr/local/bundle
          key: v1-gem-cache-${{ hashFiles('**/Gemfile.lock') }}
          restore-keys: v1-gem-cache

      - name: Install dependencies
        env:
            BUNDLE_GITHUB__COM: ${{ secrets.BUNDLE_GITHUB__COM }}:x-oauth-basic
        run: "make deps"

      - name: Lint
        run: "make lint"

      - name: Test
        env:
            RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
            RAILS_ENV: "test"
            DATABASE_URL: "postgres://postgres@<db_service_name>:5432/<db_name>"
        run: "make test"

  deploy:
    if: github.ref == 'refs/heads/master' || github.ref == 'refs/heads/staging'

    runs-on: ubuntu-latest
    needs: test

    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}

    steps:
      - uses: actions/checkout@master

      - name: Build Docker image
        env:
            BUNDLE_GITHUB__COM: ${{ secrets.BUNDLE_GITHUB__COM }}:x-oauth-basic
        run: "make build"

      - name: Deploy to staging
        if: github.ref == 'refs/heads/staging'
        run: DEPLOYER=${GITHUB_ACTOR} make deploy_to_staging

      - name: Deploy to production
        if: github.ref == 'refs/heads/master'
        run: DEPLOYER=${GITHUB_ACTOR} make deploy_to_production
```

## File Structure

[name](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#name)

Which just set the name of the workflow, in our case we named it CI since it does this part of our pipeline.

[on](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#on)

Here is an interesting setting where we can decide when this workflow will be triggered, it can be set as a `string`, an `array of events` and an `array of event types`, so it is very flexible and can do a lot of different actions.

For us, since we want to do that on every push, we just set as a string `[push]`.

[jobs](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#jobs)

A workflow run is made up of one or more jobs. We configure two different jobs that suit our environment: test and deploy.

- `test`: Run on every branch, independent of its name. In this step, we basically set up the environment and all its dependencies (Postgres, Redis and so on), install app dependencies, run code linters/tests

- `deploy`: If the target branch either `staging` or `master` we want to perform the deploy to the server, which in our case is on Amazon. So for it, we set up AWS environment variables, build the Docker Image for the app and lastly run the deploy job based on the `Makefile`.

## Notes and difficulties

### Setting up environment variables

For every step in our workflow we setup environment variables which are set like `${{ secrets.VARIABLE_NAME }}` and they can be set in the Settings tab of the repository page, in the secrets section.

### Using the cache action

When we were sure that the CI we setup is running as we want, we set up the caching action in order to make it run faster in the next time we trigger it again. The `cache` action will attempt to restore a cache based on a `key` we provide.

### Database service name

When configuring the service for the database, you need to make sure that the name of the service matches the name you set up when configuring the database URL.

### No support for private Docker Images

We used to use private docker images, and GitHub Actions doesn’t allow private images to run in its CI, yet. Also, that made us rethink why we don’t push our image to [Docker Hub](https://hub.docker.com/).

### Private gems

In our projects, we use some private gems to handle some authentication, and those led us to configure [`BUNDLE_GITHUB__COM`](https://bundler.io/v1.15/bundle_config.html) in order to use those, and this needs to be set up in the setting/secrets on your repository.

## Resources

1. Go Rails article:
https://gorails.com/episodes/github-actions-continuous-integration-ruby-on-rails

2. Workflow syntax for GitHub Actions:
https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions

---

And that’s all folks! This is a summary of how we migrate our CI service to Github Actions!

![](https://media.giphy.com/media/xUPOqo6E1XvWXwlCyQ/giphy.gif)
