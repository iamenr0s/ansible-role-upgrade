[![Molecule](https://github.com/iamenr0s/ansible-role-upgrade/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-upgrade/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_upgrade) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-upgrade/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-upgrade)

Ansible Role: Upgrade
=====================

This role performs safe, system-wide package upgrades on Debian/Ubuntu and RHEL-family (AlmaLinux/RockyLinux) and Fedora servers. It includes pre-checks to validate host stability, package manager–specific upgrade routines, and post-checks that reboot when required and verify the host returns.

Features
--------
- Validates OS support against a curated stable list (`upgrade_stable_os`).
- Checks minimum uptime and load average before proceeding.
- Supports `apt`, `dnf`, and `yum` flows with cache management.
- Allows holding packages across the upgrade (`upgrade_packages_on_hold`).
- Detects reboot hints and performs a controlled reboot, then waits for the host.

Requirements
------------
- Python 3.7+ available on the managed hosts (Ansible modules require Python).
- Appropriate package manager present (`apt`, `dnf`, or `yum`).
- For `yum`-based systems, this role uses `community.general.yum_versionlock` to hold/unhold packages. Install the `community.general` collection:
  - `ansible-galaxy collection install community.general`
- Run with privilege escalation on real hosts: `become: true` is recommended.

Supported Platforms
-------------------
The role enforces a stability guardrail via `upgrade_stable_os` (defaults):

- AlmaLinux 8, 9, 10
- Debian 10, 11, 12
- Fedora 40, 41, 42
- Rocky 8, 9, 10
- Ubuntu 20, 22, 24

Note: Platforms in `meta/main.yml` reflect Galaxy metadata; the runtime guard uses the list above to fail early on unsupported OS major versions.

Role Variables
--------------
Defined in `defaults/main.yml`:

- `upgrade_stable_os` (list): Allowed OS name + major version strings used for safety checks.
- `upgrade_min_uptime` (int): Minimum uptime in seconds before upgrade (default: `3600`).
- `upgrade_max_load_avg` (float): Maximum allowed load average (default: `1.0`).
- `upgrade_max_filesystem_usage` (int): Maximum allowed filesystem usage percentage before upgrade (default: `90`).
- `upgrade_wait_for_timeout` (int): Timeout in seconds to wait for host to return after reboot (default: `300`).
- `upgrade_reboot_delay` (int): Delay in seconds before reboot (default: `10`).
- `upgrade_wait_for_port` (int): Port used to verify connectivity after reboot (default: `22`).
- `upgrade_packages_on_hold` (list): Packages to keep on hold during upgrade (default: `[]`).
- `upgrade_skip_reboot` (bool): Skip the post-upgrade reboot and connectivity wait entirely, for environments (e.g. containers) that can't reboot or run sshd (default: `false`).

Behavior Overview
-----------------
Pre-checks (`tasks/pre-checks.yml`):
- Fails if `ansible_distribution` + major version is not in `upgrade_stable_os`.
- Gathers minimal facts and calculates uptime and 15m load average.
- Fails if uptime is below `upgrade_min_uptime` or load average exceeds `upgrade_max_load_avg`.
- Fails if any filesystem exceeds `upgrade_max_filesystem_usage`%.

Package manager flows:
- `apt` (`tasks/apt.yml`):
  - Updates cache, ensures `apt` and `apt-utils` present.
  - Holds packages via `apt-mark hold`.
  - Performs `apt` upgrade.
  - Unholds via `apt-mark unhold`.
  - Cleans cache and checks `/var/run/reboot-required`.
- `dnf` (`tasks/dnf.yml`):
  - Ensures `dnf` and `dnf-utils` present.
  - Adds/removes version locks via `dnf versionlock`.
  - Performs updates with `dnf` and cleans cache; checks reboot via `dnf needs-restarting -r`.
- `yum` (`tasks/yum.yml`):
  - Ensures `yum` and `yum-utils` present.
  - Uses `community.general.yum_versionlock` to hold/unhold packages.
  - Uses `dnf`-based modules for upgrade/cleanup where available (ensure `dnf` is present on your platform or adapt as needed).

Post-checks (`tasks/post-checks.yml`):
- Reboots when a reboot hint is detected.
- Waits for the host to come back on `upgrade_wait_for_port` within `upgrade_wait_for_timeout`.
- Pings the host and fails if it did not return cleanly.

Idempotency & Rollback
-----------------------
- Idempotent: Yes — re-running against an up-to-date, stable host reports no changes.
- Atomic: No — a failure mid-upgrade (e.g. a reboot timeout) can leave the host partially upgraded; no automatic rollback is performed.
- Rollback: Not supported by this role. Recovery relies on your own snapshot/backup strategy before running upgrades.

Tags
----
All tasks are tagged `upgrade`, allowing selective runs:
- `ansible-playbook ... --tags upgrade`
- `ansible-playbook ... --skip-tags upgrade`

Dependencies
------------
- Collections: `community.general` (required for yum version lock tasks).
- Role dependencies: none.

Example Usage
-------------
Basic run on supported hosts:

```yaml
- hosts: all
  become: true
  roles:
    - role: iamenr0s.ansible_role_upgrade
```

Hold specific packages during upgrade:

```yaml
- hosts: all
  become: true
  vars:
    upgrade_packages_on_hold:
      - kernel
      - docker-ce
  roles:
    - role: iamenr0s.ansible_role_upgrade
```

Tune stability thresholds and reboot behavior:

```yaml
- hosts: all
  become: true
  vars:
    upgrade_min_uptime: 7200         # 2 hours
    upgrade_max_load_avg: 2.5
    upgrade_reboot_delay: 30
    upgrade_wait_for_timeout: 600
    upgrade_wait_for_port: 22
  roles:
    - role: iamenr0s.ansible_role_upgrade
```

Notes
-----
- The role fails fast if uptime, load average, or filesystem usage breach the configured thresholds; tune `upgrade_min_uptime`, `upgrade_max_load_avg`, and `upgrade_max_filesystem_usage` to match your policy.
- RHEL-like systems using `yum` may rely on `dnf` modules for some operations; ensure `dnf` is available or adapt the tasks for your environment.
- Ensure the remote machines have Python 3.7+; older Python may cause Ansible module execution issues.

Contributing & Security
-----------------------
- Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).
- Report vulnerabilities privately per [SECURITY.md](SECURITY.md); do not open public issues for them.

CI & Release (maintainers)
--------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

The Galaxy API key lives in the `galaxy` GitHub environment, which only `v*` tags may target. One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --env galaxy --repo iamenr0s/ansible-role-upgrade

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --env galaxy --repo iamenr0s/ansible-role-upgrade
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

License
-------
This project is licensed under the MIT License.

Author Information
------------------
Author: iamenr0s
Galaxy: `iamenr0s.ansible_role_upgrade`
