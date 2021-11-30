---
layout: classic-docs
title: "Scheduled Pipelines"
short-title: "Scheduled Pipelines"
description: "Learn how to use scheduled pipelines"
order: 20
version:
- Cloud
- Server v3.x
- Server v2.x
suggested:
  - title: Manual job approval and scheduled workflow runs
    link: https://circleci.com/blog/manual-job-approval-and-scheduled-workflow-runs/
  - title: How to trigger a workflow
    link: https://support.circleci.com/hc/en-us/articles/360050351292?input_string=how+can+i+share+the+data+between+all+the+jobs+in+a+workflow
  - title: Conditional workflows
    link: https://support.circleci.com/hc/en-us/articles/360043638052-Conditional-steps-in-jobs-and-conditional-workflows
---

* TOC
{:toc}

## Overview
{: #overview }

Scheduled pipelines allow users to trigger pipelines periodically based on a schedule.

Since the scheduled run is based on pipelines, scheduled pipelines have all the features that come with using pipelines:

- Control the actor associated with the pipeline, which can enable the use of restricted contexts.
- Use dynamic config via setup workflows.
- Modify the schedule without having to edit `config.yml`.
- Interact with auto-cancelling of pipelines.
- Specify pipeline parameters associated with a schedule.

CircleCI has APIs that allow users to create, view, edit, and delete scheduled pipelines. At this time, a UI for scheduled pipelines is not yet available.

## Get started with scheduled pipelines in CircleCI
{: #get-started }

You have the option of setting up scheduled pipelines from scratch, or you can migrate existing scheduled workflows to scheduled pipelines.

### Start from scratch
{: #start-from-scratch }

If your project has no scheduled workflows and you would like to try out scheduled pipelines:

1. Have your CCI token ready, or create a new token by following [these steps](https://circleci.com/docs/2.0/managing-api-tokens/).
2. Create a new schedule using the API. For example:

```sh
curl --location --request POST 'https://circleci.com/api/v2/project/<project-slug>/schedule' \
--header 'circle-token: <your-cci-token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "my schedule name",
    "description": "some description",
    "attribution-actor": "system",
    "parameters": {
      "branch": "main"
      <additional pipeline parameters can be added here>
    },
    "timetable": {
        "per-hour": 3,
        "hours-of-day": [1,15],
        "days-of-week": ["MON", "WED"]
    }
}'
```

For additional information, refer to the **Schedule** section under the [open-api docs](https://circleci.com/docs/api/v2/).

### Migrate scheduled workflows to scheduled pipelines
{: #migrate-scheduled-workflows }

Currently, using scheduled workflows has some limitations:

* Cannot control the actor, so scheduled workflows can't use restricted contexts.
* Cannot control the interaction with auto-cancelling of pipelines.
* Cannot use scheduled workflows together with dynamic config without complex workarounds.
* Cannot change or cancel scheduled workflows on a branch without triggering a pipeline.
* Cannot kick off test runs for scheduled workflows without changing the schedule.
* Cannot restrict scheduled workflows from PR branches if you want the workflow to run on webhooks.

To migrate from scheduled workflows to scheduled pipelines, follow the steps below:

1. Find the scheduled trigger in your project's `.circleci/config.yml`
    For example, it might look like:

    ```yaml
    daily-run-workflow:
      triggers:
        - schedule:
            # Every day, 0421Z.
            cron: "21 4 * * *"
            filters:
              branches:
                only:
                  - main
      jobs:
        - test
        - build
    ```
2. Interpret the frequency your trigger needs to run from the cron expression.
3. Use the same step from the [Start from scratch](#start-from-scratch) section above to create the schedule via the API.
4. In the config file, remove the `triggers` section, so that it resembles a standard workflow.
    ```yaml
    daily-run-workflow:
      jobs:
        - test
        - build
    ```

#### Add workflows filtering
{: #workflows-filtering }
{:.no_toc}

As a scheduled pipeline is essentially a triggered pipeline, it will run every workflow in the config.

One way to implement workflows filtering is by using the pipeline values. For example:

```yaml
daily-run-workflow:
  when:
    and:
      - equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
      - equal: [ "my schedule name", << pipeline.schedule.name >> ]
  jobs:
    - test
    - build
```

Note that in the above example, the second `equal` under `when` is not strictly necessary. The `pipeline.schedule.name` is an available pipeline value when the pipeline is triggered by a schedule.

You may also add filtering for workflows that should NOT run when a schedule triggers:

{% raw %}
```yaml
daily-run-workflow:
  when:
    and:
      - equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
      - equal: [ "my schedule name", << pipeline.schedule.name >> ]
  jobs:
    - test
    - build

other-workflow:
  when:
    not:
      equal: [ scheduled_pipeline, << pipeline.trigger_source >> ]
  jobs:
   - build
   - deploy
```
{% endraw %}

## FAQs
{: #faq }

**Q:** How do I find the schedules that I have created?

**A:** As scheduled pipelines are stored directly in CircleCI, there is a UUID associated with each schedule. You can also list all the schedules under a single project:

```sh
curl --location --request GET 'https://circleci.com/api/v2/project/<project-slug>/schedule' \
--header 'circle-token: <PERSONAL_API_KEY>'
```

**Q:** Why is my scheduled pipeline not running?

**A:** There could be a few possible reasons:
* Is the actor who is set for the scheduled pipelines still part of the organization?
* Is the branch set for the schedule deleted?
* Is your GitHub organization using SAML protection? SAML tokens expire often, which can cause requests to GitHub to fail.