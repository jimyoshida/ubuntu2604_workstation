# Playbook Policy

The rules every playbook under [`playbooks/`](.) must satisfy, and the reasons behind
them. This is the document to check a new or changed playbook against.

The governing rule:

> Shared tooling goes to root-owned system paths. Shell configuration goes to `/etc`
> drop-ins. Only things that are genuinely per-identity stay in `$HOME`, and those take an
> explicit user list rather than reading `$USER`.

For *which* mechanism installs a given tool — apt, vendor repository, upstream release
artifact, pipx, npm — see [INSTALL-MECHANISMS.md](INSTALL-MECHANISMS.md). This document covers
everything else.

## How to read this document

Rules are numbered in three series, and the numbering is load-bearing: playbook comments and
directory READMEs cite these identifiers directly (`per A5`, `which B2 forbids`, `policy point
6`), so the identifiers are stable and are not renumbered.

| Series | Covers |
| --- | --- |
| **1–10** | The core rules. Every playbook satisfies all ten. |
| **A1–A5** | Identity, secrets and vendor apt repositories. Several tighten a numbered point. |
| **B1–B5** | Privilege grants, per-user runtime state and shared shell configuration. |
| **C1–C6** | Guards, dry runs and verification mechanics. |

Where a rule tightens an earlier one, the earlier one is named.

## Operating environment

These playbooks push to a remote host: `hosts: "{{ host }}"`, `remote_user: ubuntu`,
`become: true`. They are run from `playbooks/`, because Ansible reads `ansible.cfg` from the
working directory.

```bash
cd playbooks
ansible -m ping <host>                                    # confirm reachability first
ansible-playbook misc/gomplate.yml -e host=workstations    # a group
ansible-playbook misc/gomplate.yml -e host=ws01            # one host
```

| | |
| --- | --- |
| Control node | Ubuntu 24.04 (ansible-core **2.16**) or Ubuntu 26.04 (ansible-core 2.20) |
| Targets | Ubuntu 26.04, Python 3.14 |
| Inventory | `playbooks/inventory.ini`, gitignored; copy `inventory.ini.example` |

The 2.16 floor is a real constraint on what modules may be used: `deb822_repository` (2.15)
clears it, and nothing in these playbooks postdates it. `localhost` is a valid target via
`ansible_connection=local`, which makes `remote_user: ubuntu` a no-op.

Each target needs an account reachable over SSH with key authentication (override the name with
`-e ansible_user=<name>` rather than editing playbooks), passwordless sudo for that account, and
Python 3.

## Core policy

1. **Root-owned system paths only.** Binaries to `/usr/local/bin`, or apt. Shared libraries to
   `/usr/lib/<tool>` or `/usr/local/lib/<tool>`. Build and source trees to `/usr/local/src`.
2. **No Homebrew.** Every tool in scope has a distro package, a vendor apt repository, an
   upstream release binary, or an upstream `install.sh`.
3. **No writes to any `$HOME`.** Environment variables go to `/etc/environment` (applied to
   login and SSH sessions via PAM). Interactive-only settings — key bindings, completions,
   aliases — go to `/etc/profile.d/<tool>.sh`, which requires `/etc/bash.bashrc` to source it so
   that non-login interactive shells pick it up. `/etc/skel/` is not a substitute: it only
   affects accounts created afterwards. Tightened by **A1** (secrets) and **B2** (runtime state).
4. **Explicit world-readable modes.** Set `mode: 'u=rwX,go=rX'` on install trees rather than
   relying on the umask of whoever ran the playbook. Set `owner: root`, `group: root` explicitly
   on installed files.
5. **Pinned versions in the play's `vars`,** overridable with `-e`. Tightened by **A2**: the same
   applies to endpoint configuration, not only to versions.
6. **Version-aware idempotency.** Guard installs on the *installed version*, never on
   `stat.exists`. A file-existence guard makes version bumps silently no-op. Tightened by **C2**:
   the guard must also be able to see a tool that is absent.
7. **Integrity verification** for anything downloaded outside apt. Prefer resolving the hash from
   the release's published checksum file at run time over hardcoding it, so that overriding the
   version stays a one-flag change.
8. **Architecture from facts.** Map `ansible_facts['architecture']` onto the upstream asset name
   and fail loudly on an unmapped value. Never assume amd64. Use the `ansible_facts[...]`
   spelling, not bare `ansible_architecture`, which ansible-core 2.20 deprecates with a warning
   on every run that reads it.
9. **Unprivileged verification.** End with a task that exercises the tool as an arbitrary uid via
   `setpriv --reuid=65534 --regid=65534 --clear-groups`, **not** as the connecting user. This is
   the regression guard: a single-user regression must fail the run rather than pass because
   `ubuntu` happens to have the right environment. Where the tool has configuration discovery
   (search paths, library paths), leave the relevant variable unset in that task so the
   zero-configuration path is what gets proven. Refined by **A4**, **B4** and **C3**.
10. **Self-contained playbooks, not roles.** One playbook per tool, no dependency on `roles/`, so
    a workstation can be built up tool by tool. Shared prerequisites are the one exception, and
    they are *declared and checked*, never installed inline — see
    [INSTALL-MECHANISMS.md](INSTALL-MECHANISMS.md#prerequisite-not-installed-here).

## A1–A5 — Identity, secrets and vendor repositories

### A1. Secrets never leave `$HOME`. *(tightens point 3)*

Point 3 treats everything bound for `/etc` as one category. It is three:

| Kind | Example | Destination |
| --- | --- | --- |
| Shared, non-secret endpoint config | `JENKINS_URL`, `GRAFANA_SERVER`, `VAULT_ADDR` | `/etc/environment` or `/etc/profile.d/<tool>.sh`, set from an explicit play var |
| Per-identity, non-secret | `AWS_PROFILE`, `CLOUDSDK_CORE_PROJECT` | per-user `$HOME`; not set by these playbooks |
| Secret | every `*_TOKEN`, `*_PAT`, `*_API_TOKEN` | per-user `$HOME` only, mode `0600`; **never** `/etc`, never a play var, never committed |

`/etc/environment` is world-readable, and a shared workstation is exactly where that matters.

A playbook that would need a secret to complete is mis-designed: installation must not require
credentials. If verification appears to need one, the verification is testing the wrong thing —
see A4.

**A variable name taken from a source playbook's `lookup('env', ...)` is not evidence that the
tool reads it.** Confirm each name against the binary before writing it anywhere, using a
negative control: set the name, then set a plausible near-miss instead, and compare the
failures. Names that fail this test are common — `AZURE_DEVOPS_ORG`, `JIRA_URL`, `JIRA_LOGIN`
and `GOOGLE_CLOUD_PROJECT` are all read by nothing in the CLI they appear to configure. The
`GOOGLE_CLOUD_PROJECT` case is the instructive one: Google's *client libraries* really do read
it, so it is a correct variable for the wrong consumer, and it looks right in every direction
until you ask the CLI what project it is on.

A shared default under a name the tool ignores is invisible — the variable is set,
`/etc/environment` looks correct, and every user still gets an error. Where a playbook does set
one, it should assert it: run the tool as an unprivileged uid with the variable set and assert
that the "not configured" error is *not* what comes back.

Being a name the tool reads is necessary, not sufficient. There must also be a shared endpoint
worth naming, and that is rarer than the list of candidate names suggests: across all of
`playbooks/` there is one. [cloud-cli/jenkins-cli.yml](cloud-cli/jenkins-cli.yml) publishes a
default `JENKINS_URL` — in `/etc/profile.d/jenkins-cli.sh` rather than `/etc/environment`, so the
assignment can be conditional (`: "${JENKINS_URL:=...}"`) and an account that exports its own
keeps it — and it asserts the result the way the paragraph above requires, by resolving
`$JENKINS_URL` in an `env -i` login shell as an unprivileged uid.
[cloud-cli/sonar-scanner.yml](cloud-cli/sonar-scanner.yml) has an endpoint of exactly the same
kind and deliberately does not express it as a variable at all: `sonar.host.url` goes into the
distribution's own `conf/sonar-scanner.properties`. Both values come from an A2 play var, never
from `lookup('env', ...)`.

Every other variable these playbooks write is not endpoint configuration, and the distinction is
worth keeping straight: those are *search paths* into a root-owned shared install tree, so that a
tool installed once is discoverable by an account that did no setup.

| Variable | Playbook | Points at |
| --- | --- | --- |
| `NODE_PATH` | [core/eslint.yml](core/eslint.yml), [misc/jsmin.yml](misc/jsmin.yml), [misc/mocha-chai.yml](misc/mocha-chai.yml) | npm's global `node_modules`, so `require()` resolves a globally installed package |
| `BATS_LIB_PATH` | [misc/bats.yml](misc/bats.yml) | the shared BATS helper-library directory |
| `PLAYWRIGHT_BROWSERS_PATH` | [misc/playwright.yml](misc/playwright.yml) | the shared browser bundle under `/opt` |

These carry no site-specific value and no secret — the path is a fact about this host's install
layout — so `/etc/environment` being world-readable costs nothing for them, and an unconditional
`lineinfile` is the right shape. Two things still apply: the value is a path resolved at run time
(C5), not a hardcoded prefix, and where two playbooks publish the same name they write an
identical line, so whichever runs second is a no-op rather than a fight.

### A2. Endpoint config comes from a play var, not `lookup('env', ...)`. *(tightens point 5)*

Every `lookup('env', 'X')` is a `vars:` entry overridable by `-e`, exactly as tool versions are.
`-e jenkins_url=http://ci.example.com:8080` is reproducible and shows up in the run record;
`export JENKINS_URL=...` in one operator's shell does not — it silently makes one person's
environment everyone else's default.

**But the name operators set is not itself a `vars:` entry, because a play var outranks an
inventory group var.** Measured, not assumed: with `sonar_scanner_host_url` set in
`[workstations:vars]` and a play var of the same name, every task sees the play var — the
inventory value is read, ignored, and never reported. So a playbook whose endpoint should be
settable per site leaves that name undefined and resolves it once:

```yaml
vars:
  sonar_scanner_server_url: "{{ sonar_scanner_host_url | default('http://localhost:9000') }}"
```

`inventory.ini` then wins over the default and `-e` wins over both. The default is the tool's
own URL on this machine (`http://localhost:9000`, `http://localhost:8080`), never a particular
site's server: which server a host talks to is a per-site decision, and committing one makes
every copy of this repository point at it. The two instances are
[cloud-cli/jenkins-cli.yml](cloud-cli/jenkins-cli.yml) and
[cloud-cli/sonar-scanner.yml](cloud-cli/sonar-scanner.yml), and both names are listed with their
localhost defaults in [inventory.ini.example](inventory.ini.example).

### A3. Per-identity setup is out of the playbook, into the README

`az devops configure --defaults`, `jira init`, `influx config create`, `gcloud auth login`,
`gh auth login` are per-person, interactive and credential-bearing. Playbooks install the client
and stop. Keep the closing `debug` summary that tells each user the command to run for
themselves. Where a default genuinely should apply to everyone, express it as A1 shared config,
never as a write into one account's config file.

### A4. Verification proves reachability, not authentication. *(refines point 9)*

Most of these tools cannot do useful work unauthenticated. The point-9 guard therefore asserts:
*an arbitrary uid can execute the binary, load its shared libraries, extensions and JARs, and
reach its own zero-configuration code path.* Prefer, in this order:

1. **Offline real work** — `promtool check config`, `tofu fmt -check`, `helm template` on a file
   the playbook writes. Strongest guard; use it wherever the tool allows it.
2. **A subcommand that inspects the install itself** — `az extension show`, `aws configure list`,
   `docker buildx version`.
3. **`--version` alone.** Weakest; acceptable only where 1 and 2 do not exist.

Two mechanics:

- **Give the smoke test a writable `HOME`.** `az`, `gcloud`, `aws` and `influx` all write state
  on first invocation and fail outright when `HOME` is unset or unwritable. Run them as
  `setpriv ... env HOME=<scratch>` and remove the directory afterwards. See **C4** for where that
  directory goes and **C6** for when `HOME` is not enough.
- **Leave discovery variables unset.** Do not set `AZURE_EXTENSION_SYS_DIR`, `CLOUDSDK_CONFIG` or
  `AZURE_CONFIG_DIR` in the smoke test. The zero-configuration path is the thing under test.

### A5. Vendor apt repositories get one shared convention. *(tightens point 1)*

- **Use `ansible.builtin.deb822_repository`**, not `get_url` + `shell: gpg --dearmor` +
  `apt_repository`. The dearmor form reports `changed` on every run. Two mechanics the module
  documents: it needs **`python3-debian` on the target**, so add it to task 1's prerequisites,
  and it does **not** refresh the apt cache — follow it with an `apt` task doing `update_cache`
  conditioned on the repository task's `changed` state.
- **Pin the fingerprint always. Pin the key bytes too, only when the key has no expiry.** Where a
  vendor publishes a single armored key with no expiry, pin it by content, inline in `Signed-By`:
  the bytes are the trust anchor and a rotation fails apt's `Release` check loudly. Where the key
  expires, fetch it by URL each run (the module compares by checksum, so this stays idempotent)
  and assert the **fingerprint** of the non-expiring successor instead. Pinning an expiring key
  by content converts a routine rotation into a failed run.
- **Key expiry is part of the pinning decision, not a detail.** Public keyservers may serve a
  self-signature that has expired while the vendor's documented copy of the *same* key carries an
  extended one. Where a vendor publishes a key only as documentation prose, pin it in the
  playbook and assert the fingerprint after import.
- **List only keys that sign what apt reads.** A key whose user ID says it signs *providers*, or
  release artifacts, does not belong in `Signed-By`. Verify against the published `InRelease`
  rather than copying a key list forward.
- **Architecture from facts** (point 8), and **map the distro codename** where the vendor
  publishes none for the target release. Factor the map into an identical `vars:` block per
  playbook rather than a role — point 10 still holds.
- **Match the keyring path and `sources.list.d` filename** any existing repository-adding
  playbook uses, so the two do not fight. Where a playbook supersedes an earlier one, it deletes
  the predecessor's source file before writing its own: left side by side, apt reads the
  repository twice and warns.

## B1–B5 — Grants, per-user state and shared shell config

### B1. A privilege grant takes an explicit list and defaults to empty. *(tightens A2)*

`docker` group membership and `loginctl enable-linger` are neither installation nor
configuration: they are grants made to named accounts.

```yaml
docker_users: []          # accounts to add to the docker group; root-equivalent
podman_linger_users: []   # accounts whose user services survive logout
```

Empty is the default, and empty means *the playbook grants nothing*. This is the one place a
playbook deliberately does less than a single-user equivalent would: a fresh run produces a
working Docker daemon that the invoking account cannot talk to. That is the correct default on a
shared box — a root-equivalent grant should be typed out, once, per account, in the run record —
and the README must say so in those words, because the first person to hit it will read it as a
regression.

- **Never `lookup('env', 'USER')`, and never default the list to the connecting account.** The
  account Ansible connects as is an operator, not a beneficiary.
- **Grants are additive only.** `append: true`. A playbook that does not name an account must not
  remove that account's existing membership. Revoking is a deliberate act, not a side effect of
  running with a shorter list.
- **Say what the grant is worth.** The README states that `docker_users` is equivalent to
  passwordless root, and names the alternatives — rootless Docker via
  `docker-ce-rootless-extras`, or podman, which needs no grant at all.

### B2. Per-user runtime state is created by the user, not by the playbook. *(extends A3)*

A3 moves per-identity *setup commands* into the README. B2 extends that to per-account *runtime*
state: **no task may write, pre-create, or chown anything under any account's `$HOME`, including
the invoker's and including root's.** Not `~/.kube`, not `~/.minikube`, not `~/.docker`, not
`/root/.ansible`, not `/root/.dotnet`.

This is why so many playbooks read `dpkg-query` rather than running the tool they just installed:
a plain `--version` is enough to create state for several of them. `ansible`, `mise`, `dotnet`,
`pwsh` and `k9s` all write to `$HOME` on the most innocuous invocation available. Where a command
genuinely must run as root — because it writes into a root-owned system path — give it a scratch
`HOME` that does not outlive the play, and remove it afterwards.

`pipx` needs `PIPX_MAN_DIR` set for the same reason: left unset it creates
`/root/.local/share/man`, which also puts any man page a package ships somewhere no other account
can read.

Where a site-wide default genuinely is wanted, express it as A1 shared config — and only after
confirming the tool reads it, by A1's negative-control method.

### B3. Completions and aliases go to `/etc`, and are generated from the installed binary

| Content | Destination |
| --- | --- |
| `<tool> completion bash` output | `/etc/bash_completion.d/<tool>`, written once at install time |
| Aliases (`alias k=kubectl`), `PATH` additions, env-rewriting hooks (`eval "$(mise activate bash)"`) | `/etc/profile.d/<tool>.sh` |

Generate the completion file with the binary the playbook just installed and write the output. Do
not copy `source <(kubectl completion bash)` into a shared file: that re-runs the tool on every
interactive shell start, once per tool, and breaks noisily for any account whose `PATH` does not
yet have the binary. Writing the generated file also versions the completion with the install —
it changes when the pin changes, and shows up as `changed` when it does.

The distinction is between a *snapshot* and a *live hook*. Completion data is a snapshot and is
written out. A hook that rewrites `PATH` or the environment must run fresh in each shell, so it
goes to `/etc/profile.d/<tool>.sh` as a literal line, not as a snapshot of its output.

`bash-completion` is a task-1 prerequisite wherever a completion is installed. The
`/etc/profile.d` bootstrap is a hard dependency for the `profile.d` half only:

```yaml
- name: Ensure interactive non-login shells read /etc/profile.d
  ansible.builtin.blockinfile:
    path: /etc/bash.bashrc
    marker: "# {mark} ANSIBLE MANAGED BLOCK: profile.d for interactive shells"
    block: |
      for f in /etc/profile.d/*.sh; do [ -r "$f" ] && . "$f"; done
```

`/etc/profile.d/*.sh` is sourced by login shells only; an SSH session that starts a non-login
interactive bash reads `/etc/bash.bashrc` instead. Any playbook needing the hook lays it down
defensively — `blockinfile` is idempotent on its marker, so it is a no-op for every later one.

### B4. Verification proves an unprivileged account can run the client — not that a cluster exists. *(refines A4)*

A4 was written for CLIs blocked by a credential. Where the obstacle is a daemon or a cluster
instead, the guard asserts that an arbitrary uid can execute the binary, discover its plugins and
shared data, and reach its own offline code path. A4's preference order applies unchanged.

Two additions:

- **Assert the negative for a grant.** Where a playbook grants access to a socket, the smoke test
  proves the grant *is* a grant: as uid 65534, `docker info` must fail with a permission error on
  `/var/run/docker.sock`. A run where an ungranted account can drive the daemon is a finding, not
  a pass. Match the error text, not merely a non-zero exit.
- **Some prerequisites are per-account and cannot be smoke-tested.** Rootless podman needs a
  subuid/subgid range, and uid 65534 has none, so no smoke test can exercise it. Assert instead
  that every account in `podman_linger_users` has an entry in both `/etc/subuid` and
  `/etc/subgid`, and fail with the `usermod --add-subuids` remedy. `useradd` allocates ranges by
  default; `adduser --system`, cloud-init and LDAP often do not, and rootless podman then fails in
  a way that looks like a podman bug.

**State honestly how multi-user a tool actually is.** `kind` and `minikube` drive Docker, so they
are only as multi-user as the `docker` group is: a correct playbook installs a binary every
account can execute and that no account outside `docker_users` can use for anything. That is the
honest state of the tool, not a defect — but the README must say it, or the next person reads a
"multi-user" playbook as a promise it cannot keep. The same applies to per-account cost:
every account running `minikube start` gets its own `~/.minikube` with its own certificates and
kicbase image, gigabytes each, with no shared mode to migrate into.

### B5. Check for a colliding *package*, not just a colliding *repository*. *(tightens A5)*

A5 makes playbooks agree with each other about keyring paths and filenames. That is not
sufficient. `kubectl` is published by both `pkgs.k8s.io` and `packages.cloud.google.com` under
the same package name, with no `Conflicts`/`Replaces` between them, and Google's carries an
**epoch** that outranks every version upstream will ever publish. The two repository files do not
collide at all; the packages do.

Before adding any vendor repository, check what else already provides the package name:

```bash
cd playbooks
ansible -m shell -a 'apt-cache policy <package>; apt-cache madison <package>' <host>
```

Where the collision is real, an unpinned `apt: name=<pkg> state=present` is not merely unpinned —
it installs from a different vendor than the repository just configured, and reports success.
Pinning at install time is not enough either: `apt install <pkg>=<version> --allow-downgrades`
works once, and the next unrelated `apt upgrade` drags the epoch straight back. That case needs
`/etc/apt/preferences.d/<pkg>` at priority **1001** — above 1000, because the correct version is
"lower" than the incumbent and only a priority above 1000 permits a downgrade.

## C1–C6 — Guards, dry runs and verification mechanics

These are the rules that a single well-behaved host cannot teach. Each exists because a playbook
that reported `failed=0` was wrong.

### C1. A dry run must be honest

`--check` is a supported mode: a dry run must either verify or say plainly what it could not
verify. It must never fail with a misleading error, and it must never report `ok` for a check it
silently skipped.

The failure family is always the same — *a task that is skipped, or half-skipped, feeding a task
that is not*:

- **`command`/`shell` are skipped under `--check`,** so a read-only check that would prove
  something reports `ok` without having proved anything. Every task that only *reads* state must
  carry `check_mode: false`. This is the most common instance by far.
- **`chdir` is validated before check mode skips the task.** A `command` with `chdir` pointing at
  a scratch directory an earlier task creates dies on `Unable to change directory before
  execution`. Give the create and remove tasks `check_mode: false` so the directory really exists
  for the length of the run.
- **`get_url` is not skipped; it validates its destination.** Guard it with
  `not ansible_check_mode` where its staging directory would not exist.
- **`uri` *is* skipped, and registers nothing.** A checksum resolved from a published
  `checksums.txt` comes back empty, and the next guard reports the version pin as nonexistent.
  Give it `check_mode: false` — fetching a published checksum file changes nothing — and `--check`
  then genuinely validates the pin against upstream without downloading or installing.
- **A vendor-repo task reports `changed` without writing,** so apt has no candidate and the pinned
  install fails outright instead of simulating. Guard the install with
  `not (ansible_check_mode and <repo>.changed)`.

Where `--check` is the run that *would* have done the install, the later checks cannot verify
anything. Gate them on a `<tool>_can_verify` fact and emit a `debug` note saying verification was
deferred, rather than failing on a result that was never produced.

### C2. An install guard must be able to see an absent tool. *(tightens point 6)*

Point 6 rules out `stat.exists`. That is necessary, not sufficient — a version-aware guard can
still be blind to absence:

- **`dpkg-query -W -f='${Version}'` reports a version for a removed package.** A package in
  `deinstall ok config-files` state — binary gone, conffiles retained — answers with its version
  and exits 0, so the guard reads "already at the pin", skips the install, and the verification
  step then confirms the same phantom and prints it in the summary. Read `${Status}|${Version}`
  and compare against `install ok installed|<version>`. Every apt-based playbook here declares
  that prefix once as a play var:

  ```yaml
  <tool>_dpkg_ok: "install ok installed|"
  ```

  and every guard compares against `(<tool>_dpkg_ok ~ <tool>_version)`. Multi-package playbooks
  use the same var inside the per-package substring test —
  `('podman ' ~ podman_dpkg_ok ~ podman_version) not in <register>.stdout`. To find hosts in
  the state that exposes this:

  ```bash
  dpkg-query -W -f='${Status}|${Package}|${Version}\n' | grep -v '^install ok installed'
  ```

- **`npm ls -g a@1 b@2` exits 0 when *either* spec matches.** It is an OR across arguments, not an
  AND, so a guard checking two packages at once cannot tell "both pinned" from "only one of
  them". Run one `npm ls` per package and require every one to succeed.
- **A path can be owned by nothing.** `npm install -g` refuses an entire install with `EEXIST`
  rather than overwriting a file it does not own — including a dangling symlink left by a package
  that was replaced. Where a known shim may occupy the target path, `stat` it and remove only
  symlinks resolving into the tree being replaced.

### C3. A verify command must be tested against the binary being absent

This is the rule that catches the most embarrassing failure, and testing against a working
install is exactly the half that cannot catch it. A check shaped

```bash
<tool> --version 2>&1 | grep -qi '<tool>'
```

**succeeds when the tool is not installed**, because the shell's own `bash: <tool>: command not
found` contains the tool's name and `2>&1` feeds it straight into the grep.

So: **grep for version-bearing output** (`'git version 2.53.0'`), never for the tool's own name,
and **redirect stderr only where the tool genuinely writes its version there**. Combined with a
C2 blind spot, a vacuous verify command lets a host report a fully successful run while missing a
tool the summary claims to have installed.

Where a tool has no usable `--version` at all, verify something else real: the data file it
installs (present, non-empty, world-readable), or actual work it did (an ANSI escape converted to
a genuine HTML document, not merely echoed back).

### C4. Stage downloads and caches on disk, not `/tmp`

`/tmp` is a size-capped tmpfs on these hosts. Anything that downloads or caches — a vulnerability
database, a NuGet cache, a browser bundle, a release tarball — is staged under `/var/tmp`
instead. The failure mode is not a clean out-of-space error but whatever the tool does when its
own storage layer fills, such as `SQLITE_FULL`.

Stage in a per-version directory, remove it once the install is in place, and set `owner: root`
and an explicit mode on the installed file rather than leaving it to the caller's umask.

### C5. Resolve install locations at run time; never assume them

npm's global prefix is the standing example: an apt-installed `nodejs` defaults it to
`/usr/local`, a NodeSource one to `/usr`. Both are root-owned system paths, so either is fine —
but hardcoding one breaks the other. Read it from `npm config get prefix` and check the installed
version by absolute path beneath it.

Two consequences worth knowing:

- **Guard against a per-user version manager.** A global install performed under `become` lands
  wherever the shim on `PATH` points. Assert that `node` resolves under `/usr/bin` or
  `/usr/local/bin` before installing, or a per-user nvm/fnm can capture a system-wide install.
- **A prefix that moves orphans what was installed at the old one.** Because `/usr/local/bin`
  precedes `/usr/bin` on the default `PATH`, the stale copy then *shadows* the correct one — and
  a playbook that verifies by absolute path will report the new version correctly while the host
  runs the old one. Nothing tracks where a previous run installed; this is cleaned up by hand
  (`npm uninstall -g --prefix <old>`).

### C6. `$HOME` is not the only home

Several runtimes resolve their state directory from the **OS user database**, not from the `$HOME`
environment variable, so a scratch `HOME` alone does not redirect them. As uid 65534 the answer is
`/nonexistent`, which is not writable:

- **The JVM** reads `user.home` from passwd. Maven then tries to create
  `/nonexistent/.m2/repository`, and ZAP dies outright with `Unable to create home directory`.
  Pass `-Dmaven.repo.local` and `-dir` explicitly.
- **Ansible** resolves `remote_tmp` (`~/.ansible/tmp`) against the *remote user's* home as read
  from passwd, identifying that user from `USER`/`LOGNAME`. Under `become: true` those are still
  root, so overriding only `HOME` makes it try to `mkdir /root/.ansible/tmp` as uid 65534; clearing
  `USER`/`LOGNAME` instead just moves the target to `/nonexistent`. Set `ANSIBLE_REMOTE_TEMP`
  explicitly.

Also **`chdir` into the scratch directory**, not Ansible's working directory: uid 65534 cannot
stat the checkout path, and tools that stat the current directory just to start up — `git`, and
the `mvn` launcher, which walks up looking for a project base directory — fail with a permission
error that looks nothing like the real cause.

## Playbook skeleton

```yaml
---
# <what lands where, and why that is multi-user safe>
#
# Usage:
#   ansible-playbook <dir>/<tool>.yml -e host=<inventory host or group>

- name: Install <tool> system-wide
  hosts: "{{ host }}"
  become_user: root
  become_method: sudo
  become: true
  remote_user: ubuntu
  gather_facts: true

  vars:
    <tool>_version: "x.y.z"
    <tool>_verify_uid: 65534    # nobody
    <tool>_verify_gid: 65534    # nogroup

  tasks:
    - name: 1. Install prerequisites
    - name: 2. Check the currently installed <tool> version
    - name: 3. Decide whether <tool> needs installing
    - name: 4. ... install ...
    - name: N-2. Verify the installed <tool> version
    - name: N-1. <exercise the tool> as an unprivileged user
    - name: N. Display installation summary
```

Conventions: `hosts: "{{ host }}"` driven by an extra var, the `become`/`remote_user: ubuntu`
header block, numbered task names, and a header comment that states what lands where and records
any finding a future reader would otherwise have to rediscover.

Every playbook ends with a `debug` summary naming what was installed, what was verified and as
which uid, and the per-user commands the tool needs from each account (per A3).

## Review checklist

A playbook that touches anything in a "never touched" column is wrong. A playbook that makes a
grant without an explicit list is wrong.

| Tool | Per-user state (never touched) | Grant (explicit list, default empty) |
| --- | --- | --- |
| `aws` | `~/.aws/{config,credentials}` | — |
| `az` | `~/.azure/` | — |
| `gcloud` | `~/.config/gcloud/` | — |
| `gh` | `~/.config/gh/hosts.yml` | — |
| `glab` | `~/.config/glab-cli/config.yml` | — |
| `tofu` | `~/.terraform.d/` | — |
| `jira` | `~/.config/.jira/.config.yml` | — |
| `gcx` | `~/.config/gcx/`, `~/.local/state/gcx/` | — |
| `influx` | `~/.influxdbv2/configs` | — |
| `vault` | `~/.vault-token` | — |
| `az devops` | `~/.azure/azuredevops/config` | — |
| `databricks` | `~/.databrickscfg` | — |
| `azcopy` | `~/.azcopy/` — job plans, logs, cached Entra ID token | — |
| `docker` | `~/.docker/config.json`, `~/.docker/contexts` | `docker` group — **root-equivalent** |
| `podman` | `~/.local/share/containers`, `~/.config/containers` | `loginctl enable-linger`; subuid/subgid |
| `kubectl` | `~/.kube/config` | — |
| `helm` | `~/.config/helm`, `~/.cache/helm`, repo list | — |
| `kind` | `~/.kube/config` | needs `docker` group to do anything |
| `minikube` | `~/.minikube/` — certs, profiles, kicbase image | needs `docker` group to do anything |
| `k9s` / `kubelogin` | `~/.config/k9s`, `~/.local/state/k9s`, `~/.kube/cache/kubelogin` | — |
| `dotnet` | `~/.dotnet`, `~/.nuget/packages` | — |
| `pwsh` | `~/.cache/powershell`, `~/.config/powershell`, `~/.local/share/powershell` | — |
| `mvn` | `~/.m2` | — |
| `zap` | `~/.ZAP` | — |
| `mise` | `~/.cache/mise`, `~/.config/mise` | — |
| `ansible` | `~/.ansible` | — |
| `trivy` / `dvc` | `~/.cache/trivy`, `~/.config/dvc`, `~/.cache/dvc` | — |
| `mongosh` | `~/.mongodb/mongosh` | — |

Before opening a change for review, confirm:

- [ ] Version pinned in `vars:`, overridable with `-e` (5)
- [ ] Guard reads the installed version, and can see the tool absent (6, C2)
- [ ] Downloads outside apt are checksum- or signature-verified (7)
- [ ] Architecture from `ansible_facts['architecture']`, unmapped values fail loudly (8)
- [ ] Nothing is written under any `$HOME`, including root's (3, B2)
- [ ] Any grant takes an explicit list defaulting to empty, and is additive (B1)
- [ ] Completions to `/etc/bash_completion.d`, generated from the installed binary (B3)
- [ ] Verification runs as uid 65534, does real work where possible (9, A4, B4)
- [ ] Verify command was tested against the binary being **absent** (C3)
- [ ] `--check` neither fails misleadingly nor reports an unproven `ok` (C1)
- [ ] Scratch directories under `/var/tmp`, removed afterwards (C4)
- [ ] README section updated: install paths, version override, and any grant or regression a user will hit

## Known exceptions and outstanding debt

- **Three playbooks never verify under `--check`.** `core/devcontainers.yml`,
  `core/kind.yml` and `core/minikube.yml` gate their checks on a bare
  `not ansible_check_mode` rather than a `<tool>_can_verify` fact, so a dry run skips them even
  against a host that already has the pinned version. That is safe — it never reports an unproven
  `ok` and never fails misleadingly — but it is the conservative form, and the other 60 playbooks
  do better. Converting them means adding the fact and the 2a note, as C1 describes.

- **`deb822_repository` is not idempotent *across* control-node versions.** ansible-core 2.16 and
  2.20 write materially different files for identical input, so alternating control nodes reports
  `changed` on the repository task and fires the cache refresh with it. Harmless, but it explains
  a `+2 changed` that is not a real change.
- **npm-global playbooks do not sweep prefixes they no longer use.** See C5. Whether they should
  is the one open question there.
- **Most npm-global playbooks let npm write to `/root/.npm`.** `npm install -g` under `become`
  writes its cache and debug logs to `$HOME/.npm`, i.e. `/root/.npm`, which B2 forbids.
  [core/eslint.yml](core/eslint.yml) and [misc/jsmin.yml](misc/jsmin.yml) run the install with a
  scratch `HOME` under `/var/tmp` and remove it afterwards; [core/nodejs.yml](core/nodejs.yml),
  [core/markdownlint.yml](core/markdownlint.yml), [misc/mocha-chai.yml](misc/mocha-chai.yml),
  [misc/playwright.yml](misc/playwright.yml),
  [cloud-cli/auth0-deploy-cli.yml](cloud-cli/auth0-deploy-cli.yml) and
  [core/devcontainers.yml](core/devcontainers.yml) predate it and still write there.
  The tree is root-owned and root-readable, so nothing leaks, but it is state in a `$HOME` that
  no playbook should be creating.
- **Nothing removes `~/.bashrc` blocks or per-user trees written by earlier single-user
  playbooks.** Deleting a playbook removes nothing from a host that ran it. Marked blocks,
  `xhost` lines, `~/.krew` trees and Homebrew prefixes stay in every account that acquired them.
  Cleaning them would mean writing into accounts' `$HOME`, which is exactly what B2 forbids, so
  it is done by hand or not at all.
- **`apt upgrade` moves installed versions away from the pins,** which is expected rather than a
  bug: the next run surfaces the mismatch immediately instead of it drifting silently. Resolve it
  by advancing the pin, not by reverting the host — apt archives generally do not keep superseded
  `.deb`s, so reinstalling the old version usually is not possible.
