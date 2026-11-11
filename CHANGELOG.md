# Changelog

Notable changes to this role. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow [Semantic Versioning](https://semver.org/).

## Unreleased

### Fixed

- Galaxy import: call Galaxy's import API with the `libre_devops` namespace and the GitHub clone URL, because `ansible-galaxy role import` sends the GitHub organisation name, which Galaxy refuses with HTTP 500. The import now waits for the result and prints Galaxy's log, and the Release workflow can be run by hand to import a tag again.

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
