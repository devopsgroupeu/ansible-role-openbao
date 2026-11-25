# Configuration Reference

Full variable reference for the OpenBao role: all 113 variables, in the order
and sections of [`defaults/main.yml`](../defaults/main.yml). Every variable is
validated by [`meta/argument_specs.yml`](../meta/argument_specs.yml) (1:1 with
the defaults) and, before anything is installed, by the fail-closed checks of
`tasks/validate.yml`; each failure message is explained in
[TROUBLESHOOTING.md](TROUBLESHOOTING.md). Every public variable is prefixed
`openbao_`. Variables marked **Secret** have an empty default and must come
from Ansible Vault; the role writes them only to `openbao.env`.

---

## Table of Contents

- [Install](#install)
- [Paths / users](#paths--users)
- [Listener / addresses / UI](#listener--addresses--ui)
- [TLS](#tls)
- [Storage (Raft)](#storage-raft)
- [Seal / auto-unseal](#seal--auto-unseal)
- [Init / unseal](#init--unseal)
- [Telemetry / logging / TTLs / hardening](#telemetry--logging--ttls--hardening)
- [Audit](#audit)
- [systemd / permissions](#systemd--permissions)
- [Service control](#service-control)
- [Notes](#notes)

---

## Install

| Variable | Default | Description |
|---|---|---|
| `openbao_version` | `"2.7.3"` | Exact OpenBao version (`X.Y.Z`), at least 2.7.2; 2.7.0 and 2.7.1 are refused (namespace deletion bug), 2.6.x is not supported. A change is an upgrade: the role installs the new version on every node, then restarts the nodes one at a time (quorum-checked, leader last). Downgrades are refused. |
| `openbao_install_method` | `"package"` | `package` (signed APT/DNF repository at pkgs.openbao.org) or `binary` (GitHub release tarball, verified through the signed `checksums.txt` and/or a pinned checksum; for mirrors and air-gapped hosts). |
| `openbao_package_hold` | `true` | Protect the pinned version from unrelated upgrades: `dpkg` hold on the Debian family; on EL the `openbao` repository is written disabled and the role enables it only for its own install transaction. |
| `openbao_apt_repo_url` | `"https://pkgs.openbao.org/deb/"` | APT repository URL (deb822 source, suite `stable`, component `main`, `signed_by` the verified signing key file). Point it at a mirror if needed. |
| `openbao_rpm_repo_baseurl` | `"https://pkgs.openbao.org/rpm/$basearch"` | DNF repository base URL (`$basearch` is expanded by dnf; the repository has no `$releasever`). Packages are checked with `gpgcheck`; upstream does not sign the metadata, so `repo_gpgcheck` stays off. |
| `openbao_gpg_key_url` | `"https://openbao.org/assets/openbao-gpg-pub-20240618.asc"` | Signing key for packages, release checksums and KMS plugins. Downloaded only while no verified key is installed at `/usr/local/share/openbao/openbao-signing-key.asc`; the installed file is re-checked on every run and never deleted by the role. |
| `openbao_gpg_key_fingerprint` | `"66D15FDD87287219C8E15478D200CD702853E6D0"` | Expected primary fingerprint (40 hex digits, spaces allowed). The key file must hold exactly one key with this fingerprint; both GPG checks (tarball and plugin) also require the signature's primary key fingerprint to equal it. |
| `openbao_binary_base_url` | `"https://github.com/openbao/openbao/releases/download"` | Base URL for the binary method. The role downloads `<base>/v<version>/openbao_<version>_linux_<arch>.tar.gz`, `checksums.txt` and `checksums.txt.gpgsig`; a mirror (also `file://`) must keep that layout. |
| `openbao_binary_verify_gpg` | `true` | GPG-verify `checksums.txt` (binary method) and the KMS plugin archive (`.sig`) against the pinned key. Set false only together with a pinned checksum (`openbao_binary_checksum` or `openbao_seal_plugin_checksum`); otherwise the download is refused. |
| `openbao_binary_checksum` | `""` | Optional pinned SHA-256 (64 lowercase hex) of the release tarball for this host's architecture, checked by `get_url`. Set it per architecture through a map keyed by `openbao_arch` (see [Notes](#notes)). |
| `openbao_arch` | `"{{ {'x86_64': 'amd64', 'amd64': 'amd64', 'aarch64': 'arm64', 'arm64': 'arm64'}.get(ansible_facts['architecture'], '') }}"` | Upstream architecture name for repository, tarball and plugin asset names. Any architecture other than amd64 or arm64 resolves to `""` and is refused. |

---

## Paths / users

Path variables must be the same on every node of a cluster (see [Notes](#notes)).

| Variable | Default | Description |
|---|---|---|
| `openbao_config_dir` | `"/etc/openbao"` | Holds `openbao.hcl` (`openbao:openbao 0640`), `openbao.env` (`root:root 0600`) and the restart marker `.ansible-restart-pending`. The directory itself is `root:openbao 0750`. |
| `openbao_data_dir` | `"/opt/openbao/data"` | Raft data directory (`openbao:openbao 0750`, writable through `ReadWritePaths`). The guards treat a node with `vault.db` here as a node with data. |
| `openbao_tls_dir` | `"/opt/openbao/tls"` | TLS material (`root:openbao 0750`). The package's own `tls.crt` / `tls.key` pair in the default directory is left in place and unused. |
| `openbao_audit_dir` | `"/opt/openbao/audit"` | Audit log directory (`openbao:openbao 0750`), created on every node before the first start. `openbao_audit_file_path` must be inside it. |
| `openbao_plugin_dir` | `"/opt/openbao/plugins"` | External plugin directory (`openbao:openbao 0750`, as the file permission check requires). Created and rendered as `plugin_directory` only for the cloud KMS seals; must not be a symbolic link. |
| `openbao_user` | `"openbao"` | Service user. The package creates it; the binary method creates it as a system user (`/bin/false`, home `/opt/openbao`, no home directory created). |
| `openbao_group` | `"openbao"` | Service group. |
| `openbao_bin_path` | `"{{ '/usr/local/bin/bao' if openbao_install_method == 'binary' else '/usr/bin/bao' }}"` | Path of the `bao` binary, used by `ExecStart`, the `validate-config` hook and `operator init`. Override only when your distribution places it elsewhere. |

---

## Listener / addresses / UI

| Variable | Default | Description |
|---|---|---|
| `openbao_listen_address` | `"0.0.0.0:8200"` | API listener bind address, `host:port`. Its port is also the port of the default `openbao_api_addr`. A port below 1024 (API or cluster listener) grants `CAP_NET_BIND_SERVICE` in the unit. |
| `openbao_cluster_listen_address` | `"0.0.0.0:8201"` | Cluster listener bind address, `host:port` (Raft and request forwarding). |
| `openbao_api_addr` | `"{{ 'https' if openbao_tls_enabled \| bool else 'http' }}://{{ ansible_facts['default_ipv4']['address'] \| default(inventory_hostname) }}:{{ openbao_listen_address.split(':') \| last }}"` | Advertised API URL of this node, the `retry_join` target the other nodes use and the address the role probes, so the server certificate must cover its host. Scheme must match `openbao_tls_enabled`, no path. Set it per host when a node has several interfaces or should be reached by DNS name. |
| `openbao_cluster_addr` | `"https://{{ ansible_facts['default_ipv4']['address'] \| default(inventory_hostname) }}:{{ openbao_cluster_listen_address.split(':') \| last }}"` | Advertised cluster URL of this node (always `https`, the cluster port uses its own TLS). Set it per host together with `openbao_api_addr`. |
| `openbao_ui` | `true` | Enable the web UI on the API listener. |
| `openbao_cluster_name` | `""` | Cluster name. Empty omits the key and OpenBao generates one. |
| `openbao_listener_extra_parameters` | `{}` | Extra keys for the API listener stanza, rendered sorted (bool, number, string, or list of strings), for example `x_forwarded_for_authorized_addrs` or `tls_auto_reload: true` when external tooling rotates the certificates. `validate-config` rejects unknown keys, which fails the render. |

---

## TLS

| Variable | Default | Description |
|---|---|---|
| `openbao_tls_enabled` | `true` | TLS on the API listener. False renders `tls_disable = true` and `http://` addresses (dev/test only); the role never leaves both unset, which would make OpenBao start ACME. |
| `openbao_tls_cert_file` | `"{{ openbao_tls_dir }}/openbao-cert.pem"` | Server certificate (full chain PEM). BYO: must exist on the host before the run; the role sets `root:root` and `openbao_tls_cert_mode`. |
| `openbao_tls_key_file` | `"{{ openbao_tls_dir }}/openbao-key.pem"` | Server private key; the role sets `root:openbao` and `openbao_tls_key_mode`. |
| `openbao_tls_ca_file` | `"{{ openbao_tls_dir }}/openbao-ca.pem"` | CA bundle that issued the node certificates: `leader_ca_cert_file` of every `retry_join` block, CA of the role's own probes and of `operator init` (`BAO_CACERT`), and the file installed into the OS trust store. |
| `openbao_tls_min_version` | `"tls12"` | Minimum TLS version: `tls12` or `tls13`. |
| `openbao_tls_max_version` | `"tls13"` | Maximum TLS version: `tls12` or `tls13`; must not be below the minimum. |
| `openbao_tls_cipher_suites` | `"TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"` | TLS 1.2 cipher suites (GCM and ChaCha20 only). Rendered only when min and max are both `tls12`; TLS 1.3 suites are not configurable. |
| `openbao_tls_disable_client_certs` | `true` | Do not request client certificates on the API listener. With `false` the listener requests them and every `retry_join` block presents the node certificate and key (`leader_client_cert_file`, `leader_client_key_file`), so the certificate needs the `clientAuth` extended key usage too. |
| `openbao_tls_generate_certs` | `false` | Generate a CA (on the bootstrap node, CN `OpenBao Role CA`) and per-node certificates with `community.crypto`. **Dev/test only: the CA private key is copied to every node.** Certificates are re-signed when missing, expiring within 30 days, missing a SAN or issued by another CA; the CA is renewed when it expires within 90 days. A new certificate is loaded by a reload (SIGHUP), not a restart. |
| `openbao_tls_cert_key_type` | `"ECC"` | Key type of generated keys: `ECC` or `RSA`. |
| `openbao_tls_cert_curve` | `"secp384r1"` | Curve of generated ECC keys. |
| `openbao_tls_cert_size` | `4096` | Size of generated RSA keys. |
| `openbao_tls_cert_validity` | `"+825d"` | Validity of generated CA and node certificates (`community.crypto` relative time syntax). Keep it above 90 days: the role renews the CA when it expires within 90 days and a node certificate within 30 days, so a shorter validity re-issues both (and reloads every node) on every run. |
| `openbao_tls_leader_servername` | `"openbao.cluster.internal"` | Shared DNS SAN used as `leader_tls_servername` in every `retry_join` block (and as the CN of generated node certificates). BYO certificates must contain it. |
| `openbao_tls_extra_sans` | `[]` | Extra SANs for generated certificates, already prefixed (`DNS:name`, `IP:address`). Every peer name and default IPv4, the host of `openbao_api_addr`, `DNS:localhost` and `IP:127.0.0.1` are always included. |
| `openbao_tls_install_ca_to_trust_store` | `true` | Install `openbao_tls_ca_file` into the OS trust store as `openbao-ca.crt` (`update-ca-certificates` on Debian/Ubuntu, `update-ca-trust` on EL). |

---

## Storage (Raft)

| Variable | Default | Description |
|---|---|---|
| `openbao_raft_node_id` | `"{{ inventory_hostname }}"` | Raft node id. Never use a random value: the role refuses to change it on a node that holds raft data. |
| `openbao_raft_peers` | `"{{ groups['openbao'] \| default([inventory_hostname]) }}"` | Inventory names of all cluster members; each node gets one `retry_join` block per other member. With an inventory group not named `openbao` every host would fall back to a one-member cluster, which the role refuses when the play has several hosts. |
| `openbao_raft_retry_join_api_addrs` | `[]` | Explicit API URLs of all members, self included (the role drops its own URL from `retry_join`). Empty: each member's `openbao_api_addr`. Required for `--limit` day-2 runs; the quorum check before each restart then also covers the members outside the play. |
| `openbao_raft_performance_multiplier` | `1` | Raft timing multiplier. `1` is the production value. |
| `openbao_raft_extra_parameters` | `{}` | Extra keys for the `storage "raft"` stanza, rendered sorted (bool, number, string, or list of strings), for example `autopilot_reconcile_interval`. Not checked by `validate-config`, so typos pass silently. |

---

## Seal / auto-unseal

`shamir` (no stanza), `static` and `transit` are built into OpenBao 2.7. The
cloud seals `awskms`, `azurekeyvault` and `gcpckms` are external KMS plugins
since 2.7.0: for them the role downloads and verifies the plugin of
openbao/openbao-plugins, installs it into `openbao_plugin_dir` and renders
`plugin_directory` plus a `plugin "kms"` stanza. Seal stanzas never contain a
credential; every secret below is written to `openbao.env` only. The required
inputs of each seal are listed in [Notes](#notes).

### Common

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_type` | `"shamir"` | `shamir`, `static`, `transit`, `awskms`, `azurekeyvault` or `gcpckms`. The role refuses to change it on a node with raft data: a seal change is a manual seal migration. |

### Static

Built in. For test setups and keys injected by an existing trust source only:
anyone with root on a node can read the key from `openbao.env`.

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_static_current_key_id` | `""` | Permanent identifier of the static key (`current_key_id`). |
| `openbao_seal_static_current_key` | `""` | **Secret.** 32 random bytes, base64 or hex (for example `openssl rand -hex 32`). Written as `OPENBAO_SEAL_STATIC_CURRENT_KEY` to `openbao.env` and referenced as `env://OPENBAO_SEAL_STATIC_CURRENT_KEY`. |

### Transit

Built in. Unseals through the transit secrets engine of a separate OpenBao or
Vault cluster; see [`examples/autounseal_transit.yml`](../examples/autounseal_transit.yml)
for the unsealer key, policy and token.

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_transit_address` | `""` | URL of the unsealer (DNS name or VIP, never a single node). |
| `openbao_seal_transit_key_name` | `"autounseal"` | Transit key name on the unsealer. |
| `openbao_seal_transit_mount_path` | `"transit/"` | Transit secrets engine mount path on the unsealer. |
| `openbao_seal_transit_namespace` | `""` | Namespace on the unsealer. Empty omits the key. |
| `openbao_seal_transit_tls_ca_cert` | `""` | Path on the node to the CA certificate of the unsealer, readable by the service user. Empty omits the key (the OS trust store applies). |
| `openbao_seal_transit_disable_renewal` | `false` | Disable token renewal. Keep false, so the periodic token does not expire while a node runs. |
| `openbao_seal_transit_token` | `""` | **Secret.** Periodic orphan token for the unsealer. Written as `BAO_TOKEN` to `openbao.env`, never to `openbao.hcl`. |

### AWS KMS

External plugin `openbao-plugin-kms-aws`. The plugin process inherits the
server environment, credentials included.

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_awskms_region` | `"us-east-1"` | AWS region of the KMS key. |
| `openbao_seal_awskms_kms_key_id` | `""` | KMS key id, key ARN or `alias/<name>`. |
| `openbao_seal_awskms_access_key` | `""` | **Secret.** Access key id, written as `AWS_ACCESS_KEY_ID`. Empty uses the instance role (preferred). |
| `openbao_seal_awskms_secret_key` | `""` | **Secret.** Secret access key, written as `AWS_SECRET_ACCESS_KEY`. |

### Azure Key Vault

External plugin `openbao-plugin-kms-azure`.

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_azurekeyvault_tenant_id` | `""` | Azure tenant id. |
| `openbao_seal_azurekeyvault_vault_name` | `""` | Key Vault name. |
| `openbao_seal_azurekeyvault_key_name` | `""` | Key name in the Key Vault. |
| `openbao_seal_azurekeyvault_client_id` | `""` | Client id of the service principal, rendered as `client_id`. Empty uses the managed identity. |
| `openbao_seal_azurekeyvault_client_secret` | `""` | **Secret.** Client secret of the service principal, written as `AZURE_CLIENT_SECRET`. |

### GCP Cloud KMS

External plugin `openbao-plugin-kms-gcp`.

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_gcpckms_project` | `""` | GCP project id. |
| `openbao_seal_gcpckms_region` | `""` | Key ring location (for example `global`), not a compute region. |
| `openbao_seal_gcpckms_key_ring` | `""` | Key ring name. |
| `openbao_seal_gcpckms_crypto_key` | `""` | Crypto key name. |
| `openbao_seal_gcpckms_credentials` | `""` | Path on the node to a service account credentials JSON, rendered as `credentials`. The operator manages this secret file (readable by the service user); empty uses the instance identity. |

### KMS plugin

Used by the three cloud seals. The plugin archive
`<base>/kms-<provider>-v<version>/openbao-plugin-kms-<provider>_linux_<arch>_<micro>.tar.gz`
(`<micro>` is `v1` on amd64, `v8` on arm64) is verified, unpacked and
installed as `<openbao_plugin_dir>/openbao-plugin-kms-<provider>_<version>`
(`openbao:openbao 0750`). The installed name carries the version, so an
upgrade is a new file plus a config change (rolling restart).

| Variable | Default | Description |
|---|---|---|
| `openbao_seal_plugin_version` | `"0.1.1"` | Release version of the KMS plugin (tag `kms-<provider>-v<version>` in openbao/openbao-plugins). Required for cloud seals. |
| `openbao_seal_plugin_base_url` | `"https://github.com/openbao/openbao-plugins/releases/download"` | Base URL for plugin archives and their `.sig` signatures. A mirror (also `file://`) must keep the layout above. |
| `openbao_seal_plugin_checksum` | `""` | Optional pinned SHA-256 (64 lowercase hex) of the **extracted plugin binary** (not the archive) for this host's architecture, from `checksums-kms-<provider>.txt` of the release. When set the role checks it before installing, and renders it as `sha256sum`, which OpenBao re-checks before it runs the plugin. Use a per-architecture map (see [Notes](#notes)). |

---

## Init / unseal

| Variable | Default | Description |
|---|---|---|
| `openbao_init` | `true` | Run `bao operator init` on the bootstrap node when every member answered "not initialized" and the whole cluster is in the play. An unreachable node fails the run instead of counting as uninitialized. |
| `openbao_unseal` | `true` | Unseal Shamir nodes with the keys from the controller init file (after init, on re-runs and after each rolling restart). No effect with auto-unseal seals. With `false` the role leaves sealed nodes sealed and says so. |
| `openbao_init_key_shares` | `5` | Shamir key shares. |
| `openbao_init_key_threshold` | `3` | Shamir key threshold (shares needed to unseal). |
| `openbao_init_recovery_shares` | `5` | Recovery key shares (auto-unseal seals). Recovery keys authorize operations such as generating a root token; they cannot unseal. |
| `openbao_init_recovery_threshold` | `3` | Recovery key threshold (auto-unseal seals). |
| `openbao_init_bootstrap_node` | `"{{ openbao_raft_peers \| first \| default('') }}"` | Inventory name of the node that runs `operator init` (and creates the CA in generate mode). Must be one of `openbao_raft_peers`. |
| `openbao_init_pgp_keys` | `[]` | PGP public keys (paths on the bootstrap node or `keybase:` names), one per share, passed as `-pgp-keys` (Shamir) or `-recovery-pgp-keys` (auto-unseal). With Shamir this requires `openbao_unseal: false`, because the role cannot unseal with wrapped shares. Empty: plain shares, protected by `openbao_init_encrypt_output`. |
| `openbao_init_output_path` | `"{{ '~/.openbao' \| expanduser }}/.openbao_init_{{ openbao_cluster_id }}.json"` | Controller-side file for the init output (root token and unseal or recovery keys), outside any repository. Written once with mode `0600` (parent created `0700` when missing), never overwritten. Later runs read the unseal keys from it, so only a user who can read this file (and the password file) can re-unseal; pipelines must point it at persistent, access-controlled storage. |
| `openbao_init_encrypt_output` | `true` | Encrypt the init output with Ansible Vault in memory before it is written (header `$ANSIBLE_VAULT;1.2;AES256;filter_default`, readable with `ansible-vault view`). |
| `openbao_init_encrypt_password_file` | `""` | Controller-side plain-text Ansible Vault password file used to encrypt the init output and to decrypt it on later runs. Required when `openbao_init` and `openbao_init_encrypt_output` are true: the role fails before installing anything when it is empty. |
| `openbao_cluster_id` | `"{{ openbao_raft_peers \| sort \| first \| default('') }}"` | Stable id in the init file name. Sorted, so reordering the inventory does not change it; set it explicitly in production and keep it once the cluster is initialized. |

---

## Telemetry / logging / TTLs / hardening

| Variable | Default | Description |
|---|---|---|
| `openbao_telemetry_enabled` | `true` | Render the `telemetry` stanza (Prometheus metrics at `/v1/sys/metrics?format=prometheus`). |
| `openbao_telemetry_prometheus_retention_time` | `"24h"` | Prometheus retention time; must be non-zero for the Prometheus format. |
| `openbao_telemetry_disable_hostname` | `true` | Do not prefix metric names with the hostname. |
| `openbao_telemetry_unauthenticated_metrics_access` | `false` | Allow unauthenticated `/v1/sys/metrics` on the API listener. Standby nodes answer the metrics endpoint only this way; prefer a separate metrics-only listener (see [INTEGRATION.md](INTEGRATION.md#prometheus)). |
| `openbao_log_level` | `"info"` | Server log level: `trace`, `debug`, `info`, `warn` or `error`. |
| `openbao_log_format` | `"json"` | Server log format: `standard` or `json`. |
| `openbao_default_lease_ttl` | `"1h"` | Default lease TTL (the Vault role value; the upstream default is `768h`). |
| `openbao_max_lease_ttl` | `"24h"` | Maximum lease TTL (the Vault role value; the upstream default is `768h`). |
| `openbao_enable_file_permissions_check` | `true` | Write `BAO_ENABLE_FILE_PERMISSIONS_CHECK="true"` to `openbao.env`: OpenBao then checks owner and mode of `openbao.hcl`, the plugin directory and plugin binaries at startup (see [Notes](#notes)). |
| `openbao_extra_config_hcl` | `""` | Raw HCL appended verbatim to `openbao.hcl` after the audit block (extra top-level keys and stanzas, for example a metrics-only listener or a second audit device). Must not contain secrets, `disable_mlock` or an `initialize` stanza. `validate-config` checks it as part of the file; the audit guard does not cover it. |

---

## Audit

The role declares one file audit device in `openbao.hcl` (OpenBao blocks
`bao audit enable` through the API by default). The device settings are
**immutable once applied**: OpenBao refuses a changed declarative device at
the next unseal, so the role refuses the change before rendering (see
[TROUBLESHOOTING.md](TROUBLESHOOTING.md#audit-devices-cannot-be-modified-in-place)
for the two-step procedure).

| Variable | Default | Description |
|---|---|---|
| `openbao_audit_enabled` | `true` | Declare the file audit device. Setting it to false later removes the device (its HMAC salt is lost; re-enabling creates a new device). |
| `openbao_audit_path` | `"file"` | Mount path of the device (`audit "file" "<path>"`). Immutable once applied. |
| `openbao_audit_file_path` | `"{{ openbao_audit_dir }}/openbao_audit.log"` | Audit log file; must be inside `openbao_audit_dir` (`stdout` and `discard` are refused). Immutable once applied. |
| `openbao_audit_file_mode` | `"0600"` | Mode OpenBao sets on the log file. Immutable once applied. |
| `openbao_audit_options` | `{}` | Extra device options, rendered sorted as strings (a YAML bool becomes `"true"` or `"false"`), for example `hmac_accessor: "false"` or `elide_list_responses: "true"`. `file_path` and `mode` are refused (use the variables above). Immutable once applied. |
| `openbao_audit_logrotate_enabled` | `true` | Install `/etc/logrotate.d/openbao` for the audit log: daily, compressed, then `systemctl reload openbao.service` so the device reopens its file. Removed when this or `openbao_audit_enabled` is false. |
| `openbao_audit_logrotate_rotate` | `30` | Number of daily rotations to keep. |

---

## systemd / permissions

The role writes `/etc/systemd/system/openbao.service`, which shadows the
packaged unit. It always renders the upstream directives (`ProtectHome=read-only`,
`PrivateTmp`, `PrivateDevices`, `NoNewPrivileges`, `MemorySwapMax=0`) plus
`LimitCORE=0` and an empty `CapabilityBoundingSet=` (`CAP_NET_BIND_SERVICE`
only for a listener port below 1024). No `LimitNOFILE`, `LimitMEMLOCK` or
`CAP_IPC_LOCK`: OpenBao has no mlock.

| Variable | Default | Description |
|---|---|---|
| `openbao_systemd_hardening` | `true` | Add the role's sandboxing directives: `ProtectSystem=strict` with `ReadWritePaths` (data directory, plus the audit directory when auditing is enabled), `ProtectKernelTunables`, `ProtectKernelModules`, `ProtectKernelLogs`, `ProtectControlGroups`, `ProtectClock`, `ProtectHostname`, `RestrictSUIDSGID`, `RestrictRealtime`, `RestrictNamespaces`, `LockPersonality`, `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX AF_NETLINK`, `SystemCallArchitectures=native`, `UMask=0077`. **These are the role's opinion, not upstream guidance.** False renders the upstream `ProtectSystem=full` instead; set it if a directive conflicts with your environment. |
| `openbao_systemd_restart_sec` | `5` | `RestartSec` of the unit (seconds). |
| `openbao_systemd_timeout_stop_sec` | `30` | `TimeoutStopSec` of the unit (seconds). |
| `openbao_tls_key_mode` | `"0640"` | Mode of the TLS private key (owner `root:<openbao_group>`), BYO and generated. |
| `openbao_tls_cert_mode` | `"0644"` | Mode of the TLS certificate and CA files (owner `root:root`). |
| `openbao_manage_selinux` | `true` | Run `restorecon -R` on the config, data, TLS and audit directories (and the plugin directory for cloud seals) on EL hosts with SELinux enabled. No-op elsewhere. |

---

## Service control

| Variable | Default | Description |
|---|---|---|
| `openbao_service_enabled` | `true` | Enable `openbao.service` at boot. |
| `openbao_service_state` | `"started"` | `started` or `stopped`. With `stopped` (maintenance) the init, unseal and restart stages are skipped, a certificate reload does not start the unit, and a pending restart waits; the next run with `started` starts the node and unseals it. |
| `openbao_restart_health_timeout` | `300` | Seconds the rolling restart waits for a restarted node to report unsealed with a known leader before it stops the play (the restart stays pending). With Shamir and `openbao_unseal: false` this is the time an operator has to unseal by hand. |
| `openbao_config_only` | `false` | Render-only mode: renders the files and handles the TLS files (BYO check or generation) but skips install, the in-place change guards, the `validate-config` hook, the trust store, the service, init, unseal and restart; handlers are no-ops. The target needs the service user and group. Used by the `render` Molecule scenario. |

---

## Notes

- **`become: true` at the play level.** The role sets no per-task `become`.
  Controller-side tasks (init file, password file) run on `localhost` with
  `become: false`, as the user who runs `ansible-playbook`.
- **Validation runs before anything is installed.** The argument specs check
  types and choices, then `tasks/validate.yml` checks 26 rules (formats,
  cross-variable rules, required inputs per seal, fail-closed preconditions),
  also under `--tags`. With `rolespec_validate: false` the `validate.yml`
  checks still apply.
- **Required inputs by mode:**

  | Mode | Required |
  |---|---|
  | Every real run with `openbao_init: true` | `openbao_init_encrypt_password_file`, or `openbao_init_encrypt_output: false` |
  | BYO TLS (the default) | certificate, key and CA on every node at the configured paths; SANs cover the host of `openbao_api_addr` and `openbao_tls_leader_servername` |
  | `--limit` day-2 run | `openbao_raft_retry_join_api_addrs` with the API URLs of all members |
  | `static` | `openbao_seal_static_current_key_id`, `openbao_seal_static_current_key` |
  | `transit` | `openbao_seal_transit_address`, `openbao_seal_transit_token` |
  | `awskms` | `openbao_seal_awskms_kms_key_id` (credentials or an instance role) |
  | `azurekeyvault` | `openbao_seal_azurekeyvault_tenant_id`, `openbao_seal_azurekeyvault_vault_name`, `openbao_seal_azurekeyvault_key_name` (client id and secret, or a managed identity) |
  | `gcpckms` | `openbao_seal_gcpckms_project`, `openbao_seal_gcpckms_region`, `openbao_seal_gcpckms_key_ring`, `openbao_seal_gcpckms_crypto_key` |

- **Path variables must be identical on all nodes.** The rolling restart runs
  from the first play host and delegates to each node, using that host's
  paths (the restart marker in the config directory, the CA file). Per-host values
  are supported for the addresses (`openbao_api_addr`, `openbao_cluster_addr`)
  and `openbao_raft_node_id`, not for paths.
- **Audit immutability.** `openbao_audit_path`, `openbao_audit_file_path`,
  `openbao_audit_file_mode` and `openbao_audit_options` cannot change once the
  device exists. The guard compares the device of every play host (old file and
  new settings, whitespace-normalized) and fails before rendering; it sees only
  the hosts in the play. Adding the device (no old stanza anywhere) and
  removing it (`openbao_audit_enabled: false`) pass.
- **What `validate-config` does not check.** The `validate:` hook of
  `openbao.hcl` runs `bao operator validate-config`, which fails the render on
  unknown keys in the top level, listener, telemetry, audit and plugin blocks.
  The `storage` and `seal` stanzas are string maps, so typos in
  `openbao_raft_extra_parameters` and in seal settings pass and only show up
  when OpenBao starts.
- **Per-architecture checksum maps.** `openbao_binary_checksum` and
  `openbao_seal_plugin_checksum` hold the hash for the architecture of the host
  that runs the task. In a mixed fleet use a map keyed by `openbao_arch`; use
  `.get(openbao_arch, '')` rather than `[openbao_arch]`, so an unsupported
  architecture reaches the clear validation message instead of a templating
  error. Values for OpenBao 2.7.3 and `kms-aws-v0.1.1`:

  ```yaml
  # SHA-256 of the release tarball (checksums.txt of v2.7.3)
  openbao_binary_checksum: "{{ {'amd64': '131df9a21cf3dd2dff47107110b369c9974cbdaa482bcbc723b81b3403c68ecd', 'arm64': 'b505b07af4206fdf0e81b747fcb7e9e63f2c2b05e26ead973cce62d0ed07d0e0'}.get(openbao_arch, '') }}"
  # SHA-256 of the extracted plugin binary (checksums-kms-aws.txt of kms-aws-v0.1.1)
  openbao_seal_plugin_checksum: "{{ {'amd64': '0dffe0a159a5bdbeb147a892595fe517de858f48a99a64f33fabc775af0eec76', 'arm64': '986487f17d30123b4730cf412ad2267f7ef4cfe637c75252ed6d13294673bae2'}.get(openbao_arch, '') }}"
  ```

  Update the maps together with `openbao_version` and
  `openbao_seal_plugin_version`.
- **`tasks_from: install_plugin.yml`** installs only the KMS plugin of
  `openbao_seal_type` (the `autounseal` scenario uses it). It depends on role
  defaults and vars and on the signing key at
  `/usr/local/share/openbao/openbao-signing-key.asc`, which a full role run
  installs first. It renders nothing and restarts nothing; it repeats the
  "unverified plugin download" check because it skips `validate.yml`.
- **EPEL caveat (EL).** EPEL ships its own `openbao` package. If EPEL is
  enabled, add `excludepkgs=openbao` to its repository definition, so
  `dnf upgrade` cannot replace the pkgs.openbao.org package (on EL the role's
  hold is the disabled `openbao` repository, which does not stop EPEL).
- **The package TLS pair is unused and must stay.** On first install the
  package generates `/opt/openbao/tls/tls.crt` and `tls.key` (self-signed, no
  SANs). The role leaves them in place: while both exist, the package
  `postinst` of later upgrades exits early and skips its recursive `chown` of
  `/etc/openbao` and `/opt/openbao`. The role's own file names differ, so the
  package pair can never satisfy the BYO TLS check.
- **File permission check ownership.** With
  `openbao_enable_file_permissions_check: true` (and `-config=<file>`),
  OpenBao requires `openbao.hcl`, `openbao_plugin_dir` and every plugin binary
  to be owned by the uid that runs it and not writable by group or others.
  The role therefore makes them `openbao:openbao` (`0640`, `0750`, `0750`).
  The config directory and `openbao.env` stay root-owned (they are not
  checked). With `openbao_systemd_hardening: true`, `ProtectSystem=strict`
  mounts them read-only for the process, so the service user still cannot
  modify its config or plugins. Turning the check off does not change the
  ownership the role applies.
