# Misc Playbooks (multi-user workstations)

Standalone playbooks that install developer tooling on a **shared** Ubuntu workstation, to
root-owned system paths usable by every account on the host rather than into one account's
home directory. See [POLICY.md](../docs/POLICY.md) for the rules they follow.

Run from `playbooks/`:

```bash
ansible-playbook misc/<tool>.yml -e host=<inventory host or group>
```

## Conventions

Every playbook here follows the same rules, so that a tool installed once is usable by
every account on the host, including accounts created later:

1. **Root-owned system paths only.** Binaries go to `/usr/local/bin` (or apt). No
   Homebrew: `/home/linuxbrew` is owned by whoever installed it, and Homebrew upstream
   does not support multi-user installs.
2. **Shell configuration goes to `/etc`**, never to `~/.bashrc`. Environment variables in
   `/etc/environment` (applies to login and SSH sessions via PAM); interactive-only
   settings such as key bindings and completions in `/etc/profile.d/<tool>.sh`, with
   `/etc/bash.bashrc` sourcing it for non-login interactive shells.
3. **World-readable install trees.** Explicitly set `mode: 'u=rwX,go=rX'` rather than
   relying on the umask of whoever ran the playbook.
4. **Pinned versions** in the play's `vars`, overridable with `-e`.
5. **Verified unprivileged.** Each playbook ends with a check that runs the tool as an
   arbitrary uid (`setpriv --reuid=65534`), not as the connecting user, so a
   single-user regression fails the run instead of going unnoticed.

## bats.yml

Installs the [Bats](https://github.com/bats-core/bats-core) testing framework and its
helper libraries from upstream git tags.

| Path | Contents |
| --- | --- |
| `/usr/local/bin/bats` | executable (via upstream `install.sh`) |
| `/usr/local/libexec/bats-core/`, `/usr/local/lib/bats-core/` | internals |
| `/usr/local/src/bats-core/` | checked-out source, kept for upgrades |
| `/usr/lib/bats/bats-support/` | helper library |
| `/usr/lib/bats/bats-assert/` | helper library |

`/usr/lib/bats` is bats-core's built-in default for `BATS_LIB_PATH`
(`libexec/bats-core/bats`: `BATS_LIB_PATH=${BATS_LIB_PATH-/usr/lib/bats}`), so test files
resolve the helpers with no per-user configuration:

```bash
setup() {
  bats_load_library bats-support
  bats_load_library bats-assert
}
```

The playbook also writes `BATS_LIB_PATH` to `/etc/environment` to make the location
explicit and to survive an upstream change of that default.

Version overrides:

```bash
ansible-playbook misc/bats.yml -e host=ws01 -e bats_core_version=1.13.0
```

## gomplate.yml

Installs [gomplate](https://github.com/hairyhenderson/gomplate) as a single static binary
at `/usr/local/bin/gomplate`, root-owned, mode `0755`. There is no per-user state and
nothing to add to a shell profile.

- **Version-aware idempotency.** The installed version is compared against the pin before
  re-downloading.
- **Checksum verification.** The SHA-256 is read from the release's published
  `checksums-v<version>_sha256.txt` at run time, so changing the version stays a one-flag
  change instead of also requiring a hardcoded hash update.
- **Architecture from facts.** `ansible_architecture` is mapped to the release asset
  suffix (`x86_64` → `amd64`, `aarch64` → `arm64`) rather than assuming amd64. Unmapped
  architectures fail with a clear message.

Version overrides:

```bash
ansible-playbook misc/gomplate.yml -e host=ws01 -e gomplate_version=4.3.3
```

## grype-syft.yml

Installs [grype](https://github.com/anchore/grype) (vulnerability scanner) and
[syft](https://github.com/anchore/syft) (SBOM generator) as single static binaries at
`/usr/local/bin/grype` and `/usr/local/bin/syft`, root-owned, mode `0755`. There is no
per-user state and nothing to add to a shell profile.

Installed via each project's own `install.sh` rather than Homebrew — a Homebrew install
would only be usable by whichever single account owned that prefix, defeating the point of
a shared workstation:

- **Pinned to the release tag, not `main`.** The script is fetched from
  `raw.githubusercontent.com/anchore/<repo>/v<version>/install.sh`, so its content is fixed
  to what that release published, the same trust boundary as `bats.yml` pinning a git tag.
- **Checksum verification comes from the installer itself.** `install.sh` downloads the
  `checksums.txt` published alongside the release and verifies the binary's SHA-256 before
  installing it — no separate Ansible checksum step is needed.
- **Architecture resolution comes from the installer itself.** `install.sh` maps
  `uname -m` to the release asset name internally, so there is no separate arch-mapping var.
- **Version-aware idempotency.** The installed version (parsed from `grype version` /
  `syft version` output) is compared against the pinned version before re-installing.

The unprivileged verification step runs `syft dir:/etc` and `grype dir:/etc` as `nobody`
with `HOME=/tmp`. For grype, this also downloads the vulnerability database into that
throwaway `HOME`, proving the tool can create and use its own per-user cache
(`$HOME/.cache/grype/db` by default) without any root-owned shared path — this needs
network egress and can take a little while on the first run.

Version overrides:

```bash
ansible-playbook misc/grype-syft.yml -e host=ws01 \
  -e grype_syft_tools='[{"name":"grype","version":"0.116.1","repo":"anchore/grype"},{"name":"syft","version":"1.50.0","repo":"anchore/syft"}]'
```

## trivy.yml

Installs [Trivy](https://github.com/aquasecurity/trivy) (vulnerability scanner) from
Aqua Security's own apt repository, `/usr/bin/trivy`, root-owned. There is no per-user
state and nothing to add to a shell profile.

Ubuntu carries no `trivy` package at all, so this adds the vendor's apt repository:

| Path | Contents |
| --- | --- |
| `/etc/apt/sources.list.d/trivy.sources` | repository definition, written by `deb822_repository` |
| `/etc/apt/keyrings/trivy.asc` | repository signing key, fetched and checksum-compared each run |
| `/usr/bin/trivy` | installed by apt |

- **The key is fetched by URL and pinned by fingerprint,** not embedded. Aqua Security's
  key expires **2029-04-15**, and [POLICY.md's A5](../docs/POLICY.md) pins the bytes only for
  a key with no expiry — a pinned copy would simply stop working at that date, while
  fetching each run picks up whatever successor Aqua publishes. The fingerprint
  `825AD9036F7C850E6A6FED4935B8ACA44FD9CA9F` is asserted after import, so a repository
  quietly pointed at some other key fails the run instead of being trusted by apt.
- **Version-aware idempotency**, same as `shellcheck.yml`: the pin is compared against
  `dpkg-query`'s exact version string. Aqua's repo happens to publish `trivy` without a
  Debian revision suffix, so this pin is just the upstream version (`0.73.0`).
- **Repository setup is unconditional.** `deb822_repository` compares by checksum, so it is
  a no-op when nothing moved — and a host already at the pinned version still gets its
  definition migrated off the `trivy.list` this playbook used to write, which is removed.

The unprivileged verification step runs `trivy fs --scanners vuln /etc` as `nobody`
under a disk-backed scratch `HOME` (`/var/tmp/trivy-verify`), the same `/var/tmp`
workaround `grype-syft.yml` needed: Trivy's vulnerability database is roughly 100MiB,
too large for a size-capped `/tmp` tmpfs on some hosts. This also proves Trivy can
create and use its own per-user cache (`$HOME/.cache/trivy` by default) with no
root-owned shared path — it needs network egress and can take a little while on the
first run.

Version overrides:

```bash
ansible-playbook misc/trivy.yml -e host=ws01 -e trivy_version=0.73.0
```

## hadolint.yml

Installs [hadolint](https://github.com/hadolint/hadolint) as a single static binary at
`/usr/local/bin/hadolint`, root-owned, mode `0755`. There is no per-user state and
nothing to add to a shell profile.

hadolint has no apt package or vendor repository, so this installs the upstream release
binary directly, following the same pattern as `gomplate.yml`:

- **Checksum verification** from the release's published `checksums.sha256`, resolved at
  run time so that changing `hadolint_version` stays a one-flag change.
- **Architecture from facts.** `ansible_architecture` is mapped to the release asset
  suffix (`x86_64` → `x86_64`, `aarch64` → `arm64`). Unmapped architectures fail with a
  clear message.
- **Version-aware idempotency.** The installed version (parsed from `hadolint --version`
  output) is compared against the pinned version before re-downloading.

The unprivileged verification step pipes `FROM scratch` into `hadolint -` (reading from
stdin) as `nobody`, with `HOME` deliberately left unset, proving hadolint's
zero-configuration path: no `.hadolint.yaml` is discovered or required.

Version overrides:

```bash
ansible-playbook misc/hadolint.yml -e host=ws01 -e hadolint_version=2.15.1
```

## jsonnet.yml

Installs [jsonnet](https://github.com/google/jsonnet) from the Ubuntu apt package,
`/usr/bin/jsonnet`, root-owned. There is no per-user state and nothing to add to a shell
profile.

A plain apt install, pinned by exact dpkg version the same way
[`core/shellcheck.yml`](../core/README.md) is.

The unprivileged verification step pipes the expression `1 + 1` into `jsonnet -` as `nobody`
and checks that the output is `2`.

Version overrides:

```bash
ansible-playbook misc/jsonnet.yml -e host=ws01 -e jsonnet_version=0.20.0+ds-3.3build1
```

## junit2html.yml

Installs [junit2html](https://gitlab.com/inorton/junit2html) via `pipx`, root-owned:

| Path | Contents |
| --- | --- |
| `/opt/pipx` | `PIPX_HOME` — the pipx-managed virtualenv holding junit2html |
| `/usr/local/bin/junit2html` | `PIPX_BIN_DIR` — the app symlink pipx creates |
| `/usr/local/share/man` | `PIPX_MAN_DIR` |

Runs `pipx` as `root` with `PIPX_HOME`/`PIPX_BIN_DIR`/`PIPX_MAN_DIR` redirected to the root-owned paths
above — the pipx-as-root pattern in [INSTALL-MECHANISMS.md](../docs/INSTALL-MECHANISMS.md) —
rather than the per-user `~/.local/bin` / `~/.local/pipx` that a plain `pipx install` as
the connecting user would use. There is nothing to add to a shell profile: junit2html is a
plain CLI with no per-user configuration.

- **`PIPX_MAN_DIR` matters.** Left unset, pipx creates `/root/.local/share/man` on every
  run — confirmed by removing that directory, running an install with the variable set (it
  stays gone) and one without it (it comes back). Writing under root's own `$HOME` is what
  [POLICY.md's B2](../docs/POLICY.md) forbids. The playbook also clears up the directory earlier runs left
  behind, with `rmdir` rather than `state: absent` so it goes only when empty; anything since
  put there is somebody's and is left alone. [`certbot.yml`](#certbotyml), the other pipx
  user here, does the same, and the two are no-ops for each other.
- **`--check` verifies instead of failing.** The read-only checks — the pipx version read and
  the unprivileged render — carry `check_mode: false`, so a dry run against an
  already-provisioned host really exercises them; and when `--check` is the run that would have
  done the install, the `junit2html_can_verify` gate skips them with a note rather than failing
  on a report that was never rendered. Same shape as the rest of `misc/`.
- **No `--version` flag.** junit2html has no way to report its own version, so both the
  idempotency check and the post-install verification instead parse `pipx list --short`,
  which prints `junit2html <version>` once installed.
- **Pinned to the current PyPI release**, not a GitHub tag: upstream moved off GitHub to
  GitLab after `v31.0.2`, so later releases (`31.1.4` and newer) have no corresponding
  GitHub tag at all. `pip`/`pipx` verify the downloaded package against the hash PyPI
  publishes in its index as part of every install; there is no separate checksum step to
  add, the same way apt-based playbooks in this directory need none.
- **World-readable install tree.** `mode: 'u=rwX,go=rX'` is applied recursively to
  `/opt/pipx` after install, since pipx's own venv creation is subject to the umask of
  whoever ran it (`root`, in this case).

The unprivileged verification step renders a small sample JUnit XML fixture to HTML and
confirms the output file exists.

Version overrides:

```bash
ansible-playbook misc/junit2html.yml -e host=ws01 -e junit2html_version=31.1.4
```

## kube-score.yml

Installs [kube-score](https://github.com/zegl/kube-score) as a single static binary at
`/usr/local/bin/kube-score`, root-owned, mode `0755`. There is no per-user state and
nothing to add to a shell profile.

- **Checksum verification** from the release's published `checksums.txt`, resolved at run
  time so that changing `kube_score_version` stays a one-flag change.
- **Architecture from facts.** `ansible_architecture` is mapped to the release asset
  suffix (`x86_64` → `amd64`, `aarch64` → `arm64`). Unmapped architectures fail with a
  clear message.
- **Version-aware idempotency.** The installed version (parsed from `kube-score version`
  output) is compared against the pin before re-downloading.

The unprivileged verification step scores a Deployment manifest that deliberately has a
floating `:latest` image tag, no resource limits, and no security context, and checks the
output flags the image tag issue. kube-score exits non-zero whenever it finds `CRITICAL`
issues, which this manifest is written to trigger — that non-zero exit is the expected,
successful outcome of the check, not a failure of the playbook run.

Version overrides:

```bash
ansible-playbook misc/kube-score.yml -e host=ws01 -e kube_score_version=1.20.0
```

## plantuml.yml

Installs [PlantUML](https://plantuml.com/) from the upstream release jar, root-owned, with a
wrapper script that runs it. There is no per-user state and nothing to add to a shell profile.

| Path | Contents |
| --- | --- |
| `/usr/local/lib/plantuml.jar` | the release jar, mode `0644` — read by every account, executed by none |
| `/usr/local/bin/plantuml` | wrapper: `java -Djava.awt.headless=true -jar …` |

**Not the apt package, deliberately — and this changed.** This playbook used to install
Ubuntu's `plantuml`, pinned at `1:1.2020.2+ds-6build1`. That is PlantUML **1.2020.2**, a 2020
release, and it is still what resolute carries; upstream ships roughly monthly and is at
1.2026.x, so the distro package is six years of diagram syntax and rendering fixes behind. The
jar is fetched from Maven Central instead, `net.sourceforge.plantuml:plantuml` — the GPL
distribution, the same artifact the GitHub release page calls `plantuml.jar`.

Central rather than the GitHub release for one reason: it publishes a `.sha256` beside every
artifact, so the hash is resolved at run time ([POLICY.md](../docs/POLICY.md) point 7) and changing
`plantuml_version` stays a one-flag change. GitHub's assets carry only detached `.asc`
signatures and no checksum file. The two jars for a given version are *not* byte-identical —
they are packed separately — so a hash taken from one will not verify the other.

Do not read the newest version out of Central's `maven-metadata.xml`: its `<latest>` is `8059`,
from PlantUML's pre-2017 numbering, which sorts after every `1.20xx.y` release.
[The GitHub releases page](https://github.com/plantuml/plantuml/releases) is the authority.

A JRE is a prerequisite, provisioned by [`core/openjdk.yml`](../core/README.md#openjdkyml) —
this playbook only checks for `/usr/bin/java` and fails with that instruction if it is missing,
the shape [`cloud-cli/jenkins-cli.yml`](../cloud-cli/README.md#jenkins-cliyml) and
[`maven.yml`](#mavenyml) use. There is no architecture map: a jar is a jar.

**`graphviz` is now an explicit prerequisite.** The apt package pulled it in through
`Recommends`; a jar has no packaging to do that, and every diagram family except sequence and
timing — class, activity, state, component, deployment — shells out to `dot`. So the playbook
installs `graphviz` itself, and verifies PlantUML can find it, rather than leaving a host that
renders sequence diagrams and fails on everything else.

**A leftover apt `plantuml` is reported, never removed.** Hosts that ran the earlier version of
this playbook still have `/usr/bin/plantuml` at 1.2020.2. `/usr/local/bin` precedes `/usr/bin`
on the default `PATH`, so the wrapper wins — and the verification proves that rather than
assuming it. The summary names the package and the `sudo apt remove plantuml` line; running it
is a deliberate act, the same convention [`asciidoctor.yml`](#asciidoctoryml) uses for the
converters apt also ships.

### Verification

Three checks, all as `nobody`:

- **A rendered sequence diagram.** A two-line diagram is piped into `plantuml -pipe -tsvg` and
  the output must contain both `<svg` and `<?plantuml 1.2026.7?>` — PlantUML opens the SVG it
  writes with that processing instruction, so the second assertion proves *this* jar rendered
  it and not an older copy earlier on `PATH`. A sequence diagram needs only the Java renderer,
  so this half stands on its own if graphviz is unavailable.
- **`plantuml -testdot`**, PlantUML's own installation check, which must answer
  `Installation seems OK` — the graphviz half of the install, asserted on its text rather than
  its exit status.
- **A login shell**, `env -i … bash -lc`, which must resolve `plantuml` to
  `/usr/local/bin/plantuml` *and* report `PlantUML version 1.2026.7` — an apt copy winning the
  lookup fails the run.

### Per-account

Nothing is configured per account. A diagram large enough to exhaust the JVM's default heap is
a property of the diagram, not of the install, so each user sets their own heap for themselves
and the wrapper passes it through:

```bash
export PLANTUML_JAVA_OPTS=-Xmx2g
```

Version overrides:

```bash
ansible-playbook misc/plantuml.yml -e host=ws01 -e plantuml_version=1.2026.7
```

## k6.yml

Installs [k6](https://k6.io/) (load testing tool) from Grafana's own apt repository,
`/usr/bin/k6`, root-owned. There is no per-user state and nothing to add to a shell profile.

Ubuntu carries no `k6` package at all, so this adds the vendor's apt repository, the same
pattern as [`trivy.yml`](#trivyyml):

| Path | Contents |
| --- | --- |
| `/etc/apt/sources.list.d/k6.sources` | repository definition, written by `deb822_repository` |
| `/etc/apt/keyrings/k6.asc` | repository signing key, fetched and checksum-compared each run |
| `/usr/bin/k6` | installed by apt |

- **Signing key and repository setup follow `trivy.yml`** exactly: the key is fetched by URL
  each run and its fingerprint `C5AD17C747E3415A3642D57D77C6C491D6AC1D69` asserted, because
  Grafana's key expires **2033-03-10**. The repository definition is written unconditionally
  and the `k6.list` this playbook used to write is removed.
- **amd64 only.** Grafana's apt repository publishes amd64 packages exclusively (no arm64
  build), so this playbook fails with a clear message on any other architecture rather than
  letting apt report a confusing "no candidate" error.
- **Version-aware idempotency**, same as `trivy.yml`: the pin is compared against
  `dpkg-query`'s exact version string, which k6's repo publishes without a Debian revision
  suffix.

The unprivileged verification step writes a trivial script (a single always-true `check()`,
no HTTP calls) and runs it with `k6 run` as `nobody`, with `K6_NO_USAGE_REPORT=true` so the
run stays fully offline — proving the zero-configuration path the same way
`hadolint.yml`'s stdin check does, without needing a disk-backed scratch directory the way
`trivy.yml` and `grype-syft.yml` do for their vulnerability databases.

Version overrides:

```bash
ansible-playbook misc/k6.yml -e host=ws01 -e k6_version=2.2.0
```

## playwright.yml

Installs [Playwright](https://playwright.dev/) via `npm install -g`, root-owned, the same
mechanism `core/markdownlint.yml` uses for a global npm install. Node.js itself is a
prerequisite provisioned elsewhere; this playbook only checks for it, following
`markdownlint.yml`'s Node-in-PATH guard.

| Path | Contents |
| --- | --- |
| `<npm prefix>/bin/playwright` | npm's bin symlink |
| `<npm prefix>/lib/node_modules/playwright` | the npm package |
| `/opt/playwright-browsers` | shared browser binaries (Chromium by default) |

**Browser binaries are the part a plain npm install doesn't solve.** By default
`playwright install` caches browsers under the invoking account's own
`$HOME/.cache/ms-playwright` — per-user, so every account on the host would separately
re-download several hundred MiB the first time it ran a test. `PLAYWRIGHT_BROWSERS_PATH`
redirects that cache to the shared `/opt` path instead:

- **Published to `/etc/environment`** for real login/SSH sessions, the same
  `lineinfile` pattern `bats.yml`'s `BATS_LIB_PATH` uses — and passed explicitly as task
  `environment` wherever this playbook itself invokes `playwright`, since neither `become`
  nor `setpriv` sources `/etc/environment`.
- **Chromium only by default**, to keep the download and disk footprint reasonable
  (~300MiB). Override `playwright_browsers` to add `firefox` and/or `webkit`.
- **`install --with-deps`** apt-installs the OS libraries each browser needs to run
  headless in the same command that downloads it. This needs to run as root on Linux —
  satisfied here because the whole play already runs under `become`.
- **The browser install step runs unconditionally**, not gated behind the npm package's
  own version check: `playwright install` is already idempotent (it skips any revision
  already present at `PLAYWRIGHT_BROWSERS_PATH`), and gating it on the npm package's
  idempotency check alone would miss a browsers directory that was wiped or never
  populated on an otherwise up-to-date host.

**The verification does not render a page, and that is deliberate.** The obvious check —
`playwright screenshot` on a `data:` URL as an arbitrary uid — **hangs indefinitely** here:
`chrome-headless-shell` starts and its zygote, GPU, network-service and renderer children
all come up, but the screenshot never completes. It was measured at 23 minutes before being
killed, with no output file ever written, and `playwright screenshot` takes no timeout of
its own — so a playbook built on it blocks forever, the same trap
[`jenkins-cli.yml`](../cloud-cli/README.md#jenkins-cliyml) documents for `jenkins-cli -help`.

What replaced it answers the actual multi-user question more directly, in under a second:

- **`playwright install --dry-run`** as `nobody` reports where each browser *would* be
  installed without downloading anything, so it reads the same registry a real run uses.
  The reported location must be under `/opt/playwright-browsers` — proof that
  `PLAYWRIGHT_BROWSERS_PATH` is honoured for an account that did no setup of its own.
- **The browser binary is then executed** as `nobody`, so the browser itself is exercised
  rather than merely located. The `find` that locates it matches only a world-executable
  file and runs as that uid, so the check fails if the permission pass above left the tree
  unreadable to an ordinary account.

Both carry an explicit `timeout` (`playwright_smoke_timeout`, 60s), so a future hang fails
the run instead of blocking it.

Version overrides:

```bash
ansible-playbook misc/playwright.yml -e host=ws01 -e playwright_version=1.62.1
ansible-playbook misc/playwright.yml -e host=ws01 -e playwright_browsers='["chromium","firefox","webkit"]'
```

## mocha-chai.yml

Installs [Mocha](https://mochajs.org/) (test runner) and [Chai](https://www.chaijs.com/)
(assertion library) via `npm install -g`, root-owned — the same mechanism
`core/markdownlint.yml` uses. Node.js itself is a prerequisite provisioned elsewhere; this
playbook only checks for it.

| Path | Contents |
| --- | --- |
| `<npm prefix>/bin/mocha` | npm's bin symlink — Mocha has a CLI |
| `<npm prefix>/lib/node_modules/mocha` | the npm package |
| `<npm prefix>/lib/node_modules/chai` | the npm package — Chai has **no** CLI |

**Chai is a pure library, which is a new problem for this directory.** Every other
`misc/` npm-based playbook installs a CLI binary onto `PATH` and stops there. Chai has no
binary at all — a test file needs `require('chai')` to resolve, and Node's module
resolution does not search the global npm prefix by default. The fix is `NODE_PATH`:

- **Published to `/etc/environment`**, the same `lineinfile` shape `bats.yml`'s
  `BATS_LIB_PATH` and `playwright.yml`'s `PLAYWRIGHT_BROWSERS_PATH` use, and passed
  explicitly wherever this playbook itself invokes `mocha`, since neither `become` nor
  `setpriv` sources `/etc/environment`.
- **Points at the global `node_modules` directory itself**, not anything mocha/chai
  specific — the same shared location already holding markdownlint-cli, playwright, yarn
  and pnpm. Any other playbook that sets the same `NODE_PATH` line is a no-op, not a
  conflict.

**Chai 6.x ships as ESM-only** (`"type": "module"` in its `package.json`, no CommonJS
entry point), yet plain `require('chai')` from a `.js` test file still works with no extra
configuration — confirmed live on this host's Node.js 24.19.0, which supports requiring an
ESM module directly (Node 22.12+/20.19+ — see `core/nodejs.yml`). Older Node would need
`import()` or a `.mjs` test file instead.

**Version-aware idempotency checks mocha and chai separately** — one `npm ls -g
<name>@<version>` call per package — rather than a single combined
`npm ls -g mocha@x chai@y`. Confirmed live: `npm ls -g`'s exit code is 0 as soon as *any
one* of several `name@version` arguments matches something installed; it is not an AND
across arguments, so a single combined call cannot tell "both pinned" apart from "only one
of them is."

The unprivileged verification step writes a spec file that asserts with Chai
(`expect(1 + 1).to.equal(2)`) and runs it with `mocha` as `nobody`, checking for `1
passing` in the output — proving the installed binary, the version pin, and the shared
`NODE_PATH` resolution of Chai all work together for an account that did no setup of its
own.

Version overrides:

```bash
ansible-playbook misc/mocha-chai.yml -e host=ws01 \
  -e mocha_chai_packages='[{"name":"mocha","version":"11.8.0"},{"name":"chai","version":"6.2.2"}]'
```

## maven.yml

Installs [Apache Maven](https://maven.apache.org/) from the Apache binary distribution,
root-owned, with a versioned tree and a symlink — the `cloud-cli/influx-cli.yml` shape, so
`ls -l /usr/local/bin/mvn` says which version is active and a bump installs beside the old
tree.

| Path | Contents |
| --- | --- |
| `/usr/local/lib/maven/<version>/` | the unpacked distribution |
| `/usr/local/bin/mvn` | symlink to that version's `bin/mvn` |

A JDK is a prerequisite, provisioned by [`core/openjdk.yml`](../core/README.md#openjdkyml) —
this playbook only checks for `javac` and fails with that instruction if it is missing, the
shape [`misc/dotnet-tools.yml`](#dotnet-toolsyml) uses for the .NET SDK. Nothing is written to
any `$HOME`: each account gets its own `~/.m2` (local artifact repository, and optionally a
personal `settings.xml` holding repository credentials) the first time it runs a build, which is
exactly the per-user state a shared workstation should keep per-user.

**Not the apt package, deliberately.** Ubuntu 26.04 carries `maven` 3.9.12-1, four patch
releases behind the 3.9.16 pinned here; the Apache tarball is self-contained, checksum-
verified, and the version this repo pins is the version installed. Maven 4.0.0 and 3.10.0
are both still release candidates as of 2026-08-17, so the 3.9.x line is the stable choice.

### How the install is put together

- **Integrity and idempotency.** The tarball is fetched with `get_url` against Apache's
  published `.sha512`, and the whole install is skipped when the pinned version is already
  active rather than re-downloaded every run.
- **The tarball, not the zip.** Apache's `-bin.zip` does not carry Unix permission bits
  reliably; the `.tar.gz` does, and its `bin/mvn` arrives already `0755`.
- **No `settings.xml` of this repo's own.** The distribution ships one and it is left exactly
  as shipped — writing a copy here to configure nothing is the dead config [POLICY.md's A2/A3](../docs/POLICY.md)
  removed elsewhere. A host that needs a proxy or a mirror sets it there or in each account's
  own `~/.m2/settings.xml`.
- **A checked JDK prerequisite**, [`core/openjdk.yml`](../core/README.md#openjdkyml), rather than
  installing one inline — the version pin for OpenJDK now lives in one file, not three.

### The JVM reads `user.home` from the passwd database, not `$HOME`

`HOME=<scratch> java -XshowSettings:properties` prints `user.home = /nonexistent` for uid
65534, so Maven tries to create `/nonexistent/.m2/repository` and fails no matter how
carefully `$HOME` is set — the same class of trap as `core/ansible.yml`'s `remote_tmp`.
The unprivileged checks therefore pass an explicit `-Dmaven.repo.local`. They also need
`chdir`: the `mvn` script walks up looking for a project base directory, and an unprivileged
process left in a directory it cannot read prints `cd: can't cd to /home/<invoker>` — the
second half of [POLICY.md's C6](../docs/POLICY.md), here triggered by Maven's own launcher.

### The smoke test is an offline build, plus a deliberate failure

`mvn -o -B validate` on a generated project must reach `BUILD SUCCESS` with an **empty**
local repository and no network at all — proving Maven resolved the super POM, the lifecycle
mapping and the project model out of the installed distribution. A second run against the
same POM with `<version>` removed must fail with `'version' is missing`: without that
negative half, a Maven that parsed nothing would still have reported success.

Version overrides:

```bash
ansible-playbook misc/maven.yml -e host=ws01 -e maven_version=3.9.16
```

## testssl.yml

Installs [testssl.sh](https://testssl.sh/), the TLS/SSL scanner, from its upstream git tree —
versioned directory plus a symlink, the same shape as [`maven.yml`](#mavenyml).

| Path | Contents |
| --- | --- |
| `/usr/local/lib/testssl.sh/<version>/` | the upstream tree: script, `etc/` data files, bundled OpenSSL |
| `/usr/local/bin/testssl.sh` | symlink to that version's script |

testssl.sh is not a single binary — it reads its cipher mappings and CA bundles from `etc/`
and prefers the OpenSSL build in `bin/`, both resolved relative to the script's own location.
So the whole tree is installed and only the entry point goes on `PATH`; confirmed live that
invoking it through the symlink, from a directory the caller cannot read, still finds both.
No per-user state, nothing in a shell profile: scans write where the caller asks
(`--htmlfile`, `--jsonfile`) and temporary files to `$TMPDIR`.

**Not the apt package, deliberately.** apt carries `testssl.sh` 3.2.2+dfsg-1, and the `+dfsg`
repack exists precisely because it strips the bundled OpenSSL binaries — the build testssl.sh's
own output calls *"OpenSSL 1.0.2-bad"*, kept deliberately broken so it still speaks SSLv2,
SSLv3 and export ciphers. This host's OpenSSL is 3.5.5, which refuses all of them, so the
stripped package cannot detect the weak protocols the scanner exists to find.

### What is pinned is a tag *and* a commit

Checking out a branch — or even a tag by name alone — installs whatever it points at on the
day and records nothing about what that was. This pins the `v3.2.4` tag **and** the commit that
tag pointed at when the pin was taken, since tags are mutable on GitHub and commit ids are not.
The commit is re-checked on **every** run, not just at install time, so a moved tag or an
edited tree is caught rather than silently inherited.

The versioned directory plus symlink is also what makes `depth: 1` safe: a per-version
directory is never re-pointed at another tag, which is the thing shallow clones make awkward
(`bats.yml` clones in full for exactly that reason). Worth having here — 22 MB against 145 MB
of history.

### The smoke test scans a real TLS endpoint

An `openssl s_server` with a throwaway certificate is started on `127.0.0.1`, scanned by uid
65534 through the published symlink, and killed by a trap whatever happens. Both directions are
asserted — TLS 1.2 reported as **offered** and SSLv2 as **not offered** — so neither a
testssl.sh that printed a fixed table nor one that never reached the server can pass. About six
seconds; `--protocols` keeps it to the protocol section rather than a full scan.

One quirk worth knowing if you touch the version check: `--version` must be the *only* option
on the command line (`Fatal error: --version is a standalone command line option`), so colours
cannot be disabled with `--color 0` and are stripped with `sed` instead. Its first line names
the program *as invoked* — through a differently-named symlink it prints that name — so the
assertion anchors on `version <x> from https://testssl.sh/`, never on the program name.

Version overrides — bump the tag and the commit together:

```bash
git ls-remote https://github.com/testssl/testssl.sh.git 'refs/tags/v3.2.4^{}'
ansible-playbook misc/testssl.yml -e host=ws01 -e testssl_version=3.2.4 -e testssl_commit=<sha>
```

## zap.yml

Installs [OWASP ZAP](https://www.zaproxy.org/) from the upstream release tarball into a
versioned tree with a symlink, the same shape as [`maven.yml`](#mavenyml) and
[`testssl.yml`](#testsslyml).

| Path | Contents |
| --- | --- |
| `/usr/local/lib/zap/<version>/` | the distribution — jars, add-ons, language packs (~270 MB) |
| `/usr/local/bin/zap.sh` | symlink to that version's launcher |

ZAP is a tree, not a binary: `zap.sh` resolves everything else relative to its own location, so
the whole distribution is installed and only the launcher goes on `PATH` — confirmed that
invoking it through the symlink works, which is why nothing here touches `PATH` in
`/etc/environment`. A non-headless JRE is a checked prerequisite, provisioned by
[`core/openjdk.yml`](../core/README.md#openjdkyml), rather than installed inline: ZAP with no
arguments *is* its desktop UI, which throws `HeadlessException` on a headless JRE, and on these
desktop workstations that is a real use. `core/openjdk.yml`'s `default-jdk` pulls in the full
`default-jre` (not `-headless`), which is what makes it the right shared prerequisite here too.

### ZAP's "home" is per-user state, and this playbook creates none of it

ZAP keeps configuration, its session database, downloaded add-ons and `zap.log` in `~/.ZAP` —
one directory per account, created on first run. That is exactly the state a shared workstation
must not share: a single world-writable copy would put one account's session history, and any
credentials captured in it, within everyone else's reach.

That makes verification a trap twice over, and both halves were reproduced on a target:

- **The JVM reads `user.home` from the passwd database, not `$HOME`**, so for uid 65534 ZAP
  resolves its home to `/nonexistent` and dies with `Unable to create home directory:
  /nonexistent/.ZAP/` no matter how `$HOME` is set. The scan passes an explicit `-dir`. Same
  trap [`maven.yml`](#mavenyml) documents, but fatal here rather than a fallback.
- **Running `zap.sh` as root without `-dir` creates `/root/.ZAP`**, which [POLICY.md's B2](../docs/POLICY.md)
  forbids. So ZAP is never run as root at all: the install check is filesystem state, and the
  single ZAP invocation in the play is the unprivileged scan.

### One invocation, which is also the version check

`zap.sh -version` costs about 35 seconds — it starts the JVM and initialises every add-on — so
running it as a separate step would only make the play slower. Instead the scan's own JSON
report carries `"@version"`, and that is what the pin is asserted against.

The scan itself is the tool doing its job: a static page is served on `127.0.0.1` by
`python3 -m http.server`, and ZAP crawls and passively scans it as uid 65534 through the
published symlink, writing a JSON report (~25 seconds). Two things are asserted — the report's
version matches the pin, and its site list names the loopback URL, proving ZAP actually reached
and crawled the server rather than reporting on nothing. `-silent` disables ZAP's optional
outbound calls (telemetry, add-on update checks) so the scan talks to nothing but the local
server, and the server is killed by PID from a trap — a pattern kill would match the task's own
command line.

Integrity comes from the SHA-256 GitHub computes per release asset (the
[`core/yq.yml`](../core/README.md#yqyml) source), since ZAP publishes no checksums file beside
the tarball.

Version overrides:

```bash
ansible-playbook misc/zap.yml -e host=ws01 -e zap_version=2.17.0
```

## mongodb-tools.yml

Installs the MongoDB client tooling: the [Database Tools](https://www.mongodb.com/docs/database-tools/)
and the [MongoDB Shell](https://www.mongodb.com/docs/mongodb-shell/). Clients only — no `mongod`,
nothing listening.

| Path | Contents |
| --- | --- |
| `/usr/bin/bsondump`, `mongodump`, `mongorestore`, `mongoexport`, `mongoimport`, `mongofiles`, `mongostat`, `mongotop` | `mongodb-database-tools`, from MongoDB's apt repository |
| `/usr/bin/mongosh`, `/usr/lib/mongosh_crypt_v1.so` | `mongodb-mongosh`, from its own release `.deb` |
| `/etc/apt/sources.list.d/mongodb.sources` | repository definition, key by URL, fingerprint pinned |

No per-user setup: mongosh creates `~/.mongodb/mongosh` (config, history, logs) for each account
on first use, which is per-account state a shared workstation should keep per-account. Nothing
here writes into anyone's `$HOME` — a plain `mongosh --version` writes nothing, checked with a
fresh `HOME`, which is why the version check can stay a `dpkg-query`.

### The 26.04 build of the Database Tools does not run on this repo's hosts

This is why the repository points at **`noble`** (24.04) rather than this release's own
`resolute`. MongoDB's resolute build of `mongodb-database-tools` 100.18.0 declares:

```
x86 ISA needed: x86-64-baseline, x86-64-v2, x86-64-v3
```

and every binary in it dies with `CPU ISA level is lower than required` on ws01 (Core i7-3615QM,
Ivy Bridge) and ws02 (Core i5-2520M, Sandy Bridge) — neither CPU has the AVX2-era instructions
`x86-64-v3` needs. MongoDB's noble build of the *identical* 100.18.0 is `x86-64-baseline`, runs on
both, is byte-for-byte the same file as the `ubuntu2404` package on `fastdl.mongodb.org` (same
SHA-256), and depends only on `libc6` and the krb5 libraries, all present on 26.04. Set
`mongodb_tools_release` to `resolute` once MongoDB ships a baseline build there, or on a fleet
that is uniformly `x86-64-v3`.

Note that MongoDB folds the server release series into the apt **suite**, not the component:
the sources line is `<release>/mongodb-org/<series> multiverse`, and apt fetches
`dists/noble/mongodb-org/8.0/InRelease`. Series 8.0 is used because it is the one whose signing
key MongoDB publishes — `server-9.0.asc` is a 404 at both of MongoDB's key URLs as of
2026-08-17 — and `mongodb-database-tools` is identical in both series.

### The package installs its binaries owned by uid 1000

`dpkg-deb -c` on MongoDB's `.deb` shows every file recorded as `ubuntu/ubuntu`, uid and gid
1000, and dpkg honours that: a plain install leaves `/usr/bin/mongodump` and its seven siblings
owned by whoever is uid 1000 on the host. On these workstations that is a real login account,
which would then be free to rewrite binaries every other account runs — and that root runs too,
under `sudo`. The playbook corrects ownership to `root:root` on **every** run, not only after an
install, so a host that already took the package the plain way is repaired as well. The
closing assertion that every tool is a root-owned executable is not a formality here: without
that fix, it fails.

### mongosh comes from its release `.deb`, not from apt

mongosh is not in the resolute repository at all, and the only suite that carries it
(`noble/mongodb-org/9.0`) is signed by the unpublished 9.0 key, so that repository's key cannot
be pinned. The `.deb` attached to mongosh's own GitHub release is used instead, verified against
the SHA-256 GitHub computes per asset — the [`core/yq.yml`](../core/README.md#yqyml) source. The
plain package is the one chosen deliberately: unlike the `shared-openssl11`/`shared-openssl3`
variants it bundles its own OpenSSL and depends on nothing but `libc6`, which is what makes it
safe to install outside a repository.

### Verification is two pieces of real offline work

- **`bsondump`** decodes a hand-written twelve-byte BSON document (`{"a": 1}`: int32 length,
  element type `0x10`, key `a\0`, value, terminator) written with `printf` and octal escapes,
  since a `copy:` block cannot carry NUL bytes. That exercises the same BSON reader
  `mongorestore` uses, with no server involved.
- **`mongosh --nodb`** evaluates JavaScript with no server to connect to, asserting arithmetic,
  the EJSON serialiser's date encoding, and the version the process reports about itself. It
  runs through `shell` with the expression single-quoted: `command` tokenises shlex-style
  without a shell, which eats the quotes around a JavaScript string literal and splits on the
  spaces inside one — mongosh answers either mistake with a `SyntaxError`.

Version overrides:

```bash
ansible-playbook misc/mongodb-tools.yml -e host=ws01 -e mongodb_tools_version=100.18.0
ansible-playbook misc/mongodb-tools.yml -e host=ws01 -e mongosh_version=2.10.0
```

## certbot.yml

Installs [certbot](https://certbot.eff.org/) and the Route 53 DNS plugin with `pipx`, run as
root with its state redirected to root-owned, world-readable paths — the same shape
[`junit2html.yml`](#junit2htmlyml) uses.

| Path | Contents |
| --- | --- |
| `/opt/pipx/venvs/certbot/` | the virtualenv: certbot and every injected plugin |
| `/usr/local/bin/certbot` | the app symlink pipx creates |
| `/usr/local/share/man/` | `PIPX_MAN_DIR` — see below |

The client only. Nothing here obtains, renews or installs a certificate, and no ACME account is
registered for anyone: those need real DNS or a real web server and are run by hand under
`sudo`. certbot's own state lives in `/etc/letsencrypt`, `/var/lib/letsencrypt` and
`/var/log/letsencrypt` — root-owned system paths, not per-user state — and none of the three is
created here.

### pipx, not apt and not snap

- **apt** carries certbot `4.0.0-4` on 26.04 against upstream's `5.7.0` — a whole major version
  behind. certbot speaks ACME to a live CA, and that gap is exactly where its protocol-level
  fixes sit, so this is one of the tools worth taking from upstream.
- **snap** installs cleanly, but it refreshes on Canonical's schedule rather than on a pin in
  this repository, which is the opposite of what every other playbook here does. Nothing else
  in `playbooks/` uses snap.

### `PIPX_MAN_DIR` is set deliberately

Without it, pipx creates `/root/.local/share/man`. Confirmed by experiment: remove that
directory, run an install with `PIPX_MAN_DIR` set — it stays gone and the configured directory
appears instead — then run one without it, and it comes back. Writing under root's own `$HOME`
is what [POLICY.md's B2](../docs/POLICY.md) forbids, and it also puts any man page a package ships somewhere no
other account can read. Here it points at `/usr/local/share/man`, where `man certbot` finds it.

Both this playbook and [`junit2html.yml`](#junit2htmlyml) — the other pipx user here — also
clear up the directory earlier runs left behind, with `rmdir` rather than `state: absent` so it
goes only when empty: anything since put there is somebody's and is left alone.

### The plugin is injected, not installed alongside

certbot discovers plugins through the `certbot.plugins` entry point group **inside its own
virtualenv**, so a separate `pipx install certbot-dns-route53` would leave `certbot plugins`
unchanged. `pipx inject` puts the plugin in the same venv, which is what makes it loadable.
Idempotency reads `pipx list --json` — the `--short` form names only the main package, and this
playbook has to compare injected plugin versions too — and only injects what is missing or at
the wrong version.

Adding another plugin is one list entry (`certbot_plugins`); it is injected and then verified
the same way.

### Verification lists the plugins as an unprivileged user

`certbot plugins` enumerates the entry points certbot can actually load, so a plugin appearing
there is one certbot could really use. The assertion requires the pinned plugins **and**
certbot's own `standalone`/`webroot` authenticators, so a truncated or empty listing cannot
pass. certbot insists on writable config, work and log directories even for read-only
subcommands, and its real ones are root-only by design, so the check points all three at a
scratch directory it removes afterwards.

One quirk: task 11 (the recursive mode fix on the pipx tree) reports `changed` under `--check`
while reporting nothing on a real run — with `recurse`, the `file` module cannot walk a tree it
is not allowed to touch and answers conservatively.

Version overrides — plugins track `certbot_version` unless pinned individually:

```bash
ansible-playbook misc/certbot.yml -e host=ws01 -e certbot_version=5.7.0
```

## dvc.yml

Installs [DVC](https://dvc.org/) (Data Version Control) from Iterative's own apt repository.

| Path | Contents |
| --- | --- |
| `/usr/bin/dvc` | the CLI — a self-contained bundle, ~200 MB down and ~570 MB installed |
| `/etc/apt/sources.list.d/dvc.sources` | repository definition, key by URL, fingerprint pinned |

`git` is the package's only declared dependency, so nothing here has to provide a Python for
it. No per-user setup and nothing added to a shell profile: DVC writes `~/.config/dvc` (its
global config and the anonymous analytics id) and `~/.cache/dvc` for each account on first use,
and a project's own data lives in that project's `.dvc` directory.

**Version pin.** The repository's newest build, `3.67.1`, is also the current PyPI release, so
apt and upstream agree here:

```bash
curl -sSL https://dvc.org/deb/dists/stable/main/binary-amd64/Packages \
  | awk '/^Package: dvc$/{f=1} f&&/^Version:/{print $2; f=0}' | sort -V | tail -1
```

**The signing key expires 2027-03-05**, so per [POLICY.md's A5 rule](../docs/POLICY.md) it is fetched by URL each run
(the module compares by checksum, so that stays idempotent) with only the fingerprint pinned —
the `mise.yml`/`github-cli.yml` form rather than an inline key.

**amd64 only, and said so explicitly.** Iterative serves no arm64 index at all — `binary-arm64`
is a 403, not an empty index — so instead of an architecture map with a missing entry, the
playbook carries a list of architectures it can serve and fails on anything else with the
reason.

### Analytics is left to each account

DVC reports anonymous usage by default, and this playbook turns that off for nobody: it is a
per-account or per-site decision, and the file it belongs in is not one this repository owns.
The closing summary prints both the per-account command and the system-wide file
(`/etc/xdg/dvc/config`, which DVC reads ahead of each account's own config). The verification
does set `DVC_NO_ANALYTICS`, the same way [`cloud-cli/azure-cli.yml`](../cloud-cli/README.md#azure-cliyml)
keeps its smoke tests from forking a telemetry uploader.

### The smoke test tracks a file and checks the hash

As uid 65534, in a scratch directory that doubles as `HOME`: `dvc init --no-scm` (the directory
is not a git repository, and DVC otherwise refuses), then `dvc add` on a file whose content the
playbook wrote. DVC hashes the file, moves it into the project cache and writes a `.dvc`
pointer beside it — all offline, nothing contacts a remote.

The assertion then reads that pointer and compares its `md5` against the hash **computed from
the same string the file was written from**, not against a value recorded from an earlier run.
Size and tracked path are checked too. A DVC that wrote a plausible pointer without hashing
anything cannot pass.

Version overrides:

```bash
ansible-playbook misc/dvc.yml -e host=ws01 -e dvc_version=3.67.1
```

## dotnet-tools.yml

Installs four .NET global tools, root-owned, with the .NET SDK from
[`core/dotnet.yml`](../core/README.md#dotnetyml) as a prerequisite — this playbook only checks
for it and fails with that instruction if it is missing, the shape
[`core/markdownlint.yml`](../core/README.md#markdownlintyml) uses for Node.js.

| Path | Contents |
| --- | --- |
| `/usr/local/lib/dotnet-tools/` | the tools, installed with `--tool-path` |
| `/usr/local/bin/sqlpackage`, `dotnet-xdt`, `dotnet-sonarscanner`, `ps-rule` | one wrapper per command |

| Tool | Package | Version |
| --- | --- | --- |
| `sqlpackage` | `microsoft.sqlpackage` | 170.4.83 |
| `dotnet-xdt` | `dotnet-xdt` | 2.2.1 |
| `dotnet-sonarscanner` | `dotnet-sonarscanner` | 11.2.1 |
| `ps-rule` | `Microsoft.PSRule.Tool` | 2.9.0 |

### Why wrappers rather than `--tool-path /usr/local/bin`

A dotnet global tool is framework-dependent: it asks for the exact .NET major version it was
built against, and Ubuntu 26.04 ships only .NET 10. Measured here:

- **`Microsoft.PSRule.Tool` 2.9.0 asks for `Microsoft.NETCore.App` 6.0.0** and dies with *"You
  must install or update .NET to run this application"* even though 10.0.10 is installed.
- Its **3.0.0 prerelease asks for 8.0.0** and dies the same way, so tracking the prerelease
  buys nothing.
- **`DOTNET_ROLL_FORWARD=Major` makes it run on 10** — confirmed, `ps-rule --version` then
  answers 2.9.0.

That variable belongs to the tool, not the host: exporting it in `/etc/environment` would
change how *every* .NET application on the machine resolves its runtime. So the tools live in
their own `--tool-path` and each published command is a small wrapper that sets it for that one
process, honouring an account's own `DOTNET_ROLL_FORWARD` if it has set one — the
[`cloud-cli/jenkins-cli.yml`](../cloud-cli/README.md#jenkins-cliyml) shape, for the same reason.

### Notes from measuring the tools

- **`dotnet tool update --version X --tool-path P` covers all three cases** — absent, at another
  version, already at the pin — and exits 0 for each. `install` refuses an existing tool, so
  the playbook uses `update` throughout and decides what to run from `dotnet tool list`.
- **`dotnet` refuses to run when `HOME` points at a directory that does not exist** (*"The
  user's home directory could not be determined. Set the `DOTNET_CLI_HOME` environment
  variable"*), so the root-side scratch `HOME` is removed only after the verification that uses
  it, not before.
- **`dotnet-xdt` has no version flag** (`--version` answers *"Invalid argument"*), so its pin is
  held by `dotnet tool list` and its verification is a real XML transform: XDT rewrites a
  `value="dev"` attribute to `value="prod"`, and the assertion requires the new value present
  **and** the old one gone. Entirely offline.
- **The `sonar-scanner` permission fix is not needed at 11.2.1.** Earlier versions shipped a
  shell script in the tool store that needed `chmod 0755`; no such file exists in this release.
- `sqlpackage` reports a four-part version (`170.4.83.3`), so its check matches the pin as a
  prefix.

### No PSRule modules are installed for anyone

`ps-rule module add PSRule.Rules.Azure` writes into the running account's own home, and which
module set a project needs is the project's decision rather than the workstation's. The closing
summary prints the command for an account that wants it.

Version overrides — one variable per tool:

```bash
ansible-playbook misc/dotnet-tools.yml -e host=ws01 -e dotnet_sonarscanner_version=11.2.1
```

## scc.yml

Installs [scc](https://github.com/boyter/scc) — *Sloc, Cloc and Code*, a code counter with
complexity and cost estimation — as a single static binary, root-owned.

| Path | Contents |
| --- | --- |
| `/usr/local/bin/scc` | the binary, mode `0755` |
| `/etc/bash_completion.d/scc` | shared completion, generated from the installed binary |

Ubuntu publishes no `scc` package and upstream runs no apt repository, so this takes the
release tarball, the same shape [`cloud-cli/loki-cli.yml`](../cloud-cli/README.md#loki-cliyml)
uses:

- **Checksum verification** against the release's published `checksums.txt`, resolved at run
  time so that changing `scc_version` stays a one-flag change. An asset the checksums file
  does not describe fails with that as the message, rather than a Jinja type error.
- **Architecture from facts.** `ansible_facts['architecture']` is mapped to the release asset
  (`x86_64` → `scc_Linux_x86_64.tar.gz`, `aarch64` → `scc_Linux_arm64.tar.gz`). The release
  also carries i386, Darwin and Windows assets; only the two Linux ones these hosts can run
  are mapped, and an unmapped architecture fails with a clear message.
- **Version-aware idempotency.** `scc --version` prints `scc version 4.0.0` on stdout, and the
  guard compares that first line in full rather than as a substring, so `4.0.0` cannot be
  satisfied by a future `4.0.01`.
- **Staged on disk.** The tarball is flat (`LICENSE`, `README.md`, `scc`, no top-level
  directory), so it is unpacked in `/var/tmp/scc-install-<version>` — not the size-capped
  `/tmp` tmpfs — and only the binary is copied into place, root-owned at an explicit mode.

### No configuration is set for anyone

scc 4.x reads a global config from the path in `SCC_CONFIG_PATH` and a project one from
`./.sccconfig`, in precedence order global < project < CLI. Neither is a site-wide fact —
which COCOMO wage, currency or output format a run should use belongs to the person or the
repository, not to the workstation — so this playbook sets neither, and the closing summary
prints the export for an account that wants one.

There is nothing else per-user: measured under `env -i`, with neither `HOME` nor `PATH` set,
scc counts a tree and renders its git-history reports (`--hotspots`, `--coupling`,
`--by-author`, `--timeline`) without writing anything to `$HOME` — it reads `.git` itself
rather than shelling out to a `git` binary, so there is no runtime prerequisite either.

### Verification

The playbook writes a two-file sample tree (one Python, one Go) under `/var/tmp/scc-smoke`,
counts it as `nobody` with `SCC_CONFIG_PATH` unset and no `.sccconfig` anywhere — the
zero-configuration path an account with no setup of its own gets — and asserts the exact
result, language → `[lines, code, comment, blank, complexity]`:

| Language | Lines | Code | Comment | Blank | Complexity |
| --- | --- | --- | --- | --- | --- |
| Go | 8 | 6 | 1 | 1 | 1 |
| Python | 4 | 2 | 1 | 1 | 0 |

The Go `if` is what makes the complexity column non-zero, so this exercises the language
classifier and the complexity estimator rather than only the file walker. It is entirely
offline. A second check opens an interactive shell as `nobody` and requires `complete -p scc`
to answer, proving the shared completion is really loaded rather than merely written.

A version bump that changes how scc counts that sample will fail this assertion with both the
counts it got and the counts it expected; re-measure the sample and update
`scc_smoke_expected`.

Version overrides:

```bash
ansible-playbook misc/scc.yml -e host=ws01 -e scc_version=4.0.0
```

## asciidoctor.yml

Installs [AsciiDoctor](https://asciidoctor.org/) with its diagram and PDF extensions as
root-owned gems.

| Path | Contents |
| --- | --- |
| `/var/lib/gems/3.3.0/gems/` | the four pinned gems and their dependencies |
| `/usr/local/bin/asciidoctor` | the converter |
| `/usr/local/bin/asciidoctor-pdf` | the PDF converter |

| Gem | Version | What it is |
| --- | --- | --- |
| `asciidoctor` | 2.0.26 | the converter |
| `asciidoctor-diagram` | 3.2.1 | the diagram extension (`-r asciidoctor-diagram`) |
| `asciidoctor-diagram-plantuml` | 1.2026.2 | PlantUML wrapped in a gem — its version *is* PlantUML's |
| `asciidoctor-pdf` | 2.3.24 | the PDF backend |

Ruby is a prerequisite, provisioned by [`core/ruby.yml`](../core/README.md#rubyyml); this
playbook only checks for it and fails with that instruction if it is missing, the shape
[`mocha-chai.yml`](#mocha-chaiyml) uses for Node.js and [`maven.yml`](#mavenyml) for the JDK.
Both install paths above are read back from RubyGems at run time rather than assumed — they
carry the Ruby ABI version, so they move when the interpreter does.

**Gems, not apt.** Ubuntu 26.04 packages some of this — `asciidoctor` 2.0.26 and
`ruby-asciidoctor-pdf` 2.3.19 — but not all of it: there is no `ruby-asciidoctor-diagram`
package at all, and the PDF converter lags upstream. Since the diagram extension has to come
from RubyGems regardless, all four gems come from there, pinned, so one mechanism explains the
whole install.

**If those apt packages are also installed, these gems shadow them** — the intended outcome,
not a collision to resolve. RubyGems' bindir for a root install is `/usr/local/bin`, ahead of
`/usr/bin` on the default `PATH`, and where two copies of a gem are visible RubyGems activates
the higher version. The verification proves it rather than assuming it: it resolves
`asciidoctor` on the *login* `PATH` and asserts both the path and the version, so an apt
package winning the lookup fails the run. The summary lists any such packages and the
`apt remove` line for them; removing them is a deliberate act, not something the playbook does
on its own.

**The diagram extension needs Java, and its own PlantUML gem.** asciidoctor-diagram 3.x no
longer bundles the PlantUML jar the way 2.x did: it looks for the `asciidoctor-diagram-plantuml`
gem, then `DIAGRAM_PLANTUML_CLASSPATH`, then a `plantuml-native` binary on `PATH`, and raises if
it finds none. So the jar gem is pinned here alongside it — at 1.2026.2 — and rendering shells
out to `java`, which [`core/openjdk.yml`](../core/README.md#openjdkyml) provisions and this
playbook checks for. [`plantuml.yml`](#plantumlyml) is unrelated: it installs the standalone
CLI, which asciidoctor-diagram does not call, so the two PlantUML versions are pinned
independently and are not expected to match.

### Verification

As `nobody`, with the `PATH` a real login session gets, the playbook converts a document
carrying a PlantUML block and asserts four things at once: `asciidoctor` resolved to the gem
binstub, it reported the pinned version, the diagram rendered, and it was *this* PlantUML that
rendered it — PlantUML opens the SVG it writes with a `<?plantuml <version>?>` processing
instruction, which is matched against the pinned gem version. A second check converts a
document with `asciidoctor-pdf` and requires a real PDF back (`%PDF` magic, non-trivial size),
since a PDF has no marker text to grep for.

Version overrides:

```bash
ansible-playbook misc/asciidoctor.yml -e host=ws01 -e asciidoctor_version=2.0.27
ansible-playbook misc/asciidoctor.yml -e host=ws01 -e asciidoctor_pdf_version=2.3.25
```

Each gem's latest release:

```bash
curl -sS https://rubygems.org/api/v1/gems/asciidoctor.json | jq -r .version
```

## jsmin.yml

Installs [jsmin](https://www.npmjs.com/package/jsmin) — Douglas Crockford's JavaScript
minifier, ported to Node by Peteris Krumins — via `npm install -g`, root-owned, the same
mechanism [`mocha-chai.yml`](#mocha-chaiyml) and `core/eslint.yml` use. Node.js itself is a
prerequisite provisioned elsewhere (`core/nodejs.yml`); this playbook only checks for it.

| Path | Contents |
| --- | --- |
| `<npm prefix>/bin/jsmin` | npm's bin symlink |
| `<npm prefix>/lib/node_modules/jsmin` | the npm package |
| `/etc/environment` | `NODE_PATH=<npm prefix>/lib/node_modules` — for library use, not the CLI |

**Read this before choosing jsmin for anything.** It is Crockford's 2002 `jsmin.c`
algorithm, character-level and with no ES6 awareness, last published 2012-02-01 and
unmaintained since. Confirmed live on this host's Node.js 24.19.0: a `//` sequence *inside a
template literal* is taken for a line comment and everything to the end of that line is
deleted — silently, exit status 0, and the output is a `SyntaxError` when Node loads it.

```js
const greet = (name) => `hello ${name} // not a comment`;   // in
const greet=(name)=>`hello ${name}                          // out — backtick never closed
```

Nothing warns. Treat it as an ES5-era tool, check that its output still parses, and prefer a
current minifier (terser, esbuild) for modern sources. Its licence field — "Doug Crockford's
license that allows this module to be used for Good but not for Evil" — is not an OSI licence,
which is worth knowing before it reaches a licence scanner.

**jsmin has no `--version`,** and no flag that reports one: the CLI parses `-l`/`-c`/`-o`/
`--overwrite` and treats the first other argument as the input *filename*, so
`jsmin --version` tries to open a file called `--version` and dies with an uncaught `ENOENT`
and a stack trace. The install guard and the pin check therefore read npm's own metadata
(`npm ls -g jsmin@<version>`, non-zero both for a missing package and for a wrong version),
and the real proof is the minification described below — the case
[POLICY.md's C3](../docs/POLICY.md) covers with "verify something else real".

**The CLI needs no `NODE_PATH`; library use does.** `bin/jsmin` does `require('jsmin')` —
itself a global package — and that resolves because Node follows the `<prefix>/bin/jsmin`
symlink to its real path inside `<prefix>/lib/node_modules/jsmin/bin/`, from where the
enclosing `lib/node_modules` *is* an ancestor `node_modules` directory. `require('jsmin')`
from a project file anywhere else is not so lucky, so `NODE_PATH` is published to
`/etc/environment` with the identical line `mocha-chai.yml` and `core/eslint.yml` write —
pointing at the shared global `node_modules`, so whichever playbook runs second is a no-op.
It does **not** help an ESM caller: `import 'jsmin'` from a `.mjs` file still fails with
`ERR_MODULE_NOT_FOUND`, since Node ignores `NODE_PATH` for `import`.

Unlike ESLint, nothing in the archive occupies the target path — Ubuntu ships no `jsmin`
package, and `python3-jsmin` and `libghc-hjsmin-dev` are different implementations that
install no `/usr/bin/jsmin` — so the `EEXIST` collision `core/eslint.yml` documents does not
arise. The playbook asks `dpkg -S` who owns the path at run time anyway, rather than trusting
that to stay true, and fails with the removal remedy if the answer is an apt package.

### Verification

Three checks, all as `nobody` in a scratch directory under `/var/tmp`:

1. **Minify an ES5 file with `NODE_PATH` unset** — asserts the output is genuinely minified
   *and* that the comment is gone, so echoing the input back unchanged cannot pass, and the
   unset variable is what proves the CLI's zero-configuration path.
2. **Minify with `-o` and run the result under `node`** — proves the file-output path works
   for an ordinary account and that what came out is executable JavaScript, the check the
   template-literal finding above says every jsmin output deserves.
3. **`require('jsmin')` from a project file with `NODE_PATH` set** — asserts the variable the
   playbook published actually does something, rather than assuming it.

The fixture is deliberately ES5: a modern-syntax one would be testing jsmin's worst behaviour
rather than its install.

Version override:

```bash
ansible-playbook misc/jsmin.yml -e host=ws01 -e jsmin_version=1.0.1
```

The latest release (1.0.1 is also the newest there has ever been):

```bash
curl -sS https://registry.npmjs.org/jsmin | jq -r '."dist-tags".latest'
```

## exiftool.yml

Installs [exiftool](https://exiftool.org/) from the Ubuntu archive. The apt package is named
after the Perl distribution it packages, `libimage-exiftool-perl`, not after the tool.

| Path | Contents |
| --- | --- |
| `/usr/bin/exiftool` | the CLI |
| `/usr/share/perl5/Image/ExifTool/` | the modules it loads at run time |

Nothing is added to a shell profile and no per-user state is created: `~/.ExifTool_config` is
read only where an account has written one itself, which is where a personal tag definition
belongs.

### Verification

`exiftool -ver` prints the upstream version alone — no Debian revision, no `+dfsg` repack
suffix — so the pin `13.50+dfsg-1` is checked against dpkg, and `13.50` against the CLI. Both
checks run as `nobody`, the second one reading a real file's `MIMEType`, because that is what
proves the `Image::ExifTool` module tree is loadable by an account that did not install it.
A version string on its own does not show that.

Version override:

```bash
ansible-playbook misc/exiftool.yml -e host=ws01 -e exiftool_version=13.55+dfsg-1
```

The version the target's own apt sources carry:

```bash
apt-cache policy libimage-exiftool-perl
```
