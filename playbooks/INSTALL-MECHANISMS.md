# Install Mechanisms

A classification of how each playbook under [`playbooks/`](.) gets its tool onto
the filesystem.

Each playbook picks the first of the following that applies: an Ubuntu apt package, if the
distro package is current enough and named as expected; a vendor's own apt repository, if one
exists and the distro package lags; an upstream release artifact straight to
`/usr/local/bin` (or equivalent), for a single static binary or similar; an upstream git tag
plus its own install script, for a tool that ships as a git repository; pipx as root, for a
Python application; or `npm install -g` with `become`, for a Node.js application. 64 playbooks,
six canonical mechanisms plus a handful of second-layer package managers that sit on top of a
shared prerequisite rather than installing a runtime themselves.

| Mechanism | Playbooks |
| --- | ---: |
| 1. Ubuntu apt package | 10 |
| 2. Vendor apt repository | 15 |
| 3. Upstream release artifact → `/usr/local/bin` (or equivalent) | 24 |
| 4. Upstream git tag + install script / tree | 3 |
| 5. pipx as root | 2 |
| 6. `npm install -g` with `become` | 7 |
| Second-layer package manager (see below) | 3 |

## 1. Ubuntu apt package

The distro package is current enough and named as expected — `state: present` at a pinned
version, no repository work needed.

| Playbook | Package(s) |
| --- | --- |
| [core/shellcheck.yml](core/shellcheck.yml) | `shellcheck` |
| [core/openjdk.yml](core/openjdk.yml) | `default-jdk` (pulls in `default-jre` + `default-jdk-headless`) |
| [core/dotnet.yml](core/dotnet.yml) | `dotnet-sdk-10.0` — Ubuntu 26.04 carries .NET 10 directly; no Microsoft repo needed |
| [core/modern-tools.yml](core/modern-tools.yml) | 13 packages (bat→`batcat`+symlink, fd→`fdfind`+symlink, fzf+`/etc/profile.d` hook, plus 10 more) |
| [core/ansible.yml](core/ansible.yml) | `ansible-core` (Galaxy collections on top are a separate mechanism — see below) |
| [cloud-cli/promtool.yml](cloud-cli/promtool.yml) | `promtool`, `amtool` (via `prometheus-alertmanager`, daemon stopped/disabled) |
| [core/podman.yml](core/podman.yml) | `podman`, `podman-compose` |
| [misc/jsonnet.yml](misc/jsonnet.yml) | `jsonnet` |
| [misc/exiftool.yml](misc/exiftool.yml) | `libimage-exiftool-perl` — named after the Perl distribution, not after the `exiftool` CLI it ships |
| [core/ruby.yml](core/ruby.yml) | `ruby`, `ruby-dev`, `ruby3.3`, `ruby-rubygems` — the metapackages pin the *series*, `ruby3.3` the interpreter |

Two apt gotchas worth knowing before assuming a distro package is fine as-is: Ubuntu's `yq` is
the unrelated Python jq-wrapper, not mikefarah's Go `yq` — hence [core/yq.yml](core/yq.yml)
below uses mechanism 3 instead. And apt installs `bat` as `batcat` (name clash with
`bacula-console`) — `modern-tools.yml` adds the `/usr/local/bin/bat` symlink.

## 2. Vendor apt repository

The vendor publishes their own repository and it's kept current; adds a `deb822_repository`
(or equivalent) plus a pinned package.

| Playbook | Repository |
| --- | --- |
| [cloud-cli/azure-cli.yml](cloud-cli/azure-cli.yml) | `packages.microsoft.com` |
| [cloud-cli/azure-devops-cli.yml](cloud-cli/azure-devops-cli.yml) | same Microsoft repo as azure-cli.yml, self-contained by design |
| [cloud-cli/gcloud-cli.yml](cloud-cli/gcloud-cli.yml) | `packages.cloud.google.com` |
| [cloud-cli/github-cli.yml](cloud-cli/github-cli.yml) | `cli.github.com` |
| [cloud-cli/opentofu.yml](cloud-cli/opentofu.yml) | `packages.opentofu.org` |
| [cloud-cli/vault-cli.yml](cloud-cli/vault-cli.yml) | HashiCorp apt repo (de-brewed) |
| [core/docker.yml](core/docker.yml) | `download.docker.com` |
| [core/helm.yml](core/helm.yml) | Helm's own apt repo |
| [core/kubectl.yml](core/kubectl.yml) | `pkgs.k8s.io`, plus an `/etc/apt/preferences.d` pin at priority 1001 to beat `packages.cloud.google.com`'s epoched `kubectl` package — see the file's header for why a plain version pin isn't enough |
| [core/mise.yml](core/mise.yml) | mise's own apt repo |
| [core/nodejs.yml](core/nodejs.yml) | NodeSource, for Node.js itself (Yarn/pnpm are installed afterward — see the mechanism-6 footnote) |
| [misc/dvc.yml](misc/dvc.yml) | Iterative's apt repo |
| [misc/k6.yml](misc/k6.yml) | Grafana's apt repo (amd64 only) |
| [misc/mongodb-tools.yml](misc/mongodb-tools.yml) | MongoDB's official apt repo, for `mongodb-database-tools` only (`mongosh` is a standalone `.deb` — see mechanism 3) |
| [misc/trivy.yml](misc/trivy.yml) | Aqua Security's apt repo |

## 3. Upstream release artifact → `/usr/local/bin` (or equivalent)

A single binary, tarball, `.deb`, or jar fetched directly from upstream (GitHub Releases, a
vendor's own CDN, or a language ecosystem's own artifact repo) and checksum- or
signature-verified, because no apt package or vendor repository exists — or the ones that do
exist are rejected for lagging upstream, being unsigned, or being CPU-incompatible with these
hosts. Most land as a single static binary; a few are structured trees where only the entry
point goes on `PATH`.

| Playbook | Artifact |
| --- | --- |
| [core/yq.yml](core/yq.yml) | single binary — apt's `yq` is the wrong tool (see mechanism 1's gotcha) |
| [misc/gomplate.yml](misc/gomplate.yml) | single binary |
| [misc/hadolint.yml](misc/hadolint.yml) | single binary |
| [misc/scc.yml](misc/scc.yml) | release tarball, verified against the release's `checksums.txt` |
| [misc/kube-score.yml](misc/kube-score.yml) | single binary |
| [core/kind.yml](core/kind.yml) | single binary |
| [core/minikube.yml](core/minikube.yml) | single binary |
| [core/kube-tools.yml](core/kube-tools.yml) | three single binaries: kubelogin, k9s, kdash |
| [cloud-cli/databricks-cli.yml](cloud-cli/databricks-cli.yml) | single Go binary |
| [cloud-cli/gcx-cli.yml](cloud-cli/gcx-cli.yml) | release tarball (de-brewed) |
| [cloud-cli/jira-cli.yml](cloud-cli/jira-cli.yml) | release tarball (de-brewed) |
| [cloud-cli/influx-cli.yml](cloud-cli/influx-cli.yml) | tarball from `dl.influxdata.com`, not GitHub — versioned directory + symlink (de-brewed) |
| [cloud-cli/loki-cli.yml](cloud-cli/loki-cli.yml) | zip, verified against the release's `SHA256SUMS` |
| [cloud-cli/azcopy.yml](cloud-cli/azcopy.yml) | release tarball — upstream publishes no checksums file at all, so each asset's sha256 is pinned in the playbook and tied to one exact version |
| [misc/grype-syft.yml](misc/grype-syft.yml) | each project's own `install.sh`, pinned to the release tag (not `main`), which resolves and installs the binary itself |
| [misc/maven.yml](misc/maven.yml) | tarball from Apache (not GitHub) — versioned directory + symlink |
| [misc/zap.yml](misc/zap.yml) | ~270 MB distribution zip from the GitHub release — versioned directory + symlink to the launcher |
| [core/pwsh.yml](core/pwsh.yml) | tarball from Microsoft — no `powershell` package exists for Ubuntu 26.04 at all; versioned directory + symlink |
| [cloud-cli/aws-cli.yml](cloud-cli/aws-cli.yml) | vendor's own zip + `install` program (not ansible's `unarchive`) — versioned directory + symlinks, GPG-signature verified |
| [cloud-cli/aws-sam-cli.yml](cloud-cli/aws-sam-cli.yml) | the same vendor zip + `install` program as `aws-cli.yml` — versioned directory + symlinks; GPG-signature verified against a signer key whose certification by AWS's primary key is checked too |
| [cloud-cli/jenkins-cli.yml](cloud-cli/jenkins-cli.yml) | single jar from `repo.jenkins-ci.org` (a Maven repository, not GitHub) |
| [misc/plantuml.yml](misc/plantuml.yml) | single jar from Maven Central, verified against the `.sha256` beside it — the GitHub release publishes only detached `.asc` signatures; apt's `plantuml` is PlantUML 1.2020.2 |
| [cloud-cli/gitlab-cli.yml](cloud-cli/gitlab-cli.yml) | vendor `.deb` fetched and installed with `apt: deb=` — no vendor apt repo exists |
| [cloud-cli/sonar-scanner.yml](cloud-cli/sonar-scanner.yml) | zip from `binaries.sonarsource.com` (GitHub's releases carry no assets), verified against the `.sha256` beside it — versioned directory + symlink; bundles its own JRE, so no JDK prerequisite |

**Footnote:** [misc/mongodb-tools.yml](misc/mongodb-tools.yml) (mechanism 2 above,
via MongoDB's apt repo) installs `mongosh` this same `apt: deb=` way instead, inside the same
file — the one apt suite that carries `mongosh` is signed by a key MongoDB no longer publishes,
so its repository can't be trusted the way `mongodb-database-tools`' can.

## 4. Upstream git tag + install script / tree

The tool ships as a git repository rather than a packaged release.

| Playbook | Shape |
| --- | --- |
| [misc/bats.yml](misc/bats.yml) | shallow clone pinned to a tag, then upstream's own `install.sh`; `bats-support`/`bats-assert` helper libraries cloned in full alongside it |
| [misc/testssl.yml](misc/testssl.yml) | clone pinned to a tag **and** the commit it pointed to (no installer) — the whole tree is kept, since the script resolves its cipher data and bundled OpenSSL build relative to its own location, and only the entry-point script is symlinked onto `PATH` |
| [core/pyenv.yml](core/pyenv.yml) | clone pinned to a tag, kept whole (no installer) — pyenv resolves its libexec and its bundled `python-build` plugin relative to its own location, so only `bin/pyenv` is symlinked onto `PATH` |

## 5. pipx as root

A Python application, installed with `PIPX_HOME=/opt/pipx` and `PIPX_BIN_DIR=/usr/local/bin`
(plus `PIPX_MAN_DIR`, to keep pipx from writing under `/root`) rather than into the
connecting account's own `~/.local`.

| Playbook | Package |
| --- | --- |
| [misc/junit2html.yml](misc/junit2html.yml) | `junit2html` |
| [misc/certbot.yml](misc/certbot.yml) | `certbot`, plus the Route 53 DNS plugin via `pipx inject` |

## 6. `npm install -g` with `become`

A Node.js application, installed into npm's global prefix (root-owned) rather than a
per-user path. Node.js itself is always a declared prerequisite, never installed by these
playbooks — see the prerequisite table below.

| Playbook | Package(s) |
| --- | --- |
| [core/eslint.yml](core/eslint.yml) | `eslint` (needs `NODE_PATH` published so a project's config file can `require('eslint/config')`) |
| [core/markdownlint.yml](core/markdownlint.yml) | `markdownlint-cli` |
| [cloud-cli/auth0-deploy-cli.yml](cloud-cli/auth0-deploy-cli.yml) | `auth0-deploy-cli` |
| [core/devcontainers.yml](core/devcontainers.yml) | `@devcontainers/cli` |
| [misc/jsmin.yml](misc/jsmin.yml) | `jsmin` (Crockford's ES5-era minifier; no `--version`, so the pin is read from `npm ls -g`) |
| [misc/mocha-chai.yml](misc/mocha-chai.yml) | `mocha` (has a CLI) + `chai` (pure library, no CLI — needs `NODE_PATH` published for `require()` to resolve it) |
| [misc/playwright.yml](misc/playwright.yml) | `playwright`, plus its own `playwright install --with-deps` browser downloader redirected to a shared `/opt/playwright-browsers` via `PLAYWRIGHT_BROWSERS_PATH` |

**Footnote:** [core/eslint.yml](core/eslint.yml) is the one place where mechanism 1 and
mechanism 6 collide over a path. Ubuntu's universe `eslint` package (6.4.0, `.eslintrc`-era)
ships `/usr/bin/eslint`, which is also where npm puts its global bin symlink when the prefix is
`/usr` — and npm refuses the *entire* install with `EEXIST` rather than overwriting a file it
does not own (POLICY.md C2). The playbook checks for that package and fails with the purge
remedy instead of deleting an apt-owned symlink.

**Footnote:** [core/nodejs.yml](core/nodejs.yml) also uses this mechanism internally,
for Yarn and pnpm — but only after `corepack disable` removes the dispatcher shims NodeSource's
`nodejs` package ships at the same paths, which otherwise silently shadow a real pinned install.

## Second-layer package managers

These install *into* a runtime the playbooks above already installed, rather than installing a
runtime themselves — how a tool, once present, gets *its own* plugins or packages onto the box,
distinct from how the tool itself got there.

| Playbook | Mechanism | Needs |
| --- | --- | --- |
| [cloud-cli/azure-pwsh.yml](cloud-cli/azure-pwsh.yml) | `Install-Module -Scope AllUsers` from the PowerShell Gallery (8 `Az.*` modules) | [core/pwsh.yml](core/pwsh.yml) |
| [misc/dotnet-tools.yml](misc/dotnet-tools.yml) | `dotnet tool install --tool-path` | [core/dotnet.yml](core/dotnet.yml) |
| [misc/asciidoctor.yml](misc/asciidoctor.yml) | `gem install` as root, into RubyGems' own `Gem.default_dir` with binstubs in `Gem.bindir` (4 gems) | [core/ruby.yml](core/ruby.yml), [core/openjdk.yml](core/openjdk.yml) |

Two more instances of the same pattern live inside playbooks already classified above, layered
on top of their own apt package rather than a separate file:

- [core/ansible.yml](core/ansible.yml) installs `ansible-core` via apt
  (mechanism 1), then pins five Galaxy collections with
  `ansible-galaxy collection install --force-with-deps`.
- [cloud-cli/azure-devops-cli.yml](cloud-cli/azure-devops-cli.yml) installs `az` via
  apt (mechanism 2), then adds the `azure-devops` extension with `az extension add --system`.

## Prerequisite, not installed here

Several playbooks check for a shared runtime and fail with instructions rather than installing
it themselves — the runtime has its own playbook, which every consumer defers to instead of
carrying its own copy of the pin.

| Prerequisite | Provisioned by | Checked (not installed) by |
| --- | --- | --- |
| Node.js | [core/nodejs.yml](core/nodejs.yml) | eslint.yml, markdownlint.yml, auth0-deploy-cli.yml, devcontainers.yml, jsmin.yml, mocha-chai.yml, playwright.yml |
| OpenJDK | [core/openjdk.yml](core/openjdk.yml) | jenkins-cli.yml, maven.yml, zap.yml, asciidoctor.yml, plantuml.yml |
| .NET SDK | [core/dotnet.yml](core/dotnet.yml) | dotnet-tools.yml |
| PowerShell | [core/pwsh.yml](core/pwsh.yml) | azure-pwsh.yml |
| Ruby | [core/ruby.yml](core/ruby.yml) | asciidoctor.yml |

Each of these five runtimes is intentionally installed and pinned in exactly one place, so a
Java (or Node, .NET, PowerShell, Ruby) version bump happens once instead of drifting across
every playbook that happens to need it.
