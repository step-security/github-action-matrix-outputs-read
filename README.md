[![StepSecurity Maintained Action](https://raw.githubusercontent.com/step-security/maintained-actions-assets/main/assets/maintained-action-banner.png)](https://docs.stepsecurity.io/actions/stepsecurity-maintained-actions)

# This is a fork to correct a misbehavior in cloudposse's standard release
The original action from cloudposse has a misbehavior where it's utilizing artifacts.  This artifact behavior is downloading *all* workflow artifacts rather than solely the artifact that corresponds to the action, which can then cause the workflow to fail.  This fork exists to fix that behavior.

<!-- markdownlint-disable -->
# github-action-matrix-outputs-read [![Latest Release](https://img.shields.io/github/release/step-security/github-action-matrix-outputs-read.svg)](https://github.com/step-security/github-action-matrix-outputs-read/releases/latest)
<!-- markdownlint-restore -->

[Workaround implementation](https://github.com/community/community/discussions/17245#discussioncomment-3814009) - Read matrix jobs outputs


---

It's 100% Open Source and licensed under the [APACHE2](LICENSE).












## Introduction

GitHub actions have an [Jobs need a way to reference all outputs of matrix jobs](https://github.com/community/community/discussions/17245) issue.
If there is a job that runs multiple times with `strategy.matrix` only the latest iteration's output availiable for 
reference in other jobs.

There is a [workaround](https://github.com/community/community/discussions/17245#discussioncomment-3814009) to address the limitation.
We implement the workaround with two GitHub Actions:
* [Matrix Outputs Write](https://github.com/step-security/github-action-matrix-outputs-write)
* [Matrix Outputs Read](https://github.com/step-security/github-action-matrix-outputs-read@v1)




## Usage



Example how you can use workaround to reference matrix job outputs.

```yaml
  name: Pull Request
  on:
    pull_request:
      branches: [ 'main' ]
      types: [opened, synchronize, reopened, closed, labeled, unlabeled]

  jobs:
    build:
      runs-on: ubuntu-latest
      strategy:
        matrix:
          platform: ["i386", "arm64v8"]
      steps:
        - name: Checkout
          uses: actions/checkout@v7

        - name: Build
          id: build
          uses: cloudposse/github-action-docker-build-push@v3
          with:
            registry: registry.hub.docker.com
            organization: "${{ github.event.repository.owner.login }}"
            repository: "${{ github.event.repository.name }}"
            build-args: |-
              PLATFORM=${{ matrix.platform }}

        ## Write for matrix outputs workaround 
        - uses: step-security/github-action-matrix-outputs-write@v1
          id: out
          with:
            matrix-step-name: ${{ github.job }}
            matrix-key: ${{ matrix.platform }}
            outputs: |-
              image: ${{ steps.build.outputs.image }}:${{ steps.build.outputs.tag }}

    ## Read matrix outputs 
    read:
      runs-on: ubuntu-latest
      needs: [build]
      steps:
        - uses: step-security/github-action-matrix-outputs-read@v1
          id: read
          with:
            matrix-step-name: build

      outputs:
        result: "${{ steps.read.outputs.result }}"

    ## This how you can reference matrix output
    assert:
      runs-on: ubuntu-latest
      needs: [read]
      steps:
        - uses: nick-fields/assert-action@v4
          with:
            expected: ${{ registry.hub.docker.com }}/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}:i386
            ## This how you can reference matrix output
            actual: ${{ fromJson(needs.read.outputs.result).image.i386 }}

        - uses: nick-fields/assert-action@v4
          with:
            expected: ${{ registry.hub.docker.com }}/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}:arm64v8
            ## This how you can reference matrix output
            actual: ${{ fromJson(needs.read.outputs.result).image.arm64v8 }}
```

### Reusable workflow example 

Reusable workflow that support matrix outputs

`./.github/workflow/build-reusabled.yaml`

```yaml
name: Build - Reusable workflow
on:
  workflow_call:
    inputs:
      registry:
        required: true
        type: string
      organization:
        required: true
        type: string
      repository:
        required: true
        type: string
      platform:
        required: true
        type: string
      matrix-step-name:
        required: false
        type: string
      matrix-key:
        required: false
        type: string
    outputs:
      image:
        description: "Image"
        value: ${{ jobs.write.outputs.image }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Build
        id: build
        uses: cloudposse/github-action-docker-build-push@v3
        with:
          registry: ${{ inputs.registry }}
          organization: ${{ inputs.organization }}
          repository: ${{ inputs.repository }}
          build-args: |-
            PLATFORM=${{ inputs.platform }}
    outputs:
      image: ${{ needs.build.outputs.image }}:${{ needs.build.outputs.tag }}

  write:
    runs-on: ubuntu-latest
    needs: [build]
    steps:        
      ## Write for matrix outputs workaround 
      - uses: step-security/github-action-matrix-outputs-write@v1
        id: out
        with:
          matrix-step-name: ${{ inputs.matrix-step-name }}
          matrix-key: ${{ inputs.matrix-key }}
          outputs: |-
            image: ${{ needs.build.outputs.image }}

    outputs:
      image: ${{ fromJson(steps.out.outputs.result).image }}
```

Then you can use the workflow with matrix

```yaml
name: Pull Request
on:
  pull_request:
    branches: [ 'main' ]
    types: [opened, synchronize, reopened, closed, labeled, unlabeled]

jobs:
  build:
    usage: ./.github/workflow/build-reusabled.yaml
    strategy:
      matrix:
        platform: ["i386", "arm64v8"]
    with:
      registry: registry.hub.docker.com
      organization: "${{ github.event.repository.owner.login }}"
      repository: "${{ github.event.repository.name }}"
      platform: ${{ matrix.platform }}
      matrix-step-name: ${{ github.job }}
      matrix-key: ${{ matrix.platform }}

  ## Read matrix outputs 
  read:
    runs-on: ubuntu-latest
    needs: [build]
    steps:
      - uses: step-security/github-action-matrix-outputs-read@v1
        id: read
        with:
          matrix-step-name: build

    outputs:
      result: "${{ steps.read.outputs.result }}"

  ## This how you can reference matrix output
  assert:
    runs-on: ubuntu-latest
    needs: [read]
    steps:
      - uses: nick-fields/assert-action@v4
        with:
          expected: ${{ registry.hub.docker.com }}/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}:i386
          ## This how you can reference matrix output
          actual: ${{ fromJson(needs.read.outputs.result).image.i386 }}

      - uses: nick-fields/assert-action@v4
        with:
          expected: ${{ registry.hub.docker.com }}/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}:arm64v8
          ## This how you can reference matrix output
          actual: ${{ fromJson(needs.read.outputs.result).image.arm64v8 }}
```

or as a simple job

```yaml
name: Pull Request
on:
  pull_request:
    branches: [ 'main' ]
    types: [opened, synchronize, reopened, closed, labeled, unlabeled]

jobs:
  build:
    usage: ./.github/workflow/build-reusabled.yaml
    with:
      registry: registry.hub.docker.com
      organization: "${{ github.event.repository.owner.login }}"
      repository: "${{ github.event.repository.name }}"
      platform: "i386"

  ## This how you can reference single job output
  assert:
    runs-on: ubuntu-latest
    needs: [build]
    steps:
      - uses: nick-fields/assert-action@v4
        with:
          expected: ${{ registry.hub.docker.com }}/${{ github.event.repository.owner.login }}/${{ github.event.repository.name }}:i386
          ## This how you can reference matrix output
          actual: ${{ needs.build.outputs.image }}
```






<!-- markdownlint-disable -->

## Inputs

| Name | Description | Default | Required |
|------|-------------|---------|----------|
| matrix-step-name | Matrix step name | N/A | true |


## Outputs

| Name | Description |
|------|-------------|
| result | Outputs result |
<!-- markdownlint-restore -->



## Related Projects

Check out these related projects.

- [github-action-matrix-outputs-write](https://github.com/step-security/github-action-matrix-outputs-write) - Matrix outputs write


## References

For additional context, refer to some of these links.

- [github-actions-workflows](https://github.com/cloudposse/github-actions-workflows) - Reusable workflows for different types of projects
- [example-github-action-release-workflow](https://github.com/cloudposse/example-github-action-release-workflow) - Example application with complicated release workflow



## Copyright

Copyright © 2017-2024 [Cloud Posse, LLC](https://cpco.io/copyright)
Copyright (c) 2026 StepSecurity



## License

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

See [LICENSE](LICENSE) for full details.

```text
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
```









## Trademarks

All other trademarks referenced herein are the property of their respective owners.
