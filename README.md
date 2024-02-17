# _OCI charts releaser_ Gitea Action

A Gitea action for single chart or multi-chart repositories that performs push and gitea releases creation for the hosted charts.

## Usage

### Pre-requisites

1. A Gitea repo containing a directory with your Helm charts (one of the following folders named `/charts`, `/chart` or `helm`, if you want
   to maintain your charts in a different directory, you must include a `charts_dir` input in the workflow).
1. Create a workflow `.yml` file in your `.gitea/workflows` directory. An [example workflow](#example-workflow) is available below.
   For more information, reference the GitHub Help Documentation for [Creating a workflow file](https://help.github.com/en/articles/configuring-a-workflow#creating-a-workflow-file)

### Inputs

- `gitea_server`: The base url of the gitea API (default: https://gitea.com)
- `version`: The helm version to use (default: v3.14.1)
- `charts_dir`: The charts directory.
- **`oci_registry`**: The OCI registry host.
- **`oci_username`**: The username used to login to the OCI registry.
- **`oci_password`**: The OCI user's password.
- **`gitea-token`**: Gitea Actions token must be provided to manage release creation and update.
- `tag_name_pattern`: Specifies Gitea repository release naming pattern (ex. '{chartName}-chart'). For instance you chart is named as app, but you want it to be released as *app-chart-x.y.z*, use *tag_name_pattern* `{chartName}-chart`.
- `skip_helm_install`: Skip helm installation (default: false).
- `skip_dependencies`: Skip dependencies update from "Chart.yaml" to dir "charts/" before packaging (default: false).
- `skip_existing`: Skip the chart push if the Gitea release exists.

### Outputs

- `released_charts`: A comma-separated list of charts that were released on this run. Will be an empty string if no updates were detected, will be unset if `--skip_packaging` is used: in the latter case your custom packaging step is responsible for setting its own outputs if you need them.
- `chart_version`: The version of the most recently generated charts; will be set even if no charts have been updated since the last run.

### Example Workflow

Create a workflow (eg: `.gitea/workflows/helm-release.yml`):

```yaml
name: Release Charts

on:
  push:
    branches:
      - main
    paths:
      - 'helm/**'
      - 'charts/**'
      - 'chart/**'

env:
  REGISTRY_HOST: gitea.domain.tld
  REGISTRY_USERNAME: ${{ gitea.repository_owner }}
  REGISTRY_PASSWORD: ${{ secrets.PAT }}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "${{ gitea.actor }}"
          git config user.email "${{ gitea.actor }}@users.noreply.${{ env.REGISTRY_HOST }}"

      - name: Run chart-releaser
        uses: https://github.com/thpham/helm-oci-charts-releaser@v1
        with:
          oci_registry: ${{ env.REGISTRY_HOST }}/${{ gitea.repository_owner }}
          oci_username: ${{ gitea.repository_owner }}
          oci_password: ${{ secrets.PAT }}
          gitea_server: ${{ env.REGISTRY_HOST }}
          gitea_token: ${{ secrets.GITEA_TOKEN }}
          tag_name_pattern: '{chartName}-chart'
```

This uses under the hood uses Helm and [tea cli](https://gitea.com/gitea/tea) (which is available to actions). Helm is used to login and push charts into an OCI registry, while tea cli is used to create and update the repository releases.

It does this – during every push to `main` – by checking each chart in your project, and whenever there's a new chart version, creates a corresponding [Gitea release](https://help.github.com/en/github/administering-a-repository/about-releases) named for the chart version, adds Helm chart artifacts to the release, and pushes the chart into the given OCI registry.
