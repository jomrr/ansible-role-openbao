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
- Declarative file audit devices and their dedicated log directories

### Not Managed

- Recovery-key generation and export, provided by the openbao_config recovery
  entry point
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

API address advertised to clients.

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

### `openbao_api_ca_file`

Type: `path`. Required: `false`.

CA bundle on the target for bootstrap verification; omitted to use the system
trust store.

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
  appear in process arguments or task output.
- The TPM2 binding uses no PCR policy, so firmware and Secure Boot updates do
  not invalidate the credential.
- The cluster listener of the single node is bound to 127.0.0.1.
- Store the bootstrap Secret ID in Ansible Vault outside this OpenBao instance.
  The ansible-admin policy is highly privileged: it can manage policies,
  authentication, mounts and recovery operations.
- Self-initialization reads the Secret ID from a systemd credential; the server
  configuration contains only its filename. Trace logging is rejected while
  bootstrap is enabled because profile tracing exposes request data.
- Audit devices use HMAC redaction and log_raw=false. API-based audit-device
  creation remains disabled.

## Operational Notes

- With bootstrap disabled, initialize the server once after the first run, for
  example with `bao operator init -recovery-shares=5 -recovery-threshold=3` and
  BAO_ADDR and BAO_CACERT set for the API listener. Store the recovery keys and
  the root token outside the host.
- With bootstrap enabled, OpenBao creates the ansible/ AppRole mount,
  ansible-admin policy and AppRole, registers the supplied Role ID and Secret
  ID, and explicitly revokes the initial root token. The role verifies
  initialization and the management login; no root token is exported. Recovery
  keys are generated separately using jomrr.openbao_config with
  tasks_from=recovery before the instance is relied upon for production.
- Self-initialization runs only on empty storage. Existing installations require
  a working administrator to establish the management AppRole before enabling
  bootstrap verification. Failed partial initialization does not replay on
  restart: repair it with an existing administrator or recovery access. A fresh
  instance without any working access requires deliberate operator-led
  reinitialization; this role never removes raft storage.
- Changing bootstrap variables does not change an existing AppRole. Use the
  openbao_config secret_id entry point to register a new Secret ID, update
  Ansible Vault, then revoke the old Secret ID in a separate invocation. The
  encrypted bootstrap credential is created only when missing; remove it
  explicitly when reseeding it for a future reinitialization. Keep the Role ID
  stable.
- Audit paths are single lowercase mount components. Use a dedicated parent
  directory for each configured file_path. Existing audit devices cannot be
  modified in place: enable a replacement at a new audit path, verify it, then
  remove the old entry. Removing an entry disables that declarative device;
  API-created devices are not adopted or removed. An unavailable sole audit sink
  can block API requests.
- Configure log rotation outside this role and send SIGHUP to openbao.service
  after rotating audit files so the file descriptors are reopened. The role does
  not rotate or truncate audit logs.
- The role encrypts the seal key only when the credential of the selected key
  binding is missing. When the credential can no longer be decrypted, for
  example after replacing the vTPM or the host credential secret, remove the
  seal and bootstrap credential files of that binding and run the role again to
  encrypt the credentials from Ansible Vault. Credentials for the unused binding
  are removed when bootstrap is enabled.
- Changing openbao_seal_key or openbao_seal_key_id after initialization leaves
  the storage unreadable. Key rotation requires a seal migration outside this
  role.
- The distribution package determines the OpenBao version; updates come from the
  platform package manager.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |

## Example Playbook

### Self-initialization and file audit

Provision a management AppRole from Ansible Vault and retain the static seal
separately. The API CA file is on the target; openbao_config uses its own
controller-side CA file.

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
        openbao_api_ca_file: /etc/pki/ca-trust/source/anchors/internal-ca.pem
        openbao_bootstrap_enabled: true
        openbao_bootstrap_role_id: ansible-controller
        openbao_bootstrap_secret_id: >-
          {{ vault_openbao_management_secret_id }}
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
