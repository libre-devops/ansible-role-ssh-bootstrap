# Security policy

## Reporting a vulnerability

Report vulnerabilities in this role privately, through [GitHub's private vulnerability reporting](https://github.com/libre-devops/ansible-role-ssh-bootstrap/security/advisories/new). Do not open a public issue.

## How the role avoids locking you out

- `sshd` is only reloaded after the drop-in passes `sshd -t` on its own and the whole configuration passes `sshd -t`. If the second check fails, the previous drop-in is put back.
- An entry in `ssh_bootstrap_authorized_keys` with no keys stops the role, because with `exclusive` it would remove every key.
- Add your key in the same run that turns password authentication off, or earlier.

Only public keys go through this role. Never put a private key in a variable, a key file path or this repository. `molecule/fixtures` holds public test keys whose private halves were deleted.
