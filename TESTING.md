# Testing

All role convergence and verification runs target disposable Linux containers
created by Molecule. Do not converge this role against localhost or the current
host. Create and destroy playbooks use `hosts: localhost` only for Podman
control.

## Automated Molecule Matrix

Every scenario is configured to run against each image in the CI platform
matrix; this describes configured CI coverage, not successful local or current
run evidence. There is no application-version dimension for this role.

| Platform/image | Ansible or application versions | Molecule scenarios | Main coverage |
| --- | --- | --- | --- |
| RHEL UBI 9 / `registry.access.redhat.com/ubi9/ubi-init:latest` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Configured primary Enterprise Linux coverage |
| RHEL UBI 10 / `registry.access.redhat.com/ubi10/ubi-init:latest` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Configured primary Enterprise Linux coverage |
| Rocky Linux 9 / `rockylinux:9` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Configured Enterprise Linux compatibility coverage |
| Rocky Linux 10 / `quay.io/rockylinux/rockylinux:10` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Configured Enterprise Linux compatibility coverage |
| Ubuntu 22.04 / `geerlingguy/docker-ubuntu2204-ansible:latest` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Configured retained compatibility coverage; locally completed on this image |

Images are pulled and run directly by Podman. Tags are not pinned by digest.
UBI images are public but access to Red Hat's registry can be restricted; CI
does not invent or provide credentials. Rocky images test the compatible EL
implementation, not a Red Hat subscription.

## Scenario coverage

- `validation`: Runs `state: validate` with empty users and groups, a minimal valid
  user, numeric UID/GID values, and valid unrestricted sudo opt-in. It asserts
  actionable failure messages for invalid numeric values, invalid user and
  top-level groups,
  multiline and malformed public keys, unsafe sudo syntax, and sudo variants. Prepare and verify
  prove that validation does not create an account or sudoers file.
- `baseline`: Runs normal `state: present` convergence and Molecule's
  idempotence check. It verifies user creation, sudoers rendering, home marker
  ownership/content, preservation of an existing home marker state, numeric
  primary-group creation and reuse, declared group creation before supplementary
  membership, and a valid present-state check-mode run.
- `lifecycle`: Creates a user, transitions it to an updated `present` state,
  and then exercises `state: absent` with explicit user/group deletion and
  home-removal authorization. It verifies the shell transition and that the
  declared user, explicitly identified group, and marked home are removed.
- `all_absent`: Exercises `state: all_absent` with all destructive opt-ins and
  a role-managed home marker. It verifies removal of only the declared user,
  declared group, home, and sudoers file while preserving an unmanaged sudoers
  file and an undeclared home.
- `guardrails`: Verifies destructive check mode is non-mutating and that
  user deletion, group-only absent/all_absent requests, nonexistent-group
  handling, group deletion without opt-in, shared-group removal, undeclared
  primary-group ownership, omitted or mismatched GIDs, `all_absent`, unmanaged
  home removal, unmanaged or unsafe-metadata sudoers updates, and conflicting
  same-name GIDs fail with expected error messages.
  Final assertions verify unmanaged resources remain present.
- Functional scenarios also cover UID/GID drift and collisions, empty
  supplementary-group clearing (including replacement by an explicitly empty
  list), local-files NSS isolation and remote-only NSS rejection, implicit
  module-default homes and existing passwd homes, non-directory homes, and
  symlink-safe SSH, sudoers-parent, and home path handling.

All five scenarios are configured for every image listed in the matrix. No scenario currently
covers Debian 11/12, Ubuntu 24,
multiple application versions, or multi-host behavior.

## Prerequisites

- Linux host with Podman and permission to run disposable containers.
- Shared Idarsi Ansible testing environment at
  `/home/arsi/.local/share/venvs/idarsi-ansible-testing`.
- Network access to install the pinned collections and pull the test image.
- `ansible.posix==2.2.2` and `containers.podman==1.8.1` from `requirements.yml`.
- `visudo` in the test image for sudoers verification. The test-only Molecule
  create step installs Python 3 and sudo before Ansible connects; production
  role behavior is unchanged.

Molecule uses the Podman driver. Docker and a Docker daemon are not required.
Rootless Podman normally runs without a socket. If tooling explicitly requires
the API socket, enable it with:

```sh
systemctl --user enable --now podman.socket
export CONTAINER_HOST="unix:///run/user/$(id -u)/podman/podman.sock"
```

## Commands

Run the complete local check set with the shared environment:

```sh
export PATH="/home/arsi/.local/share/venvs/idarsi-ansible-testing/bin:$PATH"
ansible-galaxy collection install -r requirements.yml
ansible-playbook --syntax-check tests/syntax.yml
ansible-lint --profile production .
molecule test -s validation
molecule test -s baseline
molecule test -s lifecycle
molecule test -s all_absent
molecule test -s guardrails
```

Run an individual scenario with `molecule test -s <scenario>`, for example
`molecule test -s validation`. Destructive scenarios must run only in their
disposable Molecule containers.

## Completed Verification Evidence

Tester-reported verification records all 25/25 Molecule matrix combinations
passing: five scenarios—`validation`, `baseline`, `lifecycle`, `guardrails`,
and `all_absent`—across the five images listed above. The reported branch is
`feature/rc-readiness`.

No exact commit SHA or CI run artifact is recorded yet. This evidence therefore
does not claim a traceable CI run or replace the need for release/version
migration metadata before RC.

The runs emitted non-blocking warnings about a duplicate collection,
Ansible/Molecule deprecations, and missing optional Molecule files. None of
these warnings caused a scenario failure. The warnings do not expand the
documented platform or version coverage.

## CI Execution

The GitHub Actions workflow runs on every push and pull request. It executes
syntax/lint once, then runs all five Molecule scenarios for each of the five
images (25 matrix jobs). Baseline and guardrails include the check-mode coverage
described above; Molecule also performs its normal idempotence phase where
applicable.

There is currently no separate scheduled or release matrix. The push and pull
request matrix covers the five images listed above, which are the complete
supported-platform contract.

After a successful CI job, its exact repository, workflow run, commit, and test
context are appended to the GitHub Actions job summary and uploaded as a
run-specific artifact named `ci-evidence-*`.

## Supported Versus Tested Platforms

The role supports Ubuntu 22 and Enterprise Linux 9/10 (including the
documented Red Hat and Rocky normalization). RHEL UBI 9/10,
Rocky Linux 9/10, and Ubuntu 22.04 are configured in this repository's CI
matrix. Tester-reported evidence records all 25 matrix combinations passing on
branch `feature/rc-readiness`, but does not yet include an exact commit SHA or
CI run artifact.
Platform support is defined by `iac_supported_os` in `defaults/main.yml`; all
supported platforms are represented in the automated CI matrix.

## Limitations And Coverage Gaps

- The test image uses the mutable `latest` tag and is not digest-pinned.
- GitHub Actions references use major tags and are also mutable; CI is not a
  fully digest-pinned supply-chain boundary.
- Debian 11/12, Ubuntu 24, and EL 8 are outside the supported-platform contract
  and are not covered by the configured CI matrix. This is an intentional
  compatibility narrowing; operators needing those platforms must remain on a
  previous compatible role revision or qualify a future support change.
- There is no multi-host, cluster, replication, upgrade, or application-version
  matrix because this role manages local identities only.
- Check-mode behavior beyond the tested present and destructive paths follows
  the capabilities and limitations of the Ansible user, group, file, and
  authorized-key modules.
- The 25/25 result is tester-reported and not yet traceable to an exact commit
  SHA or CI run artifact.
