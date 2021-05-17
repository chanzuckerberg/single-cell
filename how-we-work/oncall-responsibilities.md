# On-call Responsibilities

## Purpose

On-call shifts last one week and represent a period of time during which the designated engineer is responsible for diagnosing, mitigating, fixing, or escalating incidents as needed. In addition, they are available for non-urgent production duties such as deploying services as part of their regular release trains.

All engineers, both frontend and backend, will be on the on-call rotation to reduce the frequency that any one person is on call and to build shared ownership over our services.

## Practice and Process

An on-call rotation begins on Monday at 9am PST. Each week, there is a primary and secondary on-call engineer. You will get an email and Slack notification from PagerDuty (PagerDuty manages our on-call schedule) designating you as the on-call engineer. The manager will always be on tertiary on-call duty.

### Pre-Requisites

- Please make sure you have access to Sentry and Datadog so that you can check out our error reporting. You can check if you have Sentry access by logging [here](https://czi-duo.okta.com/) and seeing if you have the app assigned. You can check if you have Datadog access by checking your [Okta App Homepage](https://czi.okta.com/app/UserHome) -- you'll want to make sure even if you have the app assigned that you can login and access `hosted-cellxgene-prod`. If you do not have access to either, ask in the [#help-infra](https://chanzuckerbergteam.slack.com/archives/C94RQ5SBV) Slack channel or reach out to your manager.

### Primary Responsibilities

#### Deployment

##### Deployment summary table

|                   | dev                            | staging                                                                                  | prod                                                                                     | dev-canary                                                |
| ----------------- | ------------------------------ | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| single-cell-infra | TFE auto                       | TFE auto                                                                                 | TFE manual                                                                               | TFE manual                                                |
| data-portal       | Need to set up auto deployment | Github Action Auto Deploy                                                                | happy/script Deploy Manual                                                               | no deployment                                             |
| explorer          | Github Action Auto Deploy      | single-cell-infra deploy script to kick off github action (also possible to run locally) | single-cell-infra deploy script to kick off github action (also possible to run locally) | Github Action Auto Deploy (does not auto rebase off main) |

The principal responsibility of the primary on-call is to coordinate deployments on Wednesday (from `main` to `staging`) and Thursday (from `staging` to `prod`). Both the cellxgene Data Portal and the cellxgene Explorer needs to be deployed and require two different processes. On both days, please try to promote by 1pm PST.

The instructions to deploy `hosted-cellxgene` (a.k.a. cellxgene Explorer) can be found [here](https://github.com/chanzuckerberg/single-cell-infra/tree/main/terraform/modules/hosted-cellxgene#redeploying-the-application). The instructions to deploy `corpora-data-portal` (a.k.a. cellxgene Data Portal) can be found [here](https://github.com/chanzuckerberg/single-cell-infra/tree/main/terraform/modules/corpora#redeploying-the-application).

**On Wednesday**:

- Send a note to the [#single-cell-ops](https://chanzuckerbergteam.slack.com/archives/CV7PFVABC) Slack channel to see if there are any PRs that you should wait on before the `main` to `staging` push.
- Once the deployment is in progress, let the team know in the same channel and also notify when the deployment is complete. Please write a quick note on the key new features/bug fixes for the engineering team to help test on staging. If the deployment fails, coordinate the fix.
- Data Portal
  - Continuous deployment is enabled in staging for the data portal, no further action is required to update.
- Explorer
  - Please follow the deployment instructions [here](https://github.com/chanzuckerberg/single-cell-infra/tree/main/terraform/modules/hosted-cellxgene#redeploying-the-application).
  - Please manually test the cellxgene Explorer deployment, walking through the test cases in [this doc](https://docs.google.com/document/d/1nHdd8cDlmauv27oEemlMy_mEa0Dw7UMCp-w50IhNuK0/edit).

**On Thursday**:

- Send a note to the [#single-cell-ops](https://chanzuckerbergteam.slack.com/archives/CV7PFVABC) Slack channel checking to make sure that engineers have had a chance to test the staging deployment. If you get a quorum of LGTMs, promote staging to prod.
- Let the channel know when the deployment is in progress and when it is complete. You are also responsible for rolling back to a working version and coordinating any fixes if the deployment fails.
- Explorer

  - Please follow the deployment instructions [here](https://github.com/chanzuckerberg/single-cell-infra/tree/main/terraform/modules/hosted-cellxgene#redeploying-the-application).
  - Please manually test the cellxgene Explorer deployment, walking through the test cases in [this doc](https://docs.google.com/document/d/1nHdd8cDlmauv27oEemlMy_mEa0Dw7UMCp-w50IhNuK0/edit).

- Data Portal

  - Note that the name of the image includes the sha of the commit it was built off of.
  - To deploy the data portal by kicking off a github action, you will need to [create a GITHUB_TOKEN](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) with access to the corpora-data-portal repo (specifically the permission scope will need to include, admin:repo_hook, repo, workflow). Save the token as an environmental variable (GITHUB_TOKEN) in your terminal and enable SSO for it through github.

- Once the deployment is in progress, let the team know in the same channel and also notify when the deployment is complete. Please include a quick note on the key new features/bug fixes for the team to help test. If the deployment fails, coordinate the fix.

#### Other responsibilities

- If the secondary on-call engineer is new, please pair-program to help them onboard onto the process. Include them when you do deployment and point out various issues to watch out for (i.e. incoming bugs, Sentry, etc.) so that they get an understand of what the role entails.
- Watch the [#single-cell-ops](https://chanzuckerbergteam.slack.com/archives/CV7PFVABC) and [#single-cell-eng](https://chanzuckerbergteam.slack.com/archives/C01CPJURT0S) Slack channels for any questions or issues reported by any engineers, PM, or our Curation team. Please triage these issues, notify the right person, and if appropriate file an issue to keep track of the bug. For high priority issues, especially ones that do not take too much time (< 1 day effort), please create the issue and pull it into the sprint to be worked on.
- We use Infrastructure Engineering's in-house deployment of [sentry.io](https://sentry.prod.si.czi.technology/sci-sc/) for error aggregation. Keep an eye out on Sentry issues throughout the week, especially noting issues that pop up after the deployment in large quantities. This is likely a sign that there is a high priority bug that we need to address. Sentry can help you create Github issues to track these issues.
- If there is an issue with the deployment or something happened this week that took down our application, please write a postmortem and add it to the postmortems section of this repo. Please also bring up the postmortem and any lessons during the Sprint Retrospective so that we can all learn!
- If you run into a potential security issue or product incident, need to rollback a deployment, or otherwise are running into some issues, please check out the the Incident Playbook (link to come!) to figure out next steps. If all else fails, please ping the [#single-cell-eng](https://chanzuckerbergteam.slack.com/archives/C01CPJURT0S) Slack channel to start getting help from other engineers and/or notify your manager.

### Secondary and Tertiary Responsibilities

The secondary and tertiary on-call folks' responsibility is to act as backup if the primary on-call engineer is unable to attend to an issue for any reason. For incidents, PagerDuty will automatically escalate to secondary if there is no acknowledgement from primary after 10 minutes and then to tertiary if there is no acknowledgement from secondary after 30 minutes.

## References and Important Links

- Our on-call process has been inspired by [Google's On-call Guidelines](https://landing.google.com/sre/workbook/chapters/on-call/)
- [cellxgene Explorer Deployment Instructions](https://github.com/chanzuckerberg/single-cell-infra/tree/main/terraform/modules/hosted-cellxgene#redeploying-the-application)
- [cellxgene Data Portal Deployment Instructions](https://github.com/chanzuckerberg/single-cell-infra/tree/main/terraform/modules/corpora#redeploying-the-application)
- [Sentry.IO -- Prod Environment](https://sentry.prod.si.czi.technology/sci-sc/)
