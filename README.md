# runner-image

`ghcr.io/tada5hi/runner`: the [actions-runner](https://github.com/actions/runner) image plus `libatomic1` (Node 26) and the GitHub CLI, for self-hosted runners on actions-runner-controller.

The image tag is the runner version in the `FROM` line of the `Dockerfile`. A push to `main` builds and publishes it. GitHub stops queuing jobs for a runner more than 30 days behind a new release, so merge Dependabot's `FROM` bump and update the deployment within that window.

`GH_VERSION` and `GH_SHA256` are updated by hand. Take the checksum from the `gh_<version>_checksums.txt` file of the [GitHub CLI release](https://github.com/cli/cli/releases).
