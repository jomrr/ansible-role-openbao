# Ansible Role: openbao

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-openbao)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-openbao)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-openbao)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-openbao/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-openbao/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-openbao/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-openbao/actions/workflows/main.yml?query=branch%3Amain)

Ansible role for setting up a single-node OpenBao server with automatic
unsealing.

## Purpose

This role installs OpenBao from the distribution packages and runs it as a
single-node server with integrated
raft storage. The static seal reads its key from a systemd credential that
systemd-creds encrypts with the host
secret and, by default, the TPM2. An initialized server therefore unseals itself
on every start.
Optional native self-initialization provisions a management AppRole for the
separate openbao_config role.

## Scope

### Managed

- EPEL repository package on AlmaLinux
- OpenBao server package and, for TPM2 binding, the TPM2 libraries used by
  systemd-creds
- Encrypted seal key credential and the systemd drop-in that loads it
- OpenBao configuration with raft storage, TLS listener and static seal
- Raft storage directory
- OpenBao service enablement and runtime state
- Optional self-initialization with a reserved Ansible management AppRole and
  explicit root-token revocation
- Controller reconciliation of the reserved management policy and CIDR bindings
  with rollback on failed verification
- Declarative file audit devices and their dedicated log directories

### Not Managed

- Recovery-key generation and export
- Application auth methods, secrets engines and policies, provided by
  openbao_config
- Audit log rotation and shipping
- TLS certificate issuance and distribution
- Firewall policy
- Multi-node raft clusters
- Seal key rotation and seal migration
- Raft snapshots and backups

## Requirements

- A TPM2 device, such as a vTPM, for the default host+tpm2 key binding.
- TLS certificate and private key on the target, readable by the openbao service
  account.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: ansible.posix
    version: '>=2.0.0'
```

## Role Variables

### `openbao_seal_key`

Type: `str`. Required: `true`.

Static seal key, 32 bytes encoded as 64 hexadecimal or 44 base64 characters.
The key is the only way to decrypt the storage; keep it in Ansible Vault and
never change it after initialization.

### `openbao_seal_key_id`

Type: `str`. Required: `false`.

Permanent identifier of the static seal key; keep it unchanged after
initialization.

Default:

```yaml
openbao_seal_key_id: openbao-seal-1
```

### `openbao_seal_credential_key`

Type: `str`. Required: `false`.

Key binding used by systemd-creds to encrypt the seal key credential.
host+tpm2 requires a TPM2 device such as a vTPM; host binds only to the host
credential secret.

Default:

```yaml
openbao_seal_credential_key: host+tpm2
```

### `openbao_tls_cert_file`

Type: `path`. Required: `true`.

TLS certificate file on the target presented by the API listener.

### `openbao_tls_key_file`

Type: `path`. Required: `true`.

TLS private key file on the target, readable by the openbao service account.

### `openbao_listener_address`

Type: `str`. Required: `false`.

Address and port of the TLS API listener.

Default:

```yaml
openbao_listener_address: 0.0.0.0:8200
```

### `openbao_api_addr`

Type: `str`. Required: `false`.

API address advertised to clients and reachable from the controller for
bootstrap.

Default:

```yaml
openbao_api_addr: https://{{ ansible_facts.fqdn }}:8200
```

### `openbao_raft_path`

Type: `path`. Required: `false`.

Directory of the integrated raft storage.

Default:

```yaml
openbao_raft_path: /var/lib/openbao/raft
```

### `openbao_raft_node_id`

Type: `str`. Required: `false`.

Raft node identifier; keep it unchanged after initialization.

Default:

```yaml
openbao_raft_node_id: '{{ ansible_facts.hostname }}'
```

### `openbao_ui`

Type: `bool`. Required: `false`.

Serve the web UI on the API listener.

Default:

```yaml
openbao_ui: true
```

### `openbao_log_level`

Type: `str`. Required: `false`.

Server log level.

Default:

```yaml
openbao_log_level: info
```

### `openbao_bootstrap_enabled`

Type: `bool`. Required: `false`.

Initialize empty storage with the reserved ansible management AppRole and revoke
the initial root token.
Reconcile the reserved management policy and CIDR bindings from the controller
on subsequent runs.

Default:

```yaml
openbao_bootstrap_enabled: false
```

### `openbao_bootstrap_role_id`

Type: `str`. Required: `false`.

Required management Role ID when bootstrap is enabled; unchanged on already
initialized storage.

### `openbao_bootstrap_secret_id`

Type: `str`. Required: `false`.

Required random management Secret ID of at least 32 characters, supplied from
Ansible Vault when bootstrap is enabled.
Registered only during self-initialization; later changes require the
openbao_config secret_id entry point.

### `openbao_bootstrap_bound_cidrs`

Type: `list`. Required: `false`.

Networks allowed to log in and use management tokens; an empty list leaves
access unbound.
Applied during self-initialization and reconciled on existing management
AppRoles with rollback on failed verification.

Default:

```yaml
openbao_bootstrap_bound_cidrs: []
```

### `openbao_controller_ca_file`

Type: `path`. Required: `false`.

CA bundle on the controller for bootstrap API calls; omitted to use the
controller trust store.

### `openbao_audit_devices`

Type: `list`. Required: `false`.

Declarative file audit devices; removal of an entry disables the device on
restart.
Existing devices cannot be modified in place; create a replacement at a new
audit path first.

Default:

```yaml
openbao_audit_devices: []
```

## Managed Files

- `/etc/credstore.encrypted/openbao-bootstrap-secret-id.<binding>.cred` Optional
  bootstrap Secret ID encrypted with the selected host or host-tpm2 binding.
- `/etc/openbao.d/openbao.hcl` OpenBao configuration on AlmaLinux and Fedora.
- `/etc/openbao/openbao.hcl` OpenBao configuration on openSUSE Tumbleweed.
- `/etc/credstore.encrypted/openbao-seal-key.host-tpm2.cred` Encrypted seal key
  for host+tpm2; removed when host is selected.
- `/etc/credstore.encrypted/openbao-seal-key.host.cred` Encrypted seal key for
  host; removed when host+tpm2 is selected.
- `/etc/systemd/system/openbao.service.d/seal-credential.conf` Loads the
  encrypted seal key credential into the service.
- `/var/lib/openbao/raft` Default raft storage directory.
- `Configured audit log directories` Dedicated directories owned by openbao with
  mode 0750; the service creates audit files with mode 0600.

## Check Mode

Check mode requires a host on which the role has run before; on a fresh host the
package, service account and seal key credential do not exist yet.

- Encrypting the seal key with systemd-creds is not simulated in check mode.
- Bootstrap API verification and reconciliation are skipped in check mode.

## Service Behavior

Changes to the configuration, the drop-in or the seal key credential restart
OpenBao, after systemd has reloaded a changed drop-in. An initialized server
unseals itself after the restart.

### Handlers

- OPENBAO | Reload systemd units
- OPENBAO | Restart service

## Security Notes

- Store openbao_seal_key in Ansible Vault. It is the only key that decrypts the
  storage; recovery keys do not replace it.
- The target stores the seal key only as systemd credential encrypted with the
  host secret and, with host+tpm2, the TPM2. systemd provides the decrypted key
  only in the credential directory of the running service.
- The role passes the seal key to systemd-creds on standard input; it does not
  appear in process arguments or task output. Both credential encryption tasks
  use task-local pipelining and are skipped before module execution when the
  encrypted file already exists. Pipelining requires connection-plugin support
  and ANSIBLE_KEEP_REMOTE_FILES must remain disabled.
- The TPM2 binding uses no PCR policy, so firmware and Secure Boot updates do
  not invalidate the credential.
- The cluster listener of the single node is bound to 127.0.0.1.
- Store the bootstrap Secret ID in Ansible Vault outside this OpenBao instance.
  The ansible-admin policy is highly privileged: it can manage policies,
  authentication, mounts, recovery operations and KV v2 data and metadata. Raft
  snapshot access requires a separate identity and policy. The + wildcard covers
  single-component mount paths.
- Self-initialization reads the Secret ID from a systemd credential; the server
  configuration contains only its filename. Trace logging is rejected while
  bootstrap is enabled because profile tracing exposes request data.
- Audit devices use HMAC redaction and log_raw=false. API-based audit-device
  creation remains disabled.

## Operational Notes

- `openbao_bootstrap_enabled: false` (the default) provisions and starts the
  service without initializing storage or managing bootstrap access. A new
  instance remains uninitialized after the role completes.
- To initialize empty storage through the role, set `openbao_bootstrap_enabled:
  true` and supply `openbao_bootstrap_role_id` and
  `openbao_bootstrap_secret_id`. The role configures the management AppRole and
  verifies access from the controller. The initial root token is revoked and is
  not available as a role output; subsequent automation uses the supplied
  AppRole credentials.
- Enabling bootstrap on initialized storage requires an existing management
  AppRole matching the supplied credentials. Rerunning the role does not replay
  initialization requests, including those from a partially completed
  self-initialization. The role never removes raft storage.
- Role ID and Secret ID are registered only during self-initialization. Changing
  their role variables does not rotate existing credentials. After
  initialization, keep the Role ID stable and supply a Secret ID already
  registered with OpenBao. The encrypted bootstrap credential is created only
  when missing; remove it explicitly when reseeding it for a future
  reinitialization.
- With bootstrap enabled, each normal run reconciles ansible-admin and both
  management CIDR bindings from the controller. Other AppRole settings are
  preserved. openbao_bootstrap_bound_cidrs defaults to [] (unbound); configure
  trusted admin and VPN subnets that include the controller address as observed
  by OpenBao after NAT.
- CIDR updates retain the existing session while a fresh login and authenticated
  read verify the new bindings. If writing or verification fails, the role
  restores and reads back both previous bindings, then fails with a clear
  message. A failed rollback is reported separately. First initialization has no
  previous session; incorrect initial CIDRs require another administrator or
  recovery access. Correct rejected inventory values before retrying; the
  generated initialize block is used only on empty storage.
- Bootstrap API calls run on localhost. Rename openbao_api_ca_file to
  openbao_controller_ca_file and provide that CA file on the controller; without
  it, the controller trust store is used. openbao_api_addr must be reachable
  from the controller and covered by the server certificate. TLS verification
  remains enabled.
- Audit paths are single lowercase mount components. Use a dedicated parent
  directory for each configured file_path. Existing audit devices cannot be
  modified in place: enable a replacement at a new audit path, verify it, then
  remove the old entry. Removing an entry disables that declarative device;
  API-created devices are not adopted or removed. An unavailable sole audit sink
  can block API requests.
- The role encrypts the seal key only when the credential of the selected key
  binding is missing. When the credential can no longer be decrypted, for
  example after replacing the vTPM or the host credential secret, remove the
  seal and bootstrap credential files of that binding and run the role again to
  encrypt the credentials from Ansible Vault. Credentials for the unused binding
  are removed when bootstrap is enabled.
- Changing openbao_seal_key or openbao_seal_key_id after initialization leaves
  the storage unreadable. Key rotation requires a seal migration outside this
  role.
- The role installs the distribution OpenBao package with `state: present`.
  Rerunning the role ensures the package is installed; it does not request an
  upgrade to the latest version.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |

## Example Playbook

### Self-initialization and file audit

Provision a management AppRole from Ansible Vault and retain the static seal
separately. Both roles use CA files on the controller. Replace the example
networks with the actual admin and VPN subnets seen by OpenBao.

```yaml
---
- name: Bootstrap OpenBao
  hosts: openbao
  gather_facts: true
  roles:
    - role: jomrr.openbao
      vars:
        openbao_seal_key: "{{ vault_openbao_seal_key }}"
        openbao_tls_cert_file: /etc/pki/tls/certs/openbao.crt
        openbao_tls_key_file: /etc/pki/tls/private/openbao.key
        openbao_controller_ca_file: /etc/pki/ca-trust/source/anchors/internal-ca.pem
        openbao_bootstrap_enabled: true
        openbao_bootstrap_bound_cidrs: [192.0.2.0/24, 198.51.100.0/24]
        openbao_bootstrap_role_id: ansible-controller
        openbao_bootstrap_secret_id: "{{ vault_openbao_management_secret_id }}"
        openbao_audit_devices:
          - path: file
            file_path: /var/log/openbao/audit.json
```

### Single-node OpenBao with TPM2-bound automatic unsealing

Seal key from Ansible Vault and a certificate deployed before the role runs.

```yaml
---
- name: Configure OpenBao
  hosts: openbao
  gather_facts: true
  roles:
    - role: jomrr.openbao
      vars:
        openbao_seal_key: "{{ vault_openbao_seal_key }}"
        openbao_tls_cert_file: /etc/pki/tls/certs/openbao.crt
        openbao_tls_key_file: /etc/pki/tls/private/openbao.key
```

### Host-bound credential without TPM2

For hosts without a TPM2 device; the credential is bound to the host
credential secret only.

```yaml
---
- name: Configure OpenBao without TPM2
  hosts: openbao
  gather_facts: true
  roles:
    - role: jomrr.openbao
      vars:
        openbao_seal_key: "{{ vault_openbao_seal_key }}"
        openbao_seal_credential_key: host
        openbao_tls_cert_file: /etc/pki/tls/certs/openbao.crt
        openbao_tls_key_file: /etc/pki/tls/private/openbao.key
```

## References

- [OpenBao self-initialization](https://openbao.org/docs/configuration/self-init/)
- [OpenBao declarative audit](https://openbao.org/docs/configuration/audit/)
- [OpenBao static seal](https://openbao.org/docs/configuration/seal/static/)
- [OpenBao integrated storage](https://openbao.org/docs/configuration/storage/raft/)
- [systemd-creds](https://www.freedesktop.org/software/systemd/man/latest/systemd-creds.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
