# Windows Server 2019 CIS Benchmark

Ansible role to apply the CIS Benchmark for Windows Server 2019.

## Upstream Source

<https://github.com/ansible-lockdown/Windows-2019-CIS>

**Author:** MindPoint Group
**License:** MIT

## CIS Level Tags

Use Ansible tags (or AAP Job Tags) to run specific CIS levels:

| Tag | Description |
|-----|-------------|
| `level1-domaincontroller` | Level 1 controls for Domain Controllers |
| `level1-memberserver` | Level 1 controls for Member Servers |
| `level1-domainmember` | Level 1 controls for Domain Members |
| `level2-domaincontroller` | Level 2 controls for Domain Controllers |
| `level2-memberserver` | Level 2 controls for Member Servers |
| `ngws-domaincontroller` | Next Generation Windows Security controls for DCs |
| `ngws-memberserver` | Next Generation Windows Security controls for Member Servers |

### Examples

```bash
# Run Level 1 Member Server controls only
ansible-playbook soe-win2019-cis-hardenning.yml --tags "level1-memberserver"

# Run all Level 2 controls
ansible-playbook soe-win2019-cis-hardenning.yml --tags "level2-domaincontroller,level2-memberserver"
```
