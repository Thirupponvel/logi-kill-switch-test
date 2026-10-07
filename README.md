# Logitech CloudHub 2.0 Firewall Kill-Switch

## Overview

This repository provides an automated firewall control mechanism for **MuleSoft CloudHub 2.0 Private Spaces**.

The solution supports two operations:

- **Kill-Switch** – Restricts inbound API traffic during an emergency or security incident.
- **Restored** – Restores the approved firewall configuration after the incident is resolved.

The automation is executed through **GitHub Actions** and uses the **Anypoint Platform API** to update the Private Space firewall configuration.

---

## Repository Structure


logitech-cloudhub-killswitch/
│
├── .github/
│   └── workflows/
│       └── CloudHub-Firewall-Control-V3.yml
│
├── cloudhub2-firewall-killswitch
│
└── README.md


### Files

| File | Purpose |
|---|---|
| `.github/workflows/CloudHub-Firewall-Control-V3.yml` | GitHub Actions workflow used to execute the firewall operation |
| `cloudhub2-firewall-killswitch` | Bash script that authenticates with Anypoint Platform, retrieves the current firewall configuration, creates the backup, and applies the selected firewall configuration |
| `README.md` | Documentation for the kill-switch process |

---

# How the Kill-Switch Works

The process is initiated manually from GitHub Actions.


User
  |
  v
GitHub Actions
  |
  |-- Select Environment
  |      Sandbox / Prod
  |
  |-- Select Operation
  |      Kill-Switch / Restored
  |
  v
Bash Script
  |
  v
Authenticate with Anypoint Platform
  |
  v
Get CloudHub 2.0 Private Space
  |
  v
Get Current Firewall Configuration
  |
  +-----------------------------+
  |                             |
  | Kill-Switch                 | Restored
  |                             |
  v                             v
Create Firewall Backup          No Backup Required
  |
  v
Select Kill-Switch Rules        Select Restored Rules
  |                             |
  +-------------+---------------+
                |
                v
       PATCH Private Space
                |
                v
       Firewall Configuration
                |
                v
        GitHub Actions Result
                |
                v
       Backup uploaded as
       GitHub Actions Artifact


---

# Kill-Switch Process

When **Kill-Switch** is selected:

### 1. Select Environment

The user selects one of:


Sandbox
Prod


The selected GitHub Environment determines which credentials and firewall configuration are used.

---

### 2. Select Kill-Switch

The user selects:
kill-switch

from the workflow input.

---

### 3. Authenticate

The Bash script authenticates with Anypoint Platform using the configured Connected App credentials.

The authentication uses:


Client ID
Client Secret


The credentials are stored as **GitHub Environment Secrets**.

---

### 4. Retrieve Private Space Configuration

The script retrieves the current CloudHub 2.0 Private Space configuration using the organization ID and Private Space ID.

The API endpoint follows:

/runtimefabric/api/organizations/{orgId}/privatespaces/{privateSpaceId}


---

### 5. Create Firewall Backup

Before applying the kill-switch, the current firewall configuration is backed up.

The backup is temporarily created on the GitHub Actions runner:

/tmp/cloudhub-firewall-backup/


The backup is **not committed to the repository**.

Example:
firewall-backup-20261007-120000.json


---

### 6. Apply Kill-Switch Firewall

The workflow applies the configured:


KILL_SWITCH_FIREWALL_JSON


The kill-switch configuration is designed to restrict inbound traffic while preserving the required outbound configuration.

The exact firewall rules are maintained as an environment-specific GitHub variable.

---

### 7. Upload Backup as Artifact

After the firewall operation, the backup is uploaded as a **GitHub Actions Artifact**.

Example artifact name:

firewall-backup-Sandbox-123


The artifact is retained for:


90 days


This allows the pre-kill-switch firewall configuration to be retained for recovery and audit purposes.

---

# Restore Process

When normal API traffic needs to be restored:

1. Open the GitHub Actions workflow.
2. Select the required environment.
3. Select:

```text
restored
```

4. The workflow retrieves the Private Space.
5. The configured `RESTORED_FIREWALL_JSON` is selected.
6. The approved firewall configuration is applied.
7. No new backup is created for the restore operation.

---

# GitHub Environments

The repository uses two GitHub Environments:

Sandbox
Prod


Each environment contains its own credentials and firewall configuration.

## Environment Secrets

The following secrets are required:


ANYPOINT_CLIENT_ID
ANYPOINT_CLIENT_SECRET
ANYPOINT_ORG_ID
ANYPOINT_PRIVATE_SPACE_ID


These values must be configured separately for Sandbox and Prod.

Secrets must **never** be committed to the repository.

---

# Environment Variables

The following GitHub Environment Variables are required:

RESTORED_FIREWALL_JSON
KILL_SWITCH_FIREWALL_JSON


### RESTORED_FIREWALL_JSON

Contains the approved firewall configuration used to restore normal operation.

### KILL_SWITCH_FIREWALL_JSON

Contains the emergency firewall configuration used to restrict inbound traffic.



# GitHub Actions Workflow

The workflow is manually triggered using:


Actions
  |
  v
CloudHub 2.0 Firewall Control V3
  |
  v
Run workflow


The user selects:

Environment:
    Sandbox
    Prod

Operation:
    restored
    kill-switch



# Backup and Recovery

The backup is created **before** the kill-switch firewall configuration is applied.


Current Firewall
      |
      v
Backup JSON
      |
      v
/tmp/cloudhub-firewall-backup/
      |
      v
GitHub Actions Artifact


The repository does not contain a permanent `backups/` directory.

This prevents operational firewall backups from being committed to source control.

---

# Security

The following security practices are followed:

- Anypoint Platform credentials are stored in GitHub Environment Secrets.
- Credentials are not stored in the Bash script.
- Credentials are not committed to Git.
- Sandbox and Production configurations are separated.
- Kill-switch execution is manually initiated.
- The existing firewall configuration is backed up before the kill-switch operation.
- Backup files are stored as GitHub Actions Artifacts.
- Repository access is restricted to authorized project team members.

---

# Emergency Kill-Switch Flow


Emergency identified
        |
        v
Open GitHub Actions
        |
        v
Select Sandbox / Prod
        |
        v
Select kill-switch
        |
        v
Authenticate
        |
        v
Retrieve current firewall
        |
        v
Create backup
        |
        v
Apply kill-switch firewall
        |
        v
Verify HTTP response
        |
        v
Backup uploaded as Artifact


---

# Restoration Flow


Incident resolved
        |
        v
Open GitHub Actions
        |
        v
Select Sandbox / Prod
        |
        v
Select restored
        |
        v
Authenticate
        |
        v
Apply approved firewall
        |
        v
Verify HTTP response
        |
        v
Normal traffic restored


---

# Important Operational Notes

1. Always verify the selected environment before executing the workflow.
2. Use **Sandbox** for testing before Production execution.
3. Verify the firewall JSON configuration before making changes.
4. The kill-switch backup is created before applying the emergency configuration.
5. Check the GitHub Actions run result after execution.
6. In case of a failed PATCH operation, the backup remains available on the runner for artifact upload.
7. Do not manually modify firewall configuration directly in the script.
8. Do not store credentials in the repository.

---

# Ownership

**Project:** Logitech MuleSoft Integration

**Platform:** MuleSoft CloudHub 2.0

**Automation:** GitHub Actions

**Purpose:** Emergency CloudHub 2.0 Private Space Firewall Control

**Environments:** Sandbox, Production
