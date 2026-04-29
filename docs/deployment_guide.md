# Compliance Workshop - Deployment Guide

This guide walks you through deploying the Compliance Workshop to your Ansible Automation Platform (AAP) instance using Configuration as Code (CaC).

## Prerequisites

- AAP 2.5+ with admin access
- `ansible-core` 2.16+ installed locally (for CLI method) or an Execution Environment with the required collections
- SSH access to your AAP gateway (or network connectivity to the API)
- Red Hat Automation Hub token (for certified collections)
- Git SSH key for the project repository

## Deployment Methods

You can deploy the CaC setup in two ways:

| Method | When to use |
|--------|-------------|
| **CLI** (ansible-playbook) | Running from your workstation or a CI/CD pipeline |
| **AAP** (Job Template) | Running the setup from within AAP itself (self-bootstrapping) |

Follow Steps 1-4 for both methods. Step 5 differs depending on which method you choose.

---

## 1. Clone the Repository

```bash
git clone git@github.com:<your-org>/workshop-compliance.git
cd workshop-compliance
```

## 2. Install Collections

```bash
ansible-galaxy collection install -r collections/requirements.yml -p collections/
```

This installs `infra.aap_configuration`, `ansible.platform`, and the compliance playbook dependencies.

## 3. Configure Variables

Copy the template files and fill in your environment values:

```bash
cp cac/vars/common.yml.template cac/vars/common.yml
cp cac/vars/secrets.yml.template cac/vars/secrets.yml
```

### common.yml

Edit `cac/vars/common.yml` and set:

| Variable | Description | Example |
|----------|-------------|---------|
| `aap_hostname` | Your AAP gateway URL | `https://aap.example.com` |
| `aap_username` | AAP admin username | `admin` |
| `v_organization_name` | Target organization | `Default` |
| `v_git_repo_url` | SSH URL of this repo | `git@github.com:org/workshop-compliance.git` |
| `v_rhel_username` | SSH user for RHEL hosts | `ec2-user` |
| `v_windows_username` | Windows admin user | `Administrator` |
| `v_report_server` | Host that receives scan reports | `reports.example.com` |
| `v_report_server_user` | SSH user on report server | `student1` |

### secrets.yml

Edit `cac/vars/secrets.yml` and fill in the sensitive values (passwords, keys, tokens). Then encrypt it:

```bash
ansible-vault encrypt cac/vars/secrets.yml
```

## 4. Select Compliance Profiles

The setup playbook uses **Ansible tags** to let you choose which compliance profiles to deploy. You can combine any of the following:

| Tag | What it deploys | Benchmarks |
|-----|----------------|------------|
| `rhel8` | RHEL 8 harden JT + scan JTs + workflow | CIS |
| `rhel9` | RHEL 9 harden JT + scan JTs + workflow | CIS, E8 |
| `rhel10` | RHEL 10 harden JT + scan JTs + workflow | CIS, E8 |
| `win2016` | Windows 2016 harden JT | CIS |
| `win2019` | Windows 2019 harden JT | CIS |
| `win2022` | Windows 2022 harden JT | CIS |
| `win2025` | Windows 2025 harden JT | CIS |

Core resources (organization, credentials, project, inventories, labels) are **always** deployed regardless of tag selection.

## 5. Run the Setup

### Option A: CLI (ansible-playbook)

Run from your workstation or CI/CD pipeline.

**Deploy specific profiles (recommended):**

```bash
# Example: RHEL 9 CIS/E8 + Windows 2019 CIS
ansible-playbook cac/compliance-setup.yml \
  --tags "rhel9,win2019" \
  --ask-vault-pass

# Example: All RHEL versions only
ansible-playbook cac/compliance-setup.yml \
  --tags "rhel8,rhel9,rhel10" \
  --ask-vault-pass

# Example: Windows 2022 only
ansible-playbook cac/compliance-setup.yml \
  --tags "win2022" \
  --ask-vault-pass
```

**Deploy everything:**

```bash
ansible-playbook cac/compliance-setup.yml --ask-vault-pass
```

### Option B: From AAP (self-bootstrapping)

Use this method to run the CaC setup as a job inside AAP itself. This requires a minimal manual bootstrap first.

**Step B.1 -- Manual bootstrap (one-time)**

In the AAP UI, manually create:

1. **Credential** -- "CaC Vault Credential" (type: Vault) with the vault password used to encrypt `secrets.yml`
2. **Credential** -- "CaC Machine Credential" (type: Source Control) with the Git SSH key for the repo
3. **Project** -- "CaC Setup Project" pointing to this repo's Git URL with the Source Control credential
4. **Inventory** -- "CaC Localhost" with a single host `localhost` and variable `ansible_connection: local`
5. **Job Template** -- "JT - CaC Compliance Setup":
   - **Inventory**: CaC Localhost
   - **Project**: CaC Setup Project
   - **Playbook**: `cac/compliance-setup.yml`
   - **Credentials**: CaC Vault Credential
   - **Job Tags**: leave empty to deploy all, or set to `rhel9,win2019` (your selection)
   - **Extra Variables**:

```yaml
---
aap_hostname: "https://<your-aap-hostname>"
aap_username: admin
aap_password: "<your-admin-password>"
aap_validate_certs: false
```

**Step B.2 -- Launch**

1. Navigate to **Templates** and launch "JT - CaC Compliance Setup"
2. To change which profiles are deployed, edit the Job Template's **Job Tags** field
3. The job creates all remaining resources (credentials, inventories, project, JTs, workflows)

**Note:** After the initial bootstrap, subsequent runs update existing resources in place. You can re-run the setup job whenever the repo is updated to sync changes to AAP.

## 6. Verify in AAP

After the setup completes, log in to your AAP UI and verify:

1. **Organization** -- your organization exists with Galaxy/Automation Hub credentials attached
2. **Project** -- "Git Compliance" project is synced and green
3. **Inventories** -- "Workshop RHEL" and/or "Workshop Windows" exist (add your hosts)
4. **Job Templates** -- the selected JTs appear under Templates
5. **Workflows** -- RHEL compliance workflows (Pre-Scan -> Harden -> Post-Scan) are available

## 7. Add Hosts to Inventories

The setup creates empty inventories. Add your target hosts:

- **Workshop RHEL** -- add your RHEL 8/9/10 hosts
- **Workshop Windows** -- add your Windows 2016/2019/2022/2025 hosts

## 8. Run Compliance

### Using Job Templates

Launch individual job templates from the AAP UI:

1. Navigate to **Templates**
2. Click the rocket icon next to the desired JT
3. For RHEL, select the compliance profile (CIS or E8) from the survey
4. For Windows, review the **Job Tags** field (see below)
5. Monitor the job output

### CIS Level Selection (Windows)

Windows CIS Job Templates are configured with `ask_tags_on_launch: true` and default to **Level 1 - Member Server** (`level1-memberserver`). When launching a Windows JT, the Job Tags field is pre-populated with this default.

To target a different CIS profile, change the Job Tags value at launch:

| Tag | Description |
|-----|-------------|
| `level1-domaincontroller` | Level 1 controls for Domain Controllers |
| `level1-memberserver` | Level 1 controls for Member Servers |
| `level1-domainmember` | Level 1 controls for Domain Members |
| `level2-domaincontroller` | Level 2 controls for Domain Controllers |
| `level2-memberserver` | Level 2 controls for Member Servers |
| `ngws-domaincontroller` | Next Generation Windows Security controls for DCs |
| `ngws-memberserver` | Next Generation Windows Security controls for Member Servers |

**Common scenarios:**

```bash
# Level 1 Member Server (default)
level1-memberserver

# Level 1 Domain Controller
level1-domaincontroller

# Level 2 Member Server
level2-memberserver

# All levels (clear the Job Tags field entirely)
<leave empty>

# Combine multiple profiles
level1-memberserver,level2-memberserver
```

These tags apply to all Windows CIS roles (2016, 2019, 2022, 2025). The upstream roles are sourced from [ansible-lockdown](https://github.com/ansible-lockdown).

### Using Workflows (RHEL)

For the full compliance pipeline (Pre-Scan -> Harden -> Post-Scan):

1. Navigate to **Templates**
2. Launch the workflow (e.g., "WF - RHEL 9 Compliance")
3. Select the compliance profile from the survey
4. The workflow runs all three stages automatically

### Scan Reports

After scan jobs complete, reports are shipped to your report server at:

```
/home/<v_report_server_user>/oscap-report-<hostname>-<stage>.html
```

Download and open in a browser to inspect results. See `sample reports/` in this repo for examples.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Collection install fails | Ensure your Automation Hub token is set in `ansible.cfg` |
| Setup playbook can't reach AAP | Verify `aap_hostname` and network connectivity |
| Project sync fails | Check the Git SSH key in `secrets.yml` matches a deploy key on the repo |
| RHEL jobs fail with permission denied | Verify the SSH key in `vault_rhel_ssh_private_key` and `become` is working |
| Windows jobs fail | Check WinRM connectivity, `ansible_port: 5986`, and CredSSP transport |
| Scan reports not appearing | Verify `v_report_server` hostname and SSH credentials |
