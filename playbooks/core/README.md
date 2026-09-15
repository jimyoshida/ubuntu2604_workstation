# Core Playbooks (multi-user workstations)

Standalone playbooks that install Node.js, mise, ansible-core, the .NET SDK, PowerShell,
OpenJDK, Ruby and the container and Kubernetes tooling (Docker, Podman, kubectl, Helm, kind,
minikube, the Dev Containers CLI, kubelogin/k9s/kdash) on a **shared**
Ubuntu workstation, to root-owned system paths usable by every account on the host.
See [POLICY.md](../POLICY.md) for the rules they follow.

Run from `playbooks/`:

```bash
ansible-playbook core/<tool>.yml -e host=<inventory host or group>
```

## Conventions

These playbooks follow the same rules as [`misc/`](../misc/README.md) and
[`cloud-cli/`](../cloud-cli/README.md) — root-owned system paths, pinned versions, no writes to
any `$HOME`, and a closing check that runs the tool as an arbitrary uid
(`setpriv --reuid=65534`) rather than as the connecting account.

Most of what is here needs nothing beyond that: `nodejs.yml` is apt-only and root-owned,
and `mise.yml` needs nothing beyond the `/etc/profile.d` hook it installs itself. The
container and Kubernetes playbooks — [`docker.yml`](#dockeryml),
[`podman.yml`](#podmanyml), [`kubectl.yml`](#kubectlyml), [`helm.yml`](#helmyml),
[`kind.yml`](#kindyml), [`minikube.yml`](#minikubeyml),
[`devcontainers.yml`](#devcontainersyml) and [`kube-tools.yml`](#kube-toolsyml) — install
*runtimes* rather than clients, and four rules matter specifically for them:

1. **A privilege grant takes an explicit list and defaults to empty.** Adding an account
   to the `docker` group, or enabling `loginctl` lingering for it, is a grant made to a
   named person — not part of installing software. No playbook here grants anything
   unless you name the accounts, and none of them ever infers the account from whoever
   ran the playbook. See [Grants](#grants).
2. **Per-user runtime state belongs to the user.** `~/.kube/config`, `~/.minikube/`,
   `~/.docker/`, `~/.local/share/containers` — these appear the first time an account
   *uses* a tool, and it is correct that they are per-account. No playbook here creates,
   shares, or pre-populates them, including for the connecting account.
3. **Verification proves an unprivileged account can run the client, not that a cluster
   exists.** Every playbook ends by exercising the tool as uid 65534. What that can prove
   varies by tool and is stated per playbook below; where a daemon or a cluster is the
   obstacle, the check is deliberately modest rather than skipped.
4. **`kind` and `minikube` are only as multi-user as the `docker` group is.** Both drive
   Docker, so each installs a binary every account can execute, but only accounts in
   `docker_users` can actually use either for anything. That is the tools' own dependency
   on Docker, not a limitation of these playbooks.

## Grants

Two of these playbooks can hand out access that the installation itself does not need.
Both take an explicit list, both default to **empty**, and both are **additive** — an
account you leave out keeps whatever it already has, because revoking access should be a
deliberate act rather than a side effect of running with a shorter list.

| Variable | Playbook | What it grants | Worth |
| --- | --- | --- | --- |
| `docker_users` | `docker.yml` | membership of the `docker` group | **equivalent to passwordless root** on this host |
| `podman_linger_users` | `podman.yml` | `loginctl enable-linger` | the account's user services survive logout |

Both accept a comma-separated string or a YAML list — the same shape every named-account var in
this repo takes:

```bash
ansible-playbook core/podman.yml -e host=ws01 -e podman_linger_users=alice,bob
```

> **Quoting matters.** `-e podman_linger_users="alice, bob"` splits on whitespace at the
> shell and binds only `alice,`; a trailing comma drops an account the same way. Each
> playbook prints the accounts it is about to act on before it acts — read that line.

## No playbook here touches X server access

Neither `docker.yml` nor `podman.yml` grants any account access to the host X server —
that is not part of installing a container runtime. A containerised GUI application does
not reach the host X server just because an account ran one of these playbooks. An account
that wants that runs `xhost` itself, per session, having decided to:

```bash
xhost +local:            # grants local connections until the X session ends
```

If `~/.bashrc` on an account here already carries an `xhost +local:docker` or
`xhost +local:podman` line from an older setup, these playbooks will not remove it —
that is a manual, per-account cleanup.

## nodejs.yml

Installs Node.js LTS from the NodeSource apt repository, plus Yarn Classic, pnpm and two audit
report renderers as real npm-global packages.

| Path | Contents |
| --- | --- |
| `/usr/bin/node`, `/usr/bin/npm`, `/usr/bin/npx` | from the NodeSource `nodejs` package |
| `/usr/bin/yarn`, `/usr/bin/pnpm` | real npm-global installs |
| `/usr/bin/audit-export`, `/usr/bin/yarn-audit-html` | npm-global installs — render `npm`/`yarn audit --json` as HTML |
| `/etc/apt/sources.list.d/nodesource.sources` | repository definition, NodeSource's key inline |

Every npm-global package comes from one list (`nodejs_npm_globals`), which drives the install,
the per-package `npm ls` pin check and the world-readable pass alike, so adding one is a single
list entry.

**The Yarn/pnpm handling is the tricky part here.** NodeSource's `nodejs` package bundles
Corepack, which ships `/usr/bin/yarn` and `/usr/bin/pnpm` as its own dispatcher shims from
the moment `nodejs` is installed. Left alone, those shims download whatever version is
newest on the npm registry the first time any account invokes them — non-deterministic
across accounts and across time, into that account's own cache.

The fix: `corepack disable` removes Corepack's shims, then
`npm install -g yarn@<version> pnpm@<version>` — the same mechanism
`markdownlint.yml` (below) uses for a root-owned global npm install — puts real, pinned
packages at the same paths. The idempotency check never runs `yarn`/`pnpm` directly while a
Corepack shim might still be at that path, since doing so would trigger the exact
non-deterministic download this playbook exists to prevent; it resolves the binary with
`readlink -f` to check whether the path leads into Corepack's own tree, and separately asks
npm's own install metadata (`npm ls -g <pkg>@<version>`) rather than executing the binary at
all. Only once that confirms a real npm-managed install does the playbook invoke
`yarn --version` / `pnpm --version`, for the summary and the unprivileged smoke test.

### The audit report renderers

`audit-export` renders `npm audit --json` (it also accepts pnpm's and yarn's) and
`yarn-audit-html` renders `yarn audit --json` and Berry's `yarn npm audit --json`. Each is
verified by feeding it a synthetic advisory and asserting the module name survives into the
report — proof it parsed the input rather than emitting a template.

**`audit-export` embeds its findings base64-encoded** in the page rather than as markup, so its
check decodes the payload before asserting on it. Grepping the HTML for a package name finds
nothing even when the report is perfectly good — worth knowing before writing a check against
it. `yarn-audit-html` needs no such decoding, but it does need `npm ls` rather than `--version`
for its pin check, because it implements no `--version` flag at all (`error: unknown option
'--version'`).

**`npm-audit-html` is deliberately not installed**, and the reason is recorded so nobody adds it
back by reflex. Its templates read the `advisories` key of npm v6's audit JSON; npm 7 replaced
that with `auditReportVersion: 2` and a `vulnerabilities` map, and the package has had no
release since 2020-11-11. Measured here before it was dropped: a real `npm audit --json` for a
project with a **critical** lodash advisory rendered to a report **byte-for-byte identical** to
one rendered from an empty audit, with no mention of lodash in it. It installs, it runs, and it
reports nothing — worse than not having it. `audit-export` reads the same audit correctly, which
is why it takes that place.

Repository setup uses a declarative `deb822_repository`: NodeSource's signing key carries no
expiry, so it is pinned by content inline (the `docker.yml`/`azure-cli.yml` form), and
architecture comes from facts.

**A confirmed, apt-module-level limitation worth knowing before bumping this pin.** `apt`'s
`allow_downgrade: true` is necessary but not always sufficient: on this host's `python3-apt`
(3.1.0ubuntu1), the module's synthetic priority-1001 version pin does not reliably beat an
already-installed *different* version unless the requested version is also apt's current
candidate — reproduced directly against `apt_pkg.Policy.get_candidate_ver()`, independently of
Ansible, while plain `apt-get install nodejs=<version> --allow-downgrades` resolves the identical
request correctly. This is not particular to `nodejs.yml` — every `playbooks/` playbook that
pins an apt package the same way carries the same exposure — and it does not affect the normal
case this playbook is written for (installing the pinned version onto a host with an older or no
`nodejs` at all). It only bites when the pin lags what apt currently offers as the candidate;
check `apt-cache policy nodejs` before overriding `nodejs_version` to something other than the
current candidate.

Version overrides — the stream is part of the pin, since NodeSource publishes one repository per
major version:

```bash
ansible-playbook core/nodejs.yml -e host=ws01 -e nodejs_version=24.19.0-1nodesource1
```

## mise.yml

Installs mise from its own apt repository.

| Path | Contents |
| --- | --- |
| `/usr/bin/mise` | the binary |
| `/etc/apt/sources.list.d/mise.sources` | repository definition |
| `/etc/profile.d/mise.sh` | live `eval "$(mise activate bash)"`, interactive shells only |

mise's shell integration goes to `/etc/profile.d/mise.sh` system-wide — the same
destination and the same shape (a live `eval`, not a captured snapshot) as
`modern-tools.yml`'s `fzf.sh` — rather than into any one account's `~/.bashrc` (see
[POLICY.md's B3](../POLICY.md) on why a `PATH`/env-rewriting hook belongs there and not among
completions or static aliases). `/etc/bash.bashrc` needs the `/etc/profile.d` bootstrap for
non-login interactive shells to read it, which this playbook lays down defensively in case
it runs on a host no other playbook has touched yet.

**mise writes to `$HOME` on a plain `--version`, so this playbook never runs mise itself as
root.** Confirmed live: `HOME=<scratch> mise --version` creates
`<scratch>/.cache/mise/latest-version`, a self-update-check cache, from the plainest possible
invocation. The install and post-install version checks compare `dpkg-query` output instead —
running `mise --version` as root would leave `/root/.cache/mise` behind, which is exactly what
[POLICY.md's B2](../POLICY.md) forbids ("no task ... may write ... under any account's `$HOME`, including the
invoker's"). The unprivileged checks give mise a scratch, writable `HOME` for the same reason
every other tool in this repo that touches `$HOME` gets one.

**A separate, confirmed-harmless quirk.** A `[WARN] migrate: error parsing config file: ...` line
names the *real* invoking account's `~/.config/mise/config.toml` even when `$HOME` has been
overridden — confirmed with `strace` and `env -i` that this one path resolves via the OS user
database rather than `$HOME`, unlike every other piece of mise state (including the cache write
above), which does respect it. It is a read of a path that does not exist for uid 65534 or for
root, so it is noise, not a defect this playbook works around.

The repository's signing key expires 2028-01-02, so it is fetched by URL each run and only its
fingerprint is pinned (the `github-cli.yml`/`kubectl.yml` form).

Version overrides:

```bash
ansible-playbook core/mise.yml -e host=ws01 -e mise_version=2026.8.6
```

## dotnet.yml

Installs the .NET SDK from the Ubuntu archive.

| Path | Contents |
| --- | --- |
| `/usr/bin/dotnet` | the CLI |
| `/usr/lib/dotnet/` | SDK, runtimes, targeting packs, templates |

**No Microsoft apt repository and no vendor key**: Ubuntu 26.04 carries .NET 10 itself, as
`dotnet-sdk-10.0`, which pulls the host, runtime, ASP.NET runtime, targeting packs, templates
and the AOT components with it — thirteen packages in all. That is a change from earlier
releases, where `packages.microsoft.com` was the only source.

**Per-user state is the norm with .NET, and this playbook creates none of it.** Each account
gets its own `~/.dotnet` (CLI state, first-run sentinel) and `~/.nuget/packages` (the package
cache) on first use. A shared NuGet cache is deliberately not created: one account's restore
should not decide what another builds against.

That same fact makes the CLI unsafe to run as root here — `dotnet --version` alone creates
`/root/.dotnet`, which [POLICY.md's B2](../POLICY.md) forbids — so the idempotency and version checks read
`dpkg-query` (as `mise.yml` does) and every actual `dotnet` invocation is unprivileged with a
scratch `HOME`.

The unprivileged verification is the strongest form available: it creates a console project
from the SDK's own template, builds it and runs it, so the templates, the compiler, the runtime
and NuGet restore are all exercised for an account that did no setup of its own.

Telemetry is left to each account (`DOTNET_CLI_TELEMETRY_OPTOUT=1`), the same way
[`misc/dvc.yml`](../misc/README.md#dvcyml) leaves DVC's analytics alone; the playbook sets it
only for its own checks.

This is the prerequisite for [`misc/dotnet-tools.yml`](../misc/README.md#dotnet-toolsyml).

Version overrides:

```bash
ansible-playbook core/dotnet.yml -e host=ws01 -e dotnet_version=10.0.110-0ubuntu1~26.04.1
```

## pwsh.yml

Installs [PowerShell](https://learn.microsoft.com/powershell/) 7.6.5 from the upstream binary
archive, in the versioned-directory-plus-symlink shape
[`misc/maven.yml`](../misc/README.md#mavenyml) uses.

| Path | Contents |
| --- | --- |
| `/usr/local/lib/powershell/<version>/` | the unpacked release (~171 MB) |
| `/usr/local/bin/pwsh` | symlink to that version's `pwsh` |

**Not apt, and the reason is measured.** Microsoft publishes **no `powershell` package for
Ubuntu 26.04 at all** — its 26.04 repository (`packages.microsoft.com/ubuntu/26.04/prod`, suite
`resolute`) carries none at any version, checked 2026-08-17. Installing from apt would mean
pointing a 26.04 host at the 24.04 `noble` suite and hoping the dependencies line up. The binary
archive Microsoft publishes for exactly this case is self-contained: `libicu` is its only real
system dependency, and `ldd` reports nothing missing for `pwsh`. The pin can then be the current
release, verified against the SHA-256 GitHub publishes for the asset.

### Root never runs pwsh here

PowerShell writes `~/.cache/powershell` (including `telemetry.uuid` and startup profile data),
`~/.config/powershell` and `~/.local/share/powershell` — confirmed live, the cache appears on the
plainest possible invocation. Running it as root would leave that under `/root`, which
[POLICY.md's B2](../POLICY.md) forbids, so the idempotency check reads the filesystem (the versioned path and
the symlink target) and every `pwsh` invocation is unprivileged with a scratch `HOME`.

### The AllUsers module scope is asserted, not assumed

No modules are installed — which set a workstation needs is not this playbook's decision — but
the check does assert that `/usr/local/share/powershell/Modules` is on `PSModulePath` for an
unprivileged account. That is what makes a later `Install-Module -Scope AllUsers` land somewhere
every account can read rather than in one account's `~/.local/share/powershell`, and a tarball
install could plausibly have left it off.

The rest of the unprivileged check is real work: a script builds an object, round-trips it
through `ConvertTo-Json`/`ConvertFrom-Json` and reports the version, so the cmdlet pipeline and
the type system are exercised rather than a version string. It runs from a file rather than
`-Command`, which keeps PowerShell source out of four layers of YAML, Jinja and shell quoting.

Telemetry is left to each account (`POWERSHELL_TELEMETRY_OPTOUT`), the same way
[`misc/dvc.yml`](../misc/README.md#dvcyml) leaves DVC's analytics alone.

Version overrides:

```bash
ansible-playbook core/pwsh.yml -e host=ws01 -e pwsh_version=7.6.5
```

## openjdk.yml

Installs OpenJDK from the Ubuntu archive's `default-jdk` metapackage.

| Path | Contents |
| --- | --- |
| `/usr/bin/java`, `/usr/bin/javac` | via `update-alternatives` |
| `/usr/lib/jvm/java-<major>-openjdk-<arch>/` | the JDK itself |

**One pinned package for every Java-based playbook in this repository to share, not three.**
`misc/maven.yml`, `cloud-cli/jenkins-cli.yml` and `misc/zap.yml` each used to install their own
JDK or JRE package inline — `default-jdk-headless`, `default-jre-headless` and `default-jre`
respectively — which meant the same underlying package could end up pinned three different ways
across three files on one host. `default-jdk` is not headless, and pulls in the full JDK rather
than just a JRE, deliberately: its dependency chain (`default-jdk` → `default-jre` +
`default-jdk-headless`; `default-jdk-headless` → `default-jre-headless` + `openjdk-<N>-jdk`)
covers the union of what all three consumers need — `javac` for Maven, a runtime for the Jenkins
CLI, and the non-headless runtime ZAP's desktop UI needs — confirmed with
`apt-cache depends default-jdk`.

Each of the three downstream playbooks now only checks for the binary it needs (`javac` or
`java`) and fails with instructions to run this playbook first, the shape
[`misc/dotnet-tools.yml`](../misc/README.md#dotnet-toolsyml) and
[`cloud-cli/azure-pwsh.yml`](../cloud-cli/README.md#azure-pwshyml) use for
[`dotnet.yml`](#dotnetyml) and [`pwsh.yml`](#pwshyml).

**No scratch `HOME` in the verification, unlike every other Java-based playbook here.** The JVM
reads `user.home` from the OS user database, not from `$HOME` — confirmed live,
`HOME=/nonexistent java ...` still resolves `user.home` to the real account's passwd entry —
which is exactly the trap [`misc/maven.yml`](../misc/README.md#mavenyml) and
[`misc/zap.yml`](../misc/README.md#zapyml) document for their own per-tool state (`~/.m2`,
`~/.ZAP`). A plain compile-and-run never touches `user.home` at all, so the check needs only a
writable `chdir`, not a scratch `HOME`: an unprivileged account compiles a real program with
`javac` and runs it with `java`, end to end.

This is the prerequisite for [`misc/maven.yml`](../misc/README.md#mavenyml),
[`cloud-cli/jenkins-cli.yml`](../cloud-cli/README.md#jenkins-cliyml) and
[`misc/zap.yml`](../misc/README.md#zapyml).

Version overrides:

```bash
ansible-playbook core/openjdk.yml -e host=ws01 -e openjdk_version=2:1.25-77
```

## modern-tools.yml

Installs 13 modern CLI tool replacements from apt, root-owned, at `/usr/bin` (no per-user
state, nothing added to a shell profile for any of them):

| Tool | apt package | Binary |
| --- | --- | --- |
| gum, fzf, eza, lsd, duf, procs, gdu, htop, glow | same name | same name |
| ripgrep | `ripgrep` | `rg` |
| bat | `bat` | `batcat` (symlinked to `/usr/local/bin/bat`) |
| fd | `fd-find` | `fdfind` (symlinked to `/usr/local/bin/fd`) |
| dust | `du-dust` | `dust` |

**Two of them ship under a different binary name**, because the upstream name is already
taken on Debian: `bat` is installed as `batcat` (the bacula tools own `/usr/bin/bat`) and `fd`
as `fdfind` (`fdclone`, a file manager, owns `/usr/bin/fd`). Both are published under their
upstream name from `/usr/local/bin`, which precedes `/usr/bin` on the default `PATH`, rather
than left to each account to alias for itself — and the verification runs them *through* `PATH`
as `nobody`, so it proves the symlink resolves and wins rather than merely existing.

Also lays down:

| Path | Contents |
| --- | --- |
| `/etc/profile.d/fzf.sh` | `eval "$(fzf --bash)"` — fzf key bindings and completion |

`yq.yml` lives as its own playbook in this directory (below) rather than bundled here, and
`jsonnet.yml` lives in [`misc/`](../misc/README.md) — each pinned and verified independently.
`jq` was split out the same way once, then folded into the `misc-tools.yml` bundle; neither
is here any more, so no playbook in this directory installs it.

**No vendor apt repo needed for `gum` and `glow`.** Ubuntu 26.04 ("resolute") carries apt
packages for both directly (`gum` 0.17.0-1, `glow` 2.1.1-1), so a single apt install covers
the whole bundle with no vendor repository required.

**`llhttp` is not part of this bundle.** apt's `eza` package declares `libgit2-1.9` etc. as
proper package dependencies, so no separate library symlink is needed for it to find its
runtime library.

fzf's key bindings need `/etc/profile.d` to actually be read by interactive shells, which is
not true by default for a non-login shell (e.g. a plain SSH session) on stock Ubuntu — see
[POLICY.md's B3](../POLICY.md). This playbook adds that hook to `/etc/bash.bashrc`; the
task is an idempotent no-op for any other playbook that adds the same hook.

The unprivileged verification step runs every tool's `--version` as `nobody`, then separately
opens a non-login interactive `bash -i` shell as the same user and checks that fzf's
`__fzf_select__` function is defined — proving the `/etc/profile.d` → `/etc/bash.bashrc` chain
actually works end-to-end for a real interactive session, not just that the binaries exist.

apt package pins are release-specific (see `shellcheck.yml`'s note on the same issue,
below); to override one, edit `modern_tools_packages` with `-e` as a JSON list, the same
pattern [`misc/grype-syft.yml`](../misc/README.md) uses for its tool list.

## ansible.yml

Installs [ansible-core](https://github.com/ansible/ansible) from the Ubuntu apt package —
ten CLI entrypoints under `/usr/bin`, root-owned — plus five pinned Galaxy collections into
`/usr/share/ansible/collections`. There is no per-user state and nothing to add to a shell
profile: that directory is already the system-wide half of ansible-core's default
`COLLECTIONS_PATH` (`~/.ansible/collections:/usr/share/ansible/collections`), so every account
picks the collections up with no environment variable and no config of its own.

| Path | Contents |
| --- | --- |
| `/usr/bin/ansible{,-config,-console,-doc,-galaxy,-inventory,-playbook,-pull,-test,-vault}` | from the `ansible-core` apt package |
| `/usr/share/ansible/collections` | `ansible.posix`, `community.general`, `community.docker`, `community.postgresql`, `amazon.aws` |

**ansible-core, not the `ansible` community bundle.** apt carries both: `ansible`
(13.1.0+dfsg-1ubuntu1 on 26.04) is the batteries-included collection bundle that depends on
`ansible-core`, a much larger install with its own collection versioning. This playbook pins
the engine and exactly the five collections above, and neither installs nor removes the
bundle — though it does declare `Depends: ansible-core (>= 2.18.0~)`, so pinning
`ansible_core_version` under that floor makes apt refuse the transaction rather than
silently break the bundle.

**The bundle, where it is installed, is also what makes the collection handling non-trivial.**
It ships four of these five collections under `/usr/lib/python3/dist-packages/ansible_collections`
at its own versions (measured on ws01: `amazon.aws` 10.1.2, `ansible.posix` 2.1.0,
`community.docker` 5.0.4, `community.general` 12.1.0, `community.postgresql` 4.2.0 — all older
than the pins here except postgresql, which matches exactly). Two consequences the playbook is
built around:

- **`ansible-galaxy collection install` silently does nothing** when the collection is visible
  in *any* path, including that bundle's. Confirmed live with `community.postgresql` 4.2.0: the
  install reported success-by-omission and nothing landed in the target path. Hence
  `--force-with-deps` — the `deps` half matters too, or a dependency that exists only in the
  bundle's tree (`community.library_inventory_filtering_v1`, for `community.docker`) is skipped
  as well, leaving this install dependent on a package it does not own.
- **The idempotency check reads the version installed at the target path**, not whether the
  collection is installed at all — and it keys on `<path>/ansible_collections`, which is what
  `ansible-galaxy collection list --format json` reports, not the path passed to `-p`.

**`ansible` writes to `$HOME` on a plain `--version`**, so, exactly as in `mise.yml`, this
playbook never runs ansible itself as root: `dpkg-query` supplies both the idempotency check and
the version verification. `ansible-galaxy` is worse — it writes a `galaxy_token` and a
`galaxy_cache/`— and it is the one command here that *has* to run as root, so it gets a scratch
`HOME`/`ANSIBLE_HOME` that does not outlive the play.

**A scratch `$HOME` alone is not enough for the unprivileged checks**, which is the one genuinely
non-obvious finding here. ansible resolves `remote_tmp` (default `~/.ansible/tmp`) against the
*remote user's* home as read from the OS user database, and identifies that user from
`USER`/`LOGNAME` in the environment — not from `$HOME`. Under `become: true` those are still
root, so a smoke test that only overrides `HOME` has ansible try to `mkdir /root/.ansible/tmp` as
uid 65534 and fail with "Failed to create temporary directory"; clearing `USER`/`LOGNAME` instead
just moves the target to nobody's `/nonexistent`, which is no more writable. Both were reproduced
directly on a target, which is why `ANSIBLE_REMOTE_TEMP` is set explicitly rather than inferred.

The unprivileged verification is three checks as `nobody`, all of them level-2 rather than a bare
`--version`:

1. `ansible --version` reports the pinned core version.
2. For each collection, `ansible-doc -t module -F <collection>` prints the file path ansible's own
   plugin loader resolved — asserted to be under `/usr/share/ansible/collections`. This is the
   empirical proof that a configured collections path beats the bundle's copies on the PYTHONPATH,
   and incidentally that the installed tree is world-readable.
3. A real one-task playbook runs over a local connection, exercising inventory parsing, Jinja
   templating, a filter plugin loaded out of `community.general`, and the module transfer that
   needs a writable temp directory. `ANSIBLE_NOCOWS=1` keeps cowsay, where it is installed, from
   decorating the output the check parses.

Collection pins are the latest release of each on galaxy.ansible.com as of 2026-08-17, read from
the Galaxy v3 API rather than assumed. Galaxy publishes often — re-check before treating them as
current:

```bash
curl -sS https://galaxy.ansible.com/api/v3/plugin/ansible/content/published/collections/index/community/general/ | jq .highest_version
```

Version overrides — the apt pin is Ubuntu-release-specific in the same way `shellcheck.yml`'s is;
each collection has its own variable:

```bash
ansible-playbook core/ansible.yml -e host=ws01 -e ansible_core_version=2.20.1-1
ansible-playbook core/ansible.yml -e host=ws01 -e ansible_collection_community_general_version=13.3.0
```

## eslint.yml

Installs [ESLint](https://eslint.org/) via `npm install -g`, root-owned. Node.js itself is a
prerequisite provisioned elsewhere ([nodejs.yml](nodejs.yml)); this playbook only checks for
it, and fails loudly with setup instructions if it is missing. The same npm-global mechanics
as `markdownlint.yml` below apply: pinned version rather than `@latest`, version-aware
idempotency, npm's global prefix read from `npm config get prefix` rather than assumed, and
the node-in-PATH guard against a per-user version manager capturing a global install.

Three things are specific to ESLint, all confirmed live against Node.js 24.19.0:

- **Ubuntu's own `eslint` package must not be installed.** The universe package (6.4.0, still
  `.eslintrc`-only, so not a substitute anyway) owns `/usr/bin/eslint` as a symlink into
  `/usr/share/nodejs/eslint/bin/eslint.js` — which is exactly where npm puts its global bin
  symlink when the prefix is `/usr`, as it is on a NodeSource host. npm then refuses the
  *entire* install with `EEXIST` and nothing lands in `/usr/lib/node_modules` at all. The
  playbook checks for that package and fails with `sudo apt-get purge eslint` rather than
  deleting the symlink: it resolves into an apt-owned tree, not into the npm tree being
  replaced, so removing it would be a write behind dpkg's back. The guard does not consult the
  prefix first, because the other case is no better: where the prefix is `/usr/local` the npm
  install succeeds, and apt's 6.4.0 copy is then a second eslint sitting at `/usr/bin/eslint`.
- **`NODE_PATH` is published to `/etc/environment`,** the same line
  [misc/mocha-chai.yml](../misc/mocha-chai.yml) writes for Chai, pointing at the global
  `node_modules` directory. Without it a project's `eslint.config.js` cannot load: the config
  shape ESLint's docs and `eslint --init` generate begins with `require('eslint/config')`, and
  Node resolves that relative to the config file rather than to the eslint binary, so it dies
  with `Cannot find module 'eslint/config'` from any directory outside the global tree. Both
  playbooks write an identical line, so whichever runs second is a no-op.
- **An ESM config cannot use the global install.** Node ignores `NODE_PATH` for `import`, so an
  `eslint.config.mjs` — or an `eslint.config.js` in a package marked `"type": "module"` — that
  imports from `eslint` fails with `ERR_MODULE_NOT_FOUND` even with `NODE_PATH` set. Such a
  project needs either a config that imports nothing (a plain exported array of rule objects
  works in both module systems) or a project-local `eslint`. The same limit applies to plugins:
  `typescript-eslint` and the `eslint-plugin-*` family are per-project dependencies, are not
  installed here, and a global ESLint resolves only the ones that are themselves installed
  globally, from a CommonJS config.

The unprivileged verification step lints a file with an unused binding against a config that
uses the `require('eslint/config')` shape, and asserts the `no-unused-vars` report — so it
proves in one command that uid 65534 can execute the binary, read the package tree, discover
`eslint.config.js` from the working directory, and resolve ESLint's own module out of the
global prefix. ESLint exits non-zero when it reports an error, which is the expected outcome
of that check, not a failure of the run.

Two smaller differences from the older npm playbooks: the install passes `--engine-strict`, so
an unsupported Node.js (ESLint 10 declares `^20.19.0 || ^22.13.0 || >=24`) fails the run
instead of drawing npm's default `EBADENGINE` *warning* and installing anyway; and it runs with
a scratch `HOME` under `/var/tmp`, removed afterwards, so npm's cache and debug logs do not
land in `/root/.npm`.

Version overrides:

```bash
ansible-playbook core/eslint.yml -e host=ws01 -e eslint_version=10.9.1
```

## markdownlint.yml

Installs [markdownlint-cli](https://github.com/igorshubovych/markdownlint-cli) via
`npm install -g`, root-owned. Node.js itself is a prerequisite provisioned elsewhere; this
playbook only checks for it, and fails loudly with setup instructions if it is missing.

- **Pinned version, not `@latest`.**
- **Version-aware idempotency.** The installed version is compared against the pin, not
  just the binary's presence.
- **npm's global prefix is read at run time, not assumed.** `npm config get prefix`
  differs depending on how Node.js was installed: a NodeSource-installed Node.js defaults
  it to `/usr` (`/usr/lib/node_modules`, `/usr/bin/markdownlint`), while Ubuntu's own
  `nodejs` apt package defaults it to `/usr/local` (`/usr/local/lib/node_modules`,
  `/usr/local/bin/markdownlint`). Both are root-owned system paths, so either is fine —
  but hardcoding one breaks the other.
- **Node-in-PATH guard**, accepting either system prefix: fails if `which node` resolves
  outside `/usr/bin/node` or `/usr/local/bin/node`, which would indicate a per-user
  version manager (nvm, fnm, ...) shadowing the system Node.js for the connecting account.

The unprivileged verification step lints a Markdown file with a deliberate heading-space
issue and checks the output for that specific rule (`MD018`). markdownlint writes its
findings to **stderr**, not stdout, and exits non-zero when it finds issues — both are the
expected, successful outcome of this check, not a failure of the playbook run.

Version overrides:

```bash
ansible-playbook core/markdownlint.yml -e host=ws01 -e markdownlint_cli_version=0.49.1
```

## shellcheck.yml

Installs [ShellCheck](https://github.com/koalaman/shellcheck) from the Ubuntu apt
package, `/usr/bin/shellcheck`, root-owned. There is no per-user state and nothing to
add to a shell profile.

Installed from the Ubuntu apt package rather than Homebrew — apt is current enough to use
directly, and unlike Homebrew, it's usable by every account on the host:

- **Pinned by exact dpkg version**, not just the semantic version. `shellcheck_version`
  is compared against `dpkg-query -W -f='${Version}' shellcheck`, which includes the
  Debian revision suffix (e.g. `-2`), so the idempotency check is exact.
- **The pin is Ubuntu-release-specific.** apt's candidate version for `shellcheck`
  differs between Ubuntu releases (`0.11.0-2` on 26.04 "resolute", `0.9.0-1` on 24.04
  "noble"). Overriding `shellcheck_version` only helps if that exact dpkg version is
  available from the target's configured apt sources.

The unprivileged verification step writes a throwaway script with a known issue
(unquoted `$1`) to `/tmp`, lints it as `nobody`, and checks for `SC2086` in the output.

Version overrides:

```bash
ansible-playbook core/shellcheck.yml -e host=ws01 -e shellcheck_version=0.9.0-1
```

## yq.yml

Installs [mikefarah/yq](https://github.com/mikefarah/yq) as a single static binary at
`/usr/local/bin/yq`, root-owned, mode `0755`. There is no per-user state and nothing to add
to a shell profile.

This is a release binary rather than a plain apt install, deliberately: per the
apt gotcha in [INSTALL-MECHANISMS.md](../INSTALL-MECHANISMS.md), Ubuntu's apt `yq` is
`kislyuk/yq`, a Python
wrapper around `jq` with entirely different syntax from mikefarah's Go `yq` that this
playbook installs. Silently swapping one for the other under the same command name would
break any script written against the Go one.

- **Checksum verification, but not from a checksums file.** yq's own published checksums file
  uses a bespoke multi-algorithm rhash table (`checksums_hashes_order` /
  `extract-checksum.sh` in the release), not the plain `hash  filename` format
  `gomplate.yml`/`hadolint.yml` (in [`misc/`](../misc/README.md)) parse. Instead, the
  playbook reads the SHA-256 `digest` GitHub computes and serves per release asset via its
  own releases API — simpler to consume and just as authoritative.
- **Architecture from facts.** `ansible_architecture` is mapped to the release asset suffix
  (`x86_64` → `amd64`, `aarch64` → `arm64`). Unmapped architectures fail with a clear message.
- **Version-aware idempotency.** The installed version (parsed from `yq --version` output) is
  compared against the pinned version before re-downloading.

The unprivileged verification step pipes `a: 1` into `yq e '.a' -` (reading from stdin) as
`nobody` and checks the output, proving yq's zero-configuration path.

Version overrides:

```bash
ansible-playbook core/yq.yml -e host=ws01 -e yq_version=4.53.3
```

## ruby.yml

Installs Ruby and RubyGems from the Ubuntu archive.

| Path | Contents |
| --- | --- |
| `/usr/bin/ruby`, `/usr/bin/gem` | via `update-alternatives` |
| `/var/lib/gems/3.3.0/` | where a root `gem install` puts gems |
| `/usr/local/bin/<binstub>` | and where it puts their executables |

**One pinned Ruby for every gem-based playbook to share**, the shape
[`openjdk.yml`](#openjdkyml), [`nodejs.yml`](#nodejsyml), [`dotnet.yml`](#dotnetyml) and
[`pwsh.yml`](#pwshyml) already use for their runtimes:
[`misc/asciidoctor.yml`](../misc/README.md#asciidoctoryml) checks for `ruby` and fails with the
instruction to run this playbook first, rather than carrying a second copy of the pin.

**apt, not rbenv.** The single-user role this replaces cloned rbenv and ruby-build into
`~/.rbenv`, built Ruby from source there, and appended `eval "$(rbenv init - bash)"` to that one
account's `~/.bashrc` — all of it per-identity, which [POLICY.md](../POLICY.md) points 1 and 3
rule out. Ubuntu 26.04 carries Ruby 3.3.8, current enough for everything this repository
installs on top of it, so apt is the first mechanism that applies.

**Both the metapackage and the interpreter are pinned**, because they are different versions of
different things. `ruby` and `ruby-dev` are metapackages whose version (`1:3.3build1`) names the
*series*: they stay put across a 3.3.8 → 3.3.9 update, so pinning them alone pins almost
nothing. `ruby3.3` carries the real version, and the verification asserts what `ruby -v` and
`gem --version` report rather than trusting dpkg's answer for either.

`ruby-dev` is included so a gem with a C extension (nokogiri, ffi) can be built at all — it
brings `ruby3.3-dev`, the headers. It does **not** bring a compiler: `gcc` and `make` come from
`build-essential`, which this playbook deliberately does not install. Nothing installed by this
repository needs one, since every gem
[`misc/asciidoctor.yml`](../misc/README.md#asciidoctoryml) installs is pure Ruby.

**No shell configuration is written, and none is needed.** RubyGems' own default for a root
install is `Gem.bindir` = `/usr/local/bin`, already ahead of `/usr/bin` on the default `PATH`, so
a gem's executables are on every account's `PATH` with no `/etc/profile.d` drop-in. An
unprivileged `gem install` (no `sudo`) still goes to that account's own
`~/.local/share/gem`, which is where per-user state belongs; this playbook writes nothing there.

The unprivileged verification runs `ruby -v`, round-trips a JSON document through the stdlib —
real work, not just a version string — and checks `gem --version` and `gem list`, all as
`nobody` with a scratch `HOME`.

This is the prerequisite for [`misc/asciidoctor.yml`](../misc/README.md#asciidoctoryml).

Version overrides:

```bash
ansible-playbook core/ruby.yml -e host=ws01 -e ruby_interpreter_version=3.3.9
```

## devcontainers.yml

Installs the Dev Containers CLI into npm's global (root-owned) prefix.

| Path | Contents |
| --- | --- |
| `<npm prefix>/bin/devcontainer` | the CLI entry point |
| `<npm prefix>/lib/node_modules/@devcontainers/cli` | the package |

The npm prefix is **read at run time**, not assumed: an apt-installed `nodejs` defaults it
to `/usr/local`, a NodeSource one to `/usr`. Both are root-owned, but hardcoding either
breaks the other host.

**Prerequisite:** system-wide Node.js. The playbook fails early if `node` resolves outside
`/usr/bin` or `/usr/local/bin`, which is what a per-user version manager (nvm, fnm, mise)
looks like — a global npm install under that shim would land in one account's home
directory instead of a system path.

The version is pinned (`devcontainers_cli_version`), and the install guard compares the
installed version against the pin, so bumping the pin actually replaces the package.

**It needs Docker earlier than you would expect.** Even `devcontainer read-configuration`,
which looks like pure config parsing, shells out to `docker ps` to look for an existing
container — so it exits 1 for any account outside the `docker` group, printing only its
banner, with the real cause visible solely under `--log-level trace`. On a host where
nobody has been granted `docker_users`, every subcommand except `--help` and `--version`
will fail. That is also why this playbook's unprivileged check is `--help`: there is no
subcommand that does real devcontainer work without a container runtime.

Version override:

```bash
ansible-playbook core/devcontainers.yml -e host=ws01 -e devcontainers_cli_version=0.88.0
```

## podman.yml

Installs `podman` and `podman-compose` from the Ubuntu archive.

| Path | Contents |
| --- | --- |
| `/usr/bin/podman` | from the `podman` package |
| `/usr/bin/podman-compose` | from the `podman-compose` package |
| `/etc/containers/` | shared configuration, shipped by the packages |

Podman is rootless: each account gets its own containers, images and storage under
`~/.local/share/containers`, nothing is shared between accounts, and **no group membership
is needed** — unlike Docker, which needs a root-equivalent grant to be usable at all.

Both packages are pinned with `allow_downgrade` and verified after install.
`loginctl enable-linger` is run only for accounts named in `podman_linger_users` (default
empty), and only when not already enabled.

**Rootless podman needs a subuid/subgid range, and it is per-account.** `useradd`
allocates one by default; `adduser --system`, cloud-init and LDAP often do not, and
rootless podman then fails for that account in a way that looks like a podman bug (the
error names `newuidmap`, not the missing range). The playbook asserts a range exists in
both `/etc/subuid` and `/etc/subgid` for every account named in `podman_linger_users`, and
fails with the remedy before granting anything:

```bash
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 <account>
```

Verification is `podman --version` as uid 65534 — modest by design, because nearly every
other subcommand reaches for a container store, and `nobody` has no subuid range of its
own. It runs with a scratch `HOME`, which is not optional: podman stats `$HOME/.config`
before doing anything at all, so without one even `--version` fails as
`stat /root/.config: permission denied`.

Version overrides — these pins are **release-specific**, so check `apt-cache policy` on
the target before bumping:

```bash
ansible-playbook core/podman.yml -e host=ws01 \
  -e podman_version=5.7.0+ds2-3build1 -e podman_compose_version=1.5.0-2
```

## docker.yml

Installs Docker Engine from `download.docker.com`.

| Path | Contents |
| --- | --- |
| `/usr/bin/docker` | the CLI |
| `/usr/libexec/docker/cli-plugins/` | `buildx` and `compose`, shared by every account |
| `/etc/apt/sources.list.d/docker.sources` | repository definition, signing key inline |

> **A fresh run grants nobody access to the daemon.** The engine is installed and running;
> no account can talk to it until you name accounts in `docker_users` — see
> [Grants](#grants).

```bash
ansible-playbook core/docker.yml -e host=ws01 -e docker_users=alice,bob
```

Members must log out and back in for the new group to take effect.

**If you do not want a root-equivalent grant,** two real alternatives: rootless Docker, via
the `docker-ce-rootless-extras` package this playbook installs but does not configure (each
account runs `dockerd-rootless-setuptool.sh install` for itself), or
[podman](#podmanyml), which needs no grant at all.

All five packages are pinned. The repository uses `deb822_repository` with Docker's
signing key pinned inline (it carries no expiry). This playbook also removes any
auto-named `download_docker_com_linux_ubuntu.list` apt source left behind by an older
setup, to avoid apt reading the repository twice.

Verification goes past installing the binary: as uid 65534 it runs the CLI and the
`buildx` plugin (proving `/usr/libexec/docker/cli-plugins` is reachable by an ordinary
account, which is the actual multi-user question for Docker), and then asserts that the
same account is **refused** at the socket. A run where an ungranted account can drive the
daemon fails.

The pins embed the Ubuntu codename (`5:29.7.2-1~ubuntu.26.04~resolute`), so they are not
portable across releases:

```bash
ansible-playbook core/docker.yml -e host=ws01 \
  -e docker_version=5:29.7.2-1~ubuntu.26.04~resolute \
  -e containerd_version=2.3.3-1~ubuntu.26.04~resolute
```

## kubectl.yml

Installs kubectl from `pkgs.k8s.io`, **and pins apt so it stays installed.**

| Path | Contents |
| --- | --- |
| `/usr/bin/kubectl` | the client |
| `/etc/apt/sources.list.d/kubernetes.sources` | repository definition (`v1.36` stream) |
| `/etc/apt/preferences.d/kubectl` | the pin that resolves the collision below |
| `/etc/bash_completion.d/kubectl` | shared completion, generated from the binary |
| `/etc/profile.d/kubectl.sh` | the `k` alias, with completion wired to it |
| `/usr/bin/kubectx`, `/usr/bin/kubens` | from the Ubuntu archive's `kubectx` package |

**Two repositories publish a package called `kubectl`, and without the pin the wrong one
always wins.** `packages.cloud.google.com` — added by `cloud-cli/gcloud-cli.yml`, which does
not install kubectl and has no idea it is involved — publishes a Cloud SDK dispatcher build
with an **epoch** (`1:580.0.0-0`). An epoch outranks every version upstream will ever publish.
Both ship `/usr/bin/kubectl` and neither declares `Conflicts`, so exactly one can be
installed, and on any host that has run `gcloud-cli.yml` first, it would be Google's. The
giveaway is a `-dispatcher` suffix in `kubectl version --client`.

Installing the right version once is not enough: the next unrelated `apt upgrade` takes it
straight back. `/etc/apt/preferences.d/kubectl` pins the `pkgs.k8s.io` origin at priority
**1001** — above 1000, which is what permits apt to move *backwards* across the epoch. The
playbook asserts on every run that `apt-cache policy` reports the pinned version as the
candidate, so the protection is tested rather than assumed.

**Deleting that preferences file re-opens the collision.**

The stream is part of the pin: moving to `v1.37` means editing `kubectl_stream` as well as
`kubectl_version`. The `v1.36` stream currently carries exactly one version and there is no
`v1.37` stream yet.

The repository's signing key **expires 2026-12-29**, so it is fetched each run and only its
fingerprint is pinned. When it rotates, the run fails with instructions rather than trusting
a new key silently — that is intended; verify the new fingerprint and update it deliberately.

```bash
ansible-playbook core/kubectl.yml -e host=ws01 -e kubectl_version=1.36.3-1.1
```

### kubectx and kubens

The Ubuntu archive carries a `kubectx` package — one source, no vendor collision, no
repository or key to add — that installs `/usr/bin/kubectx` and `/usr/bin/kubens`
directly, with completions under the paths `bash-completion` already reads. Pinned like
everything else here (`kubectx_version`), even though nothing forces the version the way
`kubectl_version` does.

There is no `krew` here, and these do not register as `kubectl` plugins: the package names
the binaries `kubectx` and `kubens`, not `kubectl-ctx` / `kubectl-ns`, so kubectl's plugin
discovery never sees them.

```bash
ansible-playbook core/kubectl.yml -e host=ws01 -e kubectx_version=0.9.5-2build1
```

## helm.yml

Installs Helm from `packages.buildkite.com/helm-linux/helm-debian`.

| Path | Contents |
| --- | --- |
| `/usr/bin/helm` | the binary |
| `/etc/apt/sources.list.d/helm.sources` | repository definition, signing key inline |
| `/etc/bash_completion.d/helm` | shared completion, generated from the binary |

> **This installs Helm 4.** Pinning a major version was a deliberate choice — nothing in
> this repo templates a chart, so there was no compatibility constraint pulling toward
> 3.x. If you need Helm 3, it is one flag; the 3.x line is published from the same
> repository under the same package name:

```bash
ansible-playbook core/helm.yml -e host=ws01 -e helm_version=3.21.3-1
```

The playbook prints a **MAJOR VERSION CHANGE** warning when a run would move between 3.x and
4.x in either direction.

The version is pinned and verified. The repository uses `deb822_repository` with the
packagecloud key pinned inline (it carries no expiry). Verification is real offline work:
as uid 65534 it lints and renders a small chart the playbook writes.

## kind.yml

Installs the `kind` release binary.

| Path | Contents |
| --- | --- |
| `/usr/local/bin/kind` | the binary, root-owned, mode 0755 |
| `/etc/bash_completion.d/kind` | shared completion, generated from the binary |

The install guard compares the reported version against the pin, so bumping the pin
actually replaces the binary. The download is verified against the published
`kind-linux-<arch>.sha256sum`, with architecture from facts.

```bash
ansible-playbook core/kind.yml -e host=ws01 -e kind_version=0.32.0
```

## minikube.yml

Installs the `minikube` release binary.

| Path | Contents |
| --- | --- |
| `/usr/local/bin/minikube` | the binary, root-owned, mode 0755 |
| `/etc/bash_completion.d/minikube` | shared completion, generated from the binary |

The install guard compares the reported version against the pin, the download is verified
against a checksum, and architecture comes from facts.

**No driver default is set system-wide, and that is a decision.** `MINIKUBE_DRIVER` was tested
and minikube really does read it — but minikube already auto-selects the docker driver when
Docker is present, so a shared `MINIKUBE_DRIVER=docker` would be a no-op for every account that
can use Docker, and actively wrong for every account that cannot: it would push them at the one
driver they have no access to, instead of letting minikube offer podman. Each account sets its
own, once:

```bash
minikube config set driver docker    # or podman
```

**minikube is expensive per account.** Every account that runs `minikube start` gets its own
`~/.minikube` with its own certificates and its own copy of the kicbase image — gigabytes each,
with no shared mode. On a workstation with several agent accounts, budget for it. `minikube
delete` reclaims it.

```bash
ansible-playbook core/minikube.yml -e host=ws01 -e minikube_version=1.38.1
```

## kube-tools.yml

Installs three Kubernetes companion tools in one playbook: [kubelogin](https://azure.github.io/kubelogin/),
[k9s](https://k9scli.io/) and [kdash](https://kdash.cli.rs/).

| Path | Contents |
| --- | --- |
| `/usr/local/bin/kubelogin` | Azure AKS credential plugin for `kubectl` |
| `/usr/local/bin/k9s` | cluster TUI |
| `/usr/local/bin/kdash` | dashboard TUI |
| `/etc/bash_completion.d/kubelogin`, `/etc/bash_completion.d/k9s` | shared completions, generated from the binaries |

All three are single static binaries from GitHub releases, pinned and verified against the
SHA-256 GitHub publishes per asset — the [`yq.yml`](#yqyml) source. None of
the three ships a checksums file in the same shape (kubelogin one `.sha256` per asset, k9s a
combined `checksums.sha256`, kdash one `.sha256` per asset), and the per-asset digest is
identical in form for all of them, which is what lets a single loop cover the set.

**The three projects disagree about architecture names**, so each asset and archive member is
spelled out per architecture rather than assembled from one arch string: kubelogin says
`linux-amd64`, k9s says `Linux_amd64`, and kdash says plain `linux` for amd64 but
`aarch64-gnu` for arm64. kdash also ships no `completion` subcommand at all (`error: unexpected
argument 'completion' found`), so only the other two get a completion file.

### Identity stays per account, and k9s writes state on sight

All three read the kubeconfig, and kubelogin writes an Azure token cache to
`~/.kube/cache/kubelogin` — a credential, per account. Nothing here creates or touches any of
it. k9s additionally keeps `~/.config/k9s` (settings, skins, aliases) and `~/.local/state/k9s`
(logs), and **`k9s version` alone already creates the latter** — confirmed live. That is why
this playbook keeps two scratch `HOME`s and removes both: one for the root-side version checks
and completion generation, so nothing lands in `/root` (B2), and one owned by uid 65534 for the
unprivileged checks.

As with [`kind.yml`](#kindyml), these are only as multi-user as the cluster access behind them:
the playbook installs binaries every account can execute, and an account with no kubeconfig can
do nothing with them.

### The unprivileged checks do real work offline

- **kubelogin** converts a generated kubeconfig that uses the deprecated Azure `auth-provider`
  block, and the assertion is on the rewritten file: the `exec` credential plugin and its
  `get-token` argument must be present **and** `auth-provider` must be gone. No token is
  requested and nothing is contacted — the placeholders are all-zero GUIDs against
  `example.invalid`.
- **k9s** is asked for `k9s info`, which reports the paths it will use for config, logs and
  benchmarks. It needs no cluster, so it is the one thing a TUI can be asked to do offline that
  is more than printing a version — and it proves the scratch `HOME` is where its state goes.
- **kdash** is a TUI with no offline subcommand beyond `--version`, so that is what it gets.

`junit2html`, which this toolset is often paired with for kube-score reports, is **not** here:
it has its own playbook at [`misc/junit2html.yml`](../misc/README.md#junit2htmlyml), which
installs it root-owned via pipx rather than into one account's `~/.local`.

Version overrides — one variable per tool:

```bash
ansible-playbook core/kube-tools.yml -e host=ws01 -e k9s_version=0.51.0
```

## pyenv.yml

Installs [pyenv](https://github.com/pyenv/pyenv) from its upstream git tag, as one shared
read-only checkout.

| Path | Contents |
| --- | --- |
| `/usr/local/pyenv/` | the checkout at the pinned tag, root-owned |
| `/usr/local/bin/pyenv` | symlink, so `pyenv` is on every account's `PATH` |
| `/etc/profile.d/pyenv.sh` | live `eval "$(pyenv init - bash)"`, interactive bash only |

**The code is shared; the interpreters are not.** `PYENV_ROOT` is left at pyenv's own default of
`~/.pyenv`, so each account builds the versions it wants into its own home and this playbook
installs an interpreter for nobody. That split is pyenv's supported layout rather than something
done to it: `libexec/pyenv` resolves its install prefix from the real path of the script it was
invoked through, and puts both `<prefix>/plugins/*/bin` and `$PYENV_ROOT/plugins/*/bin` on
`PATH`, so the bundled `python-build` plugin — that is, `pyenv install` — is found even though
`PYENV_ROOT` is somewhere else entirely.

A single shared `PYENV_ROOT` was the alternative, and is rejected: `$PYENV_ROOT/versions` is what
`pyenv install` writes into, so a shared one is either root-owned, and every account needs root
to add an interpreter, or group-writable, and any account can replace an interpreter the others
run. Per-account roots cost a compile each, and that is the honest state of the tool rather than
a defect: `pyenv install` builds CPython from source, per account, with no shared mode to migrate
into — a 3.14.7 built this way measures 363 MB under one account's `~/.pyenv`, and every account
that wants it pays that again. [`mise.yml`](#miseyml) above also gets an account a per-account runtime, from a prebuilt
download rather than a compile; pyenv is here for projects that pin a `.python-version` and
expect `pyenv` itself to resolve it.

**The build toolchain is part of this install, not a prerequisite left to the user.**
`python-build` compiles CPython from source, and which headers it finds is what decides whether
the interpreter it produces has `ssl`, `sqlite3`, `lzma`, `bz2`, `ctypes`, `readline`, `curses`
and `zlib` in its standard library at all — a missing header is not an error there, the module is
simply dropped, and an account that cannot install packages cannot fix it afterwards. So the
playbook installs upstream's documented dependency list, naming `libncurses-dev` where upstream
says `libncursesw5-dev`: that name is a pure virtual package provided by `libncurses-dev`.

### Verification

Four checks, each as `nobody` with a scratch `HOME`:

- **`pyenv --version`** — the pin itself.
- **`pyenv install --list`** — the multi-user claim in one command. It reads the definition files
  inside the shared prefix while `PYENV_ROOT` points into the scratch home, so it passes only if
  an account that installed none of this can reach the bundled plugin. Offline: the definitions
  are files in the checkout.
- **A non-login interactive shell**, which must come out with the `pyenv` function defined *and*
  its own `~/.pyenv/shims` on `PATH` — the `/etc/bash.bashrc` → `/etc/profile.d/pyenv.sh` chain
  end to end, not merely a file that was written.
- **A syntax-only compile against those eight headers**, because the toolchain half of this
  install is the half that fails silently.

**`pyenv --version` reports the pin the same way three different ways round,** which is why the
guard and the unprivileged check can compare one exact string: from the tagged checkout it comes
from `git describe`, from a checkout whose git the caller cannot read — uid 65534 hits git's
dubious-ownership refusal on this root-owned tree — and from a tree with no `.git` at all it
comes from a version baked into `libexec/pyenv---version`. All three print `pyenv 2.8.5`.

**The hook is guarded twice, and both guards were reproduced before being written.**
`pyenv init - bash` emits bash rather than POSIX sh (it uses `source`), so a dash login shell
reading an unguarded `/etc/profile.d/pyenv.sh` fails with `source: not found` on every login. And
generating that output is not read-only — `pyenv init -` creates `$PYENV_ROOT/{shims,versions}`
on the way — so an account whose home is missing or unwritable, including uid 65534, otherwise
gets two `mkdir: cannot create directory` errors every time a shell starts.

**There is no `/etc/bash_completion.d/pyenv`,** which reads [POLICY.md's B3](../POLICY.md)
loosely: `pyenv init -` sources pyenv's own `completions/pyenv.bash` out of the install prefix
itself, so a second copy under `/etc/bash_completion.d` would be loaded twice. B3 asks for the
output of a `completion bash` subcommand to be snapshotted; pyenv has no such subcommand, only a
completion file it ships and wires up itself.

**One finding only visible by running the playbook.** The interactive-shell check runs through
ansible's `shell` module, not `command`: with `command`, that same argv reports the `PATH` the
shell started with, as though `/etc/profile.d` had never touched it, while an ordinary exported
variable set beside it in the same file does survive — so the check fails against a host where
the hook is working. The same command run by hand on the target is correct either way.

Version overrides:

```bash
ansible-playbook core/pyenv.yml -e host=ws01 -e pyenv_version=2.8.4
```

Then, per account and per project, for themselves:

```bash
pyenv install --list          # what this pyenv knows how to build
pyenv install 3.14.7          # compiles into ~/.pyenv/versions
pyenv global 3.14.7           # or `pyenv local` inside one project
```

## What is *not* here

There is no chrony playbook: Ubuntu 26.04 ships chrony pre-configured out of the box. Go and
Rust toolchains are also out of scope — this repo provisions a workstation that runs AI
coding agents rather than compiling with either. `mise.yml` above is the general-purpose way
to get a per-account language runtime.
