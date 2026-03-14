# PLAN.md — CI-Built Ubuntu WSL Rootfs Project

## Purpose

This document is intended to be ingested by an LLM-powered coding assistant to implement a project that builds a **custom Ubuntu WSL root filesystem in CI**, publishes it as a **versioned artifact**, and allows local installation via:

```bash
wsl --import <DistroName> <InstallPath> <rootfs.tar>
```

The resulting WSL distro is a **minimal host layer** for container-based development. It is **not** intended to be a full personal workstation image.

The image must include a small fixed set of host-level tools and services:

- Docker
- docker-compose
- chezmoi
- devpod-cli
- opencode-server

Everything else is managed by the user after first start.

User creation is **manual** on first launch and is **out of scope for automation**.

---

## Project Goals

The implementation must:

1. Build an **Ubuntu rootfs** in **GitHub Actions**
2. Install and configure the baked-in host tools listed above
3. Enable the image to run correctly under **WSL**
4. Produce a **tarball artifact** suitable for `wsl --import`
5. Keep the image generic and reusable
6. Avoid baking in:
   - user accounts
   - secrets
   - personal dotfiles
   - repo-specific tooling
   - project-specific development environments

This project is specifically about creating a **clean, reproducible, CI-built WSL host image**.

---

## Non-Goals

The implementation must **not** try to do the following:

- automate Windows-side WSL installation
- automate first-run user creation
- manage user-specific configuration
- configure personal shell/editor/git settings
- replace devcontainers
- act as general workstation provisioning
- build a `.wsl` packaged distribution initially
- integrate with Docker Desktop on Windows

The design assumes **Docker runs inside WSL**, with minimal Windows interaction.

---

## High-Level Architecture

The architecture should be split into four layers:

### Layer 1 — Windows / WSL substrate
Handled externally by the user:
- Windows host
- WSL installed and working
- local import via `wsl --import`

### Layer 2 — CI-built Ubuntu rootfs artifact
Handled by this project:
- build Ubuntu rootfs
- install host packages
- configure WSL-specific settings
- configure Docker service behavior
- prepare image for import
- output tar artifact

### Layer 3 — Manual first start
Handled manually by the user:
- import distro
- launch distro
- create user
- add user to required groups
- optionally set default user if desired

### Layer 4 — User-managed environment
Handled by the user after login:
- `chezmoi init --apply`
- user dotfiles
- personal CLI/editor configuration
- DevPod configuration details
- all other tools

---

## Core Design Principles

The coding assistant must preserve these principles while implementing the project:

1. **Image-first**
   - The rootfs is the product.
   - Local provisioning should be minimal.

2. **Minimal host**
   - Only stable host-level tools belong in the image.

3. **Clear boundary between system and user state**
   - System-level components are baked in.
   - User-level configuration is explicitly out of scope.

4. **WSL-native artifact**
   - Output must be compatible with `wsl --import`.

5. **CI reproducibility**
   - Build should happen in GitHub Actions with as little manual intervention as possible.

6. **No configuration-management creep**
   - Do not reintroduce Ansible-like post-install provisioning inside the WSL image.

---

## Required Deliverables

The coding assistant should implement the repository so that it contains at minimum:

1. A documented project structure
2. A rootfs build pipeline for Ubuntu
3. A package installation mechanism for required software
4. WSL-specific image configuration
5. Docker service configuration suitable for WSL with systemd enabled
6. A GitHub Actions workflow that produces a rootfs tar artifact
7. A README explaining build and import usage
8. A validation or smoke-test strategy
9. A release/versioning strategy

---

## Suggested Repository Structure

The coding assistant should create something close to the following:

```text
.
├── README.md
├── PLAN.md
├── Makefile
├── .github/
│   └── workflows/
│       ├── build-rootfs.yml
│       └── validate.yml
├── rootfs/
│   ├── packages/
│   │   ├── apt.txt
│   │   └── external-tools.md
│   ├── overlay/
│   │   ├── etc/
│   │   │   ├── wsl.conf
│   │   │   ├── docker/
│   │   │   │   └── daemon.json
│   │   │   └── systemd/
│   │   │       └── system/
│   │   │           └── opencode-server.service
│   │   ├── usr/local/bin/
│   │   │   ├── image-build.sh
│   │   │   ├── image-clean.sh
│   │   │   └── smoke-test.sh
│   │   └── etc/skel/
│   ├── scripts/
│   │   ├── build-rootfs.sh
│   │   ├── install-packages.sh
│   │   ├── install-chezmoi.sh
│   │   ├── install-devpod.sh
│   │   ├── install-opencode-server.sh
│   │   ├── configure-wsl.sh
│   │   ├── configure-docker.sh
│   │   ├── cleanup-rootfs.sh
│   │   └── archive-rootfs.sh
│   └── ubuntu/
│       └── bootstrap.sh
└── docs/
    ├── architecture.md
    ├── build.md
    ├── import.md
    └── testing.md
```

Exact layout may differ, but separation of concerns must remain clear.

---

## Base Distro Choice

Use **Ubuntu** as the initial base distro.

Recommended initial target:
- Ubuntu 24.04 LTS, unless a strong implementation reason requires another supported LTS

Rationale:
- easier rootfs bootstrap in CI
- strong package ecosystem
- straightforward WSL compatibility
- simpler operational baseline than a custom Fedora bootstrap for this use case

The coding assistant should keep the design modular enough that another distro could be substituted later, but Ubuntu is the only required implementation target.

---

## Rootfs Build Strategy

The rootfs should be built in CI using a standard Ubuntu bootstrap method.

### Recommended approach
Use `debootstrap` to create a minimal Ubuntu filesystem tree.

The pipeline should then:

1. bootstrap the rootfs
2. configure apt sources
3. chroot into the rootfs for package installation
4. apply overlay files
5. enable required services
6. clean package caches and transient state
7. archive to tar

### Build requirements
The build process must:
- be scriptable end-to-end
- run non-interactively
- be suitable for GitHub Actions Ubuntu runners
- avoid reliance on privileged nested virtualization
- avoid ad hoc manual build steps

---

## Required Baked-In Software

The resulting image must include the following baked-in tools:

### 1. Docker
Must include:
- Docker Engine
- Docker CLI
- Docker daemon
- support for running Docker inside the WSL distro
- integration with systemd for service management

### 2. docker-compose
Install the current supported compose implementation for Docker on Linux.
Prefer the Docker Compose plugin model if appropriate rather than legacy standalone tooling, but the end-user requirement is that `docker compose` and/or `docker-compose` is available in a usable form.

### 3. chezmoi
Install the chezmoi binary so the user can run:

```bash
chezmoi init --apply <username>
```

after manual account creation.

### 4. devpod-cli
Install the DevPod CLI binary and ensure it is available on PATH.

Do not apply user-specific DevPod configuration in the image.

### 5. opencode-server
Install and configure opencode-server so it is present in the image.

If it runs as a service, provide a systemd unit.
If it is merely a binary invoked by the user, install it cleanly and document usage.
If packaging/install method is non-standard, isolate that logic into a dedicated installation script.

---

## WSL-Specific Configuration

The image must be configured specifically for WSL.

### Required WSL settings
Create `/etc/wsl.conf` with a sensible minimal configuration.

At minimum, the implementation should consider:

- enabling `systemd=true`
- not forcing assumptions about a default non-root user
- keeping interop behavior explicit and minimal
- avoiding over-customization

The image should work cleanly as an imported WSL distro without requiring distro-specific launcher packaging.

---

## Docker Configuration Requirements

This is the most important host-side runtime component.

The assistant must implement Docker in a way that works reasonably inside WSL.

### Expectations
- Docker daemon should be installed in the image
- Docker should be manageable through systemd
- Docker socket should be available in the normal Linux location
- configuration should be explicit and minimal

### Important boundaries
The implementation should:
- assume Docker runs **inside** WSL
- not depend on Docker Desktop
- not assume Windows-side daemon integration
- not bake in user-specific Docker config

### Validation
The smoke-test path should verify at least:
- `docker --version`
- `docker compose version`
- Docker service can start
- a simple container command can run successfully, if feasible in the validation environment

If full daemon validation in CI is difficult, document that clearly and include the best practical offline verification possible.

---

## User Model

The image must **not** create user accounts automatically.

This is intentional.

The assistant must preserve the following behavior:

- first launch enters the distro in a manual setup state
- the user manually creates their Linux account
- the user manually adds themselves to any needed groups such as `docker`
- the user then runs Chezmoi

Do not add first-boot account automation unless explicitly requested in a later phase.

---

## Post-Import Workflow

The expected usage pattern is:

1. User installs/imports the distro:
   ```bash
   wsl --import <DistroName> <InstallPath> <rootfs.tar>
   ```

2. User starts the distro

3. User manually creates their account

4. User installs/applies personal configuration:
   ```bash
   chezmoi init --apply philbudden
   ```

5. User uses the WSL host primarily to:
   - run Docker
   - run baked-in host tools
   - launch devcontainer-based workflows

This expected workflow should be documented clearly in the README.

---

## Declarative vs Non-Declarative Boundaries

The assistant should keep these boundaries explicit.

### Declarative / image-defined
- base distro selection
- package manifest
- binary installation steps
- service units
- system config
- build scripts
- artifact naming/versioning
- CI workflows
- smoke tests
- release process

### Manual / user-managed
- user creation
- group membership for the created user
- dotfiles
- personal editor config
- personal DevPod config
- credentials
- SSH keys
- tokens
- repo-specific tooling

---

## CI Pipeline Requirements

Implement GitHub Actions workflows to build and validate the image.

### Primary workflow
The main workflow should:
- trigger on pushes to main and tags
- optionally support manual dispatch
- build the rootfs
- archive the rootfs as a tarball
- store the artifact
- optionally publish release assets on tags

### Validation workflow
A validation workflow should:
- lint shell scripts if possible
- validate overlay structure
- verify required files are present
- perform smoke checks on the built filesystem
- fail clearly when required binaries/configs are missing

### Artifact naming
Use a predictable naming convention such as:

```text
ubuntu-wsl-rootfs-<version>.tar
```

Optional compressed variants such as `.tar.gz` or `.tar.zst` are acceptable if the import workflow is still clearly documented.

---

## Versioning and Release Strategy

Use a simple release strategy.

Recommended:
- semantic-ish version tags such as `v0.1.0`
- tagged builds publish rootfs artifacts
- include checksums
- optionally attach build metadata

The version should be discoverable from:
- Git tags
- release assets
- optionally a file embedded in the image, such as `/etc/image-release`

---

## Testing Strategy

The coding assistant should implement a pragmatic testing strategy.

### Minimum required tests
1. Validate rootfs build completes in CI
2. Verify required binaries exist in expected locations
3. Verify `/etc/wsl.conf` exists and contains `systemd=true`
4. Verify Docker service definitions exist
5. Verify opencode-server installation exists
6. Verify image can be archived successfully

### Nice-to-have tests
If practical, add:
- chroot smoke test execution
- Docker daemon startup validation
- service enablement checks
- tar import instructions validated in docs/examples

Do not build an overcomplicated test harness. Keep it practical and useful.

---

## Implementation Guidance for the Coding Assistant

The coding assistant should proceed in phases.

### Phase 1 — Skeleton and docs
- create repository structure
- write README
- write architecture/build/import docs
- define package manifest and overlays

### Phase 2 — Rootfs bootstrap
- implement Ubuntu bootstrap script
- implement chroot helper logic
- implement package installation flow
- implement overlay application

### Phase 3 — Tool installation
- install Docker and compose support
- install chezmoi
- install devpod-cli
- install opencode-server
- verify PATH and service/unit integration

### Phase 4 — WSL configuration
- add `/etc/wsl.conf`
- enable systemd behavior
- add Docker config
- add service definitions as needed

### Phase 5 — Cleanup and artifact creation
- remove apt caches and transient files
- normalize filesystem state
- create importable tarball artifact

### Phase 6 — CI workflows
- implement GitHub Actions build workflow
- implement validation workflow
- add artifact/release publishing

### Phase 7 — Smoke tests and polish
- implement validation scripts
- tighten docs
- verify outputs and naming
- document manual post-import steps

---

## Constraints the Assistant Must Respect

The coding assistant must not:
- introduce Ansible
- introduce Nix
- redesign this into a generic workstation bootstrap framework
- assume Fedora
- assume Docker Desktop
- automate user creation
- move personal configuration into the image
- require Windows-side helper tooling beyond WSL itself

The assistant must:
- keep the solution image-oriented
- optimize for maintainability over cleverness
- prefer explicit scripts over hidden magic
- keep documentation suitable for a human operator rebuilding the image later

---

## Definition of Done

The project is complete when:

1. A GitHub Actions workflow can build an Ubuntu-based rootfs artifact
2. The artifact can be used with `wsl --import`
3. The image includes:
   - Docker
   - docker-compose support
   - chezmoi
   - devpod-cli
   - opencode-server
4. The image is configured for WSL with systemd enabled
5. No automated user creation is present
6. Documentation clearly explains:
   - build process
   - artifact usage
   - manual first-run user setup
   - expected Chezmoi-based personalization flow
7. The repository is structured clearly enough for ongoing maintenance

---

## Final Instruction to the Coding Assistant

Implement this project as a **clean, maintainable, CI-built Ubuntu WSL rootfs pipeline**.

Prioritize:
- clarity
- explicit scripts
- reproducible builds
- minimal host scope
- WSL compatibility
- maintainable repository structure

Do not optimize for novelty.
Do not add unnecessary abstraction.
Build the smallest solid version of this system first.
