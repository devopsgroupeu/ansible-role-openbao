# Ansible Role - OpenBao

[![CI](https://github.com/devopsgroupeu/ansible-role-openbao/actions/workflows/ci.yml/badge.svg)](https://github.com/devopsgroupeu/ansible-role-openbao/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/Ansible%20Galaxy-devopsgroupeu.openbao-blue?logo=ansible)](https://galaxy.ansible.com/ui/standalone/roles/devopsgroupeu/openbao/)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
![GitHub Forks](https://img.shields.io/github/forks/devopsgroupeu/ansible-role-openbao)
![GitHub Stars](https://img.shields.io/github/stars/devopsgroupeu/ansible-role-openbao)
![GitHub Issues](https://img.shields.io/github/issues/devopsgroupeu/ansible-role-openbao)

[![LinkedIn](https://img.shields.io/badge/linkedin-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/company/devopsgroup8/)
![OpenBao](https://img.shields.io/badge/OpenBao-336D5C?logo=openbao&logoColor=white&style=for-the-badge)

Production-ready Ansible role for installing and configuring **OpenBao** as a
TLS-enabled Raft HA cluster with idempotent init and unseal (Shamir, static,
transit, cloud KMS plugins), declarative audit, health-gated rolling
restarts, systemd hardening and Prometheus telemetry.

---

## Requirements

- **Ansible:** `ansible-core >= 2.19` (developed and tested on 2.21)
- **Collections:** `community.crypto >= 3.2.2` (generated TLS only).
  Install via: `ansible-galaxy collection install -r requirements.yml`
- **Target OS:** Ubuntu 24.04 (noble), Debian 12/13 (bookworm/trixie),
  RHEL/Rocky/Alma 9; systemd with cgroup v2; amd64 or arm64
- **OpenBao version:** 2.7.3 (default; override via `openbao_version`, minimum 2.7.2;
  the upstream RPM repository lacks some patch releases, 2.7.2 among them, so
  on EL pin such a version with the binary method or a mirror)
- **Network:** HTTPS from the targets to openbao.org (signing key) and
  pkgs.openbao.org (package method) or github.com plus
  release-assets.githubusercontent.com (binary method, and the KMS plugins of
  the cloud seals with either method; GitHub redirects every release download
  to that host and has changed it before, objects.githubusercontent.com was
  used earlier), or to your mirrors (`openbao_binary_base_url`,
  `openbao_seal_plugin_base_url`)
- **Controller:** an Ansible Vault password file for the init output (the
  output is encrypted by default)
- **Docker (Molecule):** required for running tests locally
- **`become: true` at the play level** - the role does not set `become` per task

---

## Quick start

### Install

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install devopsgroupeu.openbao
```

Or pin a version from Git:

```yaml
# requirements.yml
roles:
  - name: devopsgroupeu.openbao
    src: https://github.com/devopsgroupeu/ansible-role-openbao.git
    version: "vX.Y.Z"   # latest release tag, see GitHub Releases
    scm: git
```

The inventory group must be named `openbao` (or set `openbao_raft_peers`),
and the whole group runs in one play: see
[`examples/inventory/openbao_cluster.yml`](examples/inventory/openbao_cluster.yml).

### Single node (dev/test - generated TLS, Shamir 1/1)

```yaml
- name: Deploy OpenBao
  hosts: openbao
  become: true
  roles:
    - role: devopsgroupeu.openbao
      vars:
        openbao_cluster_id: "openbao-dev"
        openbao_tls_generate_certs: true
        openbao_init_key_shares: 1
        openbao_init_key_threshold: 1
        openbao_init_encrypt_password_file: "{{ '~/.openbao-vault-pass' | expanduser }}"
```

### Three-node Raft cluster (production - BYO TLS, Shamir 5/3)

```yaml
- name: Deploy an OpenBao cluster
  hosts: openbao           # the 'openbao' group holds all three nodes
  become: true
  roles:
    - role: devopsgroupeu.openbao
      vars:
        openbao_version: "2.7.3"
        openbao_cluster_id: "openbao-prod"
        # BYO TLS (default): these files must exist on every node.
        openbao_tls_cert_file: /opt/openbao/tls/openbao-cert.pem
        openbao_tls_key_file: /opt/openbao/tls/openbao-key.pem
        openbao_tls_ca_file: /opt/openbao/tls/openbao-ca.pem
        openbao_init_key_shares: 5
        openbao_init_key_threshold: 3
        openbao_init_encrypt_password_file: "{{ '~/.openbao-vault-pass' | expanduser }}"
```

Each certificate needs the SAN `DNS:openbao.cluster.internal`
(`openbao_tls_leader_servername`) and the host of that node's
`openbao_api_addr`; see [`examples/raft_cluster.yml`](examples/raft_cluster.yml).

### Three-node Raft cluster with transit auto-unseal

```yaml
- name: Deploy an OpenBao cluster (auto-unseal)
  hosts: openbao
  become: true
  roles:
    - role: devopsgroupeu.openbao
      vars:
        openbao_cluster_id: "openbao-transit"
        openbao_seal_type: transit
        openbao_seal_transit_address: "https://unsealer.example.internal:8200"
        openbao_seal_transit_key_name: autounseal
        openbao_seal_transit_tls_ca_cert: /opt/openbao/tls/unsealer-ca.pem
        # openbao_seal_transit_token comes from an Ansible Vault encrypted
        # group_vars file, never from the playbook.
        # Auto-unseal init returns recovery keys, not unseal keys.
        openbao_init_recovery_shares: 5
        openbao_init_recovery_threshold: 3
        openbao_init_encrypt_password_file: "{{ '~/.openbao-vault-pass' | expanduser }}"
```

[`examples/autounseal_transit.yml`](examples/autounseal_transit.yml) shows the
unsealer prerequisites (transit key, policy, periodic orphan token);
[`examples/autounseal_awskms.yml`](examples/autounseal_awskms.yml) does the same
for AWS KMS with the KMS plugin and the IAM policy.

See `examples/` for all complete playbooks.

### Upgrades and day-2 runs

- **Upgrade:** read the upstream release notes, take a raft snapshot
  (`bao operator raft snapshot save`, needs a token), raise `openbao_version`
  and run the playbook against the whole group. The role replaces the package
  or binary on every node, then restarts one node at a time: before each
  restart every other member must be healthy, after it the node must be
  unsealed and see a leader; the leader goes last. Downgrades are refused.
- **Failed runs:** a node whose config, unit, package, binary or plugin
  changed keeps a restart marker until it was restarted, so a re-run after a
  failure finishes the rolling restart. Nodes that already restarted are not
  restarted again.
- **One play, linear, no `serial`:** the whole group runs in one play with the
  default linear strategy; the role refuses `serial` and `strategy: free`
  (the rolling behaviour is inside the role).
- **`--limit`:** day-2 runs on part of the cluster need
  `openbao_raft_retry_join_api_addrs` with the API URLs of all members; the
  role refuses otherwise. Initialization always needs the whole group.
- **`--tags`:** every stage has a tag (`install`, `config`, `tls`, `service`,
  `init_unseal`, `restart`); it reaches the tasks when the role is used through
  `roles:` or `import_role`.
- **Maintenance:** `openbao_service_state: stopped` on one host stops that
  node; the others are still converged and unsealed, but no rolling restart
  (quorum check) and no init run until it is started again.
- **Replacing a node:** remove the dead peer first (`bao operator raft
  remove-peer <node id>` from a healthy member), provision a fresh host with
  an empty data directory and run the play; it joins through `retry_join` as
  a new peer and is unsealed, nothing is initialized again.
- **Limits:** a single node has downtime on every restart. With Shamir and
  `openbao_unseal: false` (or PGP-wrapped shares) the rolling restart waits
  `openbao_restart_health_timeout` seconds for you to unseal each restarted
  node, and stops the play if nobody does. Seal type, seal key and raft node
  id changes are refused on nodes with data, and so is a data directory of
  another storage backend (manual procedures in
  [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#in-place-change-guards)).
  After a Shamir rekey, update the init file (see
  [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#node-stays-sealed-after-bao-operator-rekey)).

---

## Role variables

All 113 variables are documented in full in
**[`docs/CONFIGURATION.md`](docs/CONFIGURATION.md)**. Every variable is
prefixed `openbao_`, declared in `defaults/main.yml`, and validated by
`meta/argument_specs.yml` (1:1 mapping) and, before anything is installed,
by `tasks/validate.yml`.

Key variables to be aware of:

| Variable | Default | Notes |
|---|---|---|
| `openbao_version` | `"2.7.3"` | Exact pin, minimum 2.7.2; a change is a health-gated rolling upgrade |
| `openbao_install_method` | `"package"` | `package` (signed pkgs.openbao.org) or `binary` (verified release tarball) |
| `openbao_tls_generate_certs` | `false` | **BYO TLS by default**; `true` generates a CA and node certificates (dev/test only) |
| `openbao_tls_leader_servername` | `"openbao.cluster.internal"` | Must be a DNS SAN on every node certificate |
| `openbao_api_addr` | `https://<default IPv4>:8200` | Advertised address and probe target of each node; the certificate must cover its host |
| `openbao_raft_peers` | `groups['openbao']` | All members; a differently named group is refused when it would split the cluster |
| `openbao_raft_retry_join_api_addrs` | `[]` | Required for `--limit` day-2 runs |
| `openbao_seal_type` | `"shamir"` | `static`, `transit`, `awskms`, `azurekeyvault`, `gcpckms` for auto-unseal; cannot change on a node with data |
| `openbao_cluster_id` | first peer, sorted | Names the init file; set it explicitly in production |
| `openbao_init_output_path` | `~/.openbao/.openbao_init_<cluster id>.json` | Controller-side, `0600`, never overwritten |
| `openbao_init_encrypt_password_file` | `""` | **Required** while `openbao_init_encrypt_output` is true (default) |
| `openbao_unseal` | `true` | Unseal Shamir nodes from the init file on every run |
| `openbao_audit_enabled` | `true` | Declarative file audit device; its settings are immutable once applied |
| `openbao_systemd_hardening` | `true` | Role sandboxing directives on top of the upstream unit |
| `openbao_service_state` | `"started"` | `stopped` for maintenance: no init, unseal or restart |
| `openbao_config_only` | `false` | Render-only mode (used by the `render` scenario) |

---

## Security notes

### TLS

TLS is **on by default** (minimum TLS 1.2, no client certificate requests).
Supply BYO certificates (the default) with the SAN
`openbao_tls_leader_servername` and the host of each node's `openbao_api_addr`.
**Generated mode (`openbao_tls_generate_certs: true`) copies the CA private key
to every node**; use it for dev/test only. The role re-signs and reloads
(SIGHUP, no restart) the generated certificates it manages. When you rotate BYO
files, run `systemctl reload openbao` on each node, or set `tls_auto_reload:
true` in `openbao_listener_extra_parameters`. A reload re-reads all of
`openbao.hcl`, not only the certificates (see
[Rolling restart and the restart marker](#rolling-restart-and-the-restart-marker)).

### Init output

The init output (root token and unseal or recovery keys) exists **only on the
controller**, at `openbao_init_output_path` (default
`~/.openbao/.openbao_init_<cluster id>.json`, outside any repository): mode
`0600`, parent directory `0700`, **encrypted with Ansible Vault in memory
before it touches the disk**, and never overwritten. The role fails before
installing anything when encryption is requested without a password file.
Later runs read the unseal keys from this file, so **only a user who can read
it and the password file can re-unseal Shamir nodes** and run rolling restarts
on them. CI runners and pipelines must point `openbao_init_output_path` at
persistent, access-controlled storage. Never commit the file; `.gitignore`
covers `.openbao_init_*.json`. The password file is read like
`ansible-vault --vault-password-file` reads it (surrounding whitespace
stripped); one that holds only whitespace is refused before init. During a
run the unseal keys (and right after init the init output) exist only in
host variables of the `init_unseal` and `restart` stages, which clear them
at their end.

### Unseal keys vs recovery keys

With Shamir, init returns unseal keys; the role reads them from the init file
to unseal. With an auto-unseal seal, init returns **recovery keys, which cannot
unseal**: the unsealer or KMS does. **Deleting the KMS or transit key loses the
cluster data for good**; enable deletion protection. Operators who do not want
unseal keys on the controller use PGP-wrapped shares
(`openbao_init_pgp_keys`, with `openbao_unseal: false` for Shamir) or an
auto-unseal seal.

### Root token

**The role never reads or uses the root token** after init: cluster checks use
the unauthenticated `sys/seal-status`, `sys/leader` and `sys/health` endpoints.
Set up your own auth method and an admin policy, then revoke the root token
(`bao token revoke -self`); later runs keep converging, unsealing and
restarting. Since OpenBao 2.6.0 generating a new root token needs an
authenticated caller, so keep an admin login that may use
`sys/generate-root-token/*`.

### Transit token

Use a **periodic orphan token** with only `update` on the transit encrypt and
decrypt paths (see `examples/autounseal_transit.yml`). The role writes it as
`BAO_TOKEN` to `openbao.env` only. A periodic token that is not renewed within
its period (for example while every node is down) expires; nodes then stay
sealed until you install a new token.

### Static seal

The `static` seal keeps a 32-byte key in `openbao.env` on every node: anyone
with root on a node can unseal the data. It is meant for test setups and for
keys injected by an existing trust source, not as a general auto-unseal. The
role cannot rotate the static key (no `previous_key` support): keep the key
and its id once the cluster is initialized.

### Signing key pin

Packages, the release `checksums.txt` and the KMS plugin archives are trusted
only through the OpenBao signing key with the pinned primary fingerprint
(`openbao_gpg_key_fingerprint`). The key file is installed only when it holds
exactly one key with that fingerprint, both signature checks bind the signer to
the pin, and an installed key is never replaced or deleted silently. Mirrors
and air-gapped sites can pin SHA-256 checksums instead
(`openbao_binary_checksum`, `openbao_seal_plugin_checksum`); unverified
downloads are refused.

### Audit device immutability

The role declares the audit device in `openbao.hcl` (OpenBao blocks creating
audit devices through the API). **Its settings cannot change in place**:
OpenBao refuses a modified device when a node becomes active. The role refuses
such a change before rendering; change it in two runs (disable, then re-enable
with the new values) as described in
[TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#audit-devices-cannot-be-modified-in-place).

### systemd hardening is role opinion

On top of the upstream unit (`ProtectSystem=full`, `ProtectHome=read-only`,
`PrivateTmp`, `PrivateDevices`, `NoNewPrivileges`, `MemorySwapMax=0`) the role
adds `LimitCORE=0`, an empty capability set and, with
`openbao_systemd_hardening: true`, `ProtectSystem=strict` with
`ReadWritePaths` for the data and audit directories plus kernel, namespace and
address family restrictions. **These extra directives are this role's opinion,
not upstream guidance**; set `openbao_systemd_hardening: false` if one
conflicts with your environment.

### File ownership and the permission check

With `openbao_enable_file_permissions_check: true` OpenBao refuses to start
unless `openbao.hcl`, the plugin directory and each plugin binary are owned by
the service user and not writable by group or others, so the role makes them
`openbao:openbao`. With hardening on, `ProtectSystem=strict` mounts them
read-only for the process, so the service still cannot modify its config or
plugins. With hardening off, `ProtectSystem=full` leaves `/opt` writable for
the service user. Secrets live in `openbao.env` (`root:root 0600`), which only
systemd reads.

### Swap and mlock

OpenBao has no mlock: `disable_mlock = false` is a fatal config error, so the
role never renders the key and refuses it in `openbao_extra_config_hcl`.
`MemorySwapMax=0` in the unit keeps the process out of swap (cgroup v2); the
role does not touch host swap.

### Rolling restart and the restart marker

The role never restarts all nodes at once. A pending config or unit change
writes `/etc/openbao/.ansible-restart-pending` on that node before the file
changes, and a package, binary or plugin change right after the install; the
rolling restart restarts the flagged nodes one at a time, standbys first and
the leader last, after a quorum check and with a health gate (unsealed and
sees a leader). A failed check stops the play with quorum intact, and the
markers make the next run finish the job. With Shamir and `openbao_unseal: true`
the role restarts nothing while the unseal keys cannot be read from the init
file. Certificate changes reload instead of restarting. A reload (SIGHUP, also
the nightly logrotate reload) makes OpenBao re-read `openbao.hcl` and apply
the log level, audit device additions and removals and KMS plugin stanzas at
once; storage, listener addresses, seal and `openbao.env` changes wait for
the restart.

### Check mode

`--check --diff` is supported on hosts that were converged by this role at
least once. It reports pending file and package changes and the nodes that a
real run would restart (`Check mode: would restart ...`, counting the changes
of the same check run), never initializes, unseals, restarts or reloads
anything, and fails only for real validation errors. Check mode on a fresh host is not supported (the service user and
binary do not exist yet).

### Secrets the role cannot keep off the targets

Unseal and recovery keys, the root token, the transit token, AWS and Azure
credentials and the static seal key never appear in Ansible output or in
`openbao.hcl`; on the targets they exist only in `openbao.env` and the service
environment. By design some secrets still live on the targets: **TLS private
keys** (the node key on every node, and in generated mode also the CA key on
every node), the **GCP credentials JSON** at `openbao_seal_gcpckms_credentials`
(managed by you), and the **environment of KMS plugin processes**, which
inherit the server environment including the cloud credentials.

---

## Tests (Molecule)

```bash
# Install test dependencies
pip install -r requirements.txt

# Offline rendering of every config branch and the fail-closed input checks
molecule test -s render

# Single node: signed package install, generated TLS, init, unseal, audit
# off and on again, check mode, certificate and key rotation, maintenance
# stop, --check and --tags through roles:, in-place change guards, foreign
# data directory, blank password file
molecule test -s default

# Single node: verified binary install, static auto-unseal, seal key guard,
# KMS plugin install, validate-config of the cloud seal configs, a hardened
# instance that starts the AWS KMS plugin, download verification failures,
# signing key pin
molecule test -s autounseal

# Three nodes: 2.7.2 to 2.7.3 rolling upgrade, quorum check, healing after a
# failed run, one member in maintenance, serial and free strategy refused,
# node replacement with remove-peer, inventory group guard
molecule test -s raft-ha
```

Every scenario includes the `idempotence` step. The images default to
`debian12`; set `MOLECULE_DISTRO=ubuntu2404`, `debian13` or `rockylinux9` for
the others. The CI pipeline (`ci.yml`) runs on every push and pull request to
`main` and `develop`: `yamllint` and `ansible-lint --profile production`, then
`render` on `debian12`, `ubuntu2404` and `debian13`, `default` on those three
and `rockylinux9`, `autounseal` on `debian12` and `ubuntu2404`, and `raft-ha`
on `debian12`. `MOLECULE_DISTRO=rockylinux9 molecule test -s default` passed
locally (arm64) before it joined CI; it runs the EL 9 package path (`dnf`,
`yum_repository`, `rpm_key`), while the SELinux tasks (`semanage fcontext`,
`restorecon`) and `update-ca-trust` are covered by lint only (SELinux is off
in the container and the scenario does not touch the trust store). The
SELinux part was checked against the EL 9 targeted policy (selinux-policy
38.1.86) in a Rocky Linux 9 container: `sesearch` shows that `logrotate_t`
may only read `usr_t` files but may create, rename and unlink `var_log_t`
ones, and `semanage fcontext --add` / `--list --locallist` behave as the role
expects.

---

## File layout

```
ansible-role-openbao/
|-- defaults/main.yml              # 113 public variables
|-- vars/main.yml                  # internal variables (OS-family maps, plugin names)
|-- meta/main.yml                  # Galaxy metadata
|-- meta/argument_specs.yml        # 1:1 with defaults (113 options)
|-- handlers/main.yml              # reload openbao (certificate changes)
|-- tasks/
|   |-- main.yml                   # orchestrator
|   |-- validate.yml               # fail-closed input checks (26 rules)
|   |-- facts.yml                  # API and retry_join addresses, --limit guard
|   |-- install.yml                # dependencies, signing key, method dispatch, plugin
|   |-- install_package.yml        # pkgs.openbao.org (deb822 / dnf), no downgrade, hold
|   |-- install_binary.yml         # GPG- and checksum-verified release tarball
|   |-- install_plugin.yml         # verified KMS plugin for the cloud seals
|   |-- config.yml                 # directories, guards, openbao.env, openbao.hcl, unit, marker
|   |-- tls.yml                    # generated or BYO certificates, OS trust store
|   |-- tls_generate.yml           # community.crypto CA and node certificates
|   |-- tls_byo.yml                # existence check, owner and mode
|   |-- service.yml                # start and enable openbao.service
|   |-- probe.yml                  # unauthenticated seal-status and leader probes
|   |-- load_keys.yml              # unseal keys from the controller init file
|   |-- init_unseal.yml            # init, encrypted persist, unseal, formation check
|   |-- restart.yml                # restart order, leader last
|   `-- restart_node.yml           # quorum check, restart, unseal, health gate
|-- templates/
|   |-- openbao.hcl.j2             # server config
|   |-- audit.hcl.j2               # declarative audit device between marker lines
|   |-- openbao.env.j2             # the only file with secrets
|   |-- openbao.service.j2         # hardened unit
|   `-- logrotate.j2               # audit log rotation
|-- molecule/{render,default,autounseal,raft-ha}/
|-- examples/{single_node,raft_cluster,autounseal_transit,autounseal_awskms}.yml
|   `-- inventory/openbao_cluster.yml
|-- docs/{ARCHITECTURE,CONFIGURATION,INTEGRATION,TROUBLESHOOTING}.md
`-- .github/workflows/{ci,galaxy,pre-commit,release,release-drafter}.yml
```

---

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) to get started.

## Code of Conduct

Read our [Code of Conduct](CODE_OF_CONDUCT.md).

## Changelog

See the [GitHub Releases](https://github.com/devopsgroupeu/ansible-role-openbao/releases) page for release history.

## License

```
Copyright 2026 DevOpsGroup

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

> For more information or support, please refer to the official documentation or contact us at info@devopsgroup.sk
