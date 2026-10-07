# Logitech CloudHub 2.0 Firewall Kill-Switch

## 1. Overview

This repository provides an automated firewall control mechanism for **MuleSoft CloudHub 2.0 Private Spaces**.

The solution supports two operations:

- **Kill-Switch** — Restricts inbound API traffic during an emergency or security incident.
- **Restored** — Restores the approved firewall configuration after the incident is resolved.

The automation is executed through **GitHub Actions** and uses the **Anypoint Platform API** to update the CloudHub 2.0 Private Space firewall configuration.

---

# 2. Repository Structure

```text
logitech-cloudhub-killswitch/
│
├── .github/
│   └── workflows/
│       └── CloudHub-Firewall-Control-V3.yml
│
├── cloudhub2-firewall-killswitch
│
└── README.md
```

| File | Purpose |
|---|---|
| `.github/workflows/CloudHub-Firewall-Control-V3.yml` | GitHub Actions workflow used to execute the firewall operation |
| `cloudhub2-firewall-killswitch` | Bash script used to authenticate, retrieve the firewall configuration, create the backup, and apply the selected firewall configuration |
| `README.md` | Documentation for the kill-switch solution |

---

# 3. High-Level Process Flow

The complete process works as follows:

```text
User
  -->
GitHub Actions
  -->
Select Environment
  -->
Sandbox / Prod
  -->
Select Operation
  -->
Kill-Switch / Restored
  -->
Bash Script
  -->
Authenticate with Anypoint Platform
  -->
Retrieve CloudHub 2.0 Private Space
  -->
Retrieve Current Firewall Configuration
  -->
Select Operation
  -->
+----------------------------+
|                            |
| Kill-Switch                | Restored
|                            |
v                            v
Create Firewall Backup       No Backup Required
  |                            |
  v                            |
Select Kill-Switch Rules      Select Restored Rules
  |                            |
  +-------------+--------------+
                |
                v
       PATCH Private Space
                |
                v
       Verify HTTP Response
                |
                v
       GitHub Actions Result
                |
                v
       Upload Backup Artifact
```

---

# 4. Kill-Switch Process

When the **Kill-Switch** operation is selected, the following process is executed.

```text
User
  -->
GitHub Actions
  -->
Select Environment
  -->
Sandbox / Prod
  -->
Select Operation
  -->
kill-switch
  -->
Authenticate with Anypoint Platform
  -->
Retrieve Private Space
  -->
Retrieve Current Firewall Configuration
  -->
Create Firewall Backup
  -->
Store Backup Temporarily
  -->
/tmp/cloudhub-firewall-backup/
  -->
Select KILL_SWITCH_FIREWALL_JSON
  -->
PATCH Private Space
  -->
Verify HTTP Response
  -->
Kill-Switch Applied
  -->
Upload Firewall Backup
  -->
GitHub Actions Artifact
```

---

# 5. Kill-Switch Detailed Steps

## Step 1 — Select Environment

The user selects one of the following:

```text
Sandbox
Prod
```

The selected GitHub Environment determines the credentials and firewall configuration used by the workflow.

---

## Step 2 — Select Operation

The user selects:

```text
kill-switch
```

from the workflow input.

---

## Step 3 — Authenticate with Anypoint Platform

The Bash script authenticates using the Anypoint Platform Connected App.

Required credentials:

```text
ANYPOINT_CLIENT_ID
ANYPOINT_CLIENT_SECRET
```

The credentials are stored as **GitHub Environment Secrets**.

```text
GitHub Environment Secrets
  -->
Connected App Credentials
  -->
Anypoint Platform Authentication
  -->
Access Token
```

---

## Step 4 — Retrieve Private Space Configuration

The script uses:

```text
ANYPOINT_ORG_ID
ANYPOINT_PRIVATE_SPACE_ID
```

to retrieve the CloudHub 2.0 Private Space configuration.

API endpoint:

```text
/runtimefabric/api/organizations/{orgId}/privatespaces/{privateSpaceId}
```

Flow:

```text
Organization ID
  +
Private Space ID
  -->
Private Space API
  -->
Current Private Space Configuration
  -->
Current Firewall Rules
```

---

## Step 5 — Create Firewall Backup

Before applying the kill-switch, the current firewall configuration is backed up.

```text
Current Firewall Configuration
  -->
Extract Firewall Rules
  -->
Create JSON Backup
  -->
/tmp/cloudhub-firewall-backup/
```

Example:

```text
firewall-backup-20261007-120000.json
```

The backup is **not committed to the Git repository**.

---

## Step 6 — Select Kill-Switch Configuration

The workflow selects:

```text
KILL_SWITCH_FIREWALL_JSON
```

Flow:

```text
KILL_SWITCH_FIREWALL_JSON
  -->
Validate JSON
  -->
Select Emergency Firewall Configuration
```

The kill-switch configuration is designed to restrict inbound API traffic while preserving the required outbound connectivity.

---

## Step 7 — Apply Kill-Switch

The selected firewall configuration is sent to the CloudHub 2.0 Private Space API using a PATCH request.

```text
KILL_SWITCH_FIREWALL_JSON
  -->
PATCH Private Space API
  -->
CloudHub 2.0 Private Space
  -->
Firewall Updated
```

---

## Step 8 — Verify Result

The workflow checks the HTTP response.

```text
PATCH Response
  -->
HTTP 2xx
  -->
Kill-Switch Successfully Applied
```

If the response is not successful:

```text
PATCH Response
  -->
HTTP Error
  -->
Firewall Update Failed
  -->
Display API Error
  -->
Preserve Backup
```

---

## Step 9 — Upload Backup Artifact

The backup is uploaded as a GitHub Actions Artifact.

```text
Backup JSON
  -->
GitHub Actions Artifact
  -->
firewall-backup-Sandbox-123
  -->
90-Day Retention
```

The backup is therefore available from the GitHub Actions run without storing it in the repository.

---

# 6. Restore Process

The restore operation is used when normal API traffic needs to resume.

```text
Incident Resolved
  -->
Open GitHub Actions
  -->
Select Environment
  -->
Sandbox / Prod
  -->
Select Operation
  -->
restored
  -->
Authenticate with Anypoint Platform
  -->
Retrieve Private Space
  -->
Select RESTORED_FIREWALL_JSON
  -->
Validate Firewall Configuration
  -->
PATCH Private Space
  -->
Verify HTTP Response
  -->
Firewall Restored
  -->
Normal API Traffic Resumes
```

No new firewall backup is created during the restore operation.

---

# 7. Kill-Switch vs Restore

| Operation | Purpose | Backup |
|---|---|---|
| `kill-switch` | Apply emergency firewall configuration | Yes |
| `restored` | Restore approved normal firewall configuration | No |

---

# 8. GitHub Environments

The repository uses two GitHub Environments:

```text
Sandbox
Prod
```

Flow:

```text
GitHub Actions
  -->
Environment Selection
  -->
+-------------------+
|                   |
v                   v
Sandbox             Prod
|                   |
v                   v
Sandbox Secrets     Prod Secrets
Sandbox Variables  Prod Variables
```

This ensures that Sandbox and Production configurations remain separated.

---

# 9. GitHub Environment Secrets

The following secrets are required in both environments:

```text
ANYPOINT_CLIENT_ID
ANYPOINT_CLIENT_SECRET
ANYPOINT_ORG_ID
ANYPOINT_PRIVATE_SPACE_ID
```

Flow:

```text
GitHub Environment
  -->
Environment Secrets
  -->
Anypoint Connected App
  -->
Authentication
```

Secrets must never be committed to the repository.

---

# 10. GitHub Environment Variables

The following variables are required:

```text
RESTORED_FIREWALL_JSON
KILL_SWITCH_FIREWALL_JSON
```

### RESTORED_FIREWALL_JSON

Contains the approved firewall configuration used to restore normal API traffic.

Flow:

```text
RESTORED_FIREWALL_JSON
  -->
Validate JSON
  -->
PATCH Private Space
  -->
Normal Firewall Configuration
```

### KILL_SWITCH_FIREWALL_JSON

Contains the approved emergency firewall configuration.

Flow:

```text
KILL_SWITCH_FIREWALL_JSON
  -->
Validate JSON
  -->
PATCH Private Space
  -->
Emergency Firewall Configuration
```

---

# 11. Production Approval Flow

Production execution should use the GitHub **Prod Environment protection rule**.

```text
User
  -->
GitHub Actions
  -->
Select Prod
  -->
Select Kill-Switch / Restored
  -->
Prod Environment Protection
  -->
Approval Required
  -->
Authorized Reviewer
  -->
Approve
  -->
Workflow Continues
  -->
Firewall Script Executes
  -->
CloudHub 2.0 Firewall Updated
```

If the reviewer rejects the deployment:

```text
Prod Selected
  -->
Approval Required
  -->
Reviewer Rejects
  -->
Workflow Stops
  -->
No Firewall Change
```

Recommended environment configuration:

```text
Sandbox
  -->
No Production Approval Required

Prod
  -->
Required Reviewer Approval
```

---

# 12. GitHub Actions Workflow

The workflow is manually triggered from:

```text
GitHub Repository
  -->
Actions
  -->
CloudHub 2.0 Firewall Control V3
  -->
Run Workflow
```

The user selects:

```text
Environment
  -->
Sandbox / Prod
```

and:

```text
Operation
  -->
restored / kill-switch
```

---

# 13. Backup and Recovery Flow

The backup process is:

```text
Current CloudHub Firewall
  -->
Retrieve Firewall Configuration
  -->
Create JSON Backup
  -->
/tmp/cloudhub-firewall-backup/
  -->
GitHub Actions Artifact
  -->
90-Day Retention
```

There is no permanent:

```text
backups/
```

directory in the repository.

---

# 14. Failure Handling

## Authentication Failure

```text
GitHub Actions
  -->
Authentication
  -->
Authentication Failed
  -->
Workflow Stops
  -->
No Firewall Change
```

---

## Private Space Retrieval Failure

```text
Authentication Successful
  -->
Get Private Space
  -->
API Error / Invalid Response
  -->
Workflow Stops
  -->
No Firewall Change
```

---

## Firewall PATCH Failure

```text
Get Current Firewall
  -->
Create Backup
  -->
PATCH Firewall
  -->
PATCH Failed
  -->
Display HTTP Error
  -->
Backup Preserved
  -->
Upload Backup Artifact
```

---

# 15. Security Flow

```text
GitHub Environment Secrets
  -->
Protected Credentials
  -->
Anypoint Platform Authentication
  -->
Access Token
  -->
Private Space API
```

Security controls:

- Credentials are stored as GitHub Environment Secrets.
- Credentials are not stored in the Bash script.
- Credentials are not committed to Git.
- Sandbox and Production configurations are separated.
- Production execution requires approval.
- Kill-switch execution is manually initiated.
- Current firewall configuration is backed up before kill-switch activation.
- Backups are stored as GitHub Actions Artifacts.
- Repository access is restricted to authorized users.

---

# 16. Emergency Kill-Switch End-to-End Flow

```text
Emergency Identified
  -->
Open GitHub Actions
  -->
Select Environment
  -->
Sandbox / Prod
  -->
Select kill-switch
  -->
If Prod --> Approval Required
  -->
Authenticate with Anypoint Platform
  -->
Retrieve Private Space
  -->
Retrieve Current Firewall
  -->
Create Backup
  -->
Validate Kill-Switch Configuration
  -->
PATCH Private Space
  -->
Verify HTTP 2xx Response
  -->
Kill-Switch Active
  -->
Upload Backup Artifact
```

---

# 17. Restoration End-to-End Flow

```text
Incident Resolved
  -->
Open GitHub Actions
  -->
Select Environment
  -->
Sandbox / Prod
  -->
Select restored
  -->
If Prod --> Approval Required
  -->
Authenticate with Anypoint Platform
  -->
Retrieve Private Space
  -->
Validate Restored Configuration
  -->
PATCH Private Space
  -->
Verify HTTP 2xx Response
  -->
Firewall Restored
  -->
Normal API Traffic Resumes
```

---

# 18. Operational Checklist

## Before Kill-Switch

```text
Verify Environment
  -->
Verify Private Space
  -->
Verify Kill-Switch Configuration
  -->
Verify Production Approval
  -->
Execute Kill-Switch
```

## After Kill-Switch

```text
Verify GitHub Actions Result
  -->
Verify HTTP Response
  -->
Verify CloudHub Firewall
  -->
Verify API Traffic Restriction
  -->
Verify Backup Artifact
```

## Before Restore

```text
Confirm Incident Resolved
  -->
Verify Restore Configuration
  -->
Obtain Production Approval if Required
  -->
Execute Restore
```

## After Restore

```text
Verify HTTP Response
  -->
Verify Firewall Configuration
  -->
Verify API Connectivity
  -->
Confirm Normal Traffic
```

---

# 19. Important Operational Notes

1. Always verify the selected environment before executing the workflow.
2. Use **Sandbox** for validation before Production execution.
3. Production execution should follow the applicable change-management and approval process.
4. Verify the firewall JSON configuration before making changes.
5. Verify the Private Space ID before execution.
6. The kill-switch backup is created before applying the emergency firewall configuration.
7. Check the GitHub Actions result after execution.
8. Verify the CloudHub 2.0 firewall configuration after a successful operation.
9. Do not store credentials in the repository.
10. Do not commit firewall backup files to the repository.
11. Only authorized users should execute the Production kill-switch.
12. Production approval should be configured through the GitHub `Prod` Environment protection rule.

---

# 20. Questions and Answers

## Q1. What is the purpose of this repository?

**Answer:**  
This repository provides an automated mechanism to control the CloudHub 2.0 Private Space firewall using GitHub Actions.

---

## Q2. What is the kill-switch?

**Answer:**  
The kill-switch is an emergency firewall configuration used to restrict inbound API traffic to the CloudHub 2.0 Private Space.

---

## Q3. Does the kill-switch disable the VPN?

**Answer:**  
No. The kill-switch modifies the configured CloudHub 2.0 Private Space firewall rules. It does not directly disable the corporate VPN.

However, traffic coming through the VPN can be affected if the corresponding inbound traffic is blocked by the CloudHub firewall.

---

## Q4. Does the kill-switch affect outbound traffic?

**Answer:**  
The configured kill-switch rules are designed to restrict inbound traffic while preserving the required outbound connectivity. The exact behavior depends on the configured `KILL_SWITCH_FIREWALL_JSON`.

---

## Q5. Is the existing firewall configuration backed up?

**Answer:**  
Yes. The current firewall configuration is retrieved and backed up before the kill-switch configuration is applied.

---

## Q6. Where is the backup stored?

**Answer:**  
The backup is temporarily created on the GitHub Actions runner:

```text
/tmp/cloudhub-firewall-backup/
```

It is then uploaded as a GitHub Actions Artifact.

---

## Q7. Is the backup stored in Git?

**Answer:**  
No. The backup is not committed to the repository.

---

## Q8. How long is the backup retained?

**Answer:**  
The GitHub Actions Artifact is configured for **90 days** retention.

---

## Q9. What happens if the PATCH operation fails?

**Answer:**  
The backup has already been created before the PATCH operation. The workflow reports the HTTP error and the backup can still be uploaded as an artifact.

---

## Q10. What happens if authentication fails?

**Answer:**  
The workflow stops before modifying the CloudHub 2.0 Private Space firewall.

---

## Q11. What is the difference between `kill-switch` and `restored`?

**Answer:**

```text
kill-switch
  -->
Create Backup
  -->
Apply Emergency Firewall

restored
  -->
No New Backup
  -->
Apply Approved Firewall
```

---

## Q12. How is Production approval handled?

**Answer:**  
The `Prod` GitHub Environment should be configured with **Required Reviewers**.

The flow is:

```text
Select Prod
  -->
GitHub Environment Protection
  -->
Approval Required
  -->
Authorized Reviewer Approves
  -->
Workflow Continues
```

If approval is rejected:

```text
Approval Rejected
  -->
Workflow Stops
  -->
No Firewall Change
```

---

## Q13. Where are the Anypoint credentials stored?

**Answer:**  
They are stored as GitHub Environment Secrets:

```text
ANYPOINT_CLIENT_ID
ANYPOINT_CLIENT_SECRET
ANYPOINT_ORG_ID
ANYPOINT_PRIVATE_SPACE_ID
```

---

## Q14. Where are the firewall configurations stored?

**Answer:**  
They are stored as GitHub Environment Variables:

```text
RESTORED_FIREWALL_JSON
KILL_SWITCH_FIREWALL_JSON
```

---

## Q15. Is the workflow automatic?

**Answer:**  
No. The workflow is manually triggered from GitHub Actions.

Production execution also requires the configured approval before the job proceeds.

---

## Q16. How do I activate the kill-switch?

**Answer:**

```text
GitHub Repository
  -->
Actions
  -->
CloudHub 2.0 Firewall Control V3
  -->
Run Workflow
  -->
Select Sandbox / Prod
  -->
Select kill-switch
  -->
Approve if Prod
  -->
Execute
```

---

## Q17. How do I restore normal traffic?

**Answer:**

```text
GitHub Repository
  -->
Actions
  -->
CloudHub 2.0 Firewall Control V3
  -->
Run Workflow
  -->
Select Sandbox / Prod
  -->
Select restored
  -->
Approve if Prod
  -->
Execute
  -->
Verify Normal Traffic
```

---

## Q18. Does restoration require the backup artifact?

**Answer:**  
No. Restoration uses the approved `RESTORED_FIREWALL_JSON` configuration.

The backup artifact is retained for recovery, audit, and reference purposes.

---

## Q19. Can the Production kill-switch be executed without approval?

**Answer:**  
No, provided the `Prod` GitHub Environment is configured with the required reviewer protection rule.

The workflow will wait for approval before the job can access the protected Production environment and proceed.

---

## Q20. Who can execute the Production kill-switch?

**Answer:**  
Only authorized users with the appropriate GitHub repository permissions and required Production approval should execute the Production kill-switch.

---

# 21. Ownership

**Project:** Logitech MuleSoft Integration

**Platform:** MuleSoft CloudHub 2.0

**Automation:** GitHub Actions

**Purpose:** Emergency CloudHub 2.0 Private Space Firewall Control

**Environments:** Sandbox, Production

**Repository:** `logitech-cloudhub-killswitch`
