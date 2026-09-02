# ansible-iac-role-local-identity

## Overview

Declaratively manages local Linux users and selected access controls. The role
manages users, home directories, SSH authorized keys, and per-user sudoers
files; it does not install sudo. Declared supplementary groups are assigned to
users but are not created by the role and must already exist.

The public input is an `iac_blueprint`. The role validates the complete
blueprint, normalizes it, and then applies the requested state.

> **Maturity level: Beta**  
> The public contract has validation and destructive-operation guardrails, and
> automated functional coverage includes RHEL UBI 9/10, Rocky Linux 9/10, and
> Ubuntu 22.04. Review the
> supported-platform and limitations sections before production use.

## Supported platforms

Operating system | Supported versions | Tested versions
-----------------|--------------------|----------------
Debian           | 11, 12             | —
Ubuntu           | 22, 24             | 22.04
Enterprise Linux (`RedHat`, `Rocky`) | 8, 9, 10 | RHEL UBI 9/10, Rocky Linux 9/10

The role supports Linux hosts only. Platform support is defined by
`iac_supported_os` in [defaults/main.yml](defaults/main.yml). For the `EL`
support record, the role normalizes Ansible distributions `RedHat` and `Rocky`
to `EL`; other distributions are not accepted through this alias. Other
supported platforms are not automatically tested by this repository. RHEL UBI
images are public UBI images, not subscribed RHEL installations; access to
`registry.access.redhat.com` is required and no registry credentials are
supplied by this role or CI.

## Supported states

State | Behavior
------|---------
`validate` | Validate inventory and sudoers content without persistent host changes.
`present` | Create or update the declared users and access configuration.
`absent` | Remove the declared users and role-managed sudoers files; homes are preserved unless explicitly requested and authorized.
`all_absent` | Perform the same declared-user removal as `absent`, with an additional explicit opt-in.

`install` and `uninstall` are not supported states. `all_absent` applies only
to users listed in the blueprint; it does not discover or remove other system
users.

## Quick start

Minimal playbook:

```yaml
---
- hosts: local_identity_hosts
  become: true
  roles:
    - role: local_identity
  vars:
    iac_blueprint:
      local_identity:
        state: present
        users:
          - name: app
            shell: /usr/sbin/nologin
```

Validate the same blueprint without applying a mutating state:

```yaml
---
- hosts: local_identity_hosts
  become: true
  roles:
    - role: local_identity
  vars:
    iac_blueprint:
      local_identity:
        state: validate
        users:
          - name: app
```

## Requirements

- Ansible Core 2.14 or newer.
- `ansible.posix` collection 2.2.2 (for `authorized_key`).
- `containers.podman` collection 1.8.1 is pinned in [requirements.yml](requirements.yml) for the repository's Podman-based test scenarios.
- `visudo` on the target when any user has `sudo_rules`.

Install the declared collections with `ansible-galaxy collection install -r
requirements.yml`.

## Usage

Keep the playbook thin; configuration belongs under `iac_blueprint.local_identity`:

```yaml
---
- hosts: local_identity_hosts
  become: true
  roles:
    - role: local_identity
```

Run it with `ansible-playbook -i inventory.yml site.yml`.

## Blueprint structure

```yaml
iac_blueprint:
  local_identity:
    state: present
    users:
      - name: deploy
        groups: [adm]
        shell: /bin/bash
        home: /home/deploy
        comment: Deployment account
        uid: 1500
        gid: 1500
        password_lock: true
        authorized_keys:
          - ssh-ed25519 AAAA... replace-with-a-real-public-key
        sudo_rules:
          - 'deploy ALL=(root) /usr/bin/systemctl restart example.service'
```

`users` is required and must be a list. User names cannot be `root`, must be
Linux-safe names of up to 32 characters, and must be unique. Values in
`groups`, `authorized_keys`, and `sudo_rules` must be single-line strings.

### Blueprint fields

Top-level field | Required | Default | Description
----------------|----------|---------|-------------
`state` | No | `present` | One of `validate`, `present`, `absent`, or `all_absent`.
`users` | Yes | — | Users to manage or remove.
`allow_user_deletion` | No | `false` | Required for `absent` and `all_absent`.
`allow_home_removal` | No | `false` | Required when any user sets `remove_home: true`.
`allow_all_absent` | No | `false` | Required for `all_absent`.
`home_removal_marker` | No | `.ansible-iac-role-local-identity-managed` | Safe filename used to prove role-managed home ownership.
`authorized_keys_exclusive` | No | `false` | Default for users; when true, declared keys replace existing keys, including clearing them with an empty list.
`sudoers_dir` | No | `/etc/sudoers.d` | Safe absolute directory for per-user sudoers files.
`sudoers_mode` | No | `0440` | Sudoers file mode; only `0400` and `0440` are accepted.

Each user supports these fields:

Field | Required | Default | Description
------|----------|---------|-------------
`name` | Yes | — | Local username.
`groups` | No | `[]` | Supplementary groups; `append` controls replacement when supplied.
`shell` | No | `/bin/bash` | Login shell.
`home` | No | Module/system default | Explicit home path; required for `remove_home`.
`comment` | No | Module default | User comment/gecos value.
`uid` | No | Module/system allocation | Positive numeric UID.
`gid` | No | — | Numeric primary GID; an existing matching group is reused, otherwise a private group named after the user is created.
`create_home` | No | `true` | Whether to create the home directory.
`remove_home` | No | `false` | Request removal of the explicitly declared home during a destructive state.
`append` | No | `true` when groups are declared | Whether supplementary groups are appended or replaced.
`password_lock` | No | Module default | Pass-through user password-lock setting.
`authorized_keys` | No | `[]` | SSH public keys to manage.
`authorized_keys_exclusive` | No | Top-level value (`false`) | Whether declared keys exclusively control the file.
`allow_unrestricted_sudo` | No | `false` | Explicitly permits unrestricted sudo command specifications containing `ALL`.
`sudo_rules` | No | `[]` | Single-line sudoers entries beginning with this user name.

## Configuration examples

See the executable examples: [minimal](docs/inventory-minimal.yml),
[security](docs/inventory-security.yml), and
[removal](docs/inventory-removal.yml).

## Security defaults

- Sudoers files are root-owned, use mode `0440` by default, are validated with
  `visudo`, and are never overwritten when a same-named unmanaged file exists.
- Broad sudo command specifications containing `ALL` require the per-user
  `allow_unrestricted_sudo: true` opt-in.
- Authorized-key exclusivity is disabled by default; an empty key list therefore
  leaves an existing file unchanged unless exclusivity is enabled.
- User removal uses `remove: false`; existing homes are not removed by default.
- Homes created by this role receive a marker with mode `0600` only when an
  explicit `home` is declared. Existing homes are never marked automatically.

## Secrets

Do not commit passwords or private keys. Use Ansible Vault or an approved secret
manager for any surrounding secret data. This role accepts public SSH keys only;
it does not provide a password field in the blueprint.

## Destructive states and guardrails

> **Warning:** `absent` and `all_absent` delete declared local users and their
> role-managed sudoers files. Confirm the target before enabling deletion.

`absent` requires `allow_user_deletion: true`; `all_absent` additionally
requires `allow_all_absent: true`. Home removal additionally requires
`allow_home_removal: true`, an explicit `home`, and `remove_home: true`.
Before removal, the home must match the user's passwd entry and be a
non-symlink, user-owned directory containing the exact marker created by this
role. Unmanaged or same-named foreign sudoers files are preserved and cause the
operation to fail.

## Validation

All inventory validation runs before host mutation. `state: validate` also
checks rendered sudoers content without writing a persistent sudoers file.
Validation covers types, names, duplicate users/UIDs/homes, safe paths, GID
values, sudo-rule ownership, and deletion guardrails.

## Testing

Testing details and prerequisites are in [TESTING.md](TESTING.md). The repository
has validation, baseline/idempotence, lifecycle, `all_absent`, and guardrail
scenarios. CI runs syntax checking, production-profile Ansible Lint, and these
Podman-based scenarios on RHEL UBI 9/10, Rocky Linux 9/10, and Ubuntu 22.04.

## Development

See [CONTRIBUTING.md](CONTRIBUTING.md). Changes to the public blueprint must
update validation, normalization, documentation, examples, and tests together.

## Known limitations

- Functional Molecule coverage does not yet exercise Debian, Ubuntu 24.04, or EL 8.
- UBI and Rocky image tags are mutable and registry availability or rate limits
  may block CI. A restricted UBI registry may require an operator-provided login.
- The role manages local identities only; it does not install sudo or manage
  authentication services, password values, firewall policy, or SSH daemon
  configuration.
- Check-mode behavior follows the capabilities and limitations of the Ansible
  user, group, file, and authorized-key modules.

## License

MIT License. See [LICENSE](LICENSE). The role metadata declares the MIT license.
