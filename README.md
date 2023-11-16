# ReversingLabs GitHub Action: rl-scanner-cloud-composite

ReversingLabs provides officially supported GitHub Actions for a faster and easier way to integrate the [secure.software Portal](https://docs.secure.software/portal/) with CI/CD workflows.

The `rl-scanner-cloud-composite` action uses the official [reversinglabs/rl-scanner-cloud](https://hub.docker.com/r/reversinglabs/rl-scanner-cloud)
Docker image to:

- upload and scan a single build artifact on a secure.software Portal instance;
- display the analysis status as one of the checks in the GitHub interface;
- return the exit status message.

This action is most suitable for users who want to quickly set up a working security scanning solution.
If you already manage workflows for different purposes and want to integrate security scanning as a step under specific conditions,
try the [rl-scanner-cloud-only](https://github.com/reversinglabs/gh-action-rl-scanner-cloud-only) GitHub Action by ReversingLabs.

Compared to `rl-scanner-cloud-only`, this action is more convenient out-of-the-box because it scans the artifact and sets the result status all at once.
In the `rl-scanner-cloud-only` action, everything except the scan has to be provided by the user creating the workflow.


## What is the secure.software Portal?

The secure.software Portal is a SaaS solution that's part of the [secure.software platform](https://www.secure.software/) - a new ReversingLabs solution for software supply chain security.
More specifically, the Portal is a web-based application for improving and managing the security of your software releases and verifying third-party software used in your organization.

With the secure.software Portal, you can:

- Scan your software packages to detect potential risks before release.
- Improve your SDLC by applying actionable advice from security scan reports to all phases of software development.
- Organize your software projects and automatically compare package versions to detect potentially dangerous behavior changes in the code.
- Manage software quality policies on the fly to ensure compliance and achieve maturity in your software releases.


# How this action works

The `rl-scanner-cloud-composite` action relies on a few different
[contexts](https://docs.github.com/en/actions/learn-github-actions/contexts) to access and reuse information across its steps, and on the following external action:

- **ouzi-dev/commit-status-updater**

The action expects that the build artifact is produced in the current workspace before the action is called.
It requires specifying the path of the artifact as the input to the action.
The path must be relative to the `github.workspace`.

When called, the action runs the following steps:

- Set the commit status to pending
- Pull the latest version of the `reversinglabs/rl-scanner-cloud` Docker image
- Connect to a secure.software Portal instance from the container and upload the artifact to the Portal for scanning
- Change the commit status from pending to success/failure depending on the scan result with a descriptive message
- Return the exit status. This allows controlling any subsequent or dependent tasks; for example, blocking the merge if the status is not "success"


## Requirements

1. **An active secure.software Portal account and a Personal Access Token generated for it.** If you don't already have a Portal account, you may need to contact the administrator of your Portal organization to [invite you](https://docs.secure.software/portal/members#invite-a-new-member).
Alternatively, if you're not a secure.software customer yet, you can [contact ReversingLabs](https://docs.secure.software/portal/#get-access-to-securesoftware-portal) to sign up for a Portal account.
When you have an account set up, follow the instructions to [generate a Personal Access Token](https://docs.secure.software/api/generate-api-token).


**Note for GitHub Enterprise users:** GitHub Actions must be enabled and appropriately configured for the repository where you want to use this action.
If you don't have access to the [repository settings](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository),
contact your GitHub organization administrators for help.


## Environment variables

This action requires authentication to a secure.software Portal instance with a Personal Access Token.
The token must be passed via the environment using the following environment variables.


| Environment variable | Description  |
| :---------           | :------      |
| `RLPORTAL_ACCESS_TOKEN` | **Required.** A Personal Access Token for authenticating requests to the secure.software Portal. Before you can use this GitHub Action, you must [create the token](https://docs.secure.software/api/generate-api-token) in your Portal settings. Tokens can expire and be revoked, in which case you'll have to update the value of this environment variable. It's strongly recommended to treat this token as a secret and manage it according to your organization's security best practices. |


ReversingLabs strongly recommends [defining secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#using-encrypted-secrets-in-a-workflow) on the level of your GitHub organization or repository.


# How to use this GitHub Action

The most common use-case for this action is to add it to the "test" stage in a workflow, after the build artifact has been created.

To use the Portal security scanning functionality, an active account on a Portal instance is required, together with a Personal Access Token for Portal API authentication.


## Compare artifacts

To compare a new version of an artifact against a previously scanned version, you can use the `rl-diff-with` parameter when scanning the new version.
Both versions must be in the same Portal project and package.
This comparison is also known as the **diff scan**.

To perform the diff scan, specify the package URL (PURL) of the previously scanned version with the `rl-diff-with` parameter.
The action will verify that the requested artifact version was actually scanned before on the Portal, and ignore the request for a diff scan if there are no results for the requested PURL.

After a successful diff scan, the analysis report of the new artifact version will contain the Diff tab with all the differences between the two versions.
In the Portal web interface, the new version will be marked as "Derived" from the previous version.


## Optional proxy configuration

In some cases, proxy configuration may be required to access the internet and connect to a secure.software Portal instance.
You can configure proxy settings with the `rl-proxy-*` parameters for any self-hosted runner, including local GitHub Enterprise setups.

When using the `rl-proxy-server` parameter, you must also specify the port with `rl-proxy-port`.

If the proxy requires authentication, the proxy credentials for authentication can be configured with `rl-proxy-user` and `rl-proxy-password`.


## Inputs

| Input parameter     | Required | Description |
| :---------          | :------ | :------ |
| `artifact-to-scan`  | **Yes** | The build artifact you want to scan. Provide the artifact file path relative to the `github.workspace`. The file must be in any of the [formats supported by secure.software](https://docs.secure.software/concepts/reference). The file size on disk must not exceed 10 GB. |
| `rl-portal-server`  | **Yes** | Name of the secure.software Portal instance to use for the scan. The Portal instance name usually matches the subdirectory of `my.secure.software` in your Portal URL. For example, if your portal URL is `my.secure.software/demo`, the instance name to use with this parameter is `demo`. |
| `rl-portal-org`     | **Yes** | The name of a secure.software Portal organization to use for the scan. The organization must exist on the Portal instance specified with `rl-portal-server`. The user account authenticated with the token must be a member of the specified organization and have the appropriate permissions to upload and scan a file. Organization names are case-sensitive. |
| `rl-portal-group`   | **Yes** | The name of a secure.software Portal group to use for the scan. The group must exist in the Portal organization specified with `rl-portal-org`. Group names are case-sensitive. |
| `rl-package-url`    | **Yes**  | The package URL (PURL) used to associate the build artifact with a project and package on the Portal. PURLs are unique identifiers in the format `<project></package><@version>`. When scanning a build artifact, you must assign a PURL to it, so that it can be placed into the specified project and package as a version. If the project and package you specified don't exist in the Portal, they will be automatically created.  |
| `rl-diff-with`      | No  | This optional parameter lets you specify a previous version against which you want to compare (diff) the artifact version you're scanning. The specified version must exist in the same project and package as the artifact you're scanning. |
| `rl-timeout`        | No  | This optional parameter lets you specify how long to wait for analysis to complete before failing (in minutes). The parameter accepts any integer from 10 to 1440. The default timeout is 20 minutes. |
| `rl-submit-only`    | No  | Set to `true` to skip waiting for the analysis result. The default is `false`. |
| `rl-verbose`        | No  | Set to`true` to provide more feedback in the output while running the scan. The default is `false`. |
| `rl-proxy-server`   | No  | Server URL for proxy configuration (IP address or DNS name). |
| `rl-proxy-port`     | No  | Network port on the proxy server for proxy configuration. Required if `rl-proxy-server` is used. |
| `rl-proxy-user`     | No  | User name for proxy authentication. |
| `rl-proxy-password` | No  | Password for proxy authentication. Required if `rl-proxy-user` is used. |


## Outputs

| Output parameter | Description |
| :--------- | :------ |
| `description` | The result of the action - a string terminating in FAIL or PASS. |
| `status` | The single-word status (as is used by the GitHub Status API), representing the result of the action. It can be any of the following: success, failure, error. **Success** indicates that the resulting string contains PASS. **Failure** indicates the resulting string contains FAIL. **Error** indicates that something went wrong during the scan and the action was not able to retrieve the resulting string. |


# Examples

The following example is a basic GitHub workflow that runs on pull requests (PRs) and commit pushes to the `main` branch in your repository.

The workflow checks out your repository, builds an artifact,
and uses the `rl-scanner-cloud-composite` GitHub action to scan the artifact on the secure.software Portal.

When the scan is done, the GitHub status is updated.

Portal users can then view the analysis report and [manage the analyzed file](https://docs.secure.software/portal/projects#work-with-package-versions-releases) from the Portal web interface or via the Portal Public APIs like any other package version.


    name: ReversingLabs rl-scanner-cloud
    run-name: rl-scanner-cloud-composite

    on:
      push:
        branches: [ "main" ]
      pull_request:
        branches: [ "main" ]

    jobs:
      checkout-build-scan-action:
        runs-on: ubuntu-latest

        permissions:
          statuses: write
          pull-requests: write

        steps:
          # Need to check out data before we can do anything
          - uses: actions/checkout@v3

          # Replace this step with your build process.
          # Need to produce one file as the build artifact in scanfile=<relative file path>.
          - name: Create the build artifact
            id: build
            shell: bash
            run: |
              # Prepare the build process
              python3 -m pip install --upgrade pip
              pip install example
              python3 -m pip install --upgrade build
              # Run the build
              python3 -m build
              # Produce a single artifact to scan and set the scanfile output parameter
              echo "scanfile=$( ls dist/*.whl )" >> $GITHUB_OUTPUT

          # Use the rl-scanner-cloud-composite action
          - name: Scan build artifact on the Portal
            id: rl-scan
            env:
              RLPORTAL_ACCESS_TOKEN: ${{ secrets.RLPORTAL_ACCESS_TOKEN }}
            uses: reversinglabs/gh-action-rl-scanner-cloud-composite@v1
            with:
              artifact-to-scan: ${{ steps.build.outputs.scanfile }}


# Useful resources

- The official `reversinglabs/rl-scanner-cloud` Docker image [on Docker Hub](https://hub.docker.com/r/reversinglabs/rl-scanner-cloud)
- The official [secure.software Portal documentation](https://docs.secure.software/portal/)
- The [rl-scanner-cloud-only](https://github.com/reversinglabs/gh-action-rl-scanner-cloud-only) GitHub Action
- Introduction to [secure software release processes](https://www.reversinglabs.com/solutions/secure-software-release-processes) with ReversingLabs

