# Troubleshooting

Failures of the `devopsgroupeu.openbao` role and of the cluster it builds.
Every fail-closed message of the role starts with a fixed first sentence; each
of those sentences is a `###` heading below, so search this page for the first
sentence of your error. The rest of the message names the values involved and
the fix. Entries follow the same shape: **Symptom**, **Cause**, **Fix**.

Checks marked "cluster-wide" run once for the whole play: the message is
printed for one host, and every host of the play fails with it.

---

## Table of Contents

- [Input validation](#input-validation)
- [Cluster membership](#cluster-membership)
- [Install and downloads](#install-and-downloads)
- [TLS](#tls)
- [In-place change guards](#in-place-change-guards)
- [Init and unseal](#init-and-unseal)
- [Rolling restart](#rolling-restart)
- [Cluster operations](#cluster-operations)
- [Useful commands](#useful-commands)

---

## Input validation

These checks (`tasks/validate.yml`) run first on every run, also under
`--tags`, before anything is installed or rendered. Nothing on the hosts has
changed when one of them fails. For variables with a fixed set of values the
role's argument specification usually reports a wrong value even earlier, as
`Validation of arguments failed: value of <variable> must be one of: ...`;
the messages below then only appear when argument validation is turned off
(`rolespec_validate: false`).

### Unsupported openbao_install_method.

**Symptom:** the run stops in "Validate openbao_install_method".

**Cause:** `openbao_install_method` is neither `package` nor `binary`.

**Fix:** use `package` (signed repository, the default) or `binary` (release
tarball, for mirrors and air-gapped hosts).

### Unsupported CPU architecture for OpenBao.

**Symptom:** the run stops in "Validate the CPU architecture"; the message
names `ansible_facts['architecture']`.

**Cause:** `openbao_arch` resolved to an empty value: OpenBao release assets
exist for amd64 (`x86_64`) and arm64 (`aarch64`) only in this role.

**Fix:** run the role on an amd64 or arm64 host. If the host is one of those
but reports another architecture name, set `openbao_arch` to `amd64` or
`arm64` for it.

### Unsupported OS family for OpenBao.

**Symptom:** the run stops in "Validate the OS family".

**Cause:** the host is not Debian family (Debian, Ubuntu) or RedHat family
(EL 9), or facts were not gathered (`gather_facts: false`), so
`ansible_facts['os_family']` is unknown.

**Fix:** gather facts in the play and run the role on a supported host.

### Unsupported openbao_version.

**Symptom:** the run stops in "Validate openbao_version".

**Cause:** the value is not an exact `X.Y.Z` version (for example `2.7` or
`v2.7.3`), or it is below 2.7.2. 2.7.0 and 2.7.1 carry the namespace deletion
bug fixed in 2.7.2; 2.6.x is not supported (it needs `sha256sum` on plugin
stanzas and lacks `tls_auto_reload`).

**Fix:** set `openbao_version` to a quoted exact version of 2.7.2 or later,
for example `"2.7.3"`.

### Unsupported openbao_seal_type.

**Symptom:** the run stops in "Validate openbao_seal_type".

**Cause:** the seal is not one of `shamir`, `static`, `transit`, `awskms`,
`azurekeyvault`, `gcpckms`. Other OpenBao seals (`pkcs11`, `kmip`, `ocikms`,
...) are not supported by the role.

**Fix:** pick a supported seal. Do not add another seal through
`openbao_extra_config_hcl`: on an initialized node the seal guard reads the
first seal stanza of `openbao.hcl` and would refuse every later run.

### Invalid OpenBao TLS version range.

**Symptom:** the run stops in "Validate the TLS version range".

**Cause:** `openbao_tls_min_version` or `openbao_tls_max_version` is not
`tls12` or `tls13`, or the minimum is `tls13` while the maximum is `tls12`.

**Fix:** use `tls12` or `tls13` for both, with the minimum not above the
maximum. TLS 1.0 and 1.1 are not offered.

### Unsupported openbao_tls_cert_key_type.

**Symptom:** the run stops in "Validate openbao_tls_cert_key_type".

**Cause:** the key type for generated certificates is not `ECC` or `RSA`
(the value is case-sensitive).

**Fix:** set `ECC` (with `openbao_tls_cert_curve`) or `RSA` (with
`openbao_tls_cert_size`).

### Invalid openbao_log_level.

**Symptom:** the run stops in "Validate openbao_log_level".

**Cause:** the level is not one of `trace`, `debug`, `info`, `warn`, `error`.

**Fix:** use one of those values.

### Invalid openbao_log_format.

**Symptom:** the run stops in "Validate openbao_log_format".

**Cause:** the format is not `standard` or `json`.

**Fix:** use `standard` (human readable) or `json` (default).

### Invalid openbao_service_state.

**Symptom:** the run stops in "Validate openbao_service_state".

**Cause:** the role supports only `started` and `stopped`. `restarted` and
`reloaded` are not service states the role manages: restarts go through the
health-gated rolling restart, certificate changes through a reload.

**Fix:** use `started`, or `stopped` for maintenance. To force a restart of a
node, see [Pending restart after a failed run](#pending-restart-after-a-failed-run).

### Invalid OpenBao listener address.

**Symptom:** the run stops in "Validate the listener addresses".

**Cause:** `openbao_listen_address` or `openbao_cluster_listen_address` is not
`host:port`, for example `0.0.0.0` without a port.

**Fix:** write both as `host:port`, for example `0.0.0.0:8200` and
`0.0.0.0:8201`.

### Invalid openbao_gpg_key_fingerprint.

**Symptom:** the run stops in "Validate openbao_gpg_key_fingerprint".

**Cause:** the value is not a 40 hex digit primary key fingerprint, for
example a 16 digit long key id or a value with a `0x` prefix.

**Fix:** use the full primary fingerprint (spaces allowed). The OpenBao
release key is `66D15FDD87287219C8E15478D200CD702853E6D0`.

### Invalid pinned checksum.

**Symptom:** the run stops in "Validate the pinned checksums".

**Cause:** `openbao_binary_checksum` or `openbao_seal_plugin_checksum` is
neither empty nor 64 lowercase hex characters: uppercase hex, a `sha256:`
prefix, or a truncated value.

**Fix:** use the bare lowercase SHA-256. For mixed architectures use a map
keyed by `openbao_arch` (see
[CONFIGURATION.md](CONFIGURATION.md#notes)).

### Invalid openbao_raft_peers.

**Symptom:** the run stops in "Validate openbao_raft_peers" on every host.

**Cause:** the peer list is empty, does not contain this host, or has one
entry while the play has several hosts. The last case is the usual one: the
inventory group is not named `openbao`, so the default falls back to
`[inventory_hostname]` on each host and every host would initialize its own
one-node cluster.

**Fix:** name the inventory group `openbao`, or set `openbao_raft_peers` to the
inventory names of all members. Run independent single nodes in separate
plays.

### openbao_init_bootstrap_node is not a cluster member.

**Symptom:** the run stops in "Validate openbao_init_bootstrap_node".

**Cause:** the node set to run `operator init` is not in `openbao_raft_peers`,
typically a renamed or removed host.

**Fix:** set it to one of the members, or remove the override (the default is
the first entry of `openbao_raft_peers`).

### Invalid Shamir or recovery share settings.

**Symptom:** the run stops in "Validate the Shamir and recovery share
settings".

**Cause:** a share count or threshold is below 1, or a threshold exceeds its
share count. Both pairs are checked, whatever the seal.

**Fix:** keep `1 <= threshold <= shares` for `openbao_init_key_threshold` /
`openbao_init_key_shares` and for `openbao_init_recovery_threshold` /
`openbao_init_recovery_shares`.

### openbao_init_pgp_keys does not match the share count.

**Symptom:** the run stops in "Validate the number of openbao_init_pgp_keys".

**Cause:** OpenBao wraps one share per PGP key. The list must have exactly
`openbao_init_key_shares` entries with Shamir, or
`openbao_init_recovery_shares` entries with an auto-unseal seal.

**Fix:** give one key per share, or leave the list empty.

### PGP-wrapped Shamir shares require openbao_unseal: false.

**Symptom:** the run stops in "Validate that PGP-wrapped Shamir shares are not
used for unsealing".

**Cause:** with PGP keys every unseal share in the init output is encrypted
for its holder, so the role cannot unseal with them.

**Fix:** set `openbao_unseal: false` and unseal by hand with the decrypted
shares (also after every restart: the rolling restart then waits
`openbao_restart_health_timeout` seconds per node for you), or leave
`openbao_init_pgp_keys` empty.

### Invalid openbao_api_addr.

**Symptom:** the run stops in "Validate openbao_api_addr".

**Cause:** the URL is not `<scheme>://<host>:<port>` without a path, or its
scheme does not match the listener: `https` is required with
`openbao_tls_enabled: true`, `http` without TLS. A per-host `https://` address
left in the inventory after turning TLS off is the usual case.

**Fix:** correct the URL. The role probes each node at this address, so it
must also be reachable from the node itself and covered by its certificate.

### Unsupported content in openbao_extra_config_hcl.

**Symptom:** the run stops in "Validate openbao_extra_config_hcl".

**Cause:** the raw HCL mentions `disable_mlock` (`disable_mlock = false` is a
fatal error on OpenBao 2.x and the key does nothing otherwise) or contains an
`initialize` stanza (declarative self-initialization is not supported; the
role runs `bao operator init`). Any mention of `disable_mlock` counts, also in
a comment.

**Fix:** remove those lines. Swap is handled by `MemorySwapMax=0` in the unit.

### openbao_audit_options must not set file_path or mode.

**Symptom:** the run stops in "Validate openbao_audit_options".

**Cause:** `file_path` or `mode` is in the extra device options.

**Fix:** use `openbao_audit_file_path` and `openbao_audit_file_mode` instead.

### openbao_audit_file_path must be inside openbao_audit_dir.

**Symptom:** the run stops in "Validate openbao_audit_file_path".

**Cause:** the audit log path is `stdout`, `discard`, outside
`openbao_audit_dir`, or contains a `..` component. The unit only makes
`openbao_audit_dir` writable (`ReadWritePaths`) and the logrotate rule
follows the file.

**Fix:** use a file below `openbao_audit_dir`, or move `openbao_audit_dir`.
On an existing cluster this is an audit device change, see
[Audit devices cannot be modified in place.](#audit-devices-cannot-be-modified-in-place).

### A secret variable contains a newline or NUL character.

**Symptom:** the run stops in "Validate that secret variables are
single-line"; the message names the variables, never the values.

**Cause:** secrets are written as one quoted line to `openbao.env`, so line
breaks and NUL bytes cannot be represented. A YAML block scalar (`key: |`)
in a vaulted vars file adds a trailing newline.

**Fix:** write the value as a plain or quoted scalar, use `|-`, or append
`| trim`.

### Missing required settings for the selected seal.

**Symptom:** the run stops in "Validate the settings required by the selected
seal"; the message lists the empty variables.

**Cause:** a seal input is empty. Required: `static` key id and key; `transit`
address and token; `awskms` key id; `azurekeyvault` tenant id, vault name and
key name; `gcpckms` project, region, key ring and crypto key. When a secret
lives in an Ansible Vault file, the file is often in a directory Ansible does
not load (it must be `group_vars/` next to the inventory or the playbook).

**Fix:** set the listed variables; secrets come from Ansible Vault.

### openbao_seal_plugin_version is required for cloud seals.

**Symptom:** the run stops in "Validate openbao_seal_plugin_version for cloud
seals".

**Cause:** `awskms`, `azurekeyvault` and `gcpckms` run as external KMS
plugins on OpenBao 2.7, and the plugin release to install is empty.

**Fix:** set `openbao_seal_plugin_version` to a release of
openbao/openbao-plugins (default `0.1.1`).

### Unverified binary download refused.

**Symptom:** the run stops in "Validate that the binary download can be
verified".

**Cause:** binary method with `openbao_binary_verify_gpg: false` and no
`openbao_binary_checksum`: nothing would verify the tarball.

**Fix:** keep GPG verification on, or pin the tarball SHA-256 for each
architecture.

### Unverified KMS plugin download refused.

**Symptom:** the run stops in "Validate that the KMS plugin download can be
verified" (full run) or in "Refuse an unverified KMS plugin download"
(`tasks_from: install_plugin.yml`).

**Cause:** a cloud seal with `openbao_binary_verify_gpg: false` and no
`openbao_seal_plugin_checksum`: nothing would verify the plugin.

**Fix:** keep GPG verification on, or pin the SHA-256 of the extracted plugin
binary for each architecture.

### Init output encryption requested but no password file set.

**Symptom:** the run stops in "Validate that encrypted init output has a
password file" before anything is installed, or, right before `operator init`,
in "Assert the Ansible Vault password file is usable".

**Cause:** `openbao_init_encrypt_output` is true but
`openbao_init_encrypt_password_file` is empty, or (second task) the file is
missing, unreadable, empty or holds only whitespace on the controller (for
example `echo "$UNSET_SECRET" > pass` in a pipeline writes a single line
break). The role reads the password like `ansible-vault
--vault-password-file` does, with leading and trailing whitespace stripped,
and checks it before init, because the init output exists only once: keys
that cannot be stored safely would be lost, and keys "encrypted" with an
empty password are not protected.

**Fix:** point `openbao_init_encrypt_password_file` at a controller-side
Ansible Vault password file (for example
`openssl rand -base64 32 > ~/.openbao-vault-pass && chmod 0600 ~/.openbao-vault-pass`),
or set `openbao_init_encrypt_output: false` to accept a plain-text init file.

---

## Cluster membership

### Cluster members are missing from this play.

**Symptom:** the run stops in "Validate that every cluster member is in the
play" right after validation, before anything is installed.

**Cause:** without explicit `openbao_raft_retry_join_api_addrs` the role
builds the `retry_join` addresses from the facts of the other members, so all
of them must be in the play. A `--limit` run, or a member that was unreachable
when facts were gathered, leaves them out.

**Fix:** run against the whole group, or, for a day-2 `--limit` run, set
`openbao_raft_retry_join_api_addrs` to the API URLs of all members (exactly
each node's `openbao_api_addr`). Initialization always needs the whole group.

### The role needs the linear strategy without serial.

**Symptom:** the run stops in "Validate the play strategy and batch size"
right after validation, before anything is installed. With `strategy: free`
the message names a member other than the first one in play order; with
`serial` it names the batch.

**Cause:** the cluster-wide tasks read the other members' facts through
`hostvars` and drive the rolling restart from `run_once` tasks. That needs
the default linear strategy with every member in one batch. Under the free
strategy Ansible ignores `run_once`, so every host would run its own rolling
restart at the same time; with `serial` the members of later batches have no
facts yet.

**Fix:** remove `strategy` and `serial` from the play. The rolling behaviour
(one node at a time, quorum check, leader last) is inside the role. For a
day-2 run on part of the cluster use `--limit` with
`openbao_raft_retry_join_api_addrs`.

---

## Install and downloads

### OpenBao signing key fingerprint mismatch.

**Symptom:** the run stops in "Assert the signing key fingerprint"; the
message names the file, the fingerprint found and the expected one.

**Cause:** the key file does not hold exactly one key with
`openbao_gpg_key_fingerprint`: a wrong pin, a proxy or captive portal that
returned something else (zero keys found), a mirror that bundles several keys,
or an installed key file that was replaced. An installed file at
`/usr/local/share/openbao/openbao-signing-key.asc` is checked on every run and
never deleted by the role.

**Fix:** check `openbao_gpg_key_url` and the pin. To rotate the key on
purpose (upstream published a new release key):

1. Confirm the new primary fingerprint out of band (openbao.org).
2. Set `openbao_gpg_key_url` and `openbao_gpg_key_fingerprint` to the new
   values.
3. Remove `/usr/local/share/openbao/openbao-signing-key.asc` on every node;
   the next run downloads, verifies and installs the new key. On EL the old
   key stays in the RPM database until you remove it with `rpm -e
   gpg-pubkey-<id>` (`rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE} %{SUMMARY}\n'`
   lists them).
4. Run the role.

### No package for the pinned version on EL

**Symptom:** on an EL host with `openbao_install_method: package` the run
stops in "Install the openbao package (RHEL/EL, exact version pin)" with
`No package openbao-<version> available`, while the same version installs on
Debian or Ubuntu.

**Cause:** the upstream RPM repository at pkgs.openbao.org does not carry
every patch release. Its metadata is updated incrementally for each release
(`createrepo_c --update --includepkg`), and a release published on the same
day as another one is missing from it: on 2026-11-25 the x86_64 and aarch64
metadata listed 2.6.6, 2.7.0, 2.7.1 and 2.7.3, but neither 2.7.2 (the
minimum this role supports) nor 2.6.5. The APT repository has them.

**Fix:** pin a version the repository lists
(`dnf --disablerepo='*' --enablerepo=openbao list --showduplicates openbao`),
or install that version with `openbao_install_method: binary` (signed release
tarball), or point `openbao_rpm_repo_baseurl` at a mirror that holds the RPM.

### Downgrade refused.

**Symptom:** the run stops in "Refuse a downgrade of the openbao package" or
"Refuse a downgrade of the bao binary".

**Cause:** the installed version is newer than `openbao_version`. OpenBao
makes no guarantee that an older version can read data written by a newer
one.

**Fix:** set `openbao_version` to the installed version or later. To really
go back, restore a raft snapshot taken with the older version on freshly
installed nodes.

### Release signature does not match the pinned key.

**Symptom:** the run stops in "Assert checksums.txt is signed by the pinned
key" (binary method) or "Assert the KMS plugin archive is signed by the
pinned key"; the message shows the gpg exit code, the signer fingerprints and
GnuPG status lines such as `BADSIG` or `NO_PUBKEY`.

**Cause:** `checksums.txt` (tarball) or the plugin archive does not carry a
valid signature whose primary key is the pin: a mirror that serves altered or
mismatched files, a truncated download, or a signing key rotation upstream.
Nothing was installed.

**Fix:** check the files behind `openbao_binary_base_url` or
`openbao_seal_plugin_base_url` against the GitHub release. After an upstream
key rotation follow
[OpenBao signing key fingerprint mismatch.](#openbao-signing-key-fingerprint-mismatch).

### Release checksum verification failed.

**Symptom:** the run stops in "Assert the tarball matches checksums.txt"; the
message shows the `sha256sum` output.

**Cause:** the downloaded tarball is missing or its hash is not the one in the
signed `checksums.txt`, for example a mirror with a rebuilt tarball. Nothing
was installed.

**Fix:** fix the mirror. A pinned `openbao_binary_checksum` that does not
match fails earlier, in "Download the release tarball", with the `get_url`
message `The checksum for <file> did not match <pin>; it was <hash>.`: pin the
hash of the right version and architecture (`checksums.txt` of the release).

### KMS plugin checksum mismatch.

**Symptom:** the run stops in "Assert the KMS plugin binary matches the pinned
checksum".

**Cause:** `openbao_seal_plugin_checksum` is not the SHA-256 of the extracted
plugin binary: usually the hash of the `.tar.gz` archive, the hash of the
other architecture, or a hash of another plugin version. The plugin is not
installed.

**Fix:** pin the binary hash from `checksums-kms-<provider>.txt` of the plugin
release (the file lists binaries and archives; take the line of
`openbao-plugin-kms-<provider>_linux_<arch>_<micro>` without `.tar.gz`), per
architecture.

### KMS plugin not found or rejected by the permission check

**Symptom:** a node with a cloud seal does not start; the journal shows
`Error configuring seal "awskms": ...` with `error while validating the command
path`, `plugin directory not configured`,
`path "/opt/openbao/plugins/..." is not owned by my uid <uid>` or
`path "..." has insecure permissions ...`.

**Cause:** the plugin binary is missing or was changed by hand, or
`openbao_plugin_dir` (or the binary) is not owned by the service user. With
`BAO_ENABLE_FILE_PERMISSIONS_CHECK=true` OpenBao requires the plugin directory,
every plugin binary and `openbao.hcl` to be owned by the uid it runs as and
not writable by group or others. The plugin directory must not be a symbolic
link.

**Fix:** re-run the role: it re-installs a missing plugin and resets the
owners and modes (`openbao:openbao 0750`). Do not chown the plugin directory to
root while the permission check is on.

### Cloud seal without a plugin

**Symptom:** a node does not start with `Error configuring seal "awskms":
unknown wrapper: awskms` (or `azurekeyvault`, `gcpckms`).

**Cause:** since OpenBao 2.7.0 the cloud seals are not built into the binary;
they need `plugin_directory`, a `plugin "kms"` stanza and the plugin binary.
The role renders and installs all three for `awskms`, `azurekeyvault` and
`gcpckms`, so this appears when a seal stanza was added by hand or through
`openbao_extra_config_hcl`.

**Fix:** set `openbao_seal_type` and the seal variables instead of writing the
stanza yourself. For a seal change on an existing cluster see
[Seal migration](#seal-migration).

---

## TLS

### BYO TLS file not found.

**Symptom:** the run stops in "Assert that the BYO TLS files exist"; the
message lists each missing `variable = path`.

**Cause:** with `openbao_tls_generate_certs: false` (the default) the
certificate, key and CA must already be on every node. The role never copies
them.

**Fix:** distribute the files first (for example a play before the role), fix
the path variables, or set `openbao_tls_generate_certs: true` for dev/test.

### x509 and SAN errors

**Symptom:** one of:

- a joining node logs `x509: certificate is valid for <names>, not
  openbao.cluster.internal` or `x509: certificate signed by unknown authority`
  during `retry_join` and never reports initialized;
- the role's probes ("Read the seal status", the unseal and wait tasks) fail
  with `CERTIFICATE_VERIFY_FAILED`, `Hostname mismatch` or `IP address
  mismatch, certificate is not valid for '<host>'`.

**Cause:** `retry_join` verifies the leader's certificate against
`openbao_tls_ca_file` with `openbao_tls_leader_servername` as the server name,
so every node certificate must carry that DNS SAN (Go ignores the CN). The
role probes each node at its own `openbao_api_addr`, so each certificate must
also cover that host (`IP:` for an address, `DNS:` for a name). All nodes must
use certificates from the CA in `openbao_tls_ca_file`.

**Fix:** check the SANs and the issuer:

```bash
openssl x509 -in /opt/openbao/tls/openbao-cert.pem -noout -ext subjectAltName -issuer
```

BYO: re-issue the certificate with `DNS:<openbao_tls_leader_servername>` and
the host of `openbao_api_addr` (plus a load balancer name if clients use one),
`serverAuth` usage, and `clientAuth` too when
`openbao_tls_disable_client_certs: false`. Generated mode: add names with
`openbao_tls_extra_sans`; the role re-signs and reloads the node.

### ACME started unexpectedly

**Symptom:** OpenBao tries to obtain a certificate from Let's Encrypt (ACME,
cache under `/var/lib/openbao/certmagic`) or binds ports 80 and 443.

**Cause:** OpenBao enables ACME on a TCP listener that has neither
`tls_cert_file` nor `tls_disable = true`. The role always renders one of them
for its API listener, so the cause is an extra listener in
`openbao_extra_config_hcl`, or the packaged `openbao.hcl` started before the
role rendered its own.

**Fix:** give every extra listener `tls_cert_file` and `tls_key_file`, or
`tls_disable = true`, and let the role render the config before the first
start.

---

## In-place change guards

These guards (`tasks/config.yml`) compare the existing `openbao.hcl` and the
data directory of each node with the new settings (the node id and seal
guards only on nodes that hold raft data) and fail before anything is
rendered. A run that a guard refused may still have installed a new package,
binary or plugin earlier in the same run; the restart that change needs is
recorded and happens on the next successful run (see
[Pending restart after a failed run](#pending-restart-after-a-failed-run)).

### Data directory holds another storage backend.

**Symptom:** the run stops in "Refuse a data directory of another storage
backend"; the message lists the files it found in `openbao_data_dir`.

**Cause:** the data directory has no `vault.db` (the raft marker) but is not
empty. Since 2.7.1 the package's own `/etc/openbao/openbao.hcl` uses
`storage "pebbledb"` in `/opt/openbao/data`, the role's default data
directory, so a node that was started once with the packaged config (the
upstream quick start) holds a pebble store there (`CURRENT`, `MANIFEST-*`,
`OPTIONS-*`, `*.sst`, `*.log`). Rendered over it, raft storage would start an
empty, uninitialized node next to that data, and a later run would initialize
a new cluster without any message about the old secrets.

**Fix:** if the data matters, move it to raft first: stop openbao and run
`bao operator migrate` with a migration config (`storage_source "pebbledb"`
with the old path, `storage_destination "raft"` with an empty directory and
`cluster_addr`, see https://openbao.org/docs/commands/operator/migrate/),
then point `openbao_data_dir` at the raft directory (or move it into place).
If it holds nothing you need, stop openbao and empty the directory
(`find /opt/openbao/data -mindepth 1 -delete`), then re-run. The guard
ignores `lost+found`, so a data directory on its own file system passes.

### Raft node id change refused.

**Symptom:** the run stops in "Refuse a raft node id change"; the message
names the old and the new node id.

**Cause:** the node has raft data and `openbao_raft_node_id` differs from the
`node_id` in its `openbao.hcl`. The default is `inventory_hostname`, so
renaming a host in the inventory changes it. Raft cannot rename a member.

**Fix:** pin the old id for that host (`openbao_raft_node_id: <old id>`), or
let the node re-join under the new id:

1. From another member with a token: `bao operator raft remove-peer <old id>`.
2. On the node: `systemctl stop openbao`, then empty `openbao_data_dir`
   (`find /opt/openbao/data -mindepth 1 -delete`).
3. Run the role against the whole group: the node joins through `retry_join`
   with the new id and is unsealed.

### Seal type change refused.

**Symptom:** the run stops in "Refuse a seal type change"; the message names
the seal in `openbao.hcl` and the requested one.

**Cause:** changing the seal of an initialized node is a seal migration, which
OpenBao performs only through `bao operator unseal -migrate`.

**Fix:** keep the old `openbao_seal_type`, or migrate by hand (see
[Seal migration](#seal-migration)). If the node was never initialized, empty
`openbao_data_dir` first.

### Seal key change refused.

**Symptom:** the run stops in "Refuse a seal key change"; the message lists
each changed setting with its old and its new value.

**Cause:** the seal type stays the same, but a setting that selects the key
changed: `current_key_id` (static), `mount_path`, `key_name` or `namespace`
(transit), `region` or `kms_key_id` (awskms), `vault_name` or `key_name`
(azurekeyvault), `project`, `region`, `key_ring` or `crypto_key` (gcpckms).
Unsealing with another key is a seal migration as well; rendered as a plain
config change it would reach every node, and each restarted node would stay
sealed. The guard compares the rendered values, so writing the same KMS key
in another form (key id, key ARN or `alias/<name>`) counts as a change too.
The transit `address` is not compared: the unsealer may move to another DNS
name.

**Fix:** restore the old values, or migrate to the new key by hand first (see
[Seal migration](#seal-migration); upstream does not support an awskms to
awskms migration). The static seal key cannot be rotated through the role:
it renders no `previous_key` / `previous_key_id`, so keep
`openbao_seal_static_current_key` and its id once the cluster is initialized.
A new static key value under the same id is not visible to the guard (the key
is a secret in `openbao.env`) and leaves restarted nodes sealed the same way.

### Audit devices cannot be modified in place.

**Symptom:** the run stops in "Refuse an in-place audit device change"
(cluster-wide); the message lists the different audit blocks it found.

**Cause:** the audit stanza in the existing `openbao.hcl` of a play host, or
the new settings of another play host, differ from each other. OpenBao refuses
a modified declarative audit device ("modifications to audit devices are not
allowed") when a node becomes active, so rolling the change out would leave
the cluster without a leader after the next failover. The guard compares
whitespace-normalized blocks and sees only the hosts in the play.

**Fix:** change the device in two runs against the whole cluster:

1. Run with `openbao_audit_enabled: false`. The rolling restart applies it, and
   the node that becomes active removes the device (its HMAC salt is lost).
   The unit keeps `openbao_audit_dir` writable meanwhile, because every node
   still opens the stored device when it unseals. A reload (SIGHUP) of the
   active node, for example the nightly logrotate, can remove the device
   before the restart reaches it.
2. Confirm the device is gone before going on: `bao audit list` (with a token)
   must no longer list `file/` (with no device left it prints
   `No audit devices are enabled.`). The removal happens only when a node with the new config became active, so if
   step 1 failed before the leader was restarted, the old device still exists
   and step 3 would hit exactly the error the guard prevents. Re-run step 1
   until it completes.
3. Run with `openbao_audit_enabled: true` and the new values. This creates a
   new device with a new HMAC salt.

A second device declared through `openbao_extra_config_hcl` is immutable in
the same way but not covered by the guard.

### Seal migration

**Symptom:** you need another seal or seal key for an initialized cluster (for
example Shamir to transit or AWS KMS, or another transit key), and the role
refuses with [Seal type change refused.](#seal-type-change-refused) or
[Seal key change refused.](#seal-key-change-refused).

**Cause:** the role does not migrate seals (out of scope); OpenBao does it with
the old and the new seal configured side by side.

**Fix:** follow the upstream seal migration procedure
(https://openbao.org/docs/concepts/seal/#seal-migration) by hand, then hand the
cluster back to the role:

1. Take a raft snapshot (`bao operator raft snapshot save <file>`).
2. For a cloud seal, install its plugin first with `tasks_from:
   install_plugin.yml` and the new `openbao_seal_type` and plugin variables.
3. On every node, add the new seal stanza (and for cloud seals
   `plugin_directory` and the `plugin "kms"` stanza) to `openbao.hcl`, mark an
   old auto-unseal stanza `disabled = "true"`, and put new credentials into
   `openbao.env`. Do not run the role during the migration: it would re-render
   both files. Do not reload either (`systemctl reload`, logrotate): a reload
   applies plugin stanzas at once.
4. Restart the standbys one at a time with this config. After each restart run
   `bao operator unseal -migrate` on that node with the old unseal keys
   (Shamir) or recovery keys (auto-unseal), and wait until it is back as a
   standby before you take the next one down, so raft keeps its quorum.
5. Step down the active node (`bao operator step-down`, needs a token). A
   migrated standby becomes active and performs the migration: wait for
   `seal migration complete` in its log, then give the migration a moment to
   replicate to all nodes.
6. On the old active node, replace the seal configuration with the new seal
   only (no old stanza, no `disabled` stanza) and restart it, without
   `-migrate`. It auto-unseals with the new seal, or needs the new unseal keys
   for Shamir.
7. On the other nodes, remove the old (disabled) seal stanza from
   `openbao.hcl` as well, so every file names only the new seal: the seal
   guards read the seal of the current file. Then set `openbao_seal_type` and
   its variables in the inventory and run the role. It renders its own
   version of the file on every node and restarts them one at a time.

The init file on the controller keeps the key set of the original init. After
a migration to Shamir (the recovery keys become the unseal keys) the role
finds no `unseal_keys_hex` there: set `openbao_unseal: false` and unseal by
hand, or add those keys to the file as `unseal_keys_hex` and
`unseal_threshold` with `ansible-vault edit`.

---

## Init and unseal

### Initialization needs the whole cluster in the play.

**Symptom:** the run stops in "Assert the whole cluster is in the play"
(cluster-wide), before `operator init`.

**Cause:** no probed play host reports initialized, but a member of
`openbao_raft_peers` was not probed: it is not in the play (`--limit`), it
failed earlier in the run (for example its API did not answer the probe), or
it runs with `openbao_service_state: stopped`. The role initializes only when
it can see every member answer "not initialized", so it never creates a
second cluster next to an existing one.

**Fix:** run the first bootstrap against all members, without `--limit` and
with every member started, and fix the member that failed.

### A previous init file exists for this cluster id.

**Symptom:** the run stops in "Assert no previous init file exists"
(cluster-wide), before `operator init`; nothing was initialized.

**Cause:** `openbao_init_output_path` already exists on the controller, but no
node reports initialized: the nodes were re-provisioned or their data
directories emptied, or two clusters share an `openbao_cluster_id`. The role
never overwrites key material.

**Fix:** if the old cluster is really gone, move the old file away (keep it
until you are sure) and re-run. If this is another cluster, give it its own
`openbao_cluster_id` (or `openbao_init_output_path`). Never remove the file
while any node may still hold data of that cluster.

### Init succeeded but the init output could not be stored.

**Symptom:** the run stops in "Fail because the init output could not be
stored", after `operator init`.

**Cause:** writing the init output (root token and keys) to the controller
failed, for example a full disk or a password file that became unreadable.
The keys of the new cluster are lost, so it cannot be unsealed after a
restart. The new cluster holds no data yet.

**Fix:** wipe and re-initialize. On every node:

```bash
systemctl stop openbao
find /opt/openbao/data -mindepth 1 -delete
```

Fix the cause (directory, permissions, password file), then run the role
again; it initializes a new cluster.

### No init file was found for an initialized, sealed cluster.

**Symptom:** the run stops in "Assert unseal keys are available for the sealed
nodes" (cluster-wide; the message lists the sealed nodes and the path it
looked at), or in "Assert unseal keys are available for the restart" (the
message lists the nodes that need a restart; nothing was restarted).

**Cause:** Shamir with `openbao_unseal: true`, and no unseal keys were found at
`openbao_init_output_path` while some nodes are sealed (after a reboot or
restart) or need a rolling restart (they would come back sealed):
another controller or user ran the bootstrap (the default path is under the
home directory of the user that ran it), the `openbao_cluster_id` changed, or
a CI runner lost its workspace.

**Fix:** point `openbao_init_output_path` (or `openbao_cluster_id`) at the init
file of this cluster, or set `openbao_unseal: false` and unseal by hand.
Pipelines must keep the init file on persistent, access-controlled storage.
Pending restarts stay recorded and run on the next run that finds the keys.

### Could not decrypt the init file.

**Symptom:** the run stops in "Fail because the init file could not be
decrypted" (cluster-wide); nothing on the nodes was changed by this step.

**Cause:** the init file is Ansible Vault encrypted and the password in
`openbao_init_encrypt_password_file` is not the one it was encrypted with
(rotated or wrong password file, or the variable is empty), or the file is
corrupt or not JSON after decryption.

**Fix:** point `openbao_init_encrypt_password_file` at the right password
file. After a password rotation, re-encrypt the init file with the new
password:

```bash
ansible-vault rekey --vault-password-file ~/.openbao-vault-pass.old \
  --new-vault-password-file ~/.openbao-vault-pass ~/.openbao/.openbao_init_<cluster id>.json
```

### The nodes did not agree on one leader.

**Symptom:** the run stops in "Assert every node reports the same leader"
(cluster-wide); the message lists the leader each node reports. This is the
"did not form a cluster" case.

**Cause:** the nodes are not one Raft cluster: some were initialized
separately (for example single-node runs with a misconfigured
`openbao_raft_peers`), their peer lists or `retry_join` addresses differ, or a
node could not reach the others on the cluster port (8201).

**Fix:** compare `bao operator raft list-peers` on the nodes (with a token)
and the rendered `retry_join` blocks in `/etc/openbao/openbao.hcl`. A node that
belongs to a separate cluster must be stopped, its data directory emptied, and
the role re-run so it joins; check the firewall between the nodes on 8200 and
8201.

### Node sealed after a reboot

**Symptom:** after a reboot or a manual restart a Shamir node reports
`sealed: true`.

**Cause:** Shamir nodes always start sealed; nothing outside an Ansible run
unseals them.

**Fix:** re-run the role: it unseals every initialized, sealed node with the
keys from the init file (`openbao_unseal: true`). By hand:

```bash
export BAO_ADDR=https://<node address>:8200 BAO_CACERT=/opt/openbao/tls/openbao-ca.pem
bao operator unseal   # repeat until the threshold is reached
```

For unattended restarts use an auto-unseal seal (`transit`, `awskms`,
`azurekeyvault`, `gcpckms`).

### Node stays sealed after bao operator rekey

**Symptom:** after a Shamir rekey (`bao operator rekey`, the rotation of the
unseal keys), the next rolling restart or post-reboot run stops in "Unseal the
restarted node" or one of the "Unseal ..." tasks with censored output (the
tasks are `no_log`), or in the health gate, and the node stays sealed.

**Cause:** the role unseals with `unseal_keys_hex` and `unseal_threshold`
from the init file written at bootstrap. A rekey replaces the key shares (and
maybe the threshold) in the cluster, not in that file, so the role submits
the old shares.

**Fix:** after every rekey, write the new shares into the init file on the
controller, before the next restart:

```bash
ansible-vault edit --vault-password-file ~/.openbao-vault-pass \
  ~/.openbao/.openbao_init_<cluster id>.json
# replace unseal_keys_hex (and unseal_keys_b64) with the new shares,
# set unseal_threshold to the new threshold
```

Or set `openbao_unseal: false` and unseal by hand from then on. A recovery
key rekey (`bao operator rekey -target=recovery`, auto-unseal seals) does not
affect the role, which never uses recovery keys.

### Auto-unseal node stays sealed

**Symptom:** a node with an auto-unseal seal starts but stays sealed; the
journal shows errors of the seal (transit token, KMS access); the role waits
in "Wait until every node is unsealed" and then fails.

**Cause:** the unsealer or KMS is unreachable, its key was disabled or
deleted, or its credential is no longer valid (for example an expired transit
token). Recovery keys cannot unseal: without the seal backend the cluster
stays sealed. A new credential in `openbao.env` reaches a node only when the
process restarts, and the role's restart stage comes after the unseal wait.

**Fix:** restore the seal backend. For a new credential (for example a new
periodic orphan token, see [`examples/autounseal_transit.yml`](../examples/autounseal_transit.yml)):
update the vaulted variable, render it with `--tags config` (this writes
`openbao.env` and records a pending restart; see
[Running only some stages with --tags](#running-only-some-stages-with---tags)),
restart `openbao` on each node by hand so it reads the new value, then run the
role normally; it restarts each flagged node once more, one at a time, and
clears the restart markers. Never delete the KMS or transit key: the cluster
data cannot be recovered without it.

---

## Rolling restart

### Quorum check failed before restart.

**Symptom:** the run stops in "Fail because a member is not healthy"
(cluster-wide); the message names the member URLs that failed and the nodes
whose restart is still pending.

**Cause:** before restarting a node the role requires every other member to
answer `sys/health?standbyok=true` with 200 (active or unsealed standby).
A member was sealed, stopped or unreachable from the node about to restart, so
restarting now could cost the cluster its quorum. Nothing was restarted.

**Fix:** bring that member back (start it, unseal it) and re-run. The pending
restarts stay recorded and run on the next run.

### Node did not become healthy after restart.

**Symptom:** the run stops in "Fail because the restarted node is not healthy"
(cluster-wide); the message names the node, the step that failed and its
error.

**Cause:** the restarted node did not start, did not answer its API, stayed
sealed, or did not see a leader within `openbao_restart_health_timeout`
seconds: a config error (see `journalctl -u openbao`), Shamir with
`openbao_unseal: false` or PGP-wrapped shares and nobody unsealed it, an
unreachable auto-unseal backend, or a TLS or network problem towards the
leader. The play stops, so no second node is restarted; the nodes before it
are done.

**Fix:** check `journalctl -u openbao` on the node, fix the cause, unseal it by
hand if needed and re-run. Its restart marker stays, so the next run restarts
the remaining nodes.

### Pending restart after a failed run

**Symptom:** a run that changes nothing still restarts a node, or check mode
reports `Check mode: would restart <nodes>`.

**Cause:** restart intent is persistent. A pending change of `openbao.hcl`,
`openbao.env` or the unit writes `/etc/openbao/.ansible-restart-pending` on
that node before the file is written (the role compares the files in check
mode first), so a run that stops in between (a failed validate hook, a lost
connection, Ctrl-C) cannot lose it. A package, binary or plugin change writes
the marker right after the install, and a node whose running version
(`sys/seal-status`) differs from `openbao_version` also needs a restart. If
the earlier run failed before its rolling restart (a guard, a quorum check, a
health gate), the next run finishes the job. In check mode the role writes no
marker; a pending render or package change in the same check run is counted
as well, so the report matches the real run.

Until the restart, a pending `openbao.hcl` is partly live already: every
reload (SIGHUP; the role's certificate reload and the nightly logrotate
reload) makes OpenBao re-read the file and apply the log level, declarative
audit device additions and removals (on the active node) and KMS plugin
stanzas (a changed plugin is respawned) on that node at once. Storage,
listener addresses, seal and `openbao.env` changes wait for the restart.

**Fix:** nothing to do: the next successful run restarts the flagged nodes one
at a time (standbys first, leader last) and removes the markers. Do not delete
a marker by hand unless you restarted that node yourself after the change. To
force a health-gated restart of a node, create the marker as root
(`touch /etc/openbao/.ansible-restart-pending`) and run the role. A raised
`openbao_version` with the `install` stage skipped (for example
`--tags config,restart`) restarts every node on each such run without
upgrading it, because the running version still differs: run the full role
once.

### Upgrading OpenBao

**Symptom:** you want to move the cluster to a newer OpenBao version, or an
upgrade stopped half way.

**Cause:** an upgrade is a software change on every node plus a rolling
restart; the role does both in one run and resumes a run that stopped.

**Fix:**

1. Read the upstream release notes of every version you cross and take a raft
   snapshot with a token: `bao operator raft snapshot save <file>`.
2. Raise `openbao_version` (and update pinned checksums for the binary method)
   and run the playbook against the whole group, without `--limit`.
3. The role replaces the package or binary on every node (running processes
   keep the old binary) and records a pending restart, then restarts the nodes
   one at a time: quorum check, restart, unseal (Shamir), health gate; the
   leader goes last and hands over on shutdown.
4. If a check stops the play, fix the node it names and re-run: nodes that
   still run the old version or keep a marker are restarted, the others are
   not. Downgrades are refused; to go back, restore the snapshot on nodes
   installed with the old version.

### Running only some stages with --tags

**Symptom:** `ansible-playbook ... --tags restart` (or `--tags config`) runs
only the validation tasks.

**Cause:** the role was included with `include_role`; Ansible does not apply
`--tags` to the tasks of a dynamic include unless the include task itself has
the tag.

**Fix:** use `roles:` or `ansible.builtin.import_role` in the play. Every task
carries its stage tag (`install`, `config`, `tls`, `service`, `init_unseal`,
`restart`), and validation and the address facts always run. With a static
import, `--tags restart` runs validation, the address facts and the rolling
restart.

---

## Cluster operations

### Replacing a failed node

**Symptom:** a node is lost (disk, VM) and must be replaced.

**Cause:** not an error; the procedure relies on `retry_join`.

**Fix:** a node that lost its data must not come back under its old raft
membership: it no longer has the log entries and votes the cluster counted
on, which is outside the safety assumptions of raft. Upstream removes an
unrecoverable server first:

1. From a healthy member, with a token:
   `bao operator raft remove-peer <node id>` (the old node id, by default the
   inventory name). Check with `bao operator raft list-peers`.
2. Provision the fresh host (same inventory name or a new one; BYO TLS: with
   its certificates in place) with an empty `openbao_data_dir`.
3. Run the role against the whole group. No node is initialized again,
   because the other members report initialized; the new node joins through
   `retry_join` as a new peer and is unsealed.

Re-joining a wiped node under its old id without `remove-peer` happens to
work when the cluster is otherwise healthy, but do not rely on it. Autopilot
`cleanup_dead_servers` is off by default. In generated TLS mode, replacing
the bootstrap node creates a new CA, and every node is re-signed and
reloaded.

### Requests fail because the audit log cannot be written

**Symptom:** API requests fail with `no audit backend succeeded in logging the
request` (or `the response`).

**Cause:** OpenBao refuses requests it cannot audit: the disk under
`openbao_audit_dir` is full, or the file is not writable for the service user.
The device is replicated, so every node must be able to write its file.

**Fix:** free space and keep `openbao_audit_logrotate_enabled: true`; when
rotating by hand, run `systemctl reload openbao` afterwards so the device
reopens its file. Check the rule with `logrotate -d /etc/logrotate.d/openbao`.
On EL with SELinux enabled the role labels `openbao_audit_dir` as a log
directory (`semanage fcontext -a -t var_log_t '<dir>(/.*)?'`, then
`restorecon`): under `/opt` the directory would be `usr_t`, which the confined
logrotate domain may read but not rename or create, so the daily rotation
would be denied and the log would grow until the disk is full. With
`openbao_manage_selinux: false` add that file context yourself; check
`ausearch -m avc -c logrotate` after the first rotation.

### No active node after an audit change

**Symptom:** after a restart every node is unsealed, but none becomes active:
`sys/leader` shows no leader, the role waits in "Wait until every node sees a
leader" or in the restart health gate, and the journal repeats
`sanity check failed; unable to open "<file>" for writing` followed by
`post-unseal setup failed` (or `read-only post-unseal setup failed` on
standbys).

**Cause:** the audit table is replicated, and a node opens every stored file
device when it unseals, before it removes the devices no longer declared in
`openbao.hcl`. If one of them points at a file the process cannot open (a
path outside `openbao_audit_dir`, which `ProtectSystem=strict` keeps
read-only, a directory that does not exist on this node, wrong owner), no
node can finish its post-unseal setup. The role keeps `openbao_audit_dir`
writable whether or not `openbao_audit_enabled` is set, so its own device
cannot cause this; a second device in `openbao_extra_config_hcl` or one
enabled by hand can.

**Fix:** make the file writable again on every node: put it back under
`openbao_audit_dir` (or create the missing directory with owner
`openbao`), then re-render and restart. A normal run stops at the leader wait
before its rolling restart, so run only the stages that apply it:
`--tags config,restart` (with `roles:` or `import_role`). Once a node is
active it removes the undeclared devices; only then move or drop the path.

### OpenBao refuses to start with a disable_mlock error

**Symptom:** the unit fails; the journal shows `OpenBao has dropped support
for mlock. Please remove the line "disable_mlock" = false from your config and
disable or encrypt swap instead.`

**Cause:** `openbao.hcl` contains `disable_mlock = false`, typically a config
copied from Vault or edited by hand. The role never renders the key and
refuses it in `openbao_extra_config_hcl`.

**Fix:** remove the line (the next role run re-renders the file anyway).
Swap protection comes from `MemorySwapMax=0` in the unit.

### VAULT_* environment variables

**Symptom:** the `bao` CLI talks to the wrong server, uses an unexpected token
or ignores a CA, although `BAO_*` looks right; or a value is honoured that you
never set for OpenBao.

**Cause:** every `BAO_*` variable falls back to the matching `VAULT_*`
variable when the `BAO_*` one is not set, so a shell with `VAULT_ADDR`,
`VAULT_TOKEN`, `VAULT_CACERT` or `VAULT_SKIP_VERIFY` from Vault work leaks
into `bao`. Some names were never renamed: the `X-Vault-Token` header, the
token file `~/.vault-token` and the `vault_` prefix of the metrics.

**Fix:** check `env | grep -E '^(VAULT|BAO)_'`, unset the `VAULT_*` variables
and always set `BAO_*`. Never source `openbao.env` into a shell: it may hold
`BAO_TOKEN` (the transit seal token).

### Check mode fails on a fresh host

**Symptom:** `ansible-playbook --check` fails on a host that the role never
converged, for example on a missing user, group or binary.

**Cause:** check mode is supported only on hosts that were converged at least
once: the service user and the binary do not exist before the first real run.

**Fix:** run the role once for real. On a converged host `--check --diff`
reports pending file and package changes and never initializes, unseals,
restarts or reloads anything.

---

## Useful commands

```bash
# Service state and server log
systemctl status openbao
journalctl -u openbao -n 100 --no-pager

# CLI environment on a node (BAO_* only, unset stray VAULT_* first).
# 127.0.0.1 works with generated certificates; with BYO certificates use an
# address the certificate covers.
export BAO_ADDR=https://127.0.0.1:8200
export BAO_CACERT=/opt/openbao/tls/openbao-ca.pem
bao status

# Unauthenticated status, as the role probes it
curl -s --cacert "$BAO_CACERT" "$BAO_ADDR/v1/sys/seal-status" | jq .
curl -s --cacert "$BAO_CACERT" "$BAO_ADDR/v1/sys/leader" | jq .
# 200 active, 429 standby, 503 sealed, 501 not initialized
curl -s -o /dev/null -w '%{http_code}\n' --cacert "$BAO_CACERT" "$BAO_ADDR/v1/sys/health"

# Raft peers, autopilot and audit devices (need a token in BAO_TOKEN)
bao operator raft list-peers
bao operator raft autopilot state
bao audit list -format=json

# Parse the rendered config (what the template validate hook runs)
bao operator validate-config -config=/etc/openbao/openbao.hcl

# Pending restart marker of this node
ls -l /etc/openbao/.ansible-restart-pending

# Certificate SANs, issuer and expiry
openssl x509 -in /opt/openbao/tls/openbao-cert.pem -noout -ext subjectAltName -issuer -enddate

# Reload in place (SIGHUP, no restart, stays unsealed): re-reads openbao.hcl
# and applies certificates, log level, audit devices and KMS plugin stanzas
systemctl reload openbao

# Read the init output on the controller
ansible-vault view --vault-password-file ~/.openbao-vault-pass \
  ~/.openbao/.openbao_init_<cluster id>.json
```
