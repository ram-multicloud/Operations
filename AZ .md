# Azure VM Backup Types

Azure provides multiple options to back up Virtual Machines (VMs). Backups help protect operating systems, application data, and business-critical workloads against accidental deletion, corruption, or system failures.

## 1. Types of Azure VM Backup

### 1. OS Disk Backup
- Backs up the operating system disk.
- Contains Windows or Linux OS files, configurations, and installed software.
- Example: Windows `C:` drive.
- Useful for operating system recovery.

### 2. Data Disk Backup
- Backs up additional managed disks attached to the VM.
- Contains application files, databases, and user data.
- Example: Windows `D:` or `E:` drives.
- Useful for recovering application data.

### 3. Full VM Backup
- Protects the VM and its supported disks using Azure Backup.
- Includes the OS disk and selected data disks.
- Supports full VM recovery and individual disk recovery.
- Recommended for production workloads.

### 4. Manual Backup
- Backup is triggered manually by an administrator.
- Can be performed before OS upgrades, application deployments, or configuration changes.
- Azure Backup supports on-demand backup.
- Disk snapshots can also be created manually.

### 5. Automatic (Scheduled) Backup
- Azure automatically performs backups based on a configured backup policy.
- Supports scheduled backup frequencies.
- Retention periods can be configured.
- Recommended for production environments.

## 2. Azure VM Backup Policy Types

Azure Backup supports two main VM backup policy types.

| Feature | Standard Policy | Enhanced Policy |
|---|---|---|
| Backup frequency | Once daily | Multiple times per day |
| Minimum interval | 24 hours | 4 hours |
| Retention | Configurable | Configurable |
| Best use | Basic workloads | Production workloads |

## 3. Azure VM Backup vs Disk Snapshot

| Feature | Azure VM Backup | Managed Disk Snapshot |
|---|---|---|
| Purpose | Disaster recovery | Disk-level recovery |
| Storage | Recovery Services vault | Azure managed storage |
| Scheduling | Automatic or manual | Manual or scripted |
| Retention | Backup policy | Managed manually |
| Recovery | VM, disks, supported files | Create disk from snapshot |
| Best use | Production workloads | Development and testing |

**Note:** A disk snapshot is not a replacement for a complete Azure Backup solution.

## 4. Practical Example

Consider an Azure VM named `WebServer01`.

| Disk | Size | Purpose |
|---|---:|---|
| OS Disk | 128 GB | Windows Server |
| Data Disk 1 | 100 GB | Application files |
| Data Disk 2 | 200 GB | Database files |

### Backup Configuration

- Backup service: Azure Backup
- Backup vault: Recovery Services vault
- Backup frequency: Daily
- Backup time: 11 PM
- Retention: 30 days
- Protected disks: OS and both data disks

If the VM crashes, you can restore the VM or its backed-up disks using the available recovery options.

## 5. Configure Azure VM Backup Using Portal

1. Open Azure Portal.
2. Navigate to **Virtual Machines**.
3. Select the required VM.
4. Select **Backup** under Operations.
5. Select or create a Recovery Services vault.
6. Choose Standard or Enhanced backup policy.
7. Configure backup frequency.
8. Configure retention period.
9. Click **Enable Backup**.
10. Select **Backup now** to trigger an immediate backup.

## 6. Azure CLI Commands

### List Backup Items

```bash
az backup item list \
  --resource-group my-backup-rg \
  --vault-name my-recovery-vault \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  -o table
```

### Trigger Manual Backup

```bash
az backup protection backup-now \
  --resource-group my-backup-rg \
  --vault-name my-recovery-vault \
  --container-name my-vm-container \
  --item-name my-vm-item \
  --backup-management-type AzureIaasVM \
  --retain-until 2027-12-31
```

## 7. Best Practices

- Enable automatic backups for production VMs.
- Include all required OS and data disks.
- Configure retention based on business requirements.
- Test VM restoration periodically.
- Use snapshots before major changes.
- Protect backup vaults using appropriate security settings.
- Consider application-consistent backups for database workloads.

## 8. Important Azure Services

| Azure Service | Purpose |
|---|---|
| Azure Backup | VM backup and recovery |
| Recovery Services vault | Stores and manages backup recovery points |
| Backup Policy | Defines backup schedule and retention |
| Managed Disk Snapshot | Point-in-time disk copy |
| Azure Site Recovery | Disaster recovery and VM replication |

## 9. Official Documentation

- [Azure VM Backup](https://learn.microsoft.com/en-us/azure/backup/backup-azure-vms-introduction)
- [Configure VM Backup](https://learn.microsoft.com/en-us/azure/backup/backup-azure-vms-first-look-arm)
- [Azure Disk Snapshots](https://learn.microsoft.com/en-us/azure/virtual-machines/snapshot-copy-managed-disk)
- [Azure Site Recovery](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-about)

---
**Recommendation:** For production environments, use Azure Backup with a Recovery Services vault and an appropriate backup policy. Use disk snapshots for quick point-in-time disk recovery.
