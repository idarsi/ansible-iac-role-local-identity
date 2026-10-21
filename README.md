> **Maturity State: Release Candidate**<br>
> **RC Readiness: 100%**<br>
> **Assessment ref:** commit [`3327d8e644b7a45820e27f9839a3dcfc8eb923c1`](https://github.com/idarsi/ansible-iac-role-local-identity/commit/3327d8e644b7a45820e27f9839a3dcfc8eb923c1)

ANSIBLE-IAC-ROLE-LOCAL-IDENTITY
===============================
**COPYRIGHT** 2026 ^(ida|arsi)$ collective
**LICENSE** MIT Licence [LICENSE](LICENSE)
**AUTHORS**
- Arsi Atomi <arsi@atomi.sh>

Overview
--------

This Ansible role provides a declarative way to manage local Linux users and
their selected access controls through an `iac_blueprint`.

This maturity assessment is engineering guidance only; it is not production
approval.

The role manages:

- Local users, home directories, and supplementary group membership.
- Declared local groups.
- SSH authorized keys and per-user sudoers files.

The role does not manage:

- Sudo installation, authentication services, password values, or remote identities.
- SSH daemon configuration, firewall policy, or host-wide system administration.

Supported platforms
-------------------

| Operating system | Supported versions | Automated test coverage |
|------------------|--------------------|-------------------------|
| Ubuntu | 22 | Ubuntu 22.04 |
| Enterprise Linux (`RedHat`, `Rocky`, normalized to `EL`) | 9, 10 | RHEL UBI 9/10; Rocky Linux 9/10 |

The configured CI matrix runs five scenarios on each of the five images above.
CI run [39567526243](https://github.com/idarsi/ansible-iac-role-local-identity/actions/runs/39567526243)
for the assessed code/test reference passed all 27 jobs: 25 Molecule matrix
jobs, lint/syntax, and the aggregate Required checks. See [TESTING.md](TESTING.md)
for details. UBI images require access to the public Red Hat registry; the role
and CI do not provide registry credentials.

Supported states
----------------

| State | Behaviour |
|-------|-----------|
| `validate` | Validates the blueprint and sudo-rule policy without operational runtime preflights or persistent host changes; supported-platform facts are required. |
| `present` | Creates or updates declared users, groups, keys, and sudoers configuration. |
| `absent` | Removes declared users and role-managed sudoers files; groups and homes are removed only under their explicit safeguards. |
| `all_absent` | Declared-resource removal with an additional mandatory opt-in; it discovers and removes no undeclared users. |

`install` and `uninstall` are not supported states.

Quick start
-----------

See the [minimal Blueprint example](docs/inventory-minimal.yml) and the
[Blueprint reference](docs/blueprint.md).

Requirements
------------

Control node

- Ansible Core 2.14 or newer.
- `ansible.posix` 2.2.2 (authorized-key management).
- `containers.podman` 1.8.1 is pinned for repository test scenarios, not role runtime.

Target host

- `visudo` is required only for `state: present` when users declare sudo rules,
  for operational sudoers preflight and validation. It is not required by
  `state: validate`.

Blueprint structure
-------------------

Configuration is read from `iac_blueprint.local_identity`:

```yaml
iac_blueprint:
  local_identity:
    state: present
    users:
      - name: app
        shell: /usr/sbin/nologin
```

Top-level blueprint variables
-----------------------------

| Variable | Required (yes/no) | Default | Short description |
|----------|-------------------|---------|-------------------|
| `state` | no | `present` | One of `validate`, `present`, `absent`, `all_absent`. |
| `users` | yes | — | User records to manage or remove. |
| `groups` | no | `[]` | Local group records, created before users. |
| `allow_user_deletion` | no | `false` | Authorizes deletion in non-empty removal requests. |
| `allow_group_deletion` | no | `false` | Required to remove an existing declared group. |
| `allow_home_removal` | no | `false` | Required when a user requests home removal. |
| `allow_all_absent` | no | `false` | Required for every `all_absent` request. |
| `home_removal_marker` | no | `.ansible-iac-role-local-identity-managed` | Marker identifying a role-managed home. |
| `authorized_keys_exclusive` | no | `false` | Default key-file exclusivity for users. |
| `sudoers_dir` | no | `/etc/sudoers.d` | Directory for per-user sudoers files. |
| `sudoers_mode` | no | `0440` | Sudoers file mode (`0400` or `0440`). |
| `require_local_nss` | no | `true` | Deprecated, ignored compatibility field; local-files NSS enforcement is unconditional. |

See [the full Blueprint field reference and validation contract](docs/blueprint.md)
for all top-level, user, and group fields.

Configuration examples
----------------------

- [Minimal inventory](docs/inventory-minimal.yml)
- [Security inventory](docs/inventory-security.yml)
- [Removal inventory](docs/inventory-removal.yml)

For destructive-state safeguards and recovery guidance, see
[Security and removal](docs/security-and-removal.md). Test prerequisites and
commands are documented in [TESTING.md](TESTING.md). Development and
contribution guidance is in [CONTRIBUTING.md](CONTRIBUTING.md). The project is
licensed under the [MIT Licence](LICENSE).
