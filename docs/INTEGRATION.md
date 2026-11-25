# Integration Guide

How to consume the `devopsgroupeu.openbao` role in other projects, and how to
wire the OpenBao cluster it builds into a platform: load balancer, Prometheus,
Kubernetes and External Secrets Operator. All CLI examples use the `bao`
binary and `BAO_*` environment variables.

---

## Table of Contents

- [Consuming the role via requirements.yml](#consuming-the-role-via-requirementsyml)
- [Load balancer health check (HAProxy)](#load-balancer-health-check-haproxy)
- [Prometheus](#prometheus)
- [Kubernetes auth](#kubernetes-auth)
- [External Secrets Operator](#external-secrets-operator)
- [A second audit device](#a-second-audit-device)
- [Post-role workflow](#post-role-workflow)

---

## Consuming the role via requirements.yml

### From Ansible Galaxy

```yaml
# requirements.yml in the consuming project
collections:
  - name: community.crypto
    version: ">=3.2.2"

roles:
  - name: devopsgroupeu.openbao
    version: "X.Y.Z"   # latest release, see GitHub Releases
```

Install:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

### From Git source

```yaml
# requirements.yml - git source
roles:
  - name: devopsgroupeu.openbao
    src: https://github.com/devopsgroupeu/ansible-role-openbao.git
    version: "vX.Y.Z"   # a release tag or a commit SHA
    scm: git
```

Install:

```bash
ansible-galaxy role install -r requirements.yml -p roles/
```

The consuming project must have the install directory in `roles_path` (or
`ANSIBLE_ROLES_PATH`). Use the role through `roles:` or `import_role`, so
that `--tags` reaches its stages.

---

## Load balancer health check (HAProxy)

Clients should reach the cluster through a load balancer or VIP that passes
TLS through to the nodes (TCP mode) and checks `GET /v1/sys/health`:

| Status | Meaning |
|---|---|
| `200` | initialized, unsealed, active |
| `429` | unsealed standby (`200` with `?standbyok=true`) |
| `503` | sealed |
| `501` | not initialized |

Checking for `200` sends all traffic to the active node; add
`?standbyok=true` to the check URI to spread reads over the unsealed standbys
(they forward writes to the active node). The node certificates must also
carry the name or address clients use for the load balancer (BYO certificate,
or `openbao_tls_extra_sans` in generated mode).

With the house role `ansible-role-haproxy-keepalived`
(`devopsgroupeu.haproxy_keepalived`), one `haproxy_listens` entry on the load
balancer hosts does it. The health check runs over TLS and verifies the node
certificate against the OpenBao CA, which you copy to the load balancer hosts
first:

```yaml
haproxy_listens:
  - name: openbao
    address: "*"
    port: 8200
    mode: tcp
    balance: roundrobin
    http_check:
      send:
        method: GET
        uri: /v1/sys/health
      expect: status 200
    servers:
      - name: openbao1
        address: 10.0.1.11
      - name: openbao2
        address: 10.0.1.12
      - name: openbao3
        address: 10.0.1.13
    options:
      - check
      - check-ssl
      - verify required
      - ca-file /etc/haproxy/openbao-ca.pem
      - inter 5s
      - fall 2
      - rise 2
```

The rendered backend lines look like
`server openbao1 10.0.1.11:8200 check check-ssl verify required ca-file /etc/haproxy/openbao-ca.pem inter 5s fall 2 rise 2`.
During a rolling restart the role restarts the active node last; the check
moves traffic to the new active node within `inter` times `fall`.

---

## Prometheus

The role renders a `telemetry` stanza (`openbao_telemetry_enabled`,
`openbao_telemetry_prometheus_retention_time`). Metrics are served at
`/v1/sys/metrics` with `format=prometheus`; their names keep the `vault_`
prefix, so existing Vault dashboards and alerts keep working.

Two ways to scrape:

- **Token on the API listener.** A token with `read` and `list` on
  `sys/metrics` scrapes the active node only: standbys answer the metrics
  endpoint only without authentication. The token must stay valid; with the
  role default `openbao_max_lease_ttl: "24h"` that means a periodic token
  renewed by a job.
- **Dedicated metrics-only listener (recommended).** A second listener that
  serves nothing but `/v1/sys/metrics`, without a token, on every node.
  Declare it through `openbao_extra_config_hcl` (group vars) and restrict the
  port to the Prometheus servers with your firewall:

```yaml
openbao_extra_config_hcl: |
  listener "tcp" {
    address       = "0.0.0.0:9101"
    tls_cert_file = "/opt/openbao/tls/openbao-cert.pem"
    tls_key_file  = "/opt/openbao/tls/openbao-key.pem"
    telemetry {
      metrics_only                   = true
      unauthenticated_metrics_access = true
    }
  }
```

A reload (SIGHUP: the role's reload after a generated certificate changed, or
`systemctl reload openbao`) refreshes the certificate of this listener too.
Leave `openbao_telemetry_unauthenticated_metrics_access`
false, so the API listener keeps requiring a token for metrics.

Prometheus scrape job (the `?` of `format=prometheus` must be a parameter,
not part of the path):

```yaml
scrape_configs:
  - job_name: openbao
    metrics_path: /v1/sys/metrics
    params:
      format: ["prometheus"]
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/openbao-ca.pem
      server_name: openbao.cluster.internal   # openbao_tls_leader_servername, on every node certificate
    static_configs:
      - targets:
          - 10.0.1.11:9101
          - 10.0.1.12:9101
          - 10.0.1.13:9101
```

For the token variant, scrape port 8200 of the active node (or the load
balancer) and add `authorization: { credentials_file: /etc/prometheus/openbao-token }`.

---

## Kubernetes auth

Lets workloads and the External Secrets Operator log in with their Kubernetes
service account. Run against the cluster with an admin token:

```bash
export BAO_ADDR=https://openbao.example.internal:8200
export BAO_CACERT=/path/to/openbao-ca.pem
export BAO_TOKEN=<admin token>

# KV version 2 store for application secrets
bao secrets enable -path=secret -version=2 kv

# Kubernetes auth method
bao auth enable kubernetes

# OpenBao runs outside Kubernetes, so it needs the API server address and CA.
# Without token_reviewer_jwt each client's own JWT is used for the TokenReview
# call, so the client service accounts need the system:auth-delegator role.
bao write auth/kubernetes/config \
  kubernetes_host="https://k8s-api.example.internal:6443" \
  kubernetes_ca_cert=@/path/to/kubernetes-ca.crt

# Read-only policy for the secrets operator
bao policy write eso-read - <<'EOF'
path "secret/data/*" {
  capabilities = ["read"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
EOF

# Bind the policy to the operator's service account
bao write auth/kubernetes/role/eso \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-read \
  ttl=1h
```

---

## External Secrets Operator

The External Secrets Operator talks to OpenBao through its `vault` provider
(the API is compatible). A `ClusterSecretStore` for the role above:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: openbao
spec:
  provider:
    vault:
      server: "https://openbao.example.internal:8200"
      path: "secret"
      version: "v2"
      caBundle: "<base64 of the CA in openbao_tls_ca_file>"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso"
          serviceAccountRef:
            name: "external-secrets"
            namespace: "external-secrets"
```

An `ExternalSecret` then reads, for example, `secret/data/app/db` with
`remoteRef: { key: app/db, property: password }`.

---

## A second audit device

Upstream recommends more than one audit device, so that one failing device
does not block requests. The role declares one file device; add a second one
through `openbao_extra_config_hcl`, in the same group vars for every node:

```yaml
openbao_extra_config_hcl: |
  audit "file" "file-secondary" {
    options {
      file_path = "/opt/openbao/audit/openbao_audit_secondary.log"
      mode      = "0600"
    }
  }
```

- Keep the file inside `openbao_audit_dir`: with `openbao_systemd_hardening`
  only that directory (and the data directory) is writable for the service.
- The device is immutable like the role's own one, but the role's audit guard
  does not cover raw HCL. Never change it in place: remove it, roll that out to
  the whole cluster, then add it again with the new values.
- The role's logrotate rule covers only `openbao_audit_file_path`. Add a rule
  for the second file with the same `postrotate` reload (the reload reopens
  every file device).

---

## Post-role workflow

After the first successful run:

1. **Read the init output** on the controller (path from
   `openbao_init_output_path`, default
   `~/.openbao/.openbao_init_<cluster id>.json`):

   ```bash
   ansible-vault view --vault-password-file ~/.openbao-vault-pass \
     ~/.openbao/.openbao_init_<cluster id>.json
   ```

2. **Distribute the keys.** Hand the unseal keys (Shamir) or recovery keys
   (auto-unseal) to their holders. With Shamir and `openbao_unseal: true` the
   role needs the encrypted file on later runs to unseal restarted nodes, so
   keep it (and the password file) on persistent, access-controlled storage.
3. **Set up auth and an admin policy**, for example OIDC or userpass for
   operators, and log in with it. Include `sys/generate-root-token/*` in the
   admin policy: since OpenBao 2.6.0 generating a new root token needs an
   authenticated caller.
4. **Revoke the root token.** The role never uses it, so later runs keep
   working:

   ```bash
   BAO_TOKEN=<root token> bao token revoke -self
   ```

5. **Configure autopilot** dead server cleanup (off by default), so a node
   that is gone for good leaves the peer set without a manual
   `remove-peer`:

   ```bash
   bao operator raft autopilot set-config \
     -cleanup-dead-servers=true \
     -dead-server-last-contact-threshold=10m \
     -min-quorum=3
   ```

6. **Schedule raft snapshots** with a token whose policy allows `read` on
   `sys/storage/raft/snapshot`, and keep them off the cluster:

   ```bash
   bao operator raft snapshot save /backup/openbao-$(date +%F).snap
   ```

7. **Test failover:** stop `openbao` on the active node, check that a standby
   takes over (`bao status`, `GET /v1/sys/leader`), start the node again and
   re-run the role to unseal it (Shamir).
8. **Upgrades:** raise `openbao_version` and run the playbook against the
   whole group; the role restarts one node at a time with the leader last.
   Read the upstream release notes and take a snapshot first.
