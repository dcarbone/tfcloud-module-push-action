# Terraform Cloud Module Push Action
GitHub Action to push new module versions to a Terraform Cloud registry.

# Inputs

## Required

* `artifact-file` - Path to artifact file
* `token` - Terraform Cloud API Token with at least "Manage Modules" permissions.
* `organization` - Name of Terraform Cloud organization
* `namespace` - Your organization's namespace
* `module-name` - Name of module
* `provider-name` Name of primary provider used by module
 
## Optional 

* `registry-name` - Name of Registry to push to
  * Defaults to `private`
* `version` - Version to create in Terraform Cloud
  * If left empty, produces version from tag using
    ```
    "$(echo -n "${{ github.ref_name }}" | sed 's/^v//g')"
    ```

# Outputs

* `version` - JSON response from:
  * https://www.terraform.io/cloud-docs/api-docs/private-registry/modules#create-a-module-version
* `errors` - Any error seen during execution

# Example

```yaml
name: "Create Module Version"

on:
  push:
    tags:
      - 'v*'

jobs:
  create-version:
    runs-on: ubuntu-latest
    steps:
      # check out module code
      - uses: actions/checkout@v3
      
      # create tar.gz artifact of the module, excluding any non-essential file
      # see https://www.terraform.io/cloud-docs/api-docs/private-registry/modules#add-a-module-version-private-module
      - name: Create artifact
        id: artifact
        run: |
          # create temporary directory to house tar
          mkdir release
          # artifact filename
          _artifact_file="$(pwd)/release/${{ github.ref_name }}.tar.gz"
          
          # create artifact
          # add / remove --excludes / --includes as needed
          tar \
            -zcvf "${_artifact_file}" \
            --exclude .git \
            --exclude .github \
            --exclude .gitignore \
            --exclude release \
            --exclude tests \
            --exclude .terraform \
            .
          
          # set output value for this step
          echo "::set-output name=artifact_filepath::${_artifact_file}"
      
      # utilize this action to push new module version to Terraform Cloud
      - name: "Create module version"
        if: ${{ success() }}
        uses: dcarbone/tfcloud-module-push-action@v0.1.4
        with:
          artifact-file: ${{ steps.artifact.outputs.artifact_filepath }}
          token: ${{ secrets.TFCLOUD_API_KEY }}
          organization: { your organization }
          namespace: { your organization namespace }
          module-name: { name of module }
          provider-name: { name of primary provider used by module }
```
