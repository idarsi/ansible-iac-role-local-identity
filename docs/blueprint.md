# Blueprint reference

The public input is the mapping `iac_blueprint.local_identity`. `users` is
required, including for an empty request; `groups` defaults to an empty list.
Omitted fields use the defaults below. This reference describes the input
contract implemented by the role; module defaults remain Ansible/platform
dependent where noted.

## Top-level fields

| Field | Required | Default | Contract |
|---|---:|---|---|
| `state` | No | `present` | Exactly `validate`, `present`, `absent`, or `all_absent`. |
| `users` | Yes | — | A list of unique user records. |
| `groups` | No | `[]` | A list of unique declared local groups; created before users in `present`. |
| `allow_user_deletion` | No | `false` | Authorizes user deletion for non-empty `absent`/`all_absent` requests. |
| `allow_group_deletion` | No | `false` | Required to remove an existing declared group; see [removal safeguards](security-and-removal.md). |
| `allow_home_removal` | No | `false` | Required if any record requests `remove_home: true`. |
| `allow_all_absent` | No | `false` | Always required for `all_absent`, including empty/group-only requests. |
| `home_removal_marker` | No | `.ansible-iac-role-local-identity-managed` | Safe single filename used to identify a role-managed home. |
| `authorized_keys_exclusive` | No | `false` | Default `authorized_keys_exclusive` value for users. |
| `sudoers_dir` | No | `/etc/sudoers.d` | Safe absolute directory for per-user sudoers files. |
| `sudoers_mode` | No | `0440` | Sudoers file mode; only `0400` and `0440` are accepted. |
| `require_local_nss` | No | `true` | Deprecated compatibility field; ignored. Local-files NSS enforcement is unconditional. |

## User fields

| Field | Required | Default | Contract |
|---|---:|---|---|
| `name` | Yes | — | Unique Linux-safe name, 1–32 characters, lowercase letter or `_` first; `root` is forbidden. |
| `groups` | No | `[]` | Supplementary group names; each must resolve locally or be declared in this blueprint. |
| `shell` | No | `/bin/bash` | Non-empty login shell path/value. |
| `home` | No | Module/system default | Safe absolute home path; required when `remove_home: true`. |
| `comment` | No | Module default | User comment/gecos value. |
| `uid` | No | Module/system allocation | Unique positive integer, at most `2147483647`. |
| `gid` | No | — | Positive primary GID from `1` through `2147483647`; existing matching group is reused, otherwise a private group named for the user is created. |
| `create_home` | No | `true` | Whether to create a home directory. |
| `remove_home` | No | `false` | Request removal of this explicit home during a destructive state; additional authorization and ownership checks apply. |
| `append` | No | `true` for non-empty groups; otherwise `false` | Whether supplementary groups are appended or replaced. An explicitly empty group list clears membership by default. |
| `password_lock` | No | Module default | Boolean passed to the user module. |
| `allow_uid_gid_change` | No | `false` | Explicitly permits requested UID/primary-GID changes for an existing account. |
| `authorized_keys` | No | `[]` | Single-line OpenSSH public keys of supported types; key options and certificates are not accepted. |
| `authorized_keys_exclusive` | No | Top-level value (`false`) | Whether the declared keys exclusively control the file; an empty exclusive list clears it. |
| `allow_unrestricted_sudo` | No | `false` | Allows unrestricted sudo command specifications containing `ALL`. |
| `sudo_rules` | No | `[]` | Single-line sudoers rules, each beginning with this user's name. |

## Group fields

| Field | Required | Default | Contract |
|---|---:|---|---|
| `name` | Yes | — | Unique Linux-safe group name, 1–32 characters, lowercase letter or `_` first. |
| `gid` | No | Module/system allocation | Unique positive integer from `1` through `2147483647`. |
| `system` | No | `false` | Boolean system-group request passed to Ansible. |
| `allow_gid_change` | No | `false` | Explicitly permits changing a declared existing group's GID. |

## Validation contract

- The role requires a mapping at `iac_blueprint.local_identity`, valid mapping
  and list types, and the required `users` list. User/group names and declared
  UIDs, GIDs, and explicit homes must be unique where applicable. Boolean
  switches are type-checked; numeric IDs must be positive integers, not
  booleans.
- Names follow `^[a-z_][a-z0-9_-]{0,31}$`; user `root` is rejected. Explicit
  home and sudoers paths must meet safe absolute-path checks. A `remove_home`
  request requires an explicit `home`.
- Supplementary groups, authorized keys, and sudo rules must contain
  single-line strings. Keys must be plain supported OpenSSH public-key lines.
  Each sudo rule must begin with the associated username. `ALL` command
  specifications require that user's `allow_unrestricted_sudo: true`; `#0`
  run-as specifications are rejected.
- Remote-only identities and groups are rejected. Supplementary groups must be
  in local `/etc/group` or declared in this blueprint. `require_local_nss` is
  ignored for compatibility.
- Every state requires gathered facts identifying Ubuntu 22 or EL 9/10 (the
  role maps Ansible distributions `RedHat` and `Rocky` to `EL`). `validate`
  validates the full blueprint and sudo-rule policy, but skips operational
  runtime preflights such as NSS, filesystem path, and `visudo` checks and makes
  no persistent host changes. `present` performs its operational preflights
  before mutations.
- Destructive-state authorization, ownership and membership guards are detailed
  in [Security and removal](security-and-removal.md).
