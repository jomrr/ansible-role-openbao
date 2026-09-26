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

## Scope

### Managed

- EPEL repository package on AlmaLinux
- OpenBao server package and, for TPM2 binding, the TPM2 libraries used by
  systemd-creds
- Encrypted seal key credential and the systemd drop-in that loads it
- OpenBao configuration with raft storage, TLS listener and static seal
- Raft storage directory
- OpenBao service enablement and runtime state

### Not Managed

- Initialization, recovery keys and root token
- Auth methods, secrets engines, policies and audit devices
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

## Managed Files

- `/etc/openbao.d/openbao.hcl` OpenBao configuration on AlmaLinux and Fedora.
- `/etc/openbao/openbao.hcl` OpenBao configuration on openSUSE Tumbleweed.
- `/etc/credstore.encrypted/openbao-seal-key.host-tpm2.cred` Encrypted seal key
  for host+tpm2; removed when host is selected.
- `/etc/credstore.encrypted/openbao-seal-key.host.cred` Encrypted seal key for
  host; removed when host+tpm2 is selected.
- `/etc/systemd/system/openbao.service.d/seal-credential.conf` Loads the
  encrypted seal key credential into the service.
- `/var/lib/openbao/raft` Default raft storage directory.

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

## Operational Notes

- Initialize the server once after the first run, for example with `bao operator
  init -recovery-shares=5 -recovery-threshold=3` and BAO_ADDR and BAO_CACERT set
  for the API listener. Store the recovery keys and the root token outside the
  host.
- The role encrypts the seal key only when the credential of the selected key
  binding is missing. When the credential can no longer be decrypted, for
  example after replacing the vTPM or the host credential secret, remove the
  credential file and run the role again to encrypt the seal key from
  openbao_seal_key.
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

- [OpenBao static seal](https://openbao.org/docs/configuration/seal/static/)
- [OpenBao integrated storage](https://openbao.org/docs/configuration/storage/raft/)
- [systemd-creds](https://www.freedesktop.org/software/systemd/man/latest/systemd-creds.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
