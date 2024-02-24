# Github Workflows

Reusable workflows and actions

## Opinionated publish process for npm

> github is the source of truth for code AND releases. Get the version/tag/release right on github, then publish to npm based on that.

![](./images/plugin-release.png)

1. work on a feature branch, commiting with conventional-commits
2. merge to main
3. A push to main produces (if your commits have `fix:` or `feat:`) a bumped package.json and a tagged github release via `githubRelease`
4. A release cause `npmPublish` to run.

### githubRelease

> creates a github release based on conventional commit prefixes. Using commits like `fix: etc` (patch version) and `feat: wow` (minor version).
> A commit whose **body** (not the title) contains `BREAKING CHANGES:` will cause the action to update the packageVersion to the next major version, produce a changelog, tag and release.

```yml
name: create-github-release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: SimplySF/github-workflows/.github/workflows/create-github-release.yml@main
    secrets: inherit
    # you can also pass in values for the secrets
    # secrets:
    #  PERSONAL_ACCESS_TOKEN: gh_pat00000000
```

### Prereleases

`main` will release to `latest`. Other branches can create github prereleases and publish to other npm dist tags.

You can create a prerelease one of two ways:

1. Create a branch with the `prerelease/**` prefix. Example `prerelease/my-fix`
   1. Once a PR is opened, every commit pushed to this branch will create a prerelease
   2. The default prerelease tag will be `dev`. If another tag is desired, manually set it in your `package.json`. Example: `1.2.3-beta.0`
1. Manually run the `create-github-release` workflow in the Actions tab
   1. Click `Run workflow`
      1. Select the branch you want to create a prerelease from
      1. Enter the desired prerelease tag: `dev`, `beta`, etc

> [!NOTE]  
> Since conventional commits are used, there is no need to manually remove the prerelease tag from your `package.json`. Once the PR is merged into `main`, conventional commits will bump the version as expected (patch for `fix:`, minor for `feat:`, etc)

Setup:

1. Configure the branch rules for wherever you want to release from
1. Modify your release and publish workflows like the following

```yml
name: create-github-release

on:
  push:
    branches:
      - main
      # point at specific branches, or a naming convention via wildcard
      - prerelease/**
    tags-ignore:
      - "*"
  workflow_dispatch:
    inputs:
      prerelease:
        type: string
        description: "Name to use for the prerelease: beta, dev, etc. NOTE: If this is already set in the package.json, it does not need to be passed in here."

jobs:
  release:
    uses: SimplySF/github-workflows/.github/workflows/create-github-release.yml@main
    secrets: inherit
    with:
      prerelease: ${{ inputs.prerelease }}
      # If this is a push event, we want to skip the release if there are no semantic commits
      # However, if this is a manual release (workflow_dispatch), then we want to disable skip-on-empty
      # This helps recover from forgetting to add semantic commits ('fix:', 'feat:', etc.)
      skip-on-empty: ${{ github.event_name == 'push' }}
```

## Opinionated Testing Process

Write unit tests to tests units of code (a function/method).

Write not-unit-tests to tests larger parts of code (a command) against real environments/APIs.

Run the UT first (faster, less expensive for infrastructure/limits).

```yml
name: tests
on:
  push:
    branches-ignore: [main]
  workflow_dispatch:

jobs:
  unit-tests:
    uses: SimplySF/github-workflows/.github/workflows/unitTest.yml@main
  nuts:
    needs: unit-tests
    uses: SimplySF/github-workflows/.github/workflows/nut.yml@main
    secrets: inherit
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
      fail-fast: false
    with:
      os: ${{ matrix.os }}
```
