---
title: "Hard won lessons about Github Actions: Really fixed it this time, number 42."
description: "Really fixed it this time, number 42."
date: 2022-10-27T00:00:00+02:00
tags:
  - github actions
  - programming
  - ci/cd
draft: false
coverImage: images/belinda-fewings-_CyyAj0QboY-unsplash.jpg
---

I have spent the two weeks modernizing a long dormant project at Contiamo. Not only did this require updating various dependencies and subsequently parts of the code base, but it also meant updating an old Jenkinfile into a Github Action workflow. 😬😬😬

<!--more-->

I can't say I am proud of my commit history:

```
| | * 43adfcl final fix, final.
| | * 2e4f2c1 final fix, again.
| | * 147361f final fix
| | * f60de9f actually fixed this time
| | * 81451a8 actually fixed 2
| | * e40d511 really fixed it this time
| | * plkanaf really fixed it
| | * adf9afj fix the build
| | * 6acf203 switch to github actions
```

but I was eventually victorious! In addition to the usual silly mistakes that happen when writing a CI/CD pipeline, I stumbled on some pretty esoteric pitfalls. Here are some of the bigger lessons I learned along the way.

## Inputs are strings

Do not trust the input `type` for `workflow_dispatch`. Yes, sure, the schema _says_ you can do this to specify a boolean input

```yaml
workflow_dispatch:
  inputs:
    environment:
      description: "The environment to deploy to"
      required: true
      default: "dev"
      type: choice
      options:
        - "dev"
        - "stg"
        - "prod"
    process:
      description: "Control if the SQL/materialization processing is run."
      required: true
      default: true
      type: boolean
```

It has not completely lied to you, because this spec will allow you to manually trigger the workflow from the Github UI and it will produce a checkbox in the

{{<image src="images/workflow_dispatch_ui.png" title="Github Workflow manual trigger UI" >}}

Unfortunately, that is all it gets you because you might think that a statement like this would work

```yaml
process-sql:
  if: ${{ github.events.inputs.process }}
```

But it does not. The value `process` is defined to be a boolean but it is actually a string, the above snippet behaves like

```yaml
process-sql:
  if: "false"
```

Which is truthy, so that step/job will still execute!

To get the expected behavior I had to use a string comparison like this

```yaml
process-sql:
  if: ${{ github.events.inputs.process == 'true' }}
```

In theory, you can also use this

```yaml
process-sql:
  if: ${{ fromJSON(github.events.inputs.process) }}
```

but I haven't tested it.

Github is well aware of this and fixed it by updating an example workflow file, but I haven't found an explicit statement in the docs. See https://github.com/actions/runner/issues/1483 and https://github.com/cylc/cylc-doc/pull/317

## Inputs are strings, except for when they are not

Github recently added support for ["Reusable Workflows"](https://docs.github.com/en/actions/using-workflows/reusing-workflows). In the workflow file, this is called `workflow_call` and it looks very similar to `workflow_dispatch` (see the previous section). Here is an example definition

```yaml
workflow_call:
  inputs:
    environment:
      required: true
      type: string

    process:
      required: true
      type: boolean
      default: true
      description: "Control if the SQL/materialization processing is run."
```

A quick glance and you might think this is exactly the same before. It is certainly _very_ similar, so you might expect similar behavior. Of course, they are only similar, not the same.

First, let's discuss how to access inputs. In the previous section I used `${{ github.events.inputs.process }}`. This is essentially accessing the webhook payload that triggers the workflow. Github provides a detailed description of this [`github` context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context), there is a lot of data in there. However, this will _not_ have the input values for the `workflow_call`. Instead these are exposed from the [`inputs` context](https://docs.github.com/en/actions/learn-github-actions/contexts#inputs-context) which contains the inputs from both the `workflow_call` _and_ `workflow_dispatch`. Let's return to our boolean for a moment

```yaml
process:
  required: true
  type: boolean
```

This definition will work for both `workflow_call` _and_ `workflow_dispatch` and can be accessed as `inputs.process` but you will get two different values

- when you use `workflow_call`, `inputs.process` is an actual boolean!
- when you use `workflow_dispatch`, `inputs.process` is a string!

If you are mixing workflow triggers and use the same name for the inputs, then you can never be sure if you get a boolean or a string. This is really confusing and hard to debug ... which brings us to the next section.

P.S. A full example of using [`workflow_call` is provided later](#slack-notifications).

## Debugging json objects

Sometimes you will want to debug the workflow context, for example, to check why value `inputs.process` contains. The workflow context also contains information like the workflow event, the actor (person or bot) that triggered the event, repository information, etc. A full description of the each available context can be found [in the docs](https://docs.github.com/en/actions/learn-github-actions/contexts). A typical way to debug a single value in the context might look like this

```yaml
- name: Debug
  run: |
    echo "actor=${{github.actor}}"
```

This works really well if you want to debug a single value and you know that it is a string or number. But sometimes you want to debug the entire context, you want to see all of the inputs to your workflow, for example. You will be tempted to try this

```yaml
- name: Debug
  run: |
    echo "event.inputs: ${{ toJSON(github.event.input) }}"
```

But this will fail because `toJSON` pretty prints the JSON string, which will include quote characters and new-lines, these will cause `bash` errors.

The [suggested workaround](https://github.com/actions/runner/issues/1656) is to pass the JSON string as an env variable, this will auto-escape the string for you

```yaml
- name: Debug
  env:
    INPUT: ${{ toJSON(inputs) }}
    EVENT_INPUT: ${{ toJSON(github.event.inputs) }}
  run: |
    echo "event.inputs: $EVENT_INPUT"
    echo "inputs: $INPUT"
```

Also, note that the `github.event.inputs` is not the same as the `inputs` context. The `github.event.inputs` is the inputs to the original event. If you are using reusable workflows via `workflow_call`, these inputs are only available in the [`inputs` context](https://docs.github.com/en/actions/learn-github-actions/contexts#inputs-context). Which does not seem to be available to the [`actions/github-script`](https://github.com/actions/github-script), so you can not work around the potential BASH problems by switching to JS.

## Slack notifications

This last section is not about any kind of gotchya, rather it is a "you might like this too".

There are several slack notification apps in the [Actions Marketplace](https://github.com/marketplace?type=actions&query=slack). Some of them are going to fit your needs perfectly. But if they don't or you decide you want to customize something they don't expose, you can fallback to the [official Slack action `slack-send`](https://github.com/marketplace/actions/slack-send).

But this has a major downside, sending a formatted message to an incoming webhook is a pain and requires actually knowing the Slack API. Here is the example from the `slack-send` docs

```yaml
- name: Send custom JSON data to Slack workflow
  id: slack
  uses: slackapi/slack-github-action@v1.23.0
  with:
    # For posting a rich message using Block Kit
    payload: |
      {
        "text": "GitHub Action build result: ${{ job.status }}\n${{ github.event.pull_request.html_url || github.event.head_commit.url }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "GitHub Action build result: ${{ job.status }}\n${{ github.event.pull_request.html_url || github.event.head_commit.url }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

This might be too much customization. But I also prefer to use "officially" supported actions. It is slightly easier to trust that the action provided by the Slack team will stay up-to-date with the Slack API and Github guidelines / best practices. However, now that we have [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) ([remember earlier](#inputs-are-strings-except-for-when-they-are-not)) it is relatively easy to create our own custom actions that wrap this "official" action, to have a nicer interface. For example, I want to do something like this

```yaml
notify:
  needs: [training, process-sql, training, deploy]
  if: ${{ always() }}
  uses: ./.github/workflows/notify.yaml
  secrets: inherit
  with:
    color: ${{ needs.deploy.result }}
    message: |
      *Environment:* `${{ inputs.environment }}`
      *Process SQL:* ${{ needs.process-sql.result }}
      *Training*: ${{ needs.training.result }} (model hash: `${{ needs.training.outputs.model_tag }}`)
      *Deploy*: ${{ needs.deploy.result }}
```

Nice and simple, no need to write a JSON payload. Here is an example output

{{<image src="images/message-screenshot.png" title="Sample of the slack message." >}}

Now, we just need to create the reusable notify workflow.

```yaml
# save as .github/workflows/notify.yaml
name: Slack Notify

on:
  workflow_call:
    inputs:
      message:
        required: true
        type: string
      color:
        required: false
        default: "info"
        type: string
      title:
        required: false
        default: ""
        type: string
    secrets:
      SLACK_WEBHOOK:
        required: true

jobs:
  slack:
    runs-on: ubuntu-latest
    steps:
      - name: Debug
        env:
          INPUT: ${{ toJSON(inputs) }}
          EVENT_INPUT: ${{ toJSON(github.event.inputs) }}
        run: |
          echo "EVENT_INPUTS: $EVENT_INPUT"
          echo "INPUTS: $INPUT"

      - name: metadata
        uses: actions/github-script@v6
        id: metadata
        with:
          script: |
            const inputColor = '${{ inputs.color }}' || 'info'
            const inputTitle = '${{ inputs.title }}'
            const inputMessage = ${{ toJSON(inputs.message) }}


            const colors = {
              "success": "good",
              "failure": "danger",
              "info": "#17a2b8",
              "good": "good",
              "warning": "warning",
              "danger": "danger",
            }
            const color = colors[inputColor]
            core.setOutput("color", color)

            const repoBaseURL = `${context.serverUrl}/${context.repo.owner}/${context.repo.repo}`
            const github_ref = context.eventName == "pull_request" ? context.payload.pull_request.head.ref : context.ref
            const branch = github_ref.replace(/^refs\/heads\//, '')

            const title = inputTitle || `${context.workflow} ${context.sha.substr(0, 8)}`
            const title_url = `${repoBaseURL}/actions/runs/${context.runId}`

            core.setOutput("title", title)
            core.setOutput("title_url", title_url)

            const author_name = context.actor
            const author_link = `${context.serverUrl}/${context.actor}`
            const author_icon = `${context.serverUrl}/${context.actor}.png`

            core.setOutput("author_name", author_name)
            core.setOutput("author_link", author_link)
            core.setOutput("author_icon", author_icon)

            const footer = `*<${repoBaseURL}|${context.repo.owner}/${context.repo.repo}>* <${repoBaseURL}/tree/${branch}|${branch}>`

            const payload = {
              "attachments": [
                {
                  "color": color,
                  "title": title,
                  "title_link": title_url,
                  "author_name": author_name,
                  "author_link": author_link,
                  "author_icon": author_icon,
                  "mrkdwn_in": ["text"],
                  "text": inputMessage,
                  "fallback": inputMessage,
                  "footer": footer,
                  "footer_icon": 'https://slack.github.com/static/img/favicon-neutral.png',
                }
              ]
            }
            core.setOutput("payload", payload)

            console.log('payload', payload)

      - name: notification
        id: slack
        uses: slackapi/slack-github-action@v1.23.0
        with:
          payload: ${{ steps.metadata.outputs.payload }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

Obviously, this message payload is designed to match my own preferences, but you can easily copy or fork it to match your own messaging style.

## Conclusion

I actually really like Github Actions, it took awhile to win me over, but Github is constantly improving it and it is really good now. But, it is still a CI/CD system which means it will always take more than one `fix ci` commit and it will always have a few surprises for you. Hopefully, the few that I found will mean they are less surprising for you when _you_ find them. In the mean time, keep committing, and don't worry, you too can fix your failing build 😉

---

Cover Photo by [Belinda Fewings](https://unsplash.com/@bel2000a?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
