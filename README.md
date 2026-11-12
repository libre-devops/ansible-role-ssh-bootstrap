<div align="center">
  <a href="https://libredevops.org">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://libredevops.org/assets/libre-devops-white.png">
      <img alt="Libre DevOps" src="https://libredevops.org/assets/libre-devops-black.png" width="300">
    </picture>
  </a>
</div>

# Ansible Role: SSH bootstrap

Adds SSH public keys for existing users and hardens `sshd` with a drop-in file, checked with `sshd -t` before `sshd` reloads and with `sshd -T` after, without editing the distribution's `sshd_config`.

[![CI](https://github.com/libre-devops/ansible-role-ssh-bootstrap/actions/workflows/ci.yml/badge.svg)](https://github.com/libre-devops/ansible-role-ssh-bootstrap/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/libre-devops/ansible-role-ssh-bootstrap?sort=semver&label=release)](https://github.com/libre-devops/ansible-role-ssh-bootstrap/releases/latest)
[![Galaxy](https://img.shields.io/badge/galaxy-libre__devops.ssh__bootstrap-EE0000?logo=ansible&logoColor=white)](https://galaxy.ansible.com/ui/standalone/roles/libre_devops/ssh_bootstrap/)
[![Licence](https://img.shields.io/github/license/libre-devops/ansible-role-ssh-bootstrap)](./LICENSE)

---

## What it does

1. Optionally installs the OpenSSH server.
2. For each user in `ssh_bootstrap_authorized_keys`, checks the user exists, then adds (or removes) the given public keys in their `authorized_keys`, from strings or from files on the control node. With `exclusive`, every other key is removed.
3. Writes `ssh_bootstrap_sshd_settings` to `/etc/ssh/sshd_config.d/10-ssh-bootstrap.conf`. If `sshd_config` does not include that directory (RHEL 8), it adds the `Include` line at the top.
4. Checks the drop-in alone with `sshd -t -f`, then the whole configuration with `sshd -t`. If the whole check fails, the previous drop-in comes back and `sshd` is not reloaded, so a bad setting cannot lock anyone out.
5. Reloads `sshd`, then checks with `sshd -T` that each setting is the value `sshd` actually uses.

It does not create users, manage `known_hosts`, or write `Match` blocks.

## Contents

- [Requirements](#requirements)
- [Supported platforms](#supported-platforms)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Role variables](#role-variables)
- [Authorized keys](#authorized-keys)
- [sshd settings](#sshd-settings)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Developing](#developing)
- [Releasing](#releasing)

## Requirements

| What | Detail |
|---|---|
| Control node | `ansible-core` 2.16 or later. |
| Collections | `ansible.posix`, for `authorized_key`. `community.general` on SUSE hosts, only with `ssh_bootstrap_install_server: true`. The `ansible` package includes both. |
| Managed nodes | systemd, OpenSSH 7.3 or later (for `Include`), and a Python that your `ansible-core` supports. RHEL 8 and SLES 15 ship Python 3.6, which ansible-core 2.17 and later cannot use: run [libre_devops.python](https://github.com/libre-devops/ansible-role-python) first, as in [RHEL 8 hosts with only Python 3.6](#rhel-8-hosts-with-only-python-36), or use ansible-core 2.16, which CI tests on RHEL 8's own Python. |
| Privileges | `become: true`. |

## Supported platforms

Each platform below runs in CI on every pull request.

| Distribution | sshd configuration | Notes |
|---|---|---|
| RHEL 8 (UBI) | `/etc/ssh/sshd_config`, no `Include` | The role adds `Include /etc/ssh/sshd_config.d/*.conf` at the top. Tested twice: with Python 3.12 added under current ansible-core, and with only RHEL 8's own Python 3.6 under ansible-core 2.16. |
| RHEL 9 (UBI), Rocky 10 | `/etc/ssh/sshd_config` with `Include` | Red Hat's `50-redhat.conf` sorts after the role's drop-in. |
| Debian 12 and 13 | `/etc/ssh/sshd_config` with `Include` | |
| Ubuntu 24.04 and 26.04 | `/etc/ssh/sshd_config` with `Include` | `sshd` starts on demand through `ssh.socket`. |
| SLES 15 SP7 | `/etc/ssh/sshd_config` with `Include` | |
| SLES 16 | `/usr/etc/ssh/sshd_config` (vendor file) | There is no `/etc/ssh/sshd_config`. The role never creates one, because `sshd` would then ignore the vendor file. |

Other distributions with OpenSSH and systemd should work the same way.

## Installation

From Galaxy:

```bash
ansible-galaxy role install libre_devops.ssh_bootstrap
ansible-galaxy collection install ansible.posix
```

Or pin a release from GitHub, as in [`examples/requirements.yml`](./examples/requirements.yml).

Without Galaxy, or to keep a copy inside your organisation, see [Using the role without Galaxy](./docs/using-without-galaxy.md). It covers git, release archives, requirements files that point at your own servers, AWX and Automation Controller, execution environments, and managed nodes with Python 3.6 such as RHEL 8.

## Quick start

```yaml
- name: Bootstrap SSH access
  hosts: all
  become: true
  roles:
    - role: libre_devops.ssh_bootstrap
      vars:
        ssh_bootstrap_authorized_keys:
          - user: deploy
            key_files:
              - ~/.ssh/id_ed25519.pub
        ssh_bootstrap_sshd_settings:
          PermitEmptyPasswords: false
          PermitRootLogin: false
          PasswordAuthentication: false
```

Add your key before turning password authentication off, in the same run as here, or you may lose the only way in.

## Role variables

Every variable is validated against [`meta/argument_specs.yml`](./meta/argument_specs.yml) when the role starts, including each entry of `ssh_bootstrap_authorized_keys`. Defaults are in [`defaults/main.yml`](./defaults/main.yml).

| Variable | Default | Description |
|---|---|---|
| `ssh_bootstrap_authorized_keys` | `[]` | The keys to manage, one entry per user. See [Authorized keys](#authorized-keys). |
| `ssh_bootstrap_skip_missing_key_files` | `false` | Skip `key_files` missing on the control node instead of failing. |
| `ssh_bootstrap_install_server` | `false` | Install `openssh-server` if missing, and on RHEL and SUSE start and enable `sshd`. |
| `ssh_bootstrap_sshd_settings` | `{PermitEmptyPasswords: false}` | `sshd` settings for the drop-in. `{}` leaves `sshd` alone. See [sshd settings](#sshd-settings). |
| `ssh_bootstrap_sshd_dropin_name` | `10-ssh-bootstrap.conf` | File name of the drop-in in `/etc/ssh/sshd_config.d`. |
| `ssh_bootstrap_sshd_ensure_include` | `true` | Add the `Include` line to the top of `/etc/ssh/sshd_config` when it is missing. |
| `ssh_bootstrap_verify_effective` | `true` | After reloading, check with `sshd -T` that each single-value setting is in effect. |

## Authorized keys

Each entry of `ssh_bootstrap_authorized_keys`:

| Key | Default | Description |
|---|---|---|
| `user` | required | The account. It must exist: the role stops if it does not, rather than create it. |
| `keys` | `[]` | Public keys as strings. |
| `key_files` | `[]` | Paths on the control node to public key files. `~` is expanded. A missing file stops the role, unless `ssh_bootstrap_skip_missing_key_files` is true. |
| `exclusive` | `false` | Remove every key not in this entry from the user's `authorized_keys`. |
| `state` | `present` | `absent` removes the listed keys instead. |

All of a user's keys go into one `authorized_key` call, so `exclusive` keeps exactly the keys in the entry. An entry that ends up with no keys (for example, every file skipped) stops the role, because with `exclusive` it would remove every key the user has.

```yaml
ssh_bootstrap_authorized_keys:
  - user: deploy
    key_files:
      - ~/.ssh/id_ed25519.pub
      - files/keys/ci.pub
  - user: ops
    keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... alice@laptop
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... bob@laptop
    exclusive: true
  - user: deploy
    keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... leaver@laptop
    state: absent
```

## sshd settings

`ssh_bootstrap_sshd_settings` maps `sshd_config` keywords to values. `true` and `false` become `yes` and `no`, a list becomes a space-separated value, and anything else is written as it is:

```yaml
ssh_bootstrap_sshd_settings:
  PermitEmptyPasswords: false        # PermitEmptyPasswords no
  PermitRootLogin: prohibit-password # PermitRootLogin prohibit-password
  MaxAuthTries: 3                    # MaxAuthTries 3
  AllowUsers: [deploy, ops]          # AllowUsers deploy ops
```

The keywords are in the [`sshd_config` manual](https://man.openbsd.org/sshd_config). Check your OpenSSH version supports a keyword before using it: `sshd -t` rejects unknown ones, and the role then puts the previous drop-in back.

### Why a drop-in, and why `10-`

For most keywords `sshd` uses the first value it reads. The distribution's `sshd_config` includes `/etc/ssh/sshd_config.d/*.conf` near the top and reads the files in name order, so `10-ssh-bootstrap.conf` comes before files such as Red Hat's `50-redhat.conf` and cloud-init's `50-cloud-init.conf`. A setting above the `Include` line in `sshd_config`, or in a drop-in with a lower name, still wins. The `sshd -T` check catches that and says which setting is overridden. Rename the drop-in with `ssh_bootstrap_sshd_dropin_name` (for example `00-ssh-bootstrap.conf`) or remove the earlier line.

`sshd -T` prints some keywords (such as `AllowUsers`) one line per value, times in seconds, and a few values under older names (OpenSSH 10.0 and earlier show `PermitRootLogin prohibit-password` as `without-password`). So the check compares single-value settings only, skips times written with a unit such as `1m`, and accepts either name for the aliases it knows.

## Examples

### RHEL 8 hosts with only Python 3.6

RHEL 8 ships Python 3.6, which ansible-core 2.17 and later cannot run modules with, so every task fails before this role even starts (see [Troubleshooting](#troubleshooting)). Put [libre_devops.python](https://github.com/libre-devops/ansible-role-python) in front of this role, in the same play. It installs `python3.12` from AppStream, through your Satellite or mirror like any other package, points Ansible at it and gathers facts, and then this role runs as usual.

Install both, for example from GitHub:

```yaml
# requirements.yml
roles:
  - name: libre_devops.python
    src: https://github.com/libre-devops/ansible-role-python.git
    scm: git
    version: 1.0.0
  - name: libre_devops.ssh_bootstrap
    src: https://github.com/libre-devops/ansible-role-ssh-bootstrap.git
    scm: git
    version: 1.0.0
```

Then chain them:

```yaml
- name: Bootstrap SSH access on RHEL 8
  hosts: rhel8
  become: true
  # Gathering facts would fail on Python 3.6. libre_devops.python gathers them once it has
  # installed Python 3.12, before libre_devops.ssh_bootstrap needs them.
  gather_facts: false
  roles:
    - role: libre_devops.python
    - role: libre_devops.ssh_bootstrap
      vars:
        ssh_bootstrap_authorized_keys:
          - user: deploy
            key_files:
              - ~/.ssh/id_ed25519.pub
        ssh_bootstrap_sshd_settings:
          PermitEmptyPasswords: false
          PermitRootLogin: false
          PasswordAuthentication: false
```

Every later play in the same run uses the same Python. To skip `libre_devops.python` on later runs, set `ansible_python_interpreter: /usr/bin/python3.12` in `group_vars/rhel8.yml` once the hosts have it. For hosts with no repositories, `libre_devops.python` can push the packages from the control node instead (`python_bootstrap_source: files`), and SLES 15 gets `python311` the same way. See its README for every option.

### A typical hardening set

```yaml
ssh_bootstrap_sshd_settings:
  PermitEmptyPasswords: false
  PermitRootLogin: false
  PasswordAuthentication: false
  KbdInteractiveAuthentication: false
  X11Forwarding: false
  MaxAuthTries: 3
  LoginGraceTime: 30
  ClientAliveInterval: 300
  ClientAliveCountMax: 2
```

On OpenSSH older than 8.7, use `ChallengeResponseAuthentication` instead of `KbdInteractiveAuthentication`.

### Keys only, leave sshd alone

```yaml
ssh_bootstrap_sshd_settings: {}
```

### Lab machines where the key may not exist

```yaml
ssh_bootstrap_skip_missing_key_files: true
ssh_bootstrap_authorized_keys:
  - user: root
    key_files:
      - ~/.ssh/id_ed25519.pub
      - ~/.ssh/id_rsa.pub
```

## Troubleshooting

| Symptom | What to check |
|---|---|
| `The module interpreter '/usr/bin/python3' was not found` or `Module result deserialization failed`, on the first task | The host only has Python 3.6 (RHEL 8, SLES 15), which ansible-core 2.17 and later cannot use. Every module fails, not only this role's. Run [libre_devops.python](https://github.com/libre-devops/ansible-role-python) first in the play, as in [RHEL 8 hosts with only Python 3.6](#rhel-8-hosts-with-only-python-36). |
| `User ... does not exist` | The role only adds keys. Create the user first, for example with `ansible.builtin.user`. |
| `... does not exist on the control node` | The key file path is relative to where you run `ansible-playbook`, or `~` is a different home on the control node. |
| `sshd -t rejected the configuration` | A keyword or value is wrong for this OpenSSH version. The message has `sshd`'s own error. The previous drop-in is back and `sshd` was not reloaded. |
| `sshd is using '...', not '...'` | Something read before the drop-in sets the same keyword. See [Why a drop-in, and why `10-`](#why-a-drop-in-and-why-10-). |
| `sshd's configuration does not include /etc/ssh/sshd_config.d/*.conf` | `ssh_bootstrap_sshd_ensure_include` is false, or the vendor file in `/usr/etc/ssh` has no `Include`. Add the line yourself. |
| Ansible fails on RHEL 8 or SLES 15 with a Python version error | The host is on Python 3.6. Install `python3.12` (RHEL 8) or `python311` (SLES 15) and set `ansible_python_interpreter`. |

## Developing

You need [uv](https://docs.astral.sh/uv/), [just](https://github.com/casey/just) and [podman](https://podman.io/). CI runs the same recipes.

| Recipe | What it does |
|---|---|
| `just deps` | Installs the pinned Python tools (`uv.lock`) and the collections in `requirements.yml` into `.ansible/`. |
| `just lint` | `yamllint`, `ansible-lint` on the production profile, and `zizmor` on the workflows. |
| `just test [scenario] [platform]` | Runs a Molecule scenario on one platform, or on all of them. `just test default rhel8`. |
| `just test-python36` | The `python36` scenario: RHEL 8 with only its own Python 3.6, under ansible-core 2.16 from its own locked tool set in `.venv-python36`. |
| `just test-all` | Every scenario on every platform, one after another, then `test-python36`. |
| `just converge <scenario> <platform>` | Runs a scenario on one platform and leaves the container running, pass or fail. `just login` opens a shell in it, `just destroy` removes it. |
| `just live <inventory> [args]` | Runs [`examples/playbook.yml`](./examples/playbook.yml) against real hosts. |
| `just secrets` | Scans the git history with gitleaks. |
| `just platforms`, `just scenarios` | List what can be tested. |

The `default` scenario creates two users, gives each a key the role did not add, then runs the role: one user's entry adds a key from a file and keeps the existing key, the other's is exclusive and removes it. It writes a hardened drop-in, checks the settings with `sshd -T`, checks the `Include` line, and then runs again to prove nothing changes. The keys in `molecule/fixtures` are public test keys whose private halves were thrown away.

The `python36` scenario runs the same playbooks on RHEL 8 with only the Python it ships, 3.6, which ansible-core 2.17 and later cannot use. It runs under ansible-core 2.16, the `python36` dependency group in `pyproject.toml`, and its verify step fails unless the host really used Python 3.6.

## Releasing

1. Move the Unreleased entries in [`CHANGELOG.md`](./CHANGELOG.md) under the new version.
2. Run `just release 1.2.3`. It tags the commit and pushes the tag.
3. The Release workflow creates the GitHub release, imports the tag into Galaxy, which takes the version from the tag, and installs it back from Galaxy to check. It needs the `GALAXY_API_KEY` secret, which the libre-devops organisation provides. The import goes through Galaxy's API with [`scripts/galaxy-import.py`](./scripts/galaxy-import.py), not `ansible-galaxy role import`, because Galaxy namespaces cannot contain a hyphen: the role namespace is `libre_devops` and the GitHub organisation is `libre-devops`, which that command cannot handle. If the import fails, fix the cause and run the Release workflow by hand with the tag to import it again.

## Licence

[MIT](./LICENSE).
