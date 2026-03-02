# GitHub Actions Knowledge Base - Agent Guide

This repository is a curated collection of GitHub Actions open-source projects, organized as git submodules. It serves as a knowledge base for AI agents and developers working with GitHub Actions workflows and infrastructure.

## Quick Start

```bash
# Sync all repositories (adds missing, updates existing)
uv run sync.py

# Preview changes without making them
uv run sync.py --dry-run
```

## Repository Structure

```
actions-knowledge-base/
├── sync.py              # Repository sync tool
├── pyproject.toml       # Python project config
├── insights/            # Articles and analysis
└── repos/               # Submodules directory
    ├── runner/
    ├── runner-images/
    ├── actions-runner-controller/
    ├── checkout/
    ├── cache/
    ├── upload-artifact/
    ├── download-artifact/
    ├── setup-node/
    ├── setup-python/
    ├── setup-go/
    ├── setup-java/
    ├── github-script/
    ├── create-release/
    ├── labeler/
    ├── toolkit/
    ├── starter-workflows/
    ├── typescript-action/
    ├── javascript-action/
    ├── configure-aws-credentials/
    ├── login/
    ├── auth/
    ├── harbor/
    ├── harbor-helm/
    ├── harbor-cli/
    ├── karpenter/
    ├── kustomize/
    ├── k8s-device-plugin/
    ├── go-containerregistry/
    ├── buildkit/
    ├── setup-buildx-action/
    ├── build-push-action/
    ├── login-action/
    ├── uv/
    ├── ccache/
    └── docs/
```

## Repository Summaries

### Core Runner Infrastructure

#### `repos/runner/` - GitHub Actions Runner
**Language:** C# | **Version:** v2.321.0

The self-hosted runner application that executes GitHub Actions workflow jobs on your own infrastructure. Provides maximum control over the execution environment.

**Key paths:**
- `src/Runner.Worker/` - Job execution logic
- `src/Runner.Listener/` - Job polling and runner lifecycle
- `src/Sdk/` - Shared SDK components

**Installation:**
```bash
# Download and configure
./config.sh --url https://github.com/ORG/REPO --token TOKEN
./run.sh
```

---

#### `repos/runner-images/` - GitHub-hosted Runner Images
**Language:** Packer/PowerShell/Bash | **Tracking:** Latest

Virtual machine images used by GitHub-hosted runners. Contains the complete toolchain and software pre-installed on ubuntu-latest, windows-latest, and macos-latest.

**Key paths:**
- `images/ubuntu/` - Ubuntu image definitions
- `images/windows/` - Windows image definitions
- `images/macos/` - macOS image definitions

**Useful for:** Understanding what's pre-installed, debugging environment differences, building custom images.

---

#### `repos/actions-runner-controller/` - Actions Runner Controller (ARC)
**Language:** Go | **Version:** v0.9.3

Kubernetes operator that orchestrates self-hosted runners. Enables autoscaling runners based on workflow demand within Kubernetes clusters.

**Key paths:**
- `controllers/` - Kubernetes controller logic
- `charts/gha-runner-scale-set/` - Helm chart for runner scale sets
- `charts/gha-runner-scale-set-controller/` - Helm chart for the controller

**Installation:**
```bash
helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

helm install runner-set \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --set githubConfigUrl="https://github.com/ORG" \
  --set githubConfigSecret.github_token="TOKEN"
```

---

### Essential Actions

#### `repos/checkout/` - actions/checkout
**Language:** TypeScript | **Version:** v4.2.2

Checks out your repository code so workflows can access it. The most commonly used action.

**Key features:**
- Fetch depth control (`fetch-depth: 0` for full history)
- Submodule support
- LFS support
- Multiple repository checkout

**Usage:**
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full history for changelog generation
```

---

#### `repos/cache/` - actions/cache
**Language:** TypeScript | **Version:** v4.1.2

Caches dependencies and build outputs to speed up workflows. Supports cross-workflow and cross-branch cache sharing.

**Key paths:**
- `src/` - Core caching logic
- `src/cache.ts` - Main cache implementation

**Usage:**
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
    restore-keys: npm-
```

---

#### `repos/upload-artifact/` - actions/upload-artifact
**Language:** TypeScript | **Version:** v4.4.3

Uploads artifacts from your workflow run for sharing between jobs or downloading after completion.

**Usage:**
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 5
```

---

#### `repos/download-artifact/` - actions/download-artifact
**Language:** TypeScript | **Version:** v4.1.8

Downloads artifacts uploaded by `upload-artifact` in the same workflow run.

**Usage:**
```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: dist/
```

---

### Language Setup Actions

#### `repos/setup-node/` - actions/setup-node
**Language:** TypeScript | **Version:** v4.1.0

Sets up a Node.js environment with specified version and optional caching.

**Usage:**
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

---

#### `repos/setup-python/` - actions/setup-python
**Language:** TypeScript | **Version:** v5.3.0

Sets up a Python environment with specified version, pip caching, and optional poetry/pipenv support.

**Usage:**
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

---

#### `repos/setup-go/` - actions/setup-go
**Language:** TypeScript | **Version:** v5.1.0

Sets up a Go environment with specified version and module caching.

**Usage:**
```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true
```

---

#### `repos/setup-java/` - actions/setup-java
**Language:** TypeScript | **Version:** v4.5.0

Sets up a Java/JDK environment with support for multiple distributions (Temurin, Zulu, Corretto, etc.).

**Usage:**
```yaml
- uses: actions/setup-java@v4
  with:
    distribution: 'temurin'
    java-version: '21'
    cache: 'maven'
```

---

### Workflow Utilities

#### `repos/github-script/` - actions/github-script
**Language:** TypeScript | **Version:** v7.0.1

Run JavaScript/TypeScript scripts with access to the GitHub API via Octokit. Enables complex automation without creating a custom action.

**Usage:**
```yaml
- uses: actions/github-script@v7
  with:
    script: |
      const { data: issues } = await github.rest.issues.listForRepo({
        owner: context.repo.owner,
        repo: context.repo.repo,
        state: 'open'
      });
      console.log(`Found ${issues.length} open issues`);
```

---

#### `repos/create-release/` - actions/create-release
**Language:** TypeScript | **Version:** v1.1.4

Creates GitHub releases with release notes and assets.

**Usage:**
```yaml
- uses: actions/create-release@v1
  with:
    tag_name: ${{ github.ref_name }}
    release_name: Release ${{ github.ref_name }}
    body: |
      Changes in this release...
    draft: false
    prerelease: false
```

---

#### `repos/labeler/` - actions/labeler
**Language:** TypeScript | **Version:** v5.0.0

Automatically labels pull requests based on file paths changed.

**Usage:**
```yaml
# .github/labeler.yml
documentation:
  - 'docs/**'
  - '*.md'

frontend:
  - 'src/components/**'
```

---

### Action Development

#### `repos/toolkit/` - actions/toolkit
**Language:** TypeScript | **Tracking:** Latest

The official SDK for building GitHub Actions. Contains packages for core functionality, GitHub API access, artifact handling, caching, and more.

**Key packages:**
- `@actions/core` - Inputs, outputs, logging, secrets
- `@actions/github` - Octokit client with auth
- `@actions/exec` - Command execution
- `@actions/cache` - Caching utilities
- `@actions/artifact` - Artifact upload/download

**Installation:**
```bash
npm install @actions/core @actions/github
```

---

#### `repos/starter-workflows/` - GitHub Starter Workflows
**Language:** YAML | **Tracking:** Latest

Template workflows shown in the GitHub Actions "New workflow" UI. Great reference for language-specific CI/CD patterns.

**Key paths:**
- `ci/` - Continuous integration templates
- `deployments/` - Deployment templates
- `automation/` - Automation and bot templates
- `code-scanning/` - Security scanning templates

---

#### `repos/typescript-action/` - TypeScript Action Template
**Language:** TypeScript | **Tracking:** Latest

Official template for creating GitHub Actions with TypeScript. Includes build configuration, testing setup, and release workflow.

**Key paths:**
- `src/main.ts` - Action entry point
- `action.yml` - Action metadata
- `.github/workflows/` - CI for the action itself

---

#### `repos/javascript-action/` - JavaScript Action Template
**Language:** JavaScript | **Tracking:** Latest

Official template for creating GitHub Actions with JavaScript. Simpler setup than TypeScript for quick prototyping.

---

### Cloud Provider Actions

#### `repos/configure-aws-credentials/` - AWS Credentials
**Language:** TypeScript | **Version:** v4.0.2 | **Org:** aws-actions

Configures AWS credentials for use with AWS CLI and SDKs. Supports OIDC federation for keyless authentication.

**OIDC Usage (recommended):**
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/GitHubActions
    aws-region: us-east-1
```

**Access Key Usage:**
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

---

#### `repos/login/` - Azure Login
**Language:** TypeScript | **Version:** v2.2.0 | **Org:** azure

Authenticates with Azure using service principal, OIDC, or managed identity.

**OIDC Usage:**
```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

#### `repos/auth/` - Google Cloud Auth
**Language:** TypeScript | **Version:** v2.1.6 | **Org:** google-github-actions

Authenticates with Google Cloud using Workload Identity Federation (OIDC) or service account keys.

**OIDC Usage (recommended):**
```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/pool/providers/provider
    service_account: my-sa@project.iam.gserviceaccount.com
```

---

### Container Registry

#### `repos/harbor/` - Harbor Container Registry
**Language:** Go/TypeScript | **Version:** v2.14.2 | **Org:** goharbor

An open-source trusted cloud native registry project (CNCF graduated) that stores, signs, and scans container content. Extends Docker Distribution with security, identity, and management features.

**Key features:**
- Vulnerability scanning of container images
- Image signing and validation
- Role-based access control and multi-tenancy
- Registry replication across instances
- RESTful API and web UI

**Key paths:**
- `src/` - Main source code for Harbor components
- `api/v2.0/` - RESTful API specifications
- `make/` - Build configuration and scripts
- `docs/` - Documentation and guides
- `tests/` - Test suites and infrastructure

**Installation (Docker Compose):**
```bash
# Download the online installer
wget https://github.com/goharbor/harbor/releases/download/v2.14.2/harbor-online-installer-v2.14.2.tgz
tar xzvf harbor-online-installer-v2.14.2.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml
# Edit harbor.yml with your hostname, TLS, and storage settings
./install.sh
```

---

#### `repos/harbor-helm/` - Harbor Helm Chart
**Language:** YAML/Mustache | **Version:** v1.18.2 | **Org:** goharbor

Helm chart for deploying Harbor into Kubernetes clusters. Supports ingress, TLS, persistent storage, and multiple storage backends (S3, GCS, Azure, filesystem).

**Key paths:**
- `templates/` - Helm template files for Kubernetes manifests
- `values.yaml` - Default configuration values
- `docs/` - Deployment guides (HA, upgrades)

**Key configuration:**
- `expose.type` - Service exposure (ingress, clusterIP, nodePort, loadBalancer)
- `expose.tls.enabled` - TLS configuration (default: true)
- `persistence.imageChartStorage.type` - Storage backend (filesystem, s3, gcs, azure, swift, oss)
- `harborAdminPassword` - Initial admin credentials
- `externalURL` - External access URL

**Installation:**
```bash
helm repo add harbor https://helm.goharbor.io
helm install harbor harbor/harbor \
  --set expose.type=ingress \
  --set expose.ingress.hosts.core=core.harbor.domain \
  --set externalURL=https://core.harbor.domain \
  --set harborAdminPassword=YourPassword
```

---

#### `repos/harbor-cli/` - Harbor CLI
**Language:** Go | **Version:** v0.0.17 | **Org:** goharbor

Official command-line interface for Harbor registries. A streamlined, user-friendly alternative to the web UI for daily operations, scripting, and automation.

**Key paths:**
- `cmd/harbor/` - Command implementations
- `pkg/` - Core library packages
- `doc/` - Documentation
- `examples/config/` - Sample configuration files

**Installation:**
```bash
# macOS
brew install harbor-cli

# Or download from GitHub releases
```

---

### Documentation

#### `repos/docs/` - GitHub Documentation
**Language:** Markdown/MDX | **Tracking:** Latest | **Org:** github

The complete source for docs.github.com. Contains comprehensive documentation for GitHub Actions and all GitHub features.

**Key paths:**
- `content/actions/` - GitHub Actions documentation
- `content/actions/writing-workflows/` - Workflow authoring guides
- `content/actions/creating-actions/` - Action development guides
- `content/actions/hosting-your-own-runners/` - Self-hosted runner guides

---

### Kubernetes Infrastructure

#### `repos/karpenter/` - Karpenter Node Autoscaler
**Language:** Go | **Version:** v1.1.3 | **Org:** kubernetes-sigs

Kubernetes node autoscaler that provisions just-in-time compute resources for Kubernetes clusters. Used in ciforge to dynamically provision nodes for GitHub Actions runner pods based on workflow demand.

**Key paths:**
- `charts/karpenter/` - Helm chart for deploying Karpenter
- `pkg/apis/` - API types including NodePool, EC2NodeClass
- `pkg/providers/` - Cloud provider implementations
- `designs/` - Design documents and proposals

**Key concepts:**
- **NodePool** - Defines constraints for nodes Karpenter can provision (instance types, zones, taints)
- **EC2NodeClass** - AWS-specific node configuration (AMI, security groups, subnets)
- **Consolidation** - Automatic bin-packing and node replacement for cost optimization

---

#### `repos/kustomize/` - Kustomize
**Language:** Go | **Tracking:** Latest | **Org:** kubernetes-sigs

Kubernetes-native configuration management tool. Customizes Kubernetes manifests without templates using overlays, patches, and transformers. Built into `kubectl` as `kubectl apply -k`.

**Key paths:**
- `api/types/` - Kustomization API types
- `examples/` - Example kustomizations
- `docs/` - Documentation and guides

---

### GPU Support

#### `repos/k8s-device-plugin/` - NVIDIA Kubernetes Device Plugin
**Language:** Go | **Version:** v0.17.1 | **Org:** NVIDIA

Kubernetes device plugin that exposes NVIDIA GPUs to containerized workloads. Deployed as a DaemonSet on GPU nodes in ciforge to enable GPU-accelerated CI runner jobs.

**Key paths:**
- `deployments/helm/nvidia-device-plugin/` - Helm chart
- `cmd/nvidia-device-plugin/` - Main plugin binary
- `api/config/` - Plugin configuration schema

**Key features:**
- GPU resource advertisement (`nvidia.com/gpu`)
- GPU sharing (time-slicing, MPS, MIG)
- Health checking and topology awareness

---

### Container Tooling

#### `repos/go-containerregistry/` - go-containerregistry (crane)
**Language:** Go | **Tracking:** Latest | **Org:** google

Go library and CLI tools for interacting with container registries. The `crane` CLI is used in ciforge's image mirroring pipeline (`scripts/mirror-images.sh`) to copy upstream images to ECR.

**Key paths:**
- `cmd/crane/` - crane CLI source
- `pkg/v1/remote/` - Remote registry interaction
- `pkg/v1/mutate/` - Image mutation utilities
- `cmd/crane/doc/` - crane command documentation

**Common crane commands:**
- `crane copy SRC DST` - Copy images between registries
- `crane digest IMAGE` - Get image digest
- `crane manifest IMAGE` - Fetch image manifest
- `crane ls REPO` - List tags in a repository

---

### Docker Actions (CI/CD)

#### `repos/setup-buildx-action/` - docker/setup-buildx-action
**Language:** TypeScript | **Version:** v3.12.0 | **Org:** docker

Sets up Docker Buildx in GitHub Actions workflows. Supports local, remote, docker-container, and kubernetes drivers. Used in ciforge to connect to the in-cluster BuildKit daemon via `driver: remote`.

**Key inputs:**
- `driver` — Builder driver: `docker-container` (default), `remote`, `kubernetes`, `docker`
- `endpoint` — Remote BuildKit endpoint (e.g., `tcp://buildkitd:1234`)
- `append` — YAML list of additional builder nodes (for multi-arch with separate endpoints)
- `platforms` — Fixed platform constraints for the builder
- `driver-opts` — Driver-specific options (TLS certs for remote, etc.)

**Key paths:**
- `src/` - Action source code
- `action.yml` - Action metadata and input definitions

**Usage (remote driver):**
```yaml
- uses: docker/setup-buildx-action@v3
  with:
    driver: remote
    endpoint: tcp://buildkitd-arm64.buildkit:1234
```

---

#### `repos/build-push-action/` - docker/build-push-action
**Language:** TypeScript | **Version:** v6.19.2 | **Org:** docker

Builds and pushes container images using Buildx. Works transparently with any builder configured by `setup-buildx-action`, including remote BuildKit daemons. Registry auth is forwarded via the buildx session.

**Key inputs:**
- `context` — Build context path
- `push` — Push image after build (shorthand for `--output type=registry`)
- `tags` — Image tags (multi-line for multiple)
- `platforms` — Target platforms for multi-arch builds
- `cache-from` / `cache-to` — Build cache configuration
- `build-args` — Build-time variables
- `outputs` — Custom output configuration (alternative to `push`)

**Key paths:**
- `src/` - Action source code
- `action.yml` - Action metadata and input definitions

**Usage:**
```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: registry.example.com/my-image:latest
```

---

#### `repos/login-action/` - docker/login-action
**Language:** TypeScript | **Version:** v3.7.0 | **Org:** docker

Authenticates to container registries by writing credentials to `~/.docker/config.json`. With remote BuildKit, credentials are forwarded to the daemon via the buildx session — they never need to exist on the daemon itself.

**Key inputs:**
- `registry` — Registry server URL (default: Docker Hub)
- `username` — Registry username
- `password` — Registry password or token
- `ecr` — Set to `auto` for AWS ECR login
- `logout` — Log out at the end of the job (default: true)

**Key paths:**
- `src/` - Action source code
- `action.yml` - Action metadata and input definitions

**Usage:**
```yaml
- uses: docker/login-action@v3
  with:
    registry: harbor.example.com
    username: ${{ secrets.REGISTRY_USER }}
    password: ${{ secrets.REGISTRY_PASS }}
```

---

### Container Build

#### `repos/buildkit/` - BuildKit
**Language:** Go | **Version:** v0.27.1 | **Org:** moby

Concurrent, cache-efficient, and Dockerfile-agnostic builder toolkit. BuildKit is the next-generation container image builder used as the backend for `docker build` (via BuildX). It supports advanced features like multi-stage builds, build secrets, SSH forwarding, cache mounts, and multi-platform builds.

**Key paths:**
- `cmd/buildkitd/` - BuildKit daemon
- `cmd/buildctl/` - BuildKit CLI client
- `client/` - Go client library
- `solver/` - Build graph solver and caching logic
- `frontend/dockerfile/` - Dockerfile frontend (parser and builder)
- `examples/` - Example Dockerfiles and build configurations
- `docs/` - Documentation

**Key features:**
- Automatic garbage collection and build cache management
- Concurrent dependency resolution and parallel build steps
- Rootless execution mode
- Multiple output formats (OCI image, Docker tarball, local directory)
- Distributed workers and remote build cache (registry, S3, GitHub Actions)
- LLB (low-level builder) intermediate representation for builds

**Usage:**
```bash
# Start buildkitd daemon
buildkitd &

# Build using buildctl
buildctl build \
  --frontend dockerfile.v0 \
  --local context=. \
  --local dockerfile=. \
  --output type=image,name=myimage:latest,push=true

# Or use via Docker BuildX
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t myimage:latest --push .
```

---

### Build & Dev Tools

#### `repos/uv/` - uv (Python Package Manager)
**Language:** Rust | **Tracking:** Latest | **Org:** astral-sh

Extremely fast Python package and project manager written in Rust. Used in ciforge as the exclusive Python package manager (replaces pip/conda/poetry).

**Key paths:**
- `docs/` - Documentation and guides
- `crates/` - Rust crate source code

**Common usage:**
- `uv pip install <package>` - Install packages
- `uv venv` - Create virtual environments
- `uv run <script>` - Run Python scripts
- `uv lock` - Lock dependencies

---

#### `repos/ccache/` - ccache (Compiler Cache)
**Language:** C++ | **Tracking:** Latest | **Org:** ccache

Compiler cache that speeds up recompilation by caching previous compilations. Installed on runner nodes via the EKS bootstrap script to accelerate C/C++ build jobs.

**Key paths:**
- `doc/` - Documentation (configuration, usage)
- `src/` - Core source code
- `cmake/` - Build system configuration

**Key configuration:**
- `CCACHE_DIR` - Cache directory location
- `CCACHE_MAXSIZE` - Maximum cache size
- `CCACHE_REMOTE_STORAGE` - Remote storage backend (Redis, HTTP)

---

## Managing Repositories

### Adding a New Repository

Edit `sync.py` and add to the `ALLOWED_REPOS` list:

```python
ALLOWED_REPOS: list[str | tuple[str, str]] = [
    # Existing repos...

    # Add new repo (tracks latest):
    "new-repo-name",

    # Or pin to a specific version:
    ("new-repo-name", "v1.0.0"),

    # External org:
    ("other-org/repo-name", "v2.0.0"),
]
```

Then run `uv run sync.py` to sync.

### Updating to Latest Versions

To update pinned versions, check the latest tags:

```bash
gh api repos/actions/REPO_NAME/tags --jq '.[0].name'
```

Update the version in `sync.py` and run `uv run sync.py`.

### Removing a Repository

Remove it from `ALLOWED_REPOS` in `sync.py` and run `uv run sync.py`. The submodule will be automatically cleaned up.

---

## Tips for Navigating the Codebase

1. **Start with READMEs** - Each `repos/*/README.md` has setup instructions and examples
2. **Check examples/** - Most repos have example configurations
3. **Look for action.yml** - The action metadata file defines inputs, outputs, and behavior
4. **Use grep across repos** - `grep -r "pattern" repos/` to find implementations
5. **Check CHANGELOG.md** - Understand recent changes and breaking updates

## Common Tasks

| Task | Repository | Key File/Path |
|------|------------|---------------|
| Understand runner architecture | `runner` | `src/Runner.Worker/` |
| Check pre-installed software | `runner-images` | `images/ubuntu/scripts/` |
| Set up K8s autoscaling | `actions-runner-controller` | `charts/gha-runner-scale-set/` |
| Configure checkout options | `checkout` | `src/git-source-provider.ts` |
| Implement caching strategy | `cache` | `src/cache.ts` |
| Build custom action | `toolkit` | `packages/core/` |
| Find workflow templates | `starter-workflows` | `ci/` |
| Configure AWS OIDC | `configure-aws-credentials` | `src/` |
| Configure Azure OIDC | `login` | `src/` |
| Configure GCP OIDC | `auth` | `src/` |
| Learn workflow syntax | `docs` | `content/actions/writing-workflows/` |
| Understand contexts | `docs` | `content/actions/writing-workflows/choosing-what-your-workflow-does/` |
| Deploy container registry | `harbor` | `make/`, `docs/` |
| Deploy Harbor on K8s | `harbor-helm` | `values.yaml`, `templates/` |
| Configure Harbor via CLI | `harbor-cli` | `cmd/harbor/`, `examples/config/` |
| Harbor API reference | `harbor` | `api/v2.0/` |
| Configure Karpenter NodePools | `karpenter` | `charts/karpenter/`, `pkg/apis/` |
| Expose GPUs to K8s pods | `k8s-device-plugin` | `deployments/helm/nvidia-device-plugin/` |
| Mirror container images | `go-containerregistry` | `cmd/crane/` |
| Customize K8s manifests | `kustomize` | `api/types/`, `examples/` |
| Build container images | `buildkit` | `cmd/buildkitd/`, `frontend/dockerfile/` |
| Set up Buildx in CI | `setup-buildx-action` | `src/`, `action.yml` |
| Build & push images in CI | `build-push-action` | `src/`, `action.yml` |
| Registry auth in CI | `login-action` | `src/`, `action.yml` |
| Manage Python packages | `uv` | `docs/` |
| Speed up C/C++ builds | `ccache` | `doc/` |
