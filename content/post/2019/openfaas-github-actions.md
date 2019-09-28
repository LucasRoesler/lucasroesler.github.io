---
title: "Action Packed Functions"
summary: "A quick review of GitHub Actions and how to use them for CI/CD of your OpenFaaS Functions"
date: 2019-09-28T00:00:00+02:00
tags:
  - programming
  - openfaas
  - development
  - github
  - github actions
keywords:
  - programming
  - openfaas
  - development
  - github
  - github actions
comments: false
showPagination: false
autoThumbnailImage: true
thumbnailImagePosition: left
coverSize: partial
coverImage: /images/tools-498202_1280.jpg
---

Last month, GitHub announced new [improved CI/CD with GitHub Actions][github-actions-announcement].  Since then, I have experimented with the new Action workflows and created [an Action][openfaas-actions-repo] that wraps the `faas-cli` so that you can easily build and deploy your functions directly from GitHub.

---

My initial interactions with the new GitHub Actions were not great, they really meant it when they said "public beta". BUT!!! Having said that, there is clearly a lot of promise in GitHub Actions. People [seem to love it](https://twitter.com/search?q=github%20actions&src=typed_query). The integration into the UI is great. And, while the first week was tough, the documentation and [starter workflows have seen a lot of improvements][tweet-fixed-start-workflows].  Today, I want to introduce the OpenFaaS Actions that I wrote and show you how to set up a simple "build & deploy" workflow for your OpenFaaS functions.


Back at the beginning of the year, I wrote a short walk-through on the OpenFaaS blog showing how to cleanly [split Python functions into multiple modules][openfaas-multifile-blog]. Today, we will extend that [example repo][example-function] with a [GitHub Action Workflow][example-run-result] to make sure that the latest version of the function is [available to deploy][example-function-hub].

## TL;DR
If you are a GitHub Actions expert already and just want to see the workflow files:
```yaml
name: CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Login to Docker Hub
      uses: actions/docker/login@master
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
    - uses: LucasRoesler/openfaas-action/template_store_pull@v0.1.0
      name: Fetch Template
      with:
        name: "python3-flask"
    - uses: LucasRoesler/openfaas-action/build@v0.1.0
      name: Build
      with:
        path: "wordcount.yml"
        tag: "sha"
    - uses: LucasRoesler/openfaas-action/push@v0.1.0
      name: Push image
      if: "success() && github.ref == 'refs/heads/master' "
      with:
        path: "wordcount.yml"
        tag: "sha"
```

## The OpenFaaS Actions
Currently, the [OpenFaaS Action repo][openfaas-actions-repo] defines 6 common actions that expose the `faas-cli` commands of the same name:

1. `login` - login to an OpenFaaS cluster
2. `build` - build the functions in the named function stack configuration
3. `shrinkwrap` - assemble the function build context, merging the function implementation with the template, but don't build it
4. `template pull` - pull the function templates from the named template repo
5. `template store pull` - pull the named function template from the template store
6. `push` - push the built function docker image to a docker registry

It also provides a generic `cli` action that allows you to pass any command to the `faas-cli`.  This is the escape hatch that allows you to use the `faas-cli` directly instead of via the pre-defined actions described above.

## What is an Action?
Each step in the above workflow "uses" a different GitHub Action.  These are predefined ... well... actions that can be performed, in this cases

1. clone the project,
2. login to docker hub,
3. fetch the python function template,
4. build the function, and
5. push the function image.

Each of these actions is effectively a Docker image, a command, and some optional metadata. For example, the template pull Action is just this bit of yaml

```yaml
name: 'OpenFaaS CLI Template Repo Pull Action'
description: 'Pull templates from a remote repository'
author: 'Lucas Roesler'
inputs:
  repository_url:
    description: 'repository to pull templates from, defaults to the core templates'
    required: false
runs:
  using: 'docker'
  image: 'docker://openfaas/faas-cli:0.9.2'
  args:
    - 'faas-cli'
    - 'template'
    - 'pull'
    - '${{ inputs.repository_url }}'
```

In which I specify that it runs the `faas-cli template pull` command in the prebuilt  `openfaas/faas-cli:0.9.2` image.


## Setup the workflow

### Configure you repo secrets
Before we dive into the GitHub Action Workflow, we need to configure two [secrets for the repo][github-actions-secrets-docs] `DOCKER_USERNAME` and `DOCKER_PASSWORD`.  This is done in the Secrets section of the Settings tab, like this:

{{< image src="/images/openfaas-github-actions/github-secrets.png" title="Setup docker credentials as repository secrets" >}}

### Write the workflow

In your GitHub repo, select the Actions tab to get started. If you do not have any actions yet, it will look just like this
{{< image src="/images/openfaas-github-actions/github-actions-setup.png" title="Actions welcome screen" >}}
If you have already setup some actions, it will show the previous workflow runs. But, [as of Sept 26](https://github.blog/changelog/2019-09-26-the-actions-tab-gets-a-new-look/) GitHub has add a "New Workflow" button that brings you back to that welcome screen.

This is a custom workflow, so select "Setup a workflow yourself", which brings you to an editor
{{< image src="/images/openfaas-github-actions/github-actions-setup-writing.png" title="Actions text editor" >}}

Now we simply need to update the steps to use the new OpenFaaS Actions.

For this example, we want the workflow to

1. "clone the project",

    This step is the simplest and uses an the official GitHub action

        - uses: actions/checkout@v1

2. "login to docker hub",

    this again uses an official Action, but requires some secret parameters which are passed via the env, which we configure in the next section,

        - name: Login to Docker Hub
          uses: actions/docker/login@master
          env:
            DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
            DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

3. "fetch the python function template",

    This is the first OpenFaaS step and calls the `faas-cli template store pull` command, but is as simple as the docker login. This time we need to specify the name of the template we want to pull. Unlike the docker login action, these are passed by the `with` section because the OpenFaaS [action has predefined input parameters][github-actions-parameters-docs]

        - uses: LucasRoesler/openfaas-action/template_store_pull@v0.1.0
          name: Fetch Template
          with:
            name: "python3-flask"

    Before going further, the action name specifies a folder/action `template_store_pull` in my action repo `LucasRoesler/openfaas-action` at a specific SHA/tag `v0.1.0`. All actions have this structure, which makes it easy to find and review what the code is actually going to do before you use it or to find inspiration for your own actions.

4. "build the function",

    The `faas-cli build` action requires the relative path to the stack YAML file and it also accepts the tag flag for the build command:
      > Override latest tag on function Docker image, accepts 'latest', 'sha', or 'branch' (default "latest")

        - uses: LucasRoesler/openfaas-action/build@v0.1.0
          name: Build
          with:
            path: "wordcount.yml"
            tag: "sha"

5. "push the function image",

    Finally, with the the functions built, we can also push them to the Docker Hub using the `faas-cli push`.  This step also uses the [step conditional statement][github-actions-if-docs] to ensure that it only runs when the build step was success _AND_ when the workflow is running on the master branch.

        - uses: LucasRoesler/openfaas-action/push@v0.1.0
          name: Push image
          if: "success() && github.ref == 'refs/heads/master' "
          with:
            path: "wordcount.yml"
            tag: "sha"

Altogether, the workflow should look like

```yaml
name: CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Login to Docker Hub
      uses: actions/docker/login@master
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
    - uses: LucasRoesler/openfaas-action/template_store_pull@v0.1.0
      name: Fetch Template
      with:
        name: "python3-flask"
    - uses: LucasRoesler/openfaas-action/build@v0.1.0
      name: Build
      with:
        path: "wordcount.yml"
        tag: "sha"
    - uses: LucasRoesler/openfaas-action/push@v0.1.0
      name: Push image
      if: "success() && github.ref == 'refs/heads/master' "
      with:
        path: "wordcount.yml"
        tag: "sha"
```

### Save and Run the Workflow

Save the workflow using the "Start Commit" button, if you are feeling brave you can commit the workflow directly to `master`, but you can also easily test the workflow from any branch

{{< image classes="center" src="/images/openfaas-github-actions/github-actions-commit.png" title="Committing the workflow" >}}

Once it is committed, you can see the workflow results and logs in the Actions tab

{{< image src="/images/openfaas-github-actions/github-actions-run.png" title="Workflow logs" >}}

## Next Steps
This is just an experiment while GitHub Actions is in beta. But the next step will of course be a GitHub Action to keep the OpenFaaS action up-to-date with the `faas-cli` releases.

Additionally, there is some [interesting work currently going on][pr-template-pull-stack] that should help simplify this workflow even more. We want to add the ability to save the template dependencies to the project repository so that we can pull all of the dependencies with a single command `faas-cli template pull stack`.  For a standard OpenFaaS project using the default `stack.yml`, the basic workflow will be completely standardized and require almost no configuration, like this

```yaml
name: CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Login to Docker Hub
      uses: actions/docker/login@master
      env:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
    - uses: LucasRoesler/openfaas-action/template_pull_stack@v0.1.0
      name: Fetch Template
    - uses: LucasRoesler/openfaas-action/build@v0.1.0
      name: Build
      with:
        tag: "sha"
    - uses: LucasRoesler/openfaas-action/push@v0.1.0
      name: Push image
      if: "success() && github.ref == 'refs/heads/master'"
      with:
        tag: "sha"
```

Ultimately, I want to make the OpenFaaS experience on GitHub as easy as possible.


If you have any questions, feedback, or you want to help:

* Open an [issue or PR on the OpenFaaS Actions repo][openfaas-actions-repo-issues], or
* Join the OpenFaaS [Slack community][of-slack]

[of-slack]: https://docs.openfaas.com/community
[github-actions-announcement]: https://github.blog/2019-08-08-github-actions-now-supports-ci-cd/
[github-actions-parameters-docs]: https://help.github.com/en/articles/workflow-syntax-for-github-actions#jobsjob_idstepswith
[github-actions-if-docs]: https://help.github.com/en/articles/workflow-syntax-for-github-actions#jobsjob_idstepsif
[github-actions-secrets-docs]: https://help.github.com/en/articles/virtual-environments-for-github-actions#creating-and-using-secrets-encrypted-variables
[openfaas-actions-repo]: https://github.com/LucasRoesler/openfaas-action
[openfaas-actions-repo-issues]: https://github.com/LucasRoesler/openfaas-action/issues
[openfaas-multifile-blog]: https://www.openfaas.com/blog/multifile-python-functions/
[tweet-fixed-start-workflows]: https://twitter.com/TheAxeR/status/1163014894117163009?s=20
[example-function]: https://github.com/LucasRoesler/openfaas-multifile-example
[example-function-hub]: https://cloud.docker.com/repository/docker/theaxer/wordcount
[example-run-result]: https://github.com/LucasRoesler/openfaas-multifile-example/pull/2/checks?check_run_id=202252631
[pr-template-pull-stack]: https://github.com/openfaas/faas-cli/pull/668
[issue-stack-templates]: https://github.com/openfaas/faas-cli/pull/617
