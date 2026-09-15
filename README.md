# Ubuntu 26.04 Agent Workstation Setup

This directory contains Ansible playbooks for multi-user Ubuntu 26.04 agent workstation setup.

## Motivation

Every tool here is installed for **every** account on the host, and that one requirement is what
the repository is about. This is a shared workstation: several identities — people and agents —
reach the same box over SSH and expect the same toolchain, including accounts created after the
tools were installed. Almost every convenient way to install a developer tool on Linux installs it
for exactly one account instead, and most playbooks here are rewrites of single-user predecessors
that did just that. [POLICY.md](playbooks/POLICY.md) is the rule set that came out of those
rewrites.

### Why Homebrew is the wrong tool for a shared box

Homebrew is the usual answer to "just install it", and on Linux it works — for one user. Five
separate properties make it unusable here:

- **The prefix is owned by whoever installed it.** `/home/linuxbrew/.linuxbrew` belongs to a
  single account, and Homebrew upstream does not support multi-user installs. No mode or group
  setting turns one account's Cellar into shared infrastructure.
- **That account can then rewrite the binaries everyone else runs.** Once other users' `PATH`
  reaches into a prefix one unprivileged account owns, that account holds write access to every
  binary the rest of the host executes — a root-adjacent privilege acquired by accident of install
  order rather than granted on purpose. Contrast `docker_users` below, where a comparable grant is
  typed out per account, per run, precisely because it is worth that much.
- **Nothing reaches anyone else's `PATH` without editing their `$HOME`.** Using a brew prefix
  means a `brew shellenv` line in each account's own `~/.bashrc`: a per-identity file no playbook
  here may write (POLICY point 3, B2), repeated once per account, and absent from every account
  created afterwards. The `/etc/environment` and `/etc/profile.d` route these playbooks use has
  none of those properties — it applies to accounts that don't exist yet.
- **It refuses to run as root, so automation has to pick an owner.** A play that is `become: true`
  throughout cannot call `brew` at all; the single-user playbooks worked around it by dropping to
  `lookup('env', 'USER')`, which quietly made whoever happened to run the play the owner of the
  install. Four playbooks — [jira-cli.yml](playbooks/cloud-cli/jira-cli.yml),
  [gcx-cli.yml](playbooks/cloud-cli/gcx-cli.yml),
  [influx-cli.yml](playbooks/cloud-cli/influx-cli.yml) and
  [vault-cli.yml](playbooks/cloud-cli/vault-cli.yml) — exist in their present form specifically to
  undo that.
- **`brew install` has no version pin.** It installs whatever the tap currently carries, and
  `brew pin` only freezes what is already installed — it cannot fetch a version the tap has moved
  past. Every playbook here pins an exact version in the play's `vars` and asserts it again after
  install; see [Pinned versions and `apt upgrade` drift](#pinned-versions-and-apt-upgrade-drift).

Per-user version managers fail the same test in the same way, so they are out for the same
reasons. [core/ruby.yml](playbooks/core/ruby.yml)'s predecessor cloned rbenv and ruby-build into
`~/.rbenv`, built Ruby from source there, and appended `eval "$(rbenv init - bash)"` to that one
account's `~/.bashrc`: every part of it is per-identity, and a second account on the same host got
no `ruby` at all.

### What replaces it

Root-owned system paths (`/usr/local/bin`, `/usr/lib/<tool>`, or apt), `/etc` drop-ins for shell
configuration, exact pinned versions, world-readable modes set explicitly rather than inherited
from the operator's umask, and privilege grants that default to granting nothing. The load-bearing
rule is point 9 of [POLICY.md](playbooks/POLICY.md)'s core ten: every playbook ends by exercising
the tool as an arbitrary uid (`setpriv --reuid=65534`), never as the connecting account, so an
install that only works for the operator fails the run instead of passing unnoticed and breaking
for everyone else.

### What stays per-user on purpose

The goal is not "everything global". Identity and secrets are per-account by design —
`gh auth login`, `jira init`, `~/.databrickscfg`, every `*_TOKEN` — and `/etc/environment` is
world-readable, which is exactly why nothing secret goes there. Per-user caches stay per-user too:
Trivy still builds its vulnerability database under each account's `~/.cache/trivy`. Playbooks
install the client and stop; the interactive, credential-bearing setup each person runs for
themselves is printed in the closing summary and documented in the directory READMEs.

One consequence worth knowing: nothing here removes what a single-user install already left
behind. A surviving brew prefix still shadows the system-wide binary for any account whose
`~/.bashrc` points at it, and cleaning that up would mean writing into accounts' `$HOME`, which
this policy forbids — so it is done by hand. See
[Replacing the Homebrew installs](playbooks/cloud-cli/README.md#replacing-the-homebrew-installs).

## Prerequisites

Ubuntu 24.04 or 26.04 with system packages up to date and Ansible 2.16 or 2.20 installed respectively:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ansible
```

## Passwordless sudo and SSH

Both are host-bootstrap steps rather than part of running these playbooks, and are documented
in [Desktop Host Setup](DESKTOP-HOST-SETUP.md): [Passwordless sudo](DESKTOP-HOST-SETUP.md#passwordless-sudo)
and [Passwordless SSH](DESKTOP-HOST-SETUP.md#passwordless-ssh). Run both first if the remote host
doesn't have them yet — an AWS EC2 instance (or similar cloud image) usually already does: the
AMI grants the default user passwordless sudo and cloud-init injects your key into
`authorized_keys` at launch.

## Playbooks

Root-owned system paths and `/etc` drop-ins, not per-user Homebrew or `~/.bashrc`, so a tool
installed once is usable by every account on a **shared** workstation. Run against a remote host
(`-e host=<inventory host or group>`), not `localhost`.

| Directory | Description |
|-----------|-------------|
| [playbooks/misc/](playbooks/misc/README.md) | Developer tools (asciidoctor, bats, certbot, dotnet-tools, dvc, exiftool, gomplate, grype-syft, hadolint, jsonnet, jsmin, junit2html, k6, kube-score, maven, mocha-chai, mongodb-tools, plantuml, playwright, scc, testssl, trivy, zap) |
| [playbooks/cloud-cli/](playbooks/cloud-cli/README.md) | Cloud/service CLI tools (auth0-deploy-cli, aws, az, az devops, azcopy, databricks, gcloud, gcx, gh, glab, influx, jenkins, jira, logcli, tofu, promtool/amtool, sonar-scanner, vault, Azure PowerShell) |
| [playbooks/core/](playbooks/core/README.md) | Core CLI tools, modern CLI tool replacements, jq, yq, shellcheck, markdownlint, eslint, Node.js/Yarn/pnpm, mise, pyenv, ansible-core, .NET SDK, PowerShell, OpenJDK, Ruby, plus the container runtimes and Kubernetes tools (Docker, Podman, kubectl, Helm, kind, minikube, devcontainers, kubelogin/k9s/kdash) |

One thing to know before running `core/docker.yml`: it grants **no** account access to the
Docker socket unless you name them in `docker_users` — membership of that group is equivalent to
passwordless root, so it's typed out per account, per run. See
[Grants](playbooks/core/README.md#grants). `podman_linger_users` works the same way.

## Pinned versions and `apt upgrade` drift

Every playbook here pins an exact package version and asserts it still matches after install. A
plain `apt upgrade` (or unattended-upgrades) on a target host moves installed versions away from
those pins — that's expected, not a bug: the next playbook run, or a plain `apt-cache policy <pkg>`
/ `dpkg-query -W -f='${Version}' <pkg>`, surfaces the mismatch immediately instead of it drifting
silently.

Resolving it means advancing the pin, not reverting the host: apt archives generally don't keep
superseded `.deb`s around, so reinstalling the old pinned version usually isn't possible. Check
`apt-cache policy <pkg>` on the target, update that package's `*_version` var — and any version
string baked into its `verify_cmd` — to match, then re-run the playbook to reinstall and re-verify.

`core/kubectl.yml` is the one exception: it also pins the apt origin itself via
`/etc/apt/preferences.d/kubectl`, so `apt upgrade` can't swap it for a different vendor's package.
Even there, a version bump within that same origin still needs the pin update above.

## Policy and install mechanisms

Two reference documents describe how these playbooks are built, and are what a new or changed
playbook is checked against:

- **[POLICY.md](playbooks/POLICY.md)** — the rules every playbook satisfies: root-owned paths, pinned
  versions, no writes to any `$HOME`, privilege grants that default to empty, and verification
  that runs as an unprivileged uid. Ends with a review checklist.
- **[INSTALL-MECHANISMS.md](playbooks/INSTALL-MECHANISMS.md)** — how each playbook gets its tool onto the
  filesystem: apt package, vendor apt repository, upstream release artifact, pipx, `npm
  install -g`.
