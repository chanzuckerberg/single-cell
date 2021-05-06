# How to Upgrade Dependencies

Once a month at the beginning of the month, the oncall engineer should visit the current versions of package dependencies that we have for both the frontend (`npm`) and the backend (`pip`) for `corpora-data-portal` and `cellxgene`. We do this so that we keep up to date with any security vulnerabilities that may exist in older versions and so that we don't end up too out-of-date with improvements that occur in our dependencies causing forward-incompatibility. The sections below describe the policies and instructions for each subset of our product suite.

As a general rule please do not directly import the latest version of the downstream package that contains the vulnerability. For example, if we import package `A` and package `A` depends on package `B` which depends on package `C` which contains a vulnerability, please do not directly import the latest version of package `C` directly into our package. If a newer version of package `A` exists where they have addressed the vulnerability, directly upgrade `A`. Otherwise, wait until there is a new one available.

For an overview of current dependency issues we have in cellxgene Explorer and cellxgene Portal, you can check the two corresponding projects in [Snyk](https://app.snyk.io/org/cellxgene). Note that you should take the findings with a grain of salt as some findings may not have remediation and therefore may continue to exist after everything else has been upgraded. It's a good place to double check your work though.

Finally, please check both repos' Dependabot Alerts to ensure that all alerts have been addressed. Please do not clear the alerts unless they are either addressed (via a fully merged PR) or they are not relevant. This means that if one of our dependency packages is unable to be updated because of no available upgrade, leave the alert open so we know to update the package later once an update is available. The Portal dependabot alerts can be found [here](https://github.com/chanzuckerberg/corpora-data-portal/security/dependabot) and the Explorer dependabot alerts can be found [here](https://github.com/chanzuckerberg/cellxgene/security/dependabot).

## Upgrading cellxgene Explorer

### Upgrading cellxgene Explorer frontend

There are two commands to run to figure out which npm packages to upgrade:

`npm audit` -- this command generates a report of known vulnerabilities in dependencies described in `cellxgene/client/package.json`.

`npm outdated` -- this command generates a report of all dependencies in `cellxgene/client/package.json` and whether there are newer versions available.

For major upgrades: check the package's changelog, especially the breaking changes section first to see if such an upgrade is straightforward. If so, feel free to push the upgrade and make sure the affected functionalities still work. If breaking changes exist, please exercise more caution when updating the package to ensure that all affected functionality still operates as expected; feel free to recruit manual testing support from the larger team to ensure that the application doesn't break.

For minor and patch upgrade: In theory there should not be any breaking changes, so just upgrading the packages should not break anything.

**Important Note**: not all packages follow semantic versioning, so please notice what packages have been upgraded in `package.json` and test their related functionality accordingly to ensure everything is fine.

Run the two above commands and upgrade packages to the latest available (that do not contain a security vulnerability). Please double check that the frontend still works as expected and is not using outdated references. Finally, before committing the updates, please rebuild `package-lock.json` by first deleteing `package-lock.json` and `node_modules` and then running `npm i` to generate the new `package-lock.json`.

[Here](https://github.com/chanzuckerberg/cellxgene/pull/2167/files) is a sample PR that upgraded several frontend npm dependencies.

#### Other useful tools

Besides `npm audit` and `npm outdated`, you may take a look at [`npm-check`](https://www.npmjs.com/package/npm-check) and [`npm-check-updates`](https://www.npmjs.com/package/npm-check-updates) as tools to help upgrade packages to the latest version.

### Upgrading cellxgene Explorer CZI-hosted backend

The pip package dependency files for the CZI-hosted backend for cellxgene Explorer can be found at `cellxgene/backend/czi-hosted/requirements.txt` as well as `single-cell-infra/terraform/modules/hosted-cellxgene/common/customize/requirements.txt`. These two files should be in sync except that the file in `single-cell-infra` should pin all packages to a single version and the file in `cellxgene` should use the `>=` sign for the same version numbers. This is so that when we are testing locally, we can easily find out if a new release of a package is causing compatibility issues. We MUST pin versions in the `single-cell-infra` repo because these are the versions installed during deployment and we don't want an unexpected incompatibility due to a new package release to cause a production deployment to fail.

To perform the upgrades you can make use of `pip-upgrader` to figure out which packages have recent updates:

1. Install [pip-upgrader](https://pypi.org/project/pip-upgrader/): `pip install pip-upgrader`.

1. Upgrade the packages based on the report and verify that the application still works (please fully deploy to a `dev` or `rdev` environment to double check everything).

You can also make use of the [Snyk reports](https://app.snyk.io/org/cellxgene) to figure out which specific packages have security vulnerabilities as well as the [`safety` package](#Upgrade-cellxgene-Explorer-desktop-backend). Please note, since the `safety` package only checks currently installed package versions, please create a fresh virtual environment and install the requirements fresh from the `requirements.txt` file before running the command.

Finally, besides the `pip` installed packages, please also take a look at the Dockerfile (in the top level directory `cellxgene/Dockerfile`) to ensure that there are no security vulnerabilities there. Snyk is your friend to figure out if there are and what the remediation process is.

### Upgrading cellxgene Explorer desktop backend

The policy we have to upgrade the backend of cellxgene Explorer _desktop_ is different than any of the other policies. Because many users may have different versions of the dependent `pip` packages already installed on their laptop, and because many users do not use a virtual environment, we need to be _as permissive as possible_ when defining what the allowable versions of each package are.

This means that the allowable versions we set for each package:

1. SHOULD be the minimum version that is functionally compatible and includes no medium or high security risks.

1. MAY exclude specific versions that have medium or high security vulnerabilities if the specification is easier (e.g. `foo>=1.9.0,!=2.0.0`).

To update the `pip` packages for cellxgene Explorer desktop backend, we recommend doing the following steps:

1. Create a clean new virtual environment and install based on the `cellxgene/backend/server/requirements.txt` file. This is necessary because the package in the next step ONLY checks for vulnerabilities against the currently installed packages so you need to ensure that your versions match those in the requirements file.

1. Install [safety](https://pypi.org/project/safety/): `pip install safety`.

1. Manually install the minimum required version of all packages in `cellxgene/backend/server/requirements.txt` and `cellxgene/backend/server/requirements-prepare.txt`.

1. Run `safety`: `python -m safety check --full-report`

1. Update requirements as indicated, goto step 2 again.  Repeat until clean.

1. Check that cellxgene still works :)

Please also upgrade the requirements in `cellxgene/backend/server/requirements-dev.txt` though the policy on being as permissive as possible does not apply since only cellxgene Explorer developers need to install those requirements.

[Here](https://github.com/chanzuckerberg/cellxgene/pull/2172/files) is an example PR.

## Upgrade cellxgene Portal

### Upgrading cellxgene Portal frontend

The instructions to upgrade the Portal frontend are exactly the same as the [Explorer frontend instructions](#Upgrading-cellxgene-Explorer-frontend). The only difference is that the package location is `corpora-data-portal/frontend/package.json`.

You will also need to ensure that the frontend Docker image is up to date at `corpora-data-portal/frontend/Dockerfile`.

### Upgrading cellxgene Portal backend

The instructions to upgrade the Portal backend are pretty much identical to the [Explorer CZI-hosted backend instructions](#Upgrading-cellxgene-Explorer-czi-hosted-backend). The following is the list of files to check:

- `corpora-data-portal/requirements.txt`
- `corpora-data-portl/backend/chalice/api_server/requirements.txt`
- `corpora-data-portal/backend/chalice/cloudfront_invalidator/requirements.txt`
- `corpora-data-portal/backend/chalice/upload_failures/requirements.txt`
- `corpora-data-portal/.happy/requirements.txt`
- `corpora-data-portal/Dockerfile`
- `corpora-data-portal/Dockerfile.processing_image`
- `corpora-data-portal/Dockerfile.upload_failures`
