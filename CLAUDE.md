# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Ansible playbooks that install developer tooling on a **shared, multi-user** Ubuntu 26.04
workstation: every tool lands in root-owned system paths (`/usr/local/bin`, apt, `/usr/lib/<tool>`)
and `/etc` drop-ins, so it works for every account, including ones created later. No Homebrew, no
per-user version managers, no writes to any `$HOME`. The README's *Motivation* section explains why;
`playbooks/docs/POLICY.md` is the rule set and `playbooks/docs/INSTALL-MECHANISMS.md` classifies how
each playbook gets its tool onto disk. Read both before writing or changing a playbook; a change is
reviewed against POLICY.md's checklist.

There is no build, test suite or linter. Validation is `--syntax-check`, a `--check` dry run, and a
real run against a target host whose closing smoke test runs the tool as uid 65534.

## Commands

Everything runs from `playbooks/`, because Ansible only reads `ansible.cfg` from the working directory.
The real inventory `playbooks/inventory.ini` is gitignored; copy `inventory.ini.example` to create it.
Targets are named with `-e host=<inventory host or group>`, never hardcoded.

```bash
cd playbooks
ansible -m ping ws01                                          # reachability first
ansible-playbook core/shellcheck.yml -e host=ws01             # one host
ansible-playbook core/shellcheck.yml -e host=workstations     # a group
ansible-playbook core/shellcheck.yml -e host=ws01 --check     # dry run; must verify or say what it skipped (POLICY C1)
ansible-playbook core/shellcheck.yml -e host=ws01 -e shellcheck_version=0.11.0-2   # override a pin
ansible-playbook --syntax-check -i inventory.ini.example -e host=localhost core/shellcheck.yml
ansible -m shell -a 'apt-cache policy <pkg>; apt-cache madison <pkg>' ws01   # what the target can see
```

Ansible refuses to start when stdio is non-blocking ("Ansible requires blocking IO"), which happens
inside this tool's shell. Run it as `ansible-playbook ... </dev/null >log 2>&1` and read the log.

Control node: Ubuntu 24.04 (ansible-core 2.16) or 26.04 (2.20). Nothing may use a module newer than
2.16. Targets: Ubuntu 26.04 (`resolute`), amd64 or arm64, reached over SSH with passwordless sudo.

## Layout

- `playbooks/core/` runtimes and base tooling (Node.js, .NET, OpenJDK, Ruby, Docker, kubectl, ...)
- `playbooks/cloud-cli/` cloud and service CLIs (aws, az, gcloud, gh, terraform, vault, ...)
- `playbooks/misc/` developer tools (trivy, k6, maven, zap, bats, ...)
- `playbooks/docs/POLICY.md` numbered rules (1-10, A1-A5, B1-B5, C1-C6); playbook comments and
  READMEs cite these ids (`per A5`, `C2`), so never renumber them.
- `playbooks/docs/INSTALL-MECHANISMS.md` which of six mechanisms each playbook uses, with counts.
- Each directory has a `README.md` with one `## <tool>.yml` section per playbook (install paths,
  version override, grants, per-account setup). A new or changed playbook updates its section, and
  INSTALL-MECHANISMS.md's table and counts.
- `DESKTOP-HOST-SETUP.md` and the two `setup-passwordless-*.sh` scripts are host bootstrap, run by
  hand before any playbook.

## How a playbook is built

One self-contained playbook per tool, no roles (POLICY point 10). Shared runtimes (Node.js, OpenJDK,
.NET, PowerShell, Ruby) are installed and pinned in exactly one playbook; consumers check for them and
fail with instructions instead of installing them. Every playbook has the same shape, and the
numbered task names are part of the convention:

1. Header comment: what lands where, and any finding a future reader would otherwise rediscover.
   The `Usage:` lines include a `-e <tool>_version=...` example that is kept at the current pin.
2. `vars:` with the exact pin (`<tool>_version`), `<tool>_dpkg_ok: "install ok installed|"` for apt
   installs, arch map from `ansible_facts['architecture']` (that spelling, not
   `ansible_architecture`), and `<tool>_verify_uid/gid: 65534`.
3. Read the installed version with `check_mode: false` (POLICY C1), comparing dpkg
   `${Status}|${Version}` against the pin so a removed-but-not-purged package is seen as absent (C2).
4. `set_fact` of `<tool>_install_needed` and `<tool>_can_verify`, plus a task `2a`/`9a` debug note
   when a dry run cannot verify.
5. Install. Downloads outside apt are checksum- or signature-verified and staged under `/var/tmp`
   (`/tmp` is a small tmpfs, C4), per-version directory plus symlink where there is a tree.
6. Verify the version by matching version-bearing output, never the tool's name (C3), then exercise
   the tool as uid 65534 via `setpriv --reuid=65534 --regid=65534 --clear-groups`, with a scratch
   `HOME` under `/var/tmp` where the tool writes state on first run, removed afterwards.
7. Closing `debug` summary: what was installed, what was verified, and the per-account commands
   (`gh auth login`, `gcloud auth login`) the playbook deliberately does not run.

Privilege grants (`docker_users`, `podman_linger_users`) are explicit lists that default to empty and
are additive only (B1). Completions go to `/etc/bash_completion.d`, hooks and aliases to
`/etc/profile.d/<tool>.sh` (B3). Secrets never leave `$HOME` (A1).

### Vendor apt repositories (POLICY A5)

Use `ansible.builtin.deb822_repository` (needs `python3-debian` on the target, and an explicit
`apt: update_cache` afterwards conditioned on the repository task's `changed`). Pin the signing key's
fingerprint always; pin the key bytes inline only when the key has no expiry, otherwise fetch by URL
each run. Several repositories are deliberately pointed at `noble` rather than the host codename via a
`*_suite_map` var, each for a reason stated in that playbook's header: Microsoft publishes no
`resolute` suite, HashiCorp hosts already carry a `noble` entry for the same URI, MongoDB's `resolute`
build needs a newer ISA level. `core/kubectl.yml` additionally pins the apt origin at priority 1001
because Google's repo publishes an epoched `kubectl` (B5). `deb822_repository` writes different files
under ansible-core 2.16 and 2.20, so alternating control nodes shows a harmless `changed`.

## Pins and how they go stale

Pins are exact dpkg strings including revision and, where present, the codename suffix
(`5:29.7.2-1~ubuntu.26.04~resolute`, `2.89.0-1~noble`). Three kinds of drift:

- **Single-version vendor repos.** `cli.github.com` (gh), `mise.jdx.dev` (mise) and Aqua's
  `trivy-repo` keep exactly one version in their index, so every upstream release deletes the pin.
- **Ubuntu archive SRUs.** A superseded `-updates`/`-security` version disappears from the index
  (dotnet-sdk-10.0, ruby3.3, procs, and any other archive pin).
- **`apt upgrade` on a target** moves the installed version off the pin; the next run reports it.

Resolve all three by advancing the pin, never by reverting the host. Check the candidate with
`apt-cache policy` on a target, or fetch the repository's `Packages` index directly when no target is
handy. Update the var, the header `Usage:` example and any date-stamped "checked YYYY-MM-DD" comment
beside it. Pin bumps are committed as `chore: bump the pinned versions for <tools>` with a
`old -> new` table in the body (see `git log --grep 'bump the pinned'`).

## Commit messages

Conventional-commit subjects in lowercase imperative: `feat: add a <tool> playbook to <dir>`,
`fix: ...`, `docs: ...`, `refactor: ...`, `chore: ...`. The body explains the finding, not the diff.
