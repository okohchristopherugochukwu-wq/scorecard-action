# Scorecards' GitHub action
[![CodeQL](https://github.com/ossf/scorecard-action/actions/workflows/codeql-analysis.yml/badge.svg)](https://github.com/ossf/scorecard-action/actions/workflows/codeql-analysis.yml)
[![codecov](https://codecov.io/gh/ossf/scorecard-action/branch/main/graph/badge.svg?token=MAXISWR53I)](https://codecov.io/gh/ossf/scorecard-action)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/ossf/scorecard-action/badge)](https://scorecard.dev/viewer/?uri=github.com/ossf/scorecard-action)

> Official GitHub Action for [OSSF Scorecards](https://github.com/ossf/scorecard).

The Scorecards GitHub Action is free for all public repositories. Private repositories are supported if they have [GitHub Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security). Private repositories without GitHub Advanced Security can run Scorecards from the command line by following the [standard installation instructions](https://github.com/ossf/scorecard#using-scorecards-1).


## Breaking changes in v2

Starting from scorecard-action:v2, `GITHUB_TOKEN` permissions or job permissions needs to include
`id-token: write` for `publish_results: true`. This is needed to access GitHub's
OIDC token which verifies the authenticity of the result when publishing it. See details [here](#publishing-results)

If publishing results, scorecard-action:v2 also imposes new requirements on both the workflow and the job running the `ossf/scorecard-action` step. For full details see [here](#workflow-restrictions). 
________
[Installation](#installation)
- [Workflow Setup](#workflow-setup-required)
- [Authentication](#authentication-with-fine-grained-pat-optional)

[View Results](#view-results)
- [REST API](#rest-api)
- [Scorecard Badge](#scorecard-badge)
- [Code Scanning Alerts](#code-scanning-alerts)
- [Verify Runs](#verify-runs)
- [Troubleshooting](#troubleshooting)

[Manual Action Setup](#manual-action-setup)
- [Inputs](#inputs)
- [Publishing Results](#publishing-results)
- [Workflow Restrictions](#workflow-restrictions)
- [Uploading Artifacts](#uploading-artifacts)
- [Workflow Example](#workflow-example)

[Reporting vulnerabilities](#reporting-vulnerabilities)
________

The following GitHub triggers are supported: `push`, `schedule` (default branch only).

The `pull_request` and `workflow_dispatch` triggers are experimental.

Running the Scorecard action on a fork repository is not supported.

GitHub Enterprise repositories are not supported.

## Installation

### Workflow Setup (Required)
1. From your GitHub project's main page, click “Security” in the top ribbon.

   ![image](/images/install01.png)

2. Select “Code scanning”.

   ![image](/images/install02.png)

3. Depending on whether the repository already has a scanning tool configured, you will see "Add tool" or "Configure scanning tool.

   a. If you see "Add tool" click it and skip step `b`.

   ![image](/images/install03.png)

   If you see "Configure scanning tool" proceed to step `b`:

   ![image](/images/configurescantool.png)

   b. After clicking "Configure scanning tool", click "Explore workflows" underneath the "Other tools" section:

   ![image](/images/exploreworkflow.png)

5. Choose the "OSSF Scorecard" from the list of workflows below, and then click “Configure”. (To find it faster type "OSSF" in searchbox.)

   ![image](/images/searchingossf.png)

6. Commit the changes. (Your button might say "Commit changes..." instead of "Start commit", it does the same thing.)

   ![image](/images/install05.png)

### Authentication with Fine-grained PAT (optional)
Scorecard can run successfully with the workflow's default `GITHUB_TOKEN`, which is our recommended approach.
However, Scorecard Action requires additional permissions if you use GitHub's classic [Branch Protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) settings and want to see it reflected in your results.
You can read more about how to configure Scorecard Action for these cases [here](/docs/authentication/fine-grained-auth-token.md).

GitHub's new [Repository Rules](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets) are accessible to Scorecard Action with the workflow's default `GITHUB_TOKEN`. 
We recommend new repositories use Repository Rules so they can be read with the default GitHub token. 
Repositories that already use classic Branch Protection and wish to see their results without an admin token should consider migrating to Repository Rules.

### Additional permissions for private repositories

When running Scorecard Action on **private repositories** with the default `GITHUB_TOKEN`, add these **job-level permissions** so Scorecard can query commits and detect configured SAST tools. Without them you may see errors like:

> `Resource not accessible by integration` (e.g., during GraphQL ListCommits)

```yaml
jobs:
  analysis:
    runs-on: ubuntu-latest
    permissions:
      # Required when publishing results (badge / API / code scanning)
      security-events: write
      id-token: write
      # Recommended reads for private repos to avoid GraphQL/SAST gaps
      contents: read
      issues: read
      pull-requests: read
      checks: read
```

## View Results

The workflow is preconfigured to run on every repository contribution. After making a code change, you can view the results for the change either through the Scorecard Badge, Code Scanning Alerts or GitHub Workflow Runs.

### REST API
Starting with scorecard-action:v2, users can use a REST API to query their latest run results. This requires setting [`publish_results: true`](https://github.com/ossf/scorecard/blob/d13ba3f3355b958d5d62edc47282a2e7ed9fa7c1/.github/workflows/scorecard-analysis.yml#L39) for the action and enabling [`id-token: write`](https://github.com/ossf/scorecard/blob/d13ba3f3355b958d5d62edc47282a2e7ed9fa7c1/.github/workflows/scorecard-analysis.yml#L22) permission for the job (needed to access GitHub OIDC token). The API is available here: https://api.scorecard.dev.

### Scorecard Badge

Starting with scorecard-action:v2, users can add a Scorecard Badge to their README to display the latest status of their Scorecard results. This requires setting [`publish_results: true`](https://github.com/ossf/scorecard/blob/d13ba3f3355b958d5d62edc47282a2e7ed9fa7c1/.github/workflows/scorecard-analysis.yml#L39) for the action and enabling [`id-token: write`](https://github.com/ossf/scorecard/blob/d13ba3f3355b958d5d62edc47282a2e7ed9fa7c1/.github/workflows/scorecard-analysis.yml#L22) permission for the job (needed to access GitHub OIDC token). The badge is updated on every run of scorecard-action and points to the latest result. To add a badge to your README, copy and paste the below line, and replace the {owner} and {repo} parts.

```
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/{owner}/{repo}/badge)](https://scorecard.dev/viewer/?uri=github.com/{owner}/{repo})
```

Once this badge is added, clicking on the badge will take users to the latest run result of Scorecard.

![image](/images/badge.png)

### Code Scanning Alerts

A list of results is accessible by going in the Security tab and clicking "Code Scanning Alerts" (it can take a couple minutes for the run to complete and the results to show up). Click on the individual alerts for more information, including remediation instructions. You will need to click "Show more" to expand the full remediation instructions.

![image](/images/remediation.png)

### Verify Runs
The workflow is preconfigured to run on every repository contribution.

To verify that the Action is running successfully, click the repository's Actions tab to see the status of all recent workflow runs. This tab will also show the logs, which can help you troubleshoot if the run failed.

![image](/images/actionconfirm.png)

### Troubleshooting
If the run has failed, the most likely reason is an authentication failure. Confirm that the Personal Access Token is saved as an encrypted secret within the same repository (see [Authentication](#authentication)). Also confirm that the PAT is still valid and hasn't expired or been revoked.

If you have a valid PAT saved as an encrypted secret and the run is still failing, confirm that you have not made any changes to the workflow yaml file that affected the syntax. Review the [workflow example](#workflow-example) and reset to the default values if necessary.

## Manual Action Setup

If you prefer to manually set up the Scorecards GitHub Action, you will need to set up a [workflow file](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions).

First, [create a new file](https://docs.github.com/en/repositories/working-with-files/managing-files/creating-new-files) in t
