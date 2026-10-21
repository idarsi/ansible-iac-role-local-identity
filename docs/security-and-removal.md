# Security and removal safeguards

## Security behavior

- Sudoers files are root-owned, default to mode `0440` (only `0400` and `0440`
  are accepted), and are validated with `visudo`. A same-named unmanaged file
  is not overwritten. `visudo` is needed on the target only for `present` when
  users declare sudo rules; `validate` does not invoke it.
- Sudo rules must belong to the named user. Command specifications containing
  `ALL` need per-user `allow_unrestricted_sudo: true`; `#0` run-as rules are
  rejected. Public keys only are accepted; do not put private keys or passwords
  in this blueprint. Use Ansible Vault or an approved secret manager for other
  secret data.
- Authorized-key exclusivity defaults off. An empty key list leaves an existing
  file unchanged unless exclusivity is enabled. SSH authorized-key and sudoers
  paths reject symlinked parent components; arbitrary path aliasing is not
  supported.
- Existing homes must be directories. The role marks a home only when it
  creates it for a user with an explicitly declared `home`; it never marks an
  existing home automatically.

## Destructive states

> **Warning:** `absent` and `all_absent` delete declared local accounts and
> role-managed sudoers files. Review the target and inventory before enabling
> deletion. Run only against the intended hosts.

`absent` requires `allow_user_deletion: true` when users are declared.
`all_absent` always requires `allow_all_absent: true` (even for an empty or
group-only request), and requires `allow_user_deletion: true` when users are
declared. `all_absent` operates only on resources declared in the Blueprint; it
does not discover or remove other system users.

Declared groups are never removed implicitly. An existing declared group
requires `allow_group_deletion: true`, a declared `gid` matching its current
GID, and safe membership: every current member must also be declared for
removal in the same operation, and no undeclared user may use that GID as its
primary group. Missing groups are ignored without the opt-in. Shared groups,
mismatched or omitted GIDs, and groups with undeclared ownership are preserved;
the operation fails before user mutation. Groups created with an automatically
allocated GID are therefore present-only unless that GID is later declared.

Home removal requires all of `allow_home_removal: true`, `remove_home: true`,
and an explicit `home`. Before deleting, the role verifies the home matches the
user's passwd entry and is a non-symlink, user-owned directory containing the
exact configured marker. Unmanaged homes are preserved and removal fails.
Role-managed sudoers files can be removed; unmanaged or same-named foreign
files are preserved and cause failure rather than being replaced or deleted.

## Recovery and operational limitations

The role does not promise transactional rollback after an operational preflight
failure or a later module failure. Correct the reported issue (for example,
provide `visudo`, fix a rejected path, or reconcile identity ownership), review
what has already changed, and rerun the intended Blueprint to converge. Do not
weaken deletion or ownership checks merely to force completion. Back up
important account/group and home data before authorized destructive operations.

For scenario coverage and safe disposable-container testing, see
[TESTING.md](../TESTING.md). The executable inventories are
[minimal](inventory-minimal.yml), [security](inventory-security.yml), and
[removal](inventory-removal.yml).
