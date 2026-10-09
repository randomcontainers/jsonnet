# jsonnet

Container images with go-jsonnet, the Go implementation of [Jsonnet](https://jsonnet.org/). Jsonnet is a configuration language that extends JSON with variables, functions, imports and object inheritance. The images contain four tools: `jsonnet`, `jsonnetfmt`, `jsonnet-lint` and `jsonnet-deps`. They are compiled from the release's source tarball with a current Go release on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when go-jsonnet publishes a release, when the pinned Go release is raised and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Jsonnet project or Google. Report problems with the image in this repository and problems with go-jsonnet itself [upstream](https://github.com/google/go-jsonnet/issues).

## Quick start

Save this as `greeting.jsonnet`:

```jsonnet
local who = 'world';

{
  greeting: 'hello ' + who,
  length: std.length(who),
}
```

Evaluate it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/jsonnet greeting.jsonnet
```

```json
{
   "greeting": "hello world",
   "length": 5
}
```

A few more options:

```sh
# Libraries from vendor/ and lib/; with several -J, the right-most wins
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/jsonnet \
  -J vendor -J lib main.jsonnet

# One file per top-level field, written to out/, which -c creates
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/jsonnet \
  -J vendor -c -m out main.jsonnet

# A YAML stream with one document per element of the top-level array
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/jsonnet \
  -y manifests.jsonnet > manifests.yaml

# Top-level arguments and external variables
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/jsonnet \
  --tla-str env=prod --ext-str region=eu-west-1 main.jsonnet
```

Without arguments the image prints `jsonnet --help`. Library directories can also come from `JSONNET_PATH`, for example `-e JSONNET_PATH=vendor`. In CI, pin a minor version tag such as `0.22` or a digest: go-jsonnet has no 1.0 release yet, so the major version tag `0` follows every release.

## The other tools

The entrypoint runs `jsonnet`. Pick one of the other tools with `--entrypoint`:

```sh
# Reformat files in place
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint jsonnetfmt \
  ghcr.io/randomcontainers/jsonnet -i main.jsonnet lib/*.libsonnet

# Check the formatting without changing anything
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint jsonnetfmt \
  ghcr.io/randomcontainers/jsonnet --test main.jsonnet lib/*.libsonnet

# Lint
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint jsonnet-lint \
  ghcr.io/randomcontainers/jsonnet -J vendor main.jsonnet

# List the files main.jsonnet imports
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint jsonnet-deps \
  ghcr.io/randomcontainers/jsonnet -J vendor main.jsonnet
```

`jsonnetfmt --test` prints the files that need reformatting and exits with 2 if there are any. `jsonnet-lint` exits with 2 when it finds a problem, such as an unused variable or a syntax error, and with 1 when it cannot read a file or an option is wrong. `jsonnet-deps` prints absolute paths as the container sees them, under `/work`.

## Libraries

The image contains no Jsonnet libraries and no jsonnet-bundler (`jb`). Install libraries into `vendor/` with jsonnet-bundler outside the container, or commit them to the repository, and pass `-J vendor`. Imports are read from files only, so evaluating needs no network.

## What is in the image

- `jsonnet`, `jsonnetfmt`, `jsonnet-lint` and `jsonnet-deps` in `/usr/local/bin`, built without cgo. They are statically linked and load no libraries.
- Nothing else beyond the base image, `ca-certificates` and `tini`.

The Go version, the SHA-256 of the source tarball, the commit it was made from and every Go module with its version and checksum are in `/usr/local/share/randomcontainers/jsonnet/buildinfo`.

## Default or slim

Jsonnet's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a go-jsonnet release such as `0.22.0`. `<minor>` and `<major>` are its shorter forms, `0.22` and `0`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current go-jsonnet version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`jsonnet` writes to the working directory only with `-o` or `-m`, `jsonnetfmt` only with `-i` or `-o`, and `jsonnet-deps` only with `-o`. The files belong to the user the container runs as.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new go-jsonnet release and are rebuilt when the base image changes. Switch to root to install more, then back. This adds `make`, for a Makefile that runs `jsonnet` and `jsonnetfmt`:

```dockerfile
FROM ghcr.io/randomcontainers/jsonnet:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends make \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
ENTRYPOINT ["make"]
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache make`. The entrypoint is `["tini", "--", "jsonnet"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new go-jsonnet releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/jsonnet/` holds the version, the source URL, the build details, the license files and `runtime-deps`, the list of distro packages the tools need at run time beyond those in the base image (none).

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/jsonnet:latest \
  --repo randomcontainers/jsonnet --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/jsonnet:latest --format '{{ json .SBOM }}'
```

go-jsonnet attaches a source tarball, `go-jsonnet-v<version>.tar.gz`, to each release; it is a git archive of the release tag. Its `checksums.txt` covers only the prebuilt binaries, so the SHA-256 in `package.yml` is the digest GitHub records for the tarball. Before compiling, the build checks the tarball against it and checks that the source declares the same version. The Go toolchain is downloaded from Go's release server (dl.google.com, as linked from go.dev/dl) and checked against the SHA-256 in the Dockerfile, and the Go modules come from the Go module proxy and are checked against go-jsonnet's `go.sum`. `buildinfo` records the commit named in the tarball's header and the Go modules.

## Updates

The project checks the releases of [google/go-jsonnet](https://github.com/google/go-jsonnet/releases) every 15 minutes. A release is picked up once it is 24 hours old; pre-releases are skipped. Its source tarball is checked against the digest GitHub records for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

go-jsonnet's own binaries keep the Go release they were built with until the next go-jsonnet release. These images are compiled with the Go release pinned in the Dockerfiles instead, which is raised by hand when Go publishes a security release, and each raise rebuilds the images of the current version. A go-jsonnet release that needs a newer Go than the pin does not build until the pin is raised.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t jsonnet:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of packages Go builds in parallel.

## Licenses

go-jsonnet is released under the [Apache License 2.0](https://github.com/google/go-jsonnet/blob/master/LICENSE). The binaries also contain:

- the Go standard library, golang.org/x/crypto and golang.org/x/sys (BSD-3-Clause);
- color, go-colorable and go-isatty (MIT);
- sigs.k8s.io/yaml (MIT, with code from Go's encoding/json under BSD-3-Clause), which includes a fork of go-yaml v2 (Apache-2.0, with the files ported from libyaml under MIT).

The image's license label is `Apache-2.0 AND MIT AND BSD-3-Clause`. The license files are in `/usr/local/share/randomcontainers/jsonnet/licenses/`, with one directory per Go module; the go-yaml fork's license and NOTICE are in `sigs.k8s.io/yaml/goyaml.v2/`. Links to the source are in `/usr/local/share/randomcontainers/jsonnet/source`. The packages from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
