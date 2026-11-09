<p align="center">
    <img src="./.github/assets/logo.svg" alt="github-runner" width="128" height="128" />
</p>

<h1 align="center">github-runner</h1>

<p align="center">
    <b>The actions-runner image plus libatomic1 and the GitHub CLI, for self-hosted runners on actions-runner-controller.</b><br>
    You stop adding an <code>apt-get</code> step for Node 26 and a <code>curl</code> and <code>jq</code> replacement for <code>gh api</code> to every workflow.<br>
    The image tag is the runner version, and the two additions are all that changes.
</p>

<p align="center">
    <a href="https://github.com/tada5hi/github-runner/actions/workflows/image.yml"><img src="https://github.com/tada5hi/github-runner/actions/workflows/image.yml/badge.svg" alt="Image workflow status" /></a>
</p>

<p align="center">
    <a href="#usage"><b>Usage</b></a>
    ·
    <a href="#whats-in-the-image">What's in the image</a>
    ·
    <a href="#how-it-is-built">How it is built</a>
    ·
    <a href="#updating">Updating</a>
    ·
    <a href="#limits">Limits</a>
</p>

---

## Why github-runner?

The upstream image, `ghcr.io/actions/actions-runner`, is deliberately minimal. Two gaps show up as soon as your workflows expect what a GitHub-hosted runner has: Node 26 from `actions/setup-node` exits 127 with `error while loading shared libraries: libatomic.so.1`, and `gh` is absent.

| Concern | `actions-runner` | `github-runner` |
|---|---|---|
| **Node 26 from `actions/setup-node`** | Exits 127 until each workflow installs `libatomic1` with `apt-get` | `libatomic1` is in the image |
| **`gh` in a step** | Not installed, so each `gh api` call becomes `curl` and `jq` | GitHub CLI 2.103.3 is on the `PATH`, its tarball checked against a pinned sha256 |
| **Image tag** | The runner version | The same runner version, taken from the `FROM` line |
| **Anything else** | Nothing added | Nothing added: no `yq`, no language toolchains |

GitHub-hosted runner images have both tools, but they are large VM images and not a container you can pull. If your jobs run on GitHub-hosted runners, you do not need this image.

The same job on both images:

```yaml
# ghcr.io/actions/actions-runner
- run: sudo apt-get update && sudo apt-get install -y libatomic1
- uses: actions/setup-node@v7
  with:
      node-version: 26
- run: |
      curl -fsSL -H "Authorization: Bearer $GH_TOKEN" \
          "https://api.github.com/repos/$GITHUB_REPOSITORY/pulls" | jq -r '.[].number'
  env:
      GH_TOKEN: ${{ github.token }}
```

```yaml
# ghcr.io/tada5hi/github-runner
- uses: actions/setup-node@v7
  with:
      node-version: 26
- run: gh api "repos/$GITHUB_REPOSITORY/pulls" --jq '.[].number'
  env:
      GH_TOKEN: ${{ github.token }}
```

<p align="center">
    <img src="./.github/assets/build.svg" alt="The Dockerfile adds libatomic1 and GitHub CLI 2.103.3 to actions-runner 2.338.2. The image workflow builds it, checks both additions, and, when release-please creates a release, publishes ghcr.io/tada5hi/github-runner with the runner version as tag. A gha-runner-scale-set uses that tag on the runner and init-dind-externals containers. Each actions/runner release starts a 30-day window: Dependabot opens a pull request for the FROM line, merging it and then the release pull request publishes the new tag, and you update the tag in your deployment. After day 30 GitHub stops queuing jobs and they stay queued. A critical security update can stop queuing without that window." width="100%" />
</p>

## Usage

The image is public and needs no pull secret:

```bash
docker pull ghcr.io/tada5hi/github-runner:2.338.2
```

To see both additions, run the check the build runs:

```bash
docker run --rm --entrypoint sh ghcr.io/tada5hi/github-runner:2.338.2 \
    -c 'gh --version && ldconfig -p | grep libatomic.so.1'
```

### With actions-runner-controller

Set the image on the `runner` container and on the `init-dind-externals` init container in the `template.spec` of the `gha-runner-scale-set` chart:

```yaml
githubConfigUrl: https://github.com/your-org/your-repo
githubConfigSecret: your-github-secret
template:
    spec:
        initContainers:
            - name: init-dind-externals
              image: ghcr.io/tada5hi/github-runner:2.338.2
              command: [cp]
              args: [-r, /home/runner/externals/., /home/runner/tmpDir/]
              volumeMounts:
                  - name: dind-externals
                    mountPath: /home/runner/tmpDir
            - name: dind
              image: docker:29.9.3-dind
              restartPolicy: Always
              args:
                  - dockerd
                  - --host=unix:///var/run/docker.sock
                  - --group=$(DOCKER_GROUP_GID)
              env:
                  - name: DOCKER_GROUP_GID
                    value: "123"
              securityContext:
                  privileged: true
              startupProbe:
                  exec:
                      command: [docker, info]
                  failureThreshold: 24
                  periodSeconds: 5
              volumeMounts:
                  - name: work
                    mountPath: /home/runner/_work
                  - name: dind-sock
                    mountPath: /var/run
                  - name: dind-externals
                    mountPath: /home/runner/externals
        containers:
            - name: runner
              image: ghcr.io/tada5hi/github-runner:2.338.2
              command: [/home/runner/run.sh]
              env:
                  - name: DOCKER_HOST
                    value: unix:///var/run/docker.sock
                  - name: RUNNER_WAIT_FOR_DOCKER_IN_SECONDS
                    value: "120"
              volumeMounts:
                  - name: work
                    mountPath: /home/runner/_work
                  - name: dind-sock
                    mountPath: /var/run
        volumes:
            - name: work
              emptyDir: {}
            - name: dind-sock
              emptyDir: {}
            - name: dind-externals
              emptyDir: {}
```

- **Leave `containerMode` empty.** With `containerMode.type: dind`, chart 0.15.1 hardcodes its own pod spec, including an unpinned `docker:dind`. The block above is the same pod written out, which is what the chart's `values.yaml` tells you to do for any customization.
- **Use the same tag in both places.** `init-dind-externals` copies `/home/runner/externals` out of the image, so it has to match the runner that uses it.
- **`command` and `DOCKER_HOST` are required on `runner`:** the command is `/home/runner/run.sh` and `DOCKER_HOST` is `unix:///var/run/docker.sock`.
- **Pin the `docker` image yourself.** `29.9.3-dind` is the version this example was written against, not something this repository tracks.

## What's in the image

| Part | Version | Source |
|---|---|---|
| Runner and its base | 2.338.2 | `FROM ghcr.io/actions/actions-runner:2.338.2` |
| `libatomic1` | Whatever the base image's Ubuntu ships at build time | `apt-get`, not pinned |
| GitHub CLI | 2.103.3, in `/usr/local/bin` | Release tarball, sha256 pinned in the `Dockerfile` |

The container runs as user `runner`. From the base image you also get `git`, `curl`, `jq`, `unzip`, `sudo` (passwordless for `runner`), and the Docker CLI with `buildx`.

## How it is built

The workflow in [`.github/workflows/image.yml`](.github/workflows/image.yml) runs on a GitHub-hosted runner:

| Event | Build | Check `gh --version` and `libatomic.so.1` | release-please | Publish |
|---|---|---|---|---|
| Push to `main` | Yes | Yes | Yes | Only when this push creates a release |
| Manual run (`workflow_dispatch`) | Yes | Yes | No | No |
| Pull request | Yes | Yes | No | No |

[release-please](https://github.com/googleapis/release-please) versions this repository, with its own changelog and GitHub releases. Merging its release pull request creates a release, and only that publishes an image. Those release versions do not appear in the image tag, which is always the runner version.

[craneship](https://github.com/tada5hi/craneship) publishes the image. The only tag is `ghcr.io/tada5hi/github-runner:<runner version>`, and the version is read from the `FROM` line of the [`Dockerfile`](Dockerfile). There is no `latest`. The workflow then pulls the published tag and runs the same check on it.

## Updating

Dependabot opens pull requests for the `FROM` line (checked daily) and for the GitHub Actions in the workflow (checked weekly). The `FROM` pull requests use a `fix:` commit prefix, so a new runner version shows up in the changelog and a release.

**A new runner release:**

1. Merge Dependabot's pull request for the `FROM` line.
2. Merge the release pull request that release-please then opens. The release publishes the new tag.
3. Set the new tag in your deployment, on both `runner` and `init-dind-externals`.

Finish all three within 30 days of the release, see [Limits](#limits).

**A new GitHub CLI release:** Dependabot does not see `GH_VERSION` and `GH_SHA256`, so you update both by hand. Take the checksum for `linux_amd64.tar.gz` from the `gh_<version>_checksums.txt` file of the [GitHub CLI release](https://github.com/cli/cli/releases):

```bash
curl -fsSL https://github.com/cli/cli/releases/download/v2.103.3/gh_2.103.3_checksums.txt | grep linux_amd64.tar.gz
```

## Limits

- **You have 30 days from each actions/runner release.** [GitHub stops queuing jobs](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/autoscaling-with-self-hosted-runners) for a runner more than 30 days behind a new release, so the `FROM` bump and its release must be merged and your deployment's tag updated within that window. The symptom is jobs that stay queued.
- **A critical security update has no 30 days.** GitHub does not queue jobs for a runner until a required critical security update is applied, so the same steps can be urgent.
- **A tag can be replaced.** Every release pushes to the runner version tag, so a release on the same runner version, such as a `GH_VERSION` bump, overwrites the existing tag. A node that already pulled it keeps its copy unless the pod uses `imagePullPolicy: Always`. Pin the digest if you need a fixed image.
- **amd64 only.** The `Dockerfile` downloads the `linux_amd64` CLI tarball and the workflow builds one platform.
- **Nothing beyond the two additions.** There is no `yq` and there are no language toolchains. Jobs that need more install it themselves.
- **The image is not a security boundary.** Docker-in-Docker runners run the `dind` container privileged, which is a property of your deployment and not of this image. Treat a job on such a runner as having control of the node.
