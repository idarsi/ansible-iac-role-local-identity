# Testing

All role convergence and verification runs target disposable Linux containers
created by Molecule. Do not converge this role against localhost or the current
host. Create and destroy playbooks use `hosts: localhost` only for Podman
control.

## Automated Molecule Matrix

Every scenario runs against each image in the CI platform matrix; there is no
application-version dimension for this role.

| Platform/image | Ansible or application versions | Molecule scenarios | Main coverage |
| --- | --- | --- | --- |
| RHEL UBI 9 / `registry.access.redhat.com/ubi9/ubi-init:latest` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Primary Enterprise Linux coverage |
| RHEL UBI 10 / `registry.access.redhat.com/ubi10/ubi-init:latest` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Primary Enterprise Linux coverage |
| Rocky Linux 9 / `rockylinux:9` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Primary Enterprise Linux compatibility coverage |
| Rocky Linux 10 / `rockylinux:10` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Primary Enterprise Linux compatibility coverage |
| Ubuntu 22.04 / `geerlingguy/docker-ubuntu2204-ansible:latest` | `ansible-core==2.21.3`; application version: not applicable | `validation`, `baseline`, `lifecycle`, `guardrails`, `all_absent` | Retained compatibility coverage |

Images are pulled and run directly by Podman. Tags are not pinned by digest.
UBI images are public but access to Red Hat's registry can be restricted; CI
does not invent or provide credentials. Rocky images test the compatible EL
implementation, not a Red Hat subscription.

## Scenario coverage

- `validation`: Runs `state: validate` with an empty user list, a minimal valid
  user, numeric UID/GID values, and valid unrestricted sudo opt-in. It asserts
  actionable failure messages for invalid numeric values, invalid groups,
  multiline keys, unsafe sudo syntax, and sudo variants. Prepare and verify
  prove that validation does not create an account or sudoers file.
- `baseline`: Runs normal `state: present` convergence and Molecule's
  idempotence check. It verifies user creation, sudoers rendering, home marker
  ownership/content, preservation of an existing home marker state, numeric
  primary-group creation and reuse, and a valid present-state check-mode run.
- `lifecycle`: Creates a user and then exercises `state: absent` with explicit
  deletion and home-removal authorization. It verifies that the declared user
  and marked home are removed.
- `all_absent`: Exercises `state: all_absent` with both destructive opt-ins and
  a role-managed home marker. It verifies removal of only the declared user,
  home, and sudoers file while preserving an unmanaged sudoers file and an
  undeclared home.
- `guardrails`: Verifies destructive check mode is non-mutating and that
  deletion, `all_absent`, unmanaged home removal, unmanaged or unsafe-metadata
  sudoers updates, and conflicting same-name GIDs fail with expected error
  messages. Final assertions verify unmanaged resources remain present.

All five scenarios run on every image listed in the matrix. No scenario currently
covers Debian, Ubuntu 24.04,
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

## CI Execution

The GitHub Actions workflow runs on every push and pull request. It executes
syntax/lint once, then runs all five Molecule scenarios for each of the five
images (25 matrix jobs). Baseline and guardrails include the check-mode coverage
described above; Molecule also performs its normal idempotence phase where
applicable.

There is currently no separate scheduled or release matrix. The push and pull
request matrix covers the five images listed above; other supported platforms
are not continuously tested.

## Supported Versus Tested Platforms

The role supports Debian 11/12, Ubuntu 22/24, and Enterprise Linux 8/9/10
(including the documented Red Hat and Rocky normalization). RHEL UBI 9/10,
Rocky Linux 9/10, and Ubuntu 22.04 are automatically tested in this repository.
Platform support is defined by
`iac_supported_os` in `defaults/main.yml`; support does not imply continuous
CI coverage.

## Limitations And Coverage Gaps

- The test image uses the mutable `latest` tag and is not digest-pinned.
- Functional coverage does not exercise Debian, Ubuntu 24.04, or EL 8.
- There is no multi-host, cluster, replication, upgrade, or application-version
  matrix because this role manages local identities only.
- Check-mode behavior beyond the tested present and destructive paths follows
  the capabilities and limitations of the Ansible user, group, file, and
  authorized-key modules.
