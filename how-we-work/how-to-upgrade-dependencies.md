# How to Upgrade Dependencies

Once a month at the beginning of the month, the oncall engineer should visit the current versions of package dependencies that we have for both the frontend (`npm`) and the backend (`pip`) for `corpora-data-portal` and `cellxgene`. We do this so that we keep up to date with any security vulnerabilities that may exist in older versions and so that we don't end up too out-of-date with improvements that occur in our dependencies causing forward-incompatibility. The sections below describe the policies and instructions for each subset of our product suite.

As a general rule please do not directly import the latest version of the downstream package that contains the vulnerability. For example, if we import package `A` and package `A` depends on package `B` which depends on package `C` which contains a vulnerability, please do not directly import the latest version of package `C` directly into our package. If a newer version of package `A` exists where they have addressed the vulnerability, directly upgrade `A`. Otherwise, wait until there is a new one available.

For an overview of current dependency issues we have in cellxgene Explorer and cellxgene Portal, you can check the two corresponding projects in [Snyk](https://app.snyk.io/org/cellxgene). Note that you should take the findings with a grain of salt as some findings may not have remediation and therefore may continue to exist after everything else has been upgraded. It's a good place to double check your work though.

## Upgrading cellxgene Explorer

### Upgrading cellxgene Explorer frontend

There are two commands to run to figure out which npm packages to upgrade:

`npm audit` -- this command generates a report of known vulnerabilities in dependencies described in `cellxgene/client/package.json`.

`npm outdated` -- this command generates a report of all dependencies in `cellxgene/client/package.json` and whether there are newer versions available.

Run these two commands and upgrade packages to the latest available. Please also make any necessary changes in the client code to ensure that that the frontend still works as expected and is not using outdated references. Finally, run `npm i` to install the packages and update the corresponding `package-lock.json`.

[Here](https://github.com/chanzuckerberg/cellxgene/pull/2167/files) is a sample PR that upgraded several frontend npm dependencies.

### Upgrading cellxgene Explorer czi-hosted backend

The pip package dependency files for the CZI-hosted backend for cellxgene Explorer can be found at `cellxgene/backend/czi-hosted/requirements.txt` as well as `single-cell-infra/terraform/modules/hosted-cellxgene/common/customize/requirements.txt`. These two files should be in sync except that the file in `single-cell-infra` should pin all packages to a single version and the file in `cellxgene` should use the `>=` sign for the same version numbers. This is so that when we are testing locally, we can easily find out if a new release of a package is causing compatibility issues. We do want to pin versions in the `single-cell-infra` repo because these are the versions installed during deployment and we don't want an unexpected incompatibility to cause a production deployment to fail.

To up

### Upgrading cellxgene Explorer desktop backend

The policy we have to upgrade the backend of cellxgene Explorer _desktop_ is different than any of the other policies. Because many users may have different versions of the dependent `pip` packages already installed on their laptop, and because many users do not use a virtual environment, we need to be _as permissive as possible_ when defining what the allowable versions of each package are.

To update the `pip` packages for cellxgene Explorer desktop backend, we recommend doing the following steps:

1. Install https://pypi.org/project/safety/: `pip install safety`. Note: this will ONLY check for vulnerabilities against the currently installed packages.

2. Manually install the minimum required version of all packages in `cellxgene/backend/server/requirements.txt` and `cellxgene/backend/server/requirements-prepare.txt`.

3. Run `safety`: `python -m safety check --full-report`

4. Update requirements as indicated, goto step 2 again.  Repeat until clean.

5. Check that cellxgene still works :)

Please also upgrade the requirements in `cellxgene/backend/server/requirements-dev.txt` though the policy on being as permissive as possible does not apply since only cellxgene Explorer developers need to install those requirements.

## Upgrade cellxgene Portal

### Upgrading cellxgene Portal frontend

The instructions to upgrade the Portal frontend are exactly the same as the [Explorer frontend instructions](#Upgrading-cellxgene-Explorer-frontend). The only difference is that the package location is `corpora-data-portal/frontend/package.json`.

You will also need to ensure that the frontend Docker image is up to date at `corpora-data-portal/frontend/Dockerfile`.

### Upgrading cellxgene Portal backend

