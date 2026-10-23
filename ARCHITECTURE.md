# Architecture — EBS snapshot-based restore on Windows EC2

![Recovery architecture](../diagrams/architecture.png)

## 1. System boundary

**Region:** `eu-west-2` (London). **EC2:** Windows instance `SRV-2A`. **Storage:** system EBS volume plus 2 GiB gp3 original and 2 GiB gp3 restored data volumes.

This is a **single-instance recovery drill**, not a multi-AZ failover architecture. The restored volume was attached to the same EC2 instance for an accessible side-by-side comparison.

## 2. Role of each component

| Component | Responsibility | Evidence |
|---|---|---|
| EC2 Windows Server | Provides the OS, disk discovery and Windows file system | [EC2 attachments](../evidence/01-windows-ec2-and-attached-volumes.png) |
| Original gp3 EBS (D:) | Holds baseline and later test folders | [Windows comparison](../evidence/04-before-vs-after-recovery.png) |
| Snapshot `D-Drive-Snapshot` | Captures the point-in-time storage state | [Completed snapshot](../evidence/03-snapshot-completed.png) |
| Restored gp3 EBS (E:) | New block volume created from snapshot | [Volume list](../evidence/02-original-and-restored-ebs-volumes.png) |

## 3. Data-flow model

```text
          Windows EC2 SRV-2A (eu-west-2)
          ┌────────────────────────────────────────────────────┐
          │  Original EBS data disk D:                        │
          │    Before-Snapshot → SNAPSHOT T1                  │
          │    After-Snapshot  → added to D: after T1         │
          │                         │                          │
          └─────────────────────────┼──────────────────────────┘
                                    ▼
                      Regional EBS snapshot
                               │
                       Create new volume
                               │
                               ▼
                       Restored EBS disk E:
                       Before-Snapshot only
```

### Important AWS boundary rules

- Volumes attach to EC2 within the **same Availability Zone**.
- EBS snapshots are **regional** resources and can be used to create volumes in selected AZs in the same Region.
- Snapshot data is AWS-managed, not a user-managed Amazon S3 bucket.
- Storage operations are distinct from Windows disk online/drive-letter operations.

## 4. Why restore to another disk instead of replacing D: immediately?

**Safer comparison.** A new, separately attached disk can be inspected next to the original without overwriting the live test volume. It makes the state gap visible and keeps the original available for inspection.

In real incident response, you would consider data sensitivity, quarantine, least-privilege access, snapshots of the compromised original, and which machine should mount recovered data.

## 5. What was deliberately outside scope?

No VSS-confirmed application-consistent snapshot, database recovery, scheduled backup retention, timed RTO, multi-AZ failover, hash inventory, or production traffic was evidenced.

[Deep dive](../DEEP-DIVE.md) · [Project home](../README.md)
