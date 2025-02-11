# Qodana Scan [<img src="https://api.producthunt.com/widgets/embed-image/v1/top-post-badge.svg?post_id=304841&theme=dark&period=daily" alt="" align="right" width="190" height="41">](https://www.producthunt.com/posts/jetbrains-qodana)

[![official JetBrains project](https://jb.gg/badges/official.svg)][jb:confluence-on-gh]
[![GitHub Discussions](https://img.shields.io/github/discussions/jetbrains/qodana)][jb:discussions]
[![Twitter Follow](https://img.shields.io/twitter/follow/Qodana?style=social&logo=twitter)][jb:twitter]

**Qodana** is a code quality monitoring tool that identifies and suggests fixes for bugs, security vulnerabilities,
duplications, and imperfections. Using this GitHub Action, run Qodana with your GitHub workflow to scan your Java,
Kotlin, PHP, Python, JavaScript, TypeScript projects (
and [other supported technologies by Qodana](https://www.jetbrains.com/help/qodana/supported-technologies.html)).

**Table of Contents**

<!-- toc -->

- Qodana Scan
    - [Usage](#usage)
    - [Configuration](#configuration)
    - [Issue Tracker](#issue-tracker)
    - [License](#license)

<!-- tocstop -->

## Usage

### Basic configuration

To configure Qodana Scan, save the `.github/workflows/code_quality.yml` file containing the workflow configuration:

```yaml
name: Qodana
on:
  workflow_dispatch:
  pull_request:
  push:
    branches:
      - main
      - 'releases/*'

jobs:
  qodana:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: 'Qodana Scan'
        uses: JetBrains/qodana-action@v4.2.2
        with:
          linter: jetbrains/qodana-<linter>
```

Using this workflow, Qodana will run on the main branch, release branches, and on the pull requests coming to your
repository. Inspection results will be available in the GitHub UI. The `jetbrains/qodana-<linter>` option specifies a
[Qodana linter](linters.md).

We recommend that you have a separate workflow file for Qodana
because [different jobs run in parallel](https://help.github.com/en/actions/getting-started-with-github-actions/core-concepts-for-github-actions#job)
.

### GitHub code scanning

You can set
up [GitHub code scanning](https://docs.github.com/en/code-security/code-scanning/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning)
for your project using Qodana. To do it, add these lines to the `code_quality.yml` workflow file right below
the [basic configuration](#how-to-start-github-action) of Qodana Scan:

```yaml
      - uses: github/codeql-action/upload-sarif@v1
        with:
          sarif_file: ${{ runner.temp }}/qodana/results/qodana.sarif.json
```

This sample invokes `codeql-action` for uploading a SARIF-formatted Qodana report to GitHub, and specifies the report
file using the `sarif_file` key.

> 💡 GitHub code scanning does not export inspection results to third-party tools, which means that you cannot use this data for further processing by Qodana. In this case, you have to set up baseline and quality gate processing on the Qodana side prior to submitting inspection results to GitHub code scanning, see the
<a href="qodana-github-action.md" anchor="github-actions-quality-gate-baseline">Quality gate and baseline</a> section for details.

### Pull request quality gate

You can enforce GitHub to block the merge of pull requests if the Qodana quality gate has failed. To do it, create a
[branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule)
as described below:

1. Create a new or open an existing GitHub workflow that invokes the Qodana Scan action.
2. Set the workflow to run on `pull_request` events that target the `main` branch.

```yaml
on:
  pull_request:
    branches:
      - main
```

Instead of `main`, you can specify your branch here.

3. Set the number of problems (integer) for the Qodana action `fail-threshold` option.
4. Under your repository name, click **Settings**.
5. On the left menu, click **Branches**.
6. In the branch protection rules section, click **Add rule**.
7. Add `main` to **Branch name pattern**.
8. Select **Require status checks to pass before merging**.
9. Search for the `Qodana` status check, then check it.
10. Click **Create**.

<anchor name="github-actions-quality-gate-baseline"/>

### Quality gate and baseline

You can combine the [quality gate](quality-gate.xml) and [baseline](qodana-baseline.xml) features to manage your
technical debt, report only new problems, and block pull requests that contain too many problems.

Follow these steps to establish a baseline for your project:

1. Run Qodana [locally](docker-images.md) over your project:

```shell
docker run --rm -v <source-directory>/:/data/project/ \
  -p 8080:8080 jetbrains/qodana-<linter> --show-report
```

2. Open your report at `http://localhost:8080/`, [add detected problems](ui-overview.md#Technical+debt) to the baseline,
   and download the `qodana.sarif.json` file.

3. Upload the `qodana.sarif.json` file to your project root folder on GitHub.

4. Append this line to the Qodana Scan action configuration in the `code_quality.yml` file:

```yaml
baseline-path: qodana.sarif.json; 
```

If you want to update the baseline, you need to repeat these steps once again.

Starting from this, GitHub will generate alters only for the problems that were not added to the baseline as new.

To establish a quality gate additionally to the baseline, add this line to `code_quality.yml` right after the
`baseline-path` line:

```yaml
fail-threshold: <number-of-accepted-problems>
```

Based on this, you will be able to detect only new problems in pull requests that fall beyond the baseline. At the same
time, pull requests with **new** problems exceeding the `fail-threshold` limit will be blocked and the workflow will
fail.

### GitHub Pages

If you wish to study [Qodana reports](https://www.jetbrains.com/help/qodana/html-report.html) directly on GitHub, you
can host them on your [GitHub Pages](https://docs.github.com/en/pages) repository using this example workflow:

```yaml
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ${{ runner.temp }}/qodana/results/report
          destination_dir: ./
```

<note>Hosting of multiple Qodana reports in a single GitHub Pages repository is not supported.</note>

### Get a Qodana badge

You can set up a Qodana workflow badge in your repository:

[![Qodana](https://github.com/JetBrains/qodana-action/actions/workflows/code_scanning.yml/badge.svg)](https://github.com/JetBrains/qodana-action/actions/workflows/code_scanning.yml)

To do it, follow these steps:

1. Navigate to the workflow run that you previously configured.
2. On the workflow page, select **Create status badge**.
3. Copy the Markdown text to your repository README file.

<img src="https://user-images.githubusercontent.com/13538286/148529278-5d585f1d-adc4-4b22-9a20-769901566924.png" alt="Creating status badge" width="706"/>

## Configuration

| Name                       | Description                                                                                                                        | Default Value                           |
|----------------------------|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| `linter`                   | [Official Qodana Docker image](https://www.jetbrains.com/help/qodana/docker-images.html). Required.                                | `jetbrains/qodana-jvm-community:latest` |
| `project-dir`              | The project's root directory to be analyzed. Optional                                                                              | `${{ github.workspace }}`               |
| `results-dir`              | Directory to store the analysis results. Optional.                                                                                 | `${{ runner.temp }}/qodana/results`     |
| `cache-dir`                | Directory to store Qodana caches. Optional.                                                                                        | `${{ runner.temp }}/qodana/caches`      |
| `idea-config-dir`          | IntelliJ IDEA configuration directory. Optional.                                                                                   | -                                       |
| `gradle-settings-path`     | Provide path to gradle.properties file. An example: "/your/custom/path/gradle.properties". Optional.                               | -                                       |
| `additional-volumes`       | Mount additional volumes to Docker container. Multiline input variable: specify multiple values with newlines. Optional.                                                                            | -                                       |
| `additional-env-variables` | Pass additional environment variables to docker container. Multiline input variable: specify multiple values with newlines. Optional.                                                               | -                                       |
| `fail-threshold`           | Set the number of problems that will serve as a quality gate. If this number is reached, the pipeline run is terminated. Optional. | -                                       |
| `inspected-dir`            | Directory to be inspected. If not specified, the whole project is inspected by default. Optional.                                  | -                                       |
| `baseline-path`            | Run in baseline mode. Provide the path to an existing SARIF report to be used in the baseline state calculation. Optional.         | -                                       |
| `baseline-include-absent`  | Include the results from the baseline absent in the current Qodana run in the output report. Optional.                             | `false`                                 |
| `changes`                  | Inspect uncommitted changes and report new problems. Optional.                                                                     | `false`                                 |
| `script`                   | Override the default docker scenario. Optional.                                                                                    | -                                       |
| `profile-name`             | Name of a profile defined in the project. Optional.                                                                                | -                                       |
| `profile-path`             | Absolute path to the profile file. Optional.                                                                                       | -                                       |
| `token`                    | Qodana Cloud token, if specified, the report will be sent to Qodana Cloud. Optional.                                                  | -                                       |
| `upload-result`            | Upload Qodana results as an artifact to the job. Optional.                                                                         | `true`                                  |
| `artifact-name`            | Specify Qodana results artifact name, used for results uploading. Optional.                                                        | `Qodana report`                                  |
| `use-caches`               | Utilize GitHub caches for Qodana runs. Optional.                                                                                   | `true`                                  |
| `additional-cache-hash`    | Allows customizing the generated cache hash. Optional.                                                                             |                                         `${{ github.sha }}` |
| `use-annotations`          | Use annotation to mark the results in the GitHub user interface. Optional.                                                         | `true`                                  |
| `github-token`             | GitHub token to be used for uploading results. Optional.                                                                           | `${{ github.token }}`                   |

## Issue Tracker

All the issues, feature requests, and support related to the Qodana GitHub Action are handled on [YouTrack][youtrack].

If you'd like to file a new issue, please use the link [YouTrack | New Issue][youtrack-new-issue].

## License

### The GitHub Action repository

This repository contains source code for Qodana GitHub Action and is licensed under [Apache-2.0](./LICENSE).

### Qodana Docker images

#### Qodana Community images

View [license information](https://www.jetbrains.com/legal/?fromFooter#licensing) for the Qodana Community images.

Qodana Docker images may contain other software which is subject to other licenses, for example, Bash relating to the
base distribution or with any direct or indirect dependencies of the primary software.

As for any pre-built image usage, it is the image user's responsibility to ensure that any use of this image complies
with any relevant licenses for all software contained within.

#### Qodana EAP images

Using the Qodana EAP Docker images, you agree
to [JetBrains EAP user agreement](https://www.jetbrains.com/legal/docs/toolbox/user_eap/)
and [JetBrains privacy policy](https://www.jetbrains.com/legal/docs/privacy/privacy/). The docker image includes an
evaluation license which will expire in 30-day. Please ensure you pull a new image on time.

[gh:qodana]: https://github.com/JetBrains/qodana-action/actions/workflows/code_scanning.yml

[youtrack]: https://youtrack.jetbrains.com/issues/QD

[youtrack-new-issue]: https://youtrack.jetbrains.com/newIssue?project=QD&c=Platform%20GitHub%20Action

[jb:confluence-on-gh]: https://confluence.jetbrains.com/display/ALL/JetBrains+on+GitHub

[jb:discussions]: https://jb.gg/qodana-discussions

[jb:twitter]: https://twitter.com/Qodana

[jb:docker]: https://hub.docker.com/r/jetbrains/qodana
