# Deep Dive — AWS EBS Snapshot, Restore & Recovery Engineering

**Audience:** A learner who knows only basic cloud terms, and an engineer who wants to reason accurately about disaster recovery.  
**Project:** Windows EC2 `SRV-2A` in `eu-west-2`; 2 GiB original and restored EBS data volumes; snapshot-time data validation.

> **Mental model:** A backup is a saved checkpoint. A restore is a new copy derived from that checkpoint. Verification is proving the copy holds the state you actually expected.

## Contents

1. The block-storage mental model
2. What an EBS volume is — and is not
3. EC2 → EBS → Windows drive letters
4. Snapshots and capture boundaries
5. Incremental snapshots explained
6. Backup timeline: before / at / after
7. Recovering a new EBS volume
8. Region, Availability Zone and durability
9. Attaching and exposing a disk in Windows
10. File-system integrity and consistency
11. Crash consistency vs application consistency
12. Why snapshot data may differ from the live disk
13. Evidence-based validation
14. Checksums and stronger verification
15. RPO and RTO
16. Disk signatures, offline disks and troubleshooting
17. Recoverability vs availability vs high availability
18. Security, encryption and IAM
19. Operational risks, cleanup and costs
20. Common anti-patterns
21. A production-oriented future roadmap
22. Interview-style questions and model answers

---

## 1. The block-storage mental model

Think of an EBS volume as a virtual hard disk attached to a cloud computer. It stores **blocks**, not the Windows `D:` label. The operating system decides how to partition, format and expose those blocks as a drive letter.

**Two layers must not be confused:** AWS manages the volume attachment; Windows manages how the block device appears and whether it is usable. A volume can be attached in EC2 while its disk remains offline in Windows.

This distinction becomes crucial during restore because AWS can report `in-use` while File Explorer still has no accessible drive.

## 2. What is Amazon EBS?

Amazon Elastic Block Store (EBS) provides persistent block storage for EC2. An EBS volume can survive an instance reboot, and its lifecycle is managed independently from application processes. In this project, both data disks were **gp3** volumes with **2 GiB** capacity.

EBS is **not** Amazon S3: a volume behaves like an attached disk; S3 is an object-storage API. EBS snapshots are stored in AWS-managed snapshot storage backed by S3 infrastructure, not in a bucket you browse through your S3 console.

### EBS vs EFS vs S3

| Service | Interface | Typical use |
|---|---|---|
| EBS | Block device attached to EC2 | OS volumes, application disks, databases |
| EFS | Shared file system over NFS | Shared Linux file access |
| S3 | Object storage | Backups, artifacts, static objects |

Do not assume a Windows drive-letter workflow applies directly to EFS or S3.

## 3. Why EC2 sees D: and E:

Windows can assign a mounted file system a letter. In the project, the original data volume was visible as `D:` and the restored volume as `E:`. The exact drive letter is an OS decision: it is **not** a permanent property of the EBS snapshot.

Conceptual stack:

```text
AWS Region: eu-west-2
  └── EC2 SRV-2A in an Availability Zone
        ├── EBS system volume → Windows C:
        ├── Original EBS data volume → Windows D:
        └── Restored EBS data volume → Windows E:
```

The evidence shows the system volume plus original and restored 2 GiB volumes attached to the same instance.

## 4. What exactly is an EBS snapshot?

A snapshot records a **point-in-time state** of an EBS volume. AWS begins snapshot capture when the snapshot request is initiated. It saves the relevant written blocks of the source volume's state and eventually reports `Completed` when the snapshot is available for normal restore operations.

### Important distinction

- **Snapshot completed** means the snapshot creation operation finished.
- It does **not** mean the application was frozen, all cached writes were flushed, or a database was consistent at its transaction boundary.
- It does **not** mean writes *after* snapshot start are inside the backup.

The uploaded console screenshot shows a completed `D-Drive-Snapshot` for a **2 GiB** volume with approximately **13 MiB** of snapshot data shown. A 2 GiB capacity does not mean 2 GiB of live user content was written.

## 5. How incremental snapshots work

The first EBS snapshot logically contains all blocks needed to reconstruct the volume state. Later snapshots generally store newly written and changed blocks relative to previously snapshotted data. AWS manages the dependency bookkeeping; you do not need to restore an entire visible chain manually.

Analogy: you take a photo of a notebook. Later you edit a few pages. The next snapshot represents the full updated notebook state but may need to store only the changed portions.

**Don't confuse EBS incremental storage with conventional file copy.** AWS operates at the block layer, while File Explorer lists folders and files.

## 6. The project's most important timeline

```text
T0 — BEFORE:  D:\Before-Snapshot exists
T1 — SNAPSHOT: Capture a backup of D: at its then-current state
T2 — AFTER:   Create D:\After-Snapshot on the still-live original volume
T3 — RESTORE: Create new EBS volume from the snapshot
T4 — VERIFY:  D: has Before + After, E: has Before only
```

**Why this is valuable:** it turns a passive green status into a behavioral test. If the restored `E:` had contained the post-snapshot changes in this experiment, we would need to investigate our assumptions about what was actually captured and which volume was restored.

The same pattern generalizes to databases, but file-folder existence is not sufficient to verify database transaction consistency.

## 7. How recovery from a snapshot works

An EBS snapshot isn't a runnable Windows disk by itself. A restore typically requires these stages:

1. Select the snapshot and choose **Create volume from snapshot**.
2. Choose a volume type and an AZ matching the EC2 instance where the new volume will be attached.
3. Wait for the new volume to reach `available`.
4. Attach it to the target instance.
5. In Windows, inspect **Disk Management** or use PowerShell to identify the new device safely.
6. Bring the correct disk online where necessary, mount it without modifying the original's data, and assign a drive letter.
7. Validate expected folders, files and ideally content hashes.

**Do not initialize, format or clean a restored disk that contains data you need to recover.** New blank disks and restored data disks are different situations.

## 8. Regions and Availability Zones

Snapshots are regional resources. The source example was in **London (`eu-west-2`)**. Snapshot data is managed within the Region and may be used to create a new volume in an AZ in that Region. An attached EBS volume and its EC2 instance must be in the **same AZ**.

For cross-region disaster recovery, an EBS snapshot must be copied to the target Region before restoring there. Nothing in this project demonstrates cross-region recovery.

## 9. Windows operating-system steps

AWS `In-use` is not enough. Windows must recognize and expose the block device. Work through the layers deliberately:

```text
EBS volume exists
    ↓
Attached to correct EC2 instance
    ↓
Windows detects a disk
    ↓
Disk is online / accessible
    ↓
Partition and filesystem mount cleanly
    ↓
Drive letter present
    ↓
Expected files accessible
```

### Optional read-only discovery commands (illustrative, not saved console execution)

```powershell
Get-Disk | Format-Table Number,FriendlyName,OperationalStatus,IsOffline,IsReadOnly,PartitionStyle
Get-Volume | Format-Table DriveLetter,FileSystemLabel,FileSystem,HealthStatus,Size
Get-Partition | Format-Table DiskNumber,PartitionNumber,DriveLetter,Size
```

These commands can help identify the Windows device. **Never assume a particular disk number is the restored volume** without checking its characteristics.

## 10. Data durability vs recoverability

EBS volume durability is not equivalent to having an independent recovery point. Operational mistakes, accidental deletion, malware or application corruption can propagate to live storage. Snapshots provide separate restore points, but **their usefulness depends on how frequently they are taken and whether restores actually work.**

A recovery test must answer more than: “Did AWS say it completed?” It should ask: “Can a new disk be mounted, can expected files be read, and are the bytes correct?”

## 11. Crash consistency vs application consistency

A basic live snapshot captures written blocks at a point in time, but in-memory cached writes may not be included. This is often described as **crash-consistent** behavior. For simple static test folders, that may be enough. For transactional databases or multi-file applications, it might not be.

**Windows VSS (Volume Shadow Copy Service)** can coordinate application writers to obtain application-consistent snapshots when configured appropriately via AWS-supported solutions.

In this lab we have **no evidence that VSS was configured or run**. Do not label this experiment application-consistent.

Real design decision: if the workload is SQL Server, Active Directory, or a business application with complex writes, first establish the application's supported consistency mechanism before promising safe restores.

## 12. Why After-Snapshot did not return

The screenshot of the two Windows Explorer panes shows:

- Original `D:`: `Before-Snapshot` and `After-Snapshot`
- Restored `E:`: `Before-Snapshot` only

This is not a defect; it is the expected behavior of restoring the earlier checkpoint.

Imagine you wrote a customer order at 09:00, backed up at 09:05, and accepted more orders at 09:10. Restoring the 09:05 snapshot alone won't automatically return the 09:10 orders. To achieve a tighter recovery point you need additional backups or continuous data-protection mechanisms appropriate to your workload.

## 13. Evidence hierarchy

| Proof type | What it establishes | What it cannot establish by itself |
|---|---|---|
| `Completed` snapshot | AWS backup finished | File data integrity |
| Restored EBS attached | AWS volume attachment | Windows mount accessibility |
| D:/E: Explorer comparison | Recovered folder names and visible state | Bit-for-bit identity or filesystem consistency |
| Checksums on recovered files | Equality for specific file contents | Overall application consistency or availability SLA |
| Application functional test | User-facing operation works | Long-term recovery performance |

Your screenshots support the first three rows. They do **not** include file-level cryptographic checksums or application functional tests.

## 14. Stronger recovery verification: file hashing

If both volumes include `Before-Snapshot` test files, a stronger test is to hash selected files and compare them:

```powershell
# EXAMPLE ONLY — replace with real filenames; not executed in the supplied evidence.
Get-FileHash 'D:\Before-Snapshot\sample.txt' -Algorithm SHA256
Get-FileHash 'E:\Before-Snapshot\sample.txt' -Algorithm SHA256
```

If the hashes match, the two selected files have identical bytes with extremely high confidence. This does not by itself prove every file on the disk matches, nor that the application can start after recovery.

For directory-wide validation, write a controlled inventory-and-hash script, keep comparable relative paths, and review unreadable/locked files separately.

## 15. RPO and RTO: two questions that matter

**Recovery Point Objective (RPO)** is the maximum acceptable gap between the latest recoverable backup state and failure time.  
**Recovery Time Objective (RTO)** is the target time for service recovery.

| Scenario | RPO question | RTO question |
|---|---|---|
| Daily snapshot | How much change might occur after the most recent snapshot? | How long to create/attach volume and resume service? |
| Every 15-minute backup | How much post-backup work may still be lost? | Is restore fast enough for business needs? |
| Database with log backups | Which transactions can be replayed? | How quickly can backup and logs be restored? |

**No numeric RPO/RTO was measured** during this lab. The absence of `After-Snapshot` on `E:` is the visible demonstration of a **non-zero point-in-time gap**.

## 16. Offline disks and disk signature collisions

The learner reports encountering a Windows disk visibility/signature conflict while trying to use the original and restored disks together, followed by a change to the disk configuration.

Disk signature/ID collisions can cause Windows to offline a cloned disk to avoid ambiguity. The exact issue varies with MBR/GPT layout, Windows storage configuration and disk policies. It is **unsafe** to blindly convert partitions or run data-destructive disk commands on an important backup.

Safe diagnostic approach:

1. Confirm EBS attachment to the expected EC2 instance and AZ.
2. Open Windows **Disk Management** and identify which disk is offline.
3. Record capacity, partition details and whether the disk is MBR or GPT.
4. Check whether Windows reports a signature collision or another condition.
5. For disposable test disks only, choose the non-destructive remedy appropriate to the confirmed cause.
6. Do **not** initialize, format, clean, or delete partitions on the restored disk.
7. Mount it, assign a new drive letter if required, then verify contents.

**Evidence status:** This problem was described by the learner in the LinkedIn case-study notes; the uploaded screenshots do not include the error dialog or exact repair commands. It is documented here as a reported troubleshooting incident, not a independently reproduced fault investigation.

## 17. Recovery vs high availability

**Recovery** means restoring from a previous point after disruption. **High availability** means designing systems to continue or quickly resume service when components fail. A restored EBS data volume on one Windows EC2 proves a restoration workflow, **not** automatic failover or high availability.

This distinction is easy to blur in resumes. Keep claims precise.

## 18. Security, IAM and encryption

Production-oriented considerations:

- Restrict IAM actions such as `ec2:CreateSnapshot`, `ec2:CreateVolume`, `ec2:AttachVolume`, `ec2:DeleteSnapshot`, and relevant KMS permissions to the necessary roles and resources.
- Use EBS encryption and KMS key management appropriate to policy.
- Apply snapshot-retention rules and regularly test restores.
- Avoid exposing AWS account identifiers and instance metadata unnecessarily in portfolio evidence.
- Audit snapshot ownership and sharing before granting cross-account access.
- Be careful when copying encrypted snapshots across Regions or AWS accounts.

**Not verified in this lab:** KMS policy, automated backup lifecycle, snapshot sharing, least-privilege IAM policies, and access logging.

## 19. Costs and lifecycle

A free-tier assumption is unsafe. EC2 runtime, EBS provisioned capacity, and stored EBS snapshot data can incur charges depending on the account, Region, service pricing and credit eligibility.

After a disposable training lab, review whether the following resources remain: EC2 instance, unattached EBS volumes, snapshots and public IP allocations. Stopping an EC2 instance does not automatically remove attached EBS volumes or snapshot storage charges.

Delete **only** resources you can positively identify as unused and nonessential. Do not delete a volume or snapshot that another recovery plan depends upon.

## 20. Common anti-patterns

**Mistake 1 — “Snapshot Completed means data definitely consistent.”**  
No. For write-intensive applications you must consider flushing and application consistency.

**Mistake 2 — “My restored drive must include all later changes.”**  
No. A point-in-time snapshot does not include post-capture writes.

**Mistake 3 — “AWS reports Attached, so Windows must show a drive.”**  
No. The OS storage stack is separate.

**Mistake 4 — “Two folders with the same names prove zero data loss.”**  
No. File names are a useful demonstration but do not prove byte-for-byte data equality or completeness.

**Mistake 5 — “If a restored disk is offline, format it.”**  
Dangerous. Diagnose disk identity and OS visibility before acting.

**Mistake 6 — “A snapshot is a region-independent volume.”**  
Snapshots are regional resources; attached EBS volumes are AZ-specific.

## 21. Next experiment: production-minded DR validation

Turn this lab into a repeatable recovery drill:

1. Build a **disposable** Windows EC2 lab with a known test dataset.
2. Record baseline file inventory and SHA-256 hashes.
3. Create a snapshot at a recorded timestamp; allow all required application writes to flush if needed.
4. Introduce an explicit simulated deletion or corruption **only on throwaway data**.
5. Restore a second EBS volume and mount it on the instance.
6. Verify file inventory and hashes, and confirm data added after snapshot is absent.
7. Measure elapsed time for each stage: request, create volume, attach, mount, validate.
8. Document observed RPO/RTO constraints, limits and costs.
9. Repeat in another AZ using a new volume from the same regional snapshot.
10. If a database is involved, layer in an application-consistent method and an application test.

Treat this as a **future improvement roadmap**; it was not performed as part of the screenshot-backed lab.

## 22. Interview questions — with honest answers

**Q1. What is EBS?**  
Persistent block storage for EC2. It appears to an OS as a disk-like device.

**Q2. What's an EBS snapshot?**  
A point-in-time backup of written blocks on a volume that can be used to create another EBS volume.

**Q3. Why doesn't the restored drive contain After-Snapshot?**  
Because those changes occurred after snapshot capture; the restored volume reflects an earlier point in time.

**Q4. Is a completed snapshot automatically application-consistent?**  
No. You may need Windows VSS or app-aware processes to coordinate buffered writes/transactions.

**Q5. Can you attach an EBS volume across Availability Zones?**  
No. The EBS volume and EC2 attachment need to be in the same AZ.

**Q6. Can you restore an EBS snapshot into another AZ of the same Region?**  
Yes, by creating a new EBS volume from the snapshot in the target AZ.

**Q7. Did you prove zero data loss?**  
No. I demonstrated successful snapshot-time folder recovery, and post-snapshot data was intentionally not present on the restored volume.

**Q8. What does an 'in-use' EBS status prove?**  
The AWS volume is attached; it doesn't prove Windows has mounted a readable drive.

**Q9. What should you do if a restored disk is offline?**  
Identify the disk safely, inspect Disk Management and reported cause; do not initialize or format data-bearing disks.

**Q10. RPO vs RTO?**  
RPO is acceptable data-loss time window; RTO is acceptable recovery-time window.

**Q11. What would you improve next?**  
Hash-based validation, timed recovery, automatic backups, application-consistency testing and documented cleanup.

**Q12. What exactly did you do?**  
I created a snapshot of the data volume, introduced changes afterward, restored a new volume and compared snapshot-time versus current folders on Windows. My published screenshots support those claims, not automatic failover or full database recovery.

---

## Reference reading

- [Create Amazon EBS snapshots — AWS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html)
- [How Amazon EBS snapshots work — AWS](https://docs.aws.amazon.com/ebs/latest/userguide/how_snapshots_work.html)
- [Create an Amazon EBS volume — AWS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html)
- [Restore an EBS volume using a snapshot — AWS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-restoring-volume.html)
- [Application-consistent Windows VSS snapshots — AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/application-consistent-snapshots.html)

**Return:** [Project README](README.md) · [Validation evidence](docs/VALIDATION.md)
