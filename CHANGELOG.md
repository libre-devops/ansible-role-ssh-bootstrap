# Changelog

Notable changes to this role. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow [Semantic Versioning](https://semver.org/).

## Unreleased

### Added

- `docs/using-without-galaxy.md`, linked from the README's Installation section: installing the role without Galaxy (git, release archives, requirements files that name their own source), getting its collections with no Galaxy access, keeping a copy on your own git server or artifact store, AWX, Automation Controller and execution environments, and the ansible-core and Python versions it needs.
- A `python36` Molecule scenario and CI job: RHEL 8 with only its own Python 3.6, under ansible-core 2.16 from a separately locked tool set (`just test-python36`). It runs the default scenario's playbooks and checks the host really used Python 3.6.

### Changed

- `docs/using-without-galaxy.md` recommends `libre_devops.python` for managed nodes with only Python 3.6, ahead of a hand-written `raw` play and ansible-core 2.16.
- The README's Requirements and Troubleshooting explain managed nodes with only Python 3.6 (RHEL 8, SLES 15), which ansible-core 2.17 and later cannot manage, and link to the two ways round it.
- The Release workflow imports the release's own tag into Galaxy, rather than the default branch, and can be run by hand with a tag to import that tag again.

### Fixed

- `just converge` failed, because `molecule converge` has no `--platform-name`. It now runs the scenario on the one platform and keeps the container.

## 1.0.0 - 2026-11-11

### Added

- First release, split out of `libre-devops/mde-linux-rhel-ansible-role`.
- Public keys for existing users from strings or files on the control node, with `exclusive` and `state: absent`, one `authorized_key` call per user.
- `sshd` settings in a drop-in in `/etc/ssh/sshd_config.d`, checked with `sshd -t` alone and as part of the whole configuration, with the previous drop-in restored if the check fails.
- The `Include` line added to `sshd_config` where it is missing (RHEL 8), and the vendor configuration on SLES 16 left alone.
- A check with `sshd -T` that each single-value setting is in effect.
- Molecule scenario on RHEL 8 and 9, Rocky 10, Debian 12 and 13, Ubuntu 24.04 and 26.04, and SLES 15 and 16.

### Changed from the RHEL-only MDE role

- A missing key file now stops the role, unless `ssh_bootstrap_skip_missing_key_files` is true. The old tasks checked for the files on the managed node instead of the control node.
- The package update task is gone: updating packages is not part of SSH bootstrap.
